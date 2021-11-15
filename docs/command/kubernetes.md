### Pod

**查询所有Running状态的Pod**

```bash
kubectl get pods --field-selector=status.phase=Running --all-namespaces
```

**查询所有镜像对应不是Running状态的pod数量**

```bash
kubectl get pods --all-namespaces  --field-selector=status.phase!=Running -o jsonpath="{..image}" |\
tr -s '[[:space:]]' '\n' |\
sort |\
uniq -c
```

**查询所有pod请求的cpu数量**

```bash
# 查询所有pod请求的cpu数量
kubectl get po -o \
  custom-columns="Name:metadata.name,Namespace:metadata.namespace,CPU-request:spec.containers[*].resources.requests.cpu,CPU-limit:spec.containers[*].resources.limits.cpu" \
  --all-namespaces

# 过滤未Limit的Pod
kubectl get po -o \
  custom-columns="Name:metadata.name,Namespace:metadata.namespace,CPU-request:spec.containers[*].resources.requests.cpu,CPU-limit:spec.containers[*].resources.limits.cpu" \
  --all-namespaces \
  | awk '{if($3!="<none>")print}'
```

**查询所有pod请求的memory数量**

```bash
# 查询所有pod请求的memory数量
kubectl get po -o \
  custom-columns="Name:metadata.name,Namespace:metadata.namespace,Memory-request:spec.containers[*].resources.requests.memory,Memory-limit:spec.containers[*].resources.limits.memory" \
  --all-namespaces

# 过滤未Limit的Pod
kubectl get po -o \
  custom-columns="Name:metadata.name,Namespace:metadata.namespace,Memory-request:spec.containers[*].resources.requests.memory,Memory-limit:spec.containers[*].resources.limits.memory" \
  --all-namespaces \
  | awk '{if($3!="<none>")print}'
```

### Network

**服务端口转发**

```bash
kubectl -n seldon port-forward svc/seldon-core-analytics-grafana 3000:80 &
```

### 节点

**查询节点**

> --selector='key=value'
>
> --selector='key in (value1, value2)'

```bash
kubectl get nodes --selector='type=notebook-X2_LARGE'

kubectl get nodes --selector='type in (notebook-X2_LARGE,notebook-X4_LARGE,notebook-X8_LARGE)'

kubectl get nodes --label-columns type --selector='type in (notebook-X2_LARGE,notebook-X4_LARGE,notebook-X8_LARGE)'

kubectl get nodes --label-columns type --selector='type in (notebook-X2_LARGE,notebook-X4_LARGE,notebook-X8_LARGE)' --sort-by={.metadata.labels."type"}
```

**按照type标签排序节点**

```
kubectl get nodes --show-labels --sort-by={.metadata.labels."type"}
```

**按type标签排序节点，并显示type标签**

```
kubectl get nodes --label-columns type --sort-by={.metadata.labels."type"}
```

**获取每个节点上运行的Pod**

```bash
for n in $(kubectl get nodes --selector='type in (notebook-X2_LARGE,notebook-X4_LARGE,notebook-X8_LARGE)' --no-headers --sort-by={.metadata.labels."type"} | cut -d " " -f1); do
    kubectl get node ${n} --label-columns type --no-headers | awk {'print $1" " $6'}
    kubectl get pods --all-namespaces --no-headers --selector role=notebook --field-selector spec.nodeName=${n}
done
```

**获取节点上运行Pod数量**

```bash
for n in $(kubectl get nodes --selector='type in (notebook-X2_LARGE,notebook-X4_LARGE,notebook-X8_LARGE)' --no-headers --sort-by={.metadata.labels."type"} | cut -d " " -f1); do
    kubectl get node ${n} --label-columns type --no-headers | awk {'print $1" " $6'}
    kubectl get pods --all-namespaces --no-headers --field-selector=status.phase=Running --selector role=notebook --field-selector spec.nodeName=${n} | wc -l
done
```

### 删除正在Pending的PVC

```bash
kubectl patch pvc {PVC_NAME} -p '{"metadata":{"finalizers":null}}' -n {NAMESPACE}
```
### Released PV重新绑定PVC

```bash
kubectl patch pv {PVC_NAME} -p '{"spec":{"claimRef": null}}'
```

**按时间排序查看事件**

```bash
kubectl get event -n kubeflow --sort-by='.metadata.creationTimestamp' 
```
