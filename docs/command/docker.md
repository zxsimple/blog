**Switch docker vm**

`docker-machine env --shell=powershell ibm-dsx | Invoke-Expression `

**Docker save tar**

```bash
docker save -o xdl.tar registry.cn-hangzhou.aliyuncs.com/xdl/xdl:ubuntu-gpu-tf1.12
```

**Docker import tar**

```bash
docker load -i xdl.tar
```

> 如果报错：open /var/lib/docker/tmp/docker-import-970689518/bin/json: no such file or directory，则用以下命令导入

```bash
cat xdl.tar | docker import - xdl
```

**Stop all containers for specific image**

```bash
docker rm $(docker stop $(docker ps -a -q --filter ancestor=<image-name> --format="{{.ID}}"))
```

**删除所有exited状态的容器实例**

```bash
docker ps --filter "status=exited" | grep 'weeks ago' | awk '{print $1}' | xargs --no-run-if-empty docker rm
```

**停止所有容器**

```bash
docker ps | awk 'NR > 1 {print $1}' | xargs docker stop
```

**删除exited docker**

```bash
docker rm -v $(docker ps -a -q -f status=exited)
```

**删除dangling镜像**

```bash
docker rmi $(docker images -f "dangling=true" -q)
```
