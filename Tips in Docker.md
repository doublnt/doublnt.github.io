## Docker上一些内容记录

### **集群中查看具体错误信息**

&emsp;通过 `docker stack ps ing --no-trunc` 命令查看到具体的错误信息是"starting container failed: Address already in use"，

### **重启Docker 服务**

&emsp;通过 `sudo service docker restart` 命令

### **开发集群故障时重启服务器的办法**

```shell
sudo ssh root@dev-server-swarm
reboot
sudo reboot

```

### 查看日志

docker service logs -t  - - raw  service_name 

### 版本回滚

docker service update --rollback openapi_api 回滚到上一个镜像的版本

出现"The swarm does not have a leader"，可以等一会，看能否自动恢复

如果 docker stack ps 发现"Unable to complete atomic operation, key modified"错误，一般都需要重启服务器

### 错误Cannot connect to the Docker daemon at tcp://swarm1-node1:2375. Is the docker daemon running?

