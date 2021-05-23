## Docker上运行Consul的Demo

> demo来源于consul官网，记录仅用于自己备份，[consul-demo](https://learn.hashicorp.com/tutorials/consul/docker-container-agents?in=consul/docker)

在docker上启动consul `server`

```shell
docker run -d -p 8500:8500 -p 8600:8600/udp --name=badger consul agent -server -ui -node=server-1 -bootstrap-expect=1 -client=0.0.0.0
```

可以看到其实到``consul``为止都是docker的指令，而consul后的`agent`实际上是consul的指令了，说明docker在运行时可以在最后直接加上容器内应用的指令。

启动consul `client`

```shell
docker run -d --name=fox consul agent -node=client-1 -join=172.17.0.2
```

在搞一个测试的web服务

```shell
docker pull hashicorp/counting-service:0.0.2
docker run \
   -p 9001:9001 \
   -d \
   --name=weasel \
   hashicorp/counting-service:0.0.2
```

现在注意三个服务的名字，consul-server叫badger，consul-client叫fox，web服务叫weasel

然后再client上注册这个web服务，consul的默认配置目录为`consul/config`下的json文件，也就是按配置的方式该目录下的一个json文件代表一个服务

向consul-client中写入配置注册服务

```shell
docker exec fox /bin/sh -c "echo '{\"service\": {\"name\": \"counting\", \"tags\": [\"go\"], \"port\": 9001}}' >> /consul/config/counting.json"
```

注册之后reload consul，查看DNS服务

```shell
root@GriffithLP:/# dig @127.0.0.1 -p 8600 counting.service.consul

; <<>> DiG 9.11.3-1ubuntu1.11-Ubuntu <<>> @127.0.0.1 -p 8600 counting.service.consul
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 5173
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;counting.service.consul.       IN      A

;; ANSWER SECTION:
counting.service.consul. 0      IN      A       172.17.0.3

;; Query time: 6 msec
;; SERVER: 127.0.0.1#8600(127.0.0.1)
;; WHEN: Fri May 21 21:39:22 CST 2021
;; MSG SIZE  rcvd: 68
```

这里consul的集群网络是通过dokcer默认的docker0网络实现的，单机部署测试使用，可以使用docker-compose优化。