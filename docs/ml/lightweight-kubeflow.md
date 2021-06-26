使用Kubeflow并不容易，而Kubeflow基本实现了主流的基于Kubernetes的训练框架方案[Training Operators](https://www.kubeflow.org/docs/components/training/)。我们看一下如何在不依赖Kubeflow的情况下，在Kubernetes上调度运行[tf-operator](https://github.com/kubeflow/tf-operator)，[pytorch-operator](https://github.com/kubeflow/pytorch-operator) 和通过[mpi-operator](https://github.com/kubeflow/mpi-operator/)运行[Horovod](https://github.com/horovod/)。

### Prerequisite

首先我们单独安装`tf-operator`，`pytorch-operator`和`mpi-operator`

**安装kustomize**

```bash
wget https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv4.1.2/kustomize_v4.1.2_linux_amd64.tar.gz
tar -xvf kustomize_v4.1.2_linux_amd64.tar.gz
mv kustomize /usr/local/bin/
```

**下载manifests**

```bash
wget https://github.com/kubeflow/manifests/archive/refs/tags/v1.2.0.tar.gz
tar -xvf v1.2.0.tar.gz
MANIFESTS_DIR=$(pwd)/manifests-1.2.0
```

**安装Operator**

> 将**gcr.io**镜像换成**gcr.azk8s.cn**解决墙内镜像拉取问题

安装`tf-operator`

```bash
cd "${MANIFESTS_DIR}/tf-training/tf-job-crds/base"
kustomize build . | kubectl apply -f -
cd "${MANIFESTS_DIR}/tf-training/tf-job-operator/base"
sed -i 's/gcr.io/gcr.azk8s.cn/g' *
kustomize build . | kubectl apply -f -
```

安装`pytorch-operator`

```bash
cd "${MANIFESTS_DIR}/pytorch-job/pytorch-job-crds/base"
kustomize build . | kubectl apply -f -
cd "${MANIFESTS_DIR}/pytorch-job/pytorch-operator/base/"
sed -i 's/gcr.io/gcr.azk8s.cn/g' *
kustomize build . | kubectl apply -f -
```

安装`mpi-operator`

```bash
wget https://raw.githubusercontent.com/kubeflow/mpi-operator/master/deploy/v1/mpi-operator.yaml
kubectl apply -f mpi-operator.yaml
```

验证CRD部署成功

执行`kubectl get crd | grep kubeflow`看到`operator`已经安装，这样我们就不用依赖[Kubeflow几十个组件依赖](https://zhuanlan.zhihu.com/p/346234161)而只使用其训练组件了。

```properties
mpijobs.kubeflow.org                             2021-04-22T08:43:00Z
pytorchjobs.kubeflow.org                         2021-04-22T01:23:40Z
tfjobs.kubeflow.org                              2021-04-22T01:00:02Z
```

### 提交Tensorflow作业

通过配置[tfReplicaSpecs](https://www.kubeflow.org/docs/components/training/tftraining/)可以定义**Chief**，**Ps**，**Worker** 和**Evaluator**作业进程的资源配置和实例数量，`tf-operator`在启动pod节点时会自动配置`TF_CONFIG`环境变量说明各个节点实例的访问地址和当前节点的角色和`index`，在分布式训练的场景下Tensorflow会根据此环境变量来识别各个节点的角色。

```yaml
spec:
  containers:
  - env:
    - name: TF_CONFIG
      value: '{"cluster":{"ps":["distributed-training-ps-0.kubeflow.svc:2222"],"worker":["distributed-training-worker-0.kubeflow.svc:2222","distributed-training-worker-1.kubeflow.svc:2222"]},"task":{"type":"worker","index":0},"environment":"cloud"}'
```

**单节点训练**

首先创建任务需要的PVC

`tfevent-pv.yaml`

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: tfevent-volume
  labels:
    type: local
    app: tfjob
spec:
  capacity:
    storage: 10Gi
  storageClassName: standard
  accessModes:
    - ReadWriteMany
  hostPath:
    path: /tmp/data
```

`tfevent-pvc.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: tfevent-volume
  namespace: kubeflow
  labels:
    type: local
    app: tfjob
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard
  volumeName: tfevent-volume
```

确保PVC创建成功`kubectl get pvc -n kubeflow`

`tf_job_mnist.yaml`

```yaml
apiVersion: "kubeflow.org/v1"
kind: "TFJob"
metadata:
  name: "mnist"
  namespace: kubeflow
spec:
  cleanPodPolicy: None
  tfReplicaSpecs:
    Worker:
      replicas: 1
      restartPolicy: Never
      template:
        spec:
          containers:
            - name: tensorflow
              image: kubedl/tf-mnist-with-summaries:1.0
              command:
                - "python"
                - "/var/tf_mnist/mnist_with_summaries.py"
                - "--log_dir=/train/logs"
                - "--learning_rate=0.01"
                - "--batch_size=150"
              volumeMounts:
                - mountPath: "/train"
                  name: "training"
          volumes:
            - name: "training"
              persistentVolumeClaim:
                claimName: "tfevent-volume"
```

创建tf-operator作业

```bash
kubectl apply -f tf_job_mnist.yaml
```

查看作业

```bash
kubectl -n kubeflow get tfjob mnist -o yaml
```

查看日志

```
kubectl -n kubeflow logs `kubectl get pods -n kubeflow --selector=name=tf-job-operator -o jsonpath='{.items[0].metadata.name}'` 
```

**分布式训练**

实现[Multi-worker training with Estimator](https://tensorflow.google.cn/tutorials/distribute/multi_worker_with_estimator)训练的示例

```python
# Define DistributionStrategies and convert the Keras Model to an
# Estimator that utilizes these DistributionStrateges.
# Evaluator is a single worker, so using MirroredStrategy.
config = tf.estimator.RunConfig(
    experimental_distribute=tf.contrib.distribute.DistributeConfig(
        train_distribute=tf.contrib.distribute.CollectiveAllReduceStrategy(
            num_gpus_per_worker=0),
        eval_distribute=tf.contrib.distribute.MirroredStrategy(
            num_gpus_per_worker=0)))
keras_estimator = tf.keras.estimator.model_to_estimator(
    keras_model=model, config=config, model_dir=model_dir)
```

分布式任务定义`distributed_tfjob.yaml`，启动1个Parameter Server和2个Worker实例。

```yaml
apiVersion: "kubeflow.org/v1"
kind: "TFJob"
metadata:
  name: "distributed-training"
  namespace: "kubeflow"
spec:
  cleanPodPolicy: None
  tfReplicaSpecs:
    Ps:
      replicas: 1
      restartPolicy: "OnFailure"
      template:
        metadata:
          annotations:
            sidecar.istio.io/inject: "false"
        spec:
          containers:
          - image: "swr.cn-north-4.myhuaweicloud.com/honor-bigdata-dev/distributed_worker:1.0"
            imagePullPolicy: "IfNotPresent"
            name: "tensorflow"
          imagePullSecrets:
          - name: "default-secret"
    Worker:
      replicas: 2
      restartPolicy: "OnFailure"
      template:
        metadata:
          annotations:
            sidecar.istio.io/inject: "false"
        spec:
          containers:
          - image: "swr.cn-north-4.myhuaweicloud.com/honor-bigdata-dev/distributed_worker:1.0"
            imagePullPolicy: "IfNotPresent"
            name: "tensorflow"
          imagePullSecrets:
          - name: "default-secret"
```

### 提交PyTorch作业

通过配置[pytorchReplicaSpecs](https://www.kubeflow.org/docs/components/training/pytorch/)可以定义**Master**，**Worker** 作业进程的资源配置和实例数量，`pytorch-operator`在启动pod节点时会自动配置`MASTER_ADDR`，`WORLD_SIZE`，`RANK`环境变量分别说明master节点的地址，参与计算的节点数量以及当前节点的`index`。

```yaml
Containers:
  pytorch:
    Image:         swr.cn-north-4.myhuaweicloud.com/honor-bigdata/pytorch-dist-mnist-test:1.0
    Port:          23456/TCP
    Host Port:     0/TCP
    Args:
      --backend
      gloo
    Environment:
      MASTER_PORT:       23456
      MASTER_ADDR:       localhost
      WORLD_SIZE:        3
      RANK:              0
```

**构建镜像**

```
docker build -f Dockerfile -t swr.cn-north-4.myhuaweicloud.com/honor-bigdata/pytorch-dist-mnist-test:1.0 ./

docker build -f Dockerfile-mpi -t swr.cn-north-4.myhuaweicloud.com/honor-bigdata/pytorch-dist-mnist-test:mpi ./
```

**以gloo为通信框架进行分布式训练**

`pytorch_job_mnist_gloo.yaml`

```yaml
apiVersion: "kubeflow.org/v1"
kind: "PyTorchJob"
metadata:
  name: "pytorch-dist-mnist-gloo"
spec:
  pytorchReplicaSpecs:
    Master:
      replicas: 1
      restartPolicy: OnFailure
      template:
        metadata:
          annotations:
            sidecar.istio.io/inject: "false"
        spec:
          containers:
            - name: pytorch
              image: swr.cn-north-4.myhuaweicloud.com/honor-bigdata/pytorch-dist-mnist-test:1.0
              args: ["--backend", "gloo"]
          imagePullSecrets:
            - name: default-secret
    Worker:
      replicas: 2
      restartPolicy: OnFailure
      template:
        metadata:
          annotations:
            sidecar.istio.io/inject: "false"
        spec:
          containers: 
            - name: pytorch
              image: swr.cn-north-4.myhuaweicloud.com/honor-bigdata/pytorch-dist-mnist-test:1.0
              args: ["--backend", "gloo"]
          imagePullSecrets:
            - name: default-secret
```

**以mpi为通信框架进行分布式训练**

`pytorch_job_mnist_mpi.yaml`

```yaml
apiVersion: "kubeflow.org/v1"
kind: "PyTorchJob"
metadata:
  name: "pytorch-dist-mnist-mpi"
spec:
  pytorchReplicaSpecs:
    Master:
      replicas: 1
      restartPolicy: OnFailure
      template:
        metadata:
          annotations:
            sidecar.istio.io/inject: "false"
        spec:
          containers:
            - name: pytorch
              image: swr.cn-north-4.myhuaweicloud.com/honor-bigdata/pytorch-dist-mnist-test:mpi
              args: ["--backend", "mpi"]
    Worker:
      replicas: 2
      restartPolicy: OnFailure 
      template:
        metadata:
          annotations:
            sidecar.istio.io/inject: "false"
        spec:
          containers: 
            - name: pytorch
              image: swr.cn-north-4.myhuaweicloud.com/honor-bigdata/pytorch-dist-mnist-test:mpi
              args: ["--backend", "mpi"]
```

### 提交MPI作业

> 注意Job的命名空间必须和mpi-operator一致，参考[github](https://github.com/kubeflow/mpi-operator/issues/344)

通过配置[mpiReplicaSpecs](https://www.kubeflow.org/docs/components/training/mpi/)可以定义**Launcher**和**Worker** 作业进程的资源配置和实例数量，`mpi-operator`会创建一个`JOB_NAME-config`的ConfigMap并挂载到`/etc/mpi`目录。`JOB_NAME-config`中会定义hosts文件和Pod执行命令。

`JOB_NAME-config`

```
data:
  hostfile: |
    tensorflow-mnist-mpi-worker-0 slots=1
    tensorflow-mnist-mpi-worker-1 slots=1
  kubexec.sh: |-
    #!/bin/sh
    set -x
    POD_NAME=$1
    shift
    /opt/kube/kubectl exec ${POD_NAME} -- /bin/sh -c "$*"
```

启动有2个worker节点的mpi训练作业

```
kubectl apply -f tensorflow-benchmarks.yaml -n mpi-operator
```

```yaml
apiVersion: kubeflow.org/v1
kind: MPIJob
metadata:
  name: tensorflow-benchmarks
spec:
  slotsPerWorker: 1
  cleanPodPolicy: Running
  mpiReplicaSpecs:
    Launcher:
      replicas: 1
      template:
         spec:
           containers:
           - image: mpioperator/tensorflow-benchmarks:latest
             name: tensorflow-benchmarks
             command:
             - mpirun
             - --allow-run-as-root
             - -np
             - "2"
             - -bind-to
             - none
             - -map-by
             - slot
             - -x
             - NCCL_DEBUG=INFO
             - -x
             - LD_LIBRARY_PATH
             - -x
             - PATH
             - -mca
             - pml
             - ob1
             - -mca
             - btl
             - ^openib
             - python
             - scripts/tf_cnn_benchmarks/tf_cnn_benchmarks.py
             - --model=resnet101
             - --batch_size=64
             - --variable_update=horovod
    Worker:
      replicas: 2
      template:
        spec:
          containers:
          - image: mpioperator/tensorflow-benchmarks:latest
            name: tensorflow-benchmarks
```

### 提交Horovod作业

> 注意Job的命名空间必须和mpi-operator一致，参考[github](https://github.com/kubeflow/mpi-operator/issues/344)

以mpi方式启动Horovod分布式训练任务

`tensorflow-mnist-hvd.yaml`

```yaml
apiVersion: kubeflow.org/v1
kind: MPIJob
metadata:
  name: tensorflow-mnist
spec:
  slotsPerWorker: 1
  cleanPodPolicy: Running
  mpiReplicaSpecs:
    Launcher:
      replicas: 1
      template:
        spec:
          containers:
          - image: docker.io/kubeflow/mpi-horovod-mnist
            name: mpi-launcher
            command:
            - mpirun
            args:
            - -np
            - "2"
            - --allow-run-as-root
            - -bind-to
            - none
            - -map-by
            - slot
            - -x
            - LD_LIBRARY_PATH
            - -x
            - PATH
            - -mca
            - pml
            - ob1
            - -mca
            - btl
            - ^openib
            - python
            - /examples/tensorflow_mnist.py
            imagePullPolicy: IfNotPresent
            resources:
              limits:
                cpu: 1
                memory: 2Gi
    Worker:
      replicas: 2
      template:
        spec:
          containers:
          - image: docker.io/kubeflow/mpi-horovod-mnist
            name: mpi-worker
            resources:
              limits:
                cpu: 2
                memory: 4Gi
```
