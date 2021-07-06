### SSH

**批量SSH**

```bash
$ while read HOST ; do echo start $HOST end; done < servers.txt 

while read HOST ; 
	do ssh [user]:[password] "mkdir -p /home/${USER}" ; 
done < servers.txt 

ssh xx.xx.xx.xx >> EOF
user
passwd
EOF

# Servers
ip #1
ip #2
```

**ssh远程端口转发**

修改远程服务器SSH配置，并重启

```bash
vi /etc/ssh/sshd_config

GatewayPorts yes
PermitTunnel yes
AllowTCPForwarding yes

service sshd restart
```

ssh连接远程服务器

```bash
ssh -L [LOCAL_PORT]:localhost:[REMOTE_PORT] USER@IP

### Polyaxon
ssh -L 8888:localhost:8000 root@10.10.10.11
```

### 文本

**文本中每行出现的字符数量**

```bash
awk -F'[$WORD]' '{Print NF}' $FILE

# e.g, 冒号出现的数量
awk -F'[:]' '{Print NF}' documents.txt
```
