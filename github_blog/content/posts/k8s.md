---
title: "K8s"
date: 2023-09-03T15:20:27+08:00
draft: true
---





##  启动集群
minikube start

minikube start --kubernetes-version=v1.23.0

##  查看节点。kubectl 是一个用来跟 K8S 集群进行交互的命令行工具
kubectl get node
##  停止集群
minikube stop
##  清空集群
minikube delete --all
##  安装集群可视化 Web UI 控制台
minikube dashboard







##  部署应用到集群中

项目地址https://github.com/gzyunke/test-k8s，里面包含了一个k8s启动pod，启动development的yaml文件  



##### 1.直接命令参数 运行（在启动dokerdesk，启动minikube之后 再运行）

```
kubectl run testapp --image=ccr.ccs.tencentyun.com/k8s-tutorial/test-k8s:v1
```

运行一个pod 

pod的名字是testapp 

 镜像是  --image=ccr.ccs.tencentyun.com/k8s-tutorial/test-k8s:v1



##### **2.查看pod**

```
kubectl get pod
```

可以看到有个启动的过程 READY 0/1变成1/1

```
PS C:\WINDOWS\system32> kubectl get pod
NAME      READY   STATUS              RESTARTS   AGE
testapp   0/1     ContainerCreating   0          45s
PS C:\WINDOWS\system32> kubectl get pod
NAME      READY   STATUS    RESTARTS   AGE
testapp   1/1     Running   0          77s
```



##### 3.可以用命令配置文件运行（pod）

找到配置文件所在的目录，执行命令

```
 kubectl apply -f .\pod.yaml
```

配置文件如下

```
apiVersion: v1
kind: Pod  # 类型是Pod
metadata:
  name: test-pod #Pod的名字
spec:
  # 一个pod可以定义多个容器
  containers:
    - name: test-k8s # 容器名字
      image: ccr.ccs.tencentyun.com/k8s-tutorial/test-k8s:v1 # 镜像
```



##### 4.可以用命令配置文件运行（Deployment）

找到配置文件所在的目录，执行命令

```
 kubectl apply -f .\app.yaml
```



配置文件如下

```
apiVersion: apps/v1
kind: Deployment
metadata:
  # 部署名字
  name: test-k8s
spec:
  # 运行副本的数量，pod的数量
  replicas: 5
  # 用来查找关联的 Pod，所有标签都匹配才行
  selector:
    matchLabels:
      app: test-k8s
  # 定义 Pod 相关数据
  template:
    metadata:
      labels:
        app: test-k8s
    spec:
      # 定义容器，可以多个
      containers:
      - name: test-k8s # 容器名字
        image: ccr.ccs.tencentyun.com/k8s-tutorial/test-k8s:v1 # 镜像
```



可以看到replicas 配了几，就有几个pod

```
PS C:\WINDOWS\system32> kubectl get pod
NAME                        READY   STATUS    RESTARTS   AGE
test-k8s                    1/1     Running   0          10m
test-k8s-8598bbb8c6-8dk69   1/1     Running   0          8s
test-k8s-8598bbb8c6-hl8xj   1/1     Running   0          8s
test-k8s-8598bbb8c6-jkdtd   1/1     Running   0          8s
test-k8s-8598bbb8c6-kjrfk   1/1     Running   0          8s
test-k8s-8598bbb8c6-z4dt6   1/1     Running   0          8s
testapp                     1/1     Running   0          28m
```


加上 -o wide 可以看到pod的ip，在哪个节点上运行
```
PS C:\WINDOWS\system32> kubectl get pod -o wide
NAME                        READY   STATUS    RESTARTS   AGE     IP           NODE       NOMINATED NODE   READINESS GATES
test-k8s                    1/1     Running   0          14m     172.17.0.4   minikube   <none>           <none>
test-k8s-8598bbb8c6-8dk69   1/1     Running   0          3m47s   172.17.0.8   minikube   <none>           <none>
test-k8s-8598bbb8c6-hl8xj   1/1     Running   0          3m47s   172.17.0.7   minikube   <none>           <none>
test-k8s-8598bbb8c6-jkdtd   1/1     Running   0          3m47s   172.17.0.9   minikube   <none>           <none>
test-k8s-8598bbb8c6-kjrfk   1/1     Running   0          3m47s   172.17.0.6   minikube   <none>           <none>
test-k8s-8598bbb8c6-z4dt6   1/1     Running   0          3m47s   172.17.0.5   minikube   <none>           <none>
testapp                     1/1     Running   0          32m     172.17.0.3   minikube   <none>           <none>
```



 **Deployment 通过 label 关联起来 Pods**

![deployment.png](https://sjwx.easydoc.xyz/46901064/files/kwpt8p8o.png)

##### 5.查看 pod 详情  

```
kubectl describe pod pod-name
```

```
PS C:\WINDOWS\system32> kubectl describe pod test-k8s-8598bbb8c6-z4dt6
Name:             test-k8s-8598bbb8c6-z4dt6
Namespace:        default
Priority:         0
Service Account:  default
Node:             minikube/192.168.49.2
Start Time:       Sun, 03 Sep 2023 16:08:44 +0800
Labels:           app=test-k8s
                  pod-template-hash=8598bbb8c6
Annotations:      <none>
Status:           Running
IP:               172.17.0.5
IPs:
  IP:           172.17.0.5
Controlled By:  ReplicaSet/test-k8s-8598bbb8c6
Containers:
  test-k8s:
    Container ID:   docker://acc7370f8f643ce9ae3cad32b2e73a724b82f734990826e11cd435d20bfa7073
    Image:          ccr.ccs.tencentyun.com/k8s-tutorial/test-k8s:v1
    Image ID:       docker-pullable://ccr.ccs.tencentyun.com/k8s-tutorial/test-k8s@sha256:9b452816d6493045a21d8b3d6a851f21ca2e50c86cd57ba8c41141f001d9911d
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Sun, 03 Sep 2023 16:08:45 +0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-lm545 (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  kube-api-access-lm545:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  8m58s  default-scheduler  Successfully assigned default/test-k8s-8598bbb8c6-z4dt6 to minikube
  Normal  Pulled     8m58s  kubelet            Container image "ccr.ccs.tencentyun.com/k8s-tutorial/test-k8s:v1" already present on machine
  Normal  Created    8m58s  kubelet            Created container test-k8s
  Normal  Started    8m57s  kubelet            Started container test-k8s
```

##### 6.查看pod的log 

```
kubectl logs pod-name
```

```
PS C:\WINDOWS\system32> kubectl logs test-k8s-8598bbb8c6-z4dt6
[32m[2023-09-03T08:08:46.147] [INFO] app - [39mrun in docker
[32m[2023-09-03T08:08:46.153] [INFO] app - [39mServer started successfully and listened on 8080
```

```
kubectl logs pod-name -f  
```

持续不断的查看日志

##### 7.进入pod中的容器

```
kubectl exec -it pod-name -- bash
```

 进入 Pod 容器终端，如果一个pod里有多个容器 -c container-name 可以指定进入哪个容器。

```
PS D:\minikube> kubectl exec -it test-k8s-8598bbb8c6-z4dt6 -- bash
root@test-k8s-8598bbb8c6-z4dt6:/app# ls
app.js  docker-compose.yml  draw  log  log.js  log4js.json  node_modules  package-lock.json  package.json  yaml
root@test-k8s-8598bbb8c6-z4dt6:/app# pwd
/app
root@test-k8s-8598bbb8c6-z4dt6:/app# cd ..
root@test-k8s-8598bbb8c6-z4dt6:/# ls
app  bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
root@test-k8s-8598bbb8c6-z4dt6:/# cd app/log
root@test-k8s-8598bbb8c6-z4dt6:/app/log# ls
access  app.log  errors.log
root@test-k8s-8598bbb8c6-z4dt6:/app/log# tail -f app.log
[2021-12-02T17:22:27.289] [INFO] app - on index page
[2021-12-02T17:22:31.337] [INFO] app - on hello page
[2021-12-02T17:22:32.750] [INFO] app - on hello page
[2021-12-02T17:22:35.918] [INFO] app - on hello page
[2021-12-02T17:22:39.464] [INFO] app - on hello page
[2021-12-02T17:22:56.888] [INFO] app - on index page
[2021-12-02T17:23:09.312] [INFO] app - on index page
[2023-09-04T00:10:59.180] [INFO] app - run in docker
[2023-09-04T00:10:59.186] [INFO] app - Server started successfully and listened on 8080
http://localhost:8080
```

##### 8. 伸缩扩展副本（配置文件）
```
kubectl apply -f .\app.yaml
```
在 app.yaml中把 replicas 数量从5个改成6个，查看pod数量也发生相应的变化
```
PS G:\xrqProjects\test-k8s-main\yaml\deployment> kubectl get pod
NAME                        READY   STATUS    RESTARTS      AGE
test-k8s                    1/1     Running   1 (17m ago)   16h
test-k8s-8598bbb8c6-8dk69   1/1     Running   1 (17m ago)   16h
test-k8s-8598bbb8c6-hl8xj   1/1     Running   1 (17m ago)   16h
test-k8s-8598bbb8c6-jkdtd   1/1     Running   1 (17m ago)   16h
test-k8s-8598bbb8c6-kjrfk   1/1     Running   1 (17m ago)   16h
test-k8s-8598bbb8c6-z4dt6   1/1     Running   1 (17m ago)   16h
testapp                     1/1     Running   1 (17m ago)   16h
PS G:\xrqProjects\test-k8s-main\yaml\deployment> kubectl apply -f .\app.yaml
deployment.apps/test-k8s configured
PS G:\xrqProjects\test-k8s-main\yaml\deployment> kubectl get pod
NAME                        READY   STATUS    RESTARTS      AGE
test-k8s                    1/1     Running   1 (18m ago)   16h
test-k8s-8598bbb8c6-8dk69   1/1     Running   1 (18m ago)   16h
test-k8s-8598bbb8c6-hl8xj   1/1     Running   1 (18m ago)   16h
test-k8s-8598bbb8c6-jkdtd   1/1     Running   1 (18m ago)   16h
test-k8s-8598bbb8c6-kjrfk   1/1     Running   1 (18m ago)   16h
test-k8s-8598bbb8c6-skz8x   1/1     Running   0             2s
test-k8s-8598bbb8c6-z4dt6   1/1     Running   1 (18m ago)   16h
testapp                     1/1     Running   1 (18m ago)   16h
```




##### 9.伸缩扩展副本（命令行参数）

```
kubectl scale deployment test-k8s --replicas=10
```




```
PS G:\xrqProjects\test-k8s-main\yaml\deployment> kubectl get pod
NAME                        READY   STATUS    RESTARTS      AGE
test-k8s                    1/1     Running   1 (21m ago)   16h
test-k8s-8598bbb8c6-8dk69   1/1     Running   1 (21m ago)   16h
test-k8s-8598bbb8c6-hl8xj   1/1     Running   1 (21m ago)   16h
test-k8s-8598bbb8c6-jkdtd   1/1     Running   1 (21m ago)   16h
test-k8s-8598bbb8c6-kjrfk   1/1     Running   1 (21m ago)   16h
test-k8s-8598bbb8c6-skz8x   1/1     Running   0             3m37s
test-k8s-8598bbb8c6-z4dt6   1/1     Running   1 (21m ago)   16h
testapp                     1/1     Running   1 (21m ago)   16h
PS G:\xrqProjects\test-k8s-main\yaml\deployment> kubectl scale deployment test-k8s --replicas=10
deployment.apps/test-k8s scaled
PS G:\xrqProjects\test-k8s-main\yaml\deployment> kubectl get pod
NAME                        READY   STATUS    RESTARTS      AGE
test-k8s                    1/1     Running   1 (21m ago)   16h
test-k8s-8598bbb8c6-8dk69   1/1     Running   1 (21m ago)   16h
test-k8s-8598bbb8c6-92k7b   1/1     Running   0             4s
test-k8s-8598bbb8c6-b684w   1/1     Running   0             4s
test-k8s-8598bbb8c6-g77gw   1/1     Running   0             4s
test-k8s-8598bbb8c6-hl8xj   1/1     Running   1 (21m ago)   16h
test-k8s-8598bbb8c6-jkdtd   1/1     Running   1 (21m ago)   16h
test-k8s-8598bbb8c6-kjrfk   1/1     Running   1 (21m ago)   16h
test-k8s-8598bbb8c6-skz8x   1/1     Running   0             3m50s
test-k8s-8598bbb8c6-xhjzx   1/1     Running   0             4s
test-k8s-8598bbb8c6-z4dt6   1/1     Running   1 (21m ago)   16h
testapp                     1/1     Running   1 (21m ago)   16h

```

##### 10.把集群内端口映射到节点（把pod的端口映射到外面）

```
kubectl port-forward pod-name 8090:8080
```

8090是外面的端口，8080是容器端口

```
PS G:\xrqProjects\test-k8s-main\yaml\deployment> kubectl port-forward test-k8s-8598bbb8c6-hl8xj 8090:8080
Forwarding from 127.0.0.1:8090 -> 8080
Forwarding from [::1]:8090 -> 8080
Handling connection for 8090
Handling connection for 8090
Handling connection for 8090

```



##### 11. 查看历史

```
kubectl rollout history deployment test-k8s 
```

```
PS G:\xrqProjects\test-k8s-main\yaml\deployment> kubectl rollout history deployment test-k8s
deployment.apps/test-k8s
REVISION  CHANGE-CAUSE
1         <none>
```

现在只有一个历史版本

修改下 app.yaml 的镜像地址 image: ccr.ccs.tencentyun.com/k8s-tutorial/test-k8s:v2-with-error # 镜像，会报错的一个代码版本

```
kubectl apply -f .\app.yaml
```

重新部署下，灰度更新(灰度更新是指创建一个新pod，销毁一个旧的pod)

```
PS G:\xrqProjects\test-k8s-main\yaml\deployment> kubectl get pod
NAME                        READY   STATUS    RESTARTS      AGE
test-k8s                    1/1     Running   1 (60m ago)   17h
test-k8s-5dd85b6897-2dxs2   1/1     Running   0             88s
test-k8s-5dd85b6897-9g8md   1/1     Running   0             95s
test-k8s-5dd85b6897-b5t2k   1/1     Running   0             95s
test-k8s-5dd85b6897-h4p2x   1/1     Running   0             95s
test-k8s-5dd85b6897-hj6vj   1/1     Running   0             87s
test-k8s-5dd85b6897-vqnwd   1/1     Running   0             87s
testapp                     1/1     Running   1 (60m ago)   17h


```

查看下pod 详情

```
kubectl describe pod test-k8s-5dd85b6897-h4p2x
```

发现image已经修改

```
Events:
  Type    Reason     Age    From               Message

----    ------     ----   ----               -------

  Normal  Scheduled  2m39s  default-scheduler  Successfully assigned default/test-k8s-5dd85b6897-h4p2x to minikube
  Normal  Pulling    2m39s  kubelet            Pulling image "ccr.ccs.tencentyun.com/k8s-tutorial/test-k8s:v2-with-error"
  Normal  Pulled     2m33s  kubelet            Successfully pulled image "ccr.ccs.tencentyun.com/k8s-tutorial/test-k8s:v2-with-error" in 6.110508402s
  Normal  Created    2m33s  kubelet            Created container test-k8s
  Normal  Started    2m32s  kubelet            Started container test-k8s
```



再次查看历史

```
kubectl rollout history deployment test-k8s 
```
```
PS G:\xrqProjects\test-k8s-main\yaml\deployment> kubectl rollout history deployment test-k8s
deployment.apps/test-k8s
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

发现已经有2个版本了

为了检验下新版本的代码功能，把集群内端口映射到节点（把pod的端口映射到外面）

```
kubectl port-forward test-k8s-5dd85b6897-b5t2k 8080:8080
```

在主机访问接口 http://localhost:8080/hello/aasdasd 发现是报错的，需要紧急的回退



##### 12.  回到上个版本（灰度更新）

```
kubectl rollout undo deployment test-k8s 
```



```
PS G:\xrqProjects\test-k8s-main\yaml\deployment> kubectl rollout undo deployment test-k8s
deployment.apps/test-k8s rolled back
PS G:\xrqProjects\test-k8s-main\yaml\deployment> kubectl get pod
NAME                        READY   STATUS              RESTARTS      AGE
test-k8s                    1/1     Running             1 (73m ago)   17h
test-k8s-5dd85b6897-2dxs2   1/1     Running             0             14m
test-k8s-5dd85b6897-9g8md   1/1     Terminating         0             14m
test-k8s-5dd85b6897-b5t2k   1/1     Running             0             14m
test-k8s-5dd85b6897-h4p2x   1/1     Running             0             14m
test-k8s-5dd85b6897-hj6vj   1/1     Running             0             14m
test-k8s-5dd85b6897-vqnwd   1/1     Terminating         0             14m
test-k8s-8598bbb8c6-crjwp   0/1     ContainerCreating   0             2s
test-k8s-8598bbb8c6-kxj4q   0/1     ContainerCreating   0             1s
test-k8s-8598bbb8c6-sgmnv   1/1     Running             0             2s
test-k8s-8598bbb8c6-wqj99   0/1     ContainerCreating   0             2s
testapp                     1/1     Running             1 (73m ago)   17h
PS G:\xrqProjects\test-k8s-main\yaml\deployment> kubectl get pod
NAME                        READY   STATUS    RESTARTS      AGE
test-k8s                    1/1     Running   1 (74m ago)   17h
test-k8s-8598bbb8c6-crjwp   1/1     Running   0             108s
test-k8s-8598bbb8c6-jkw9g   1/1     Running   0             106s
test-k8s-8598bbb8c6-kxj4q   1/1     Running   0             107s
test-k8s-8598bbb8c6-rfpfl   1/1     Running   0             106s
test-k8s-8598bbb8c6-sgmnv   1/1     Running   0             108s
test-k8s-8598bbb8c6-wqj99   1/1     Running   0             108s
testapp                     1/1     Running   1 (74m ago)   17h

```

转发一个新的pod ，检查其代码

```
PS G:\xrqProjects\test-k8s-main\yaml\deployment> kubectl port-forward test-k8s-8598bbb8c6-rfpfl  8080:8080
Forwarding from 127.0.0.1:8090 -> 8080
Forwarding from [::1]:8090 -> 8080
Handling connection for 8090
Handling connection for 8090
```

在主机访问接口 http://localhost:8080/hello/aasdasd 发现是正常的版本了

现在又多了 3 号版本了，但是1号版本没了？

```
PS G:\xrqProjects\test-k8s-main\yaml\deployment> kubectl rollout history deployment test-k8s
deployment.apps/test-k8s
REVISION  CHANGE-CAUSE
2         <none>
3         <none>
```





##### 13.回到指定版本

```
kubectl rollout undo deployment test-k8s --to-revision=2 
```

##### 14.回到指定版本

```
kubectl delete deployment test-k8s
```

##### 更多命令

```
# 查看全部
kubectl get all
# 重新部署
kubectl rollout restart deployment test-k8s
# 命令修改镜像，--record 表示把这个命令记录到操作历史中
kubectl set image deployment test-k8s test-k8s=ccr.ccs.tencentyun.com/k8s-tutorial/test-k8s:v2-with-error --record
# 暂停运行，暂停后，对 deployment 的修改不会立刻生效，恢复后才应用设置
kubectl rollout pause deployment test-k8s
# 恢复
kubectl rollout resume deployment test-k8s
# 输出到文件
kubectl get deployment test-k8s -o yaml >> app2.yaml
# 删除全部资源
kubectl delete all --all
```



### 现存问题

- 每次只能访问一个 pod，没有负载均衡自动转发到不同 pod
- 访问还需要端口转发
- Pod 重创后 IP 变了，名字也变了

##  Service

### 特性

- Service 通过 label 关联对应的 Pod
- Servcie 生命周期不跟 Pod 绑定，不会因为 Pod 重创改变 IP
- 提供了负载均衡功能，自动转发流量到不同 Pod
- 可对集群外部提供访问端口
- 集群内部可通过服务名字访问

### 启动 Service

创建 一个 Service，通过标签`test-k8s`跟对应的 Pod 关联上
`service.yaml`

```
apiVersion: v1
kind: Service
metadata:
  name: test-k8s
spec:
  selector:
    app: test-k8s   # 要和pod的名字对应起来
  type: ClusterIP
  ports:  # 数组类型 可以多个
    - port: 8080        # 本 Service 暴露的端口
      targetPort: 8080  # 容器的端口
```

先启动pod

```
PS G:\xrqProjects\test-k8s-main\yaml\service>  kubectl apply -f .\app.yaml
deployment.apps/test-k8s created
```

再启动service

```
PS G:\xrqProjects\test-k8s-main\yaml\service> kubectl apply -f .\service.yaml
service/test-k8s created
```

```
PS G:\xrqProjects\test-k8s-main\yaml\service> kubectl get pod
NAME                        READY   STATUS    RESTARTS   AGE
test-k8s-8598bbb8c6-25d2c   1/1     Running   0          9m21s
test-k8s-8598bbb8c6-2llv2   1/1     Running   0          9m21s
test-k8s-8598bbb8c6-2wcgm   1/1     Running   0          9m21s
test-k8s-8598bbb8c6-ggljl   1/1     Running   0          9m21s
test-k8s-8598bbb8c6-px5n7   1/1     Running   0          9m21s
```

### 查看服务

```
PS G:\xrqProjects\test-k8s-main\yaml\service> kubectl get service
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP    12m
test-k8s     ClusterIP   10.108.160.222   <none>        8080/TCP   44s
```

 test-k8s 这个service 的 ip  10.108.160.22是固定的

也可以用 `kubectl get svc`

```
PS G:\xrqProjects\test-k8s-main\yaml\service> kubectl get svc
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP    20m
test-k8s     ClusterIP   10.108.160.222   <none>        8080/TCP   8m44s
```

查看服务详情 `kubectl describe svc test-k8s`， test-k8s是service名字可以发现 Endpoints 是各个 Pod 的 IP，也就是他会把流量转发到这些节点。

```
PS G:\xrqProjects\test-k8s-main\yaml\service> kubectl describe svc test-k8s
Name:              test-k8s
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          app=test-k8s
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.108.160.222
IPs:               10.108.160.222
Port:              <unset>  8080/TCP
TargetPort:        8080/TCP
Endpoints:         172.17.0.2:8080,172.17.0.3:8080,172.17.0.4:8080 + 2 more...
Session Affinity:  None
Events:            <none>
```

```
PS G:\xrqProjects\test-k8s-main\yaml\service> kubectl get pod -o wide
NAME                        READY   STATUS    RESTARTS   AGE   IP           NODE       NOMINATED NODE   READINESS GATES
test-k8s-8598bbb8c6-25d2c   1/1     Running   0          25m   172.17.0.6   minikube   <none>           <none>
test-k8s-8598bbb8c6-2llv2   1/1     Running   0          25m   172.17.0.7   minikube   <none>           <none>
test-k8s-8598bbb8c6-2wcgm   1/1     Running   0          25m   172.17.0.4   minikube   <none>           <none>
test-k8s-8598bbb8c6-ggljl   1/1     Running   0          25m   172.17.0.3   minikube   <none>           <none>
test-k8s-8598bbb8c6-px5n7   1/1     Running   0          25m   172.17.0.2   minikube   <none>           <none>
```





### 服务的默认类型是`ClusterIP`，只能在集群内部访问，我们可以进入到 Pod 里面访问： 意思是进入到 pod里面 可以访问 sevice

`kubectl exec -it pod-name -- bash`
`curl http://test-k8s:8080`

```
PS G:\xrqProjects\test-k8s-main\yaml\service> kubectl exec -it test-k8s-8598bbb8c6-ggljl -- bash
root@test-k8s-8598bbb8c6-ggljl:/app# curl http://test-k8s:8080
index page

IP lo172.17.0.4, hostname: test-k8s-8598bbb8c6-2wcgm
```

### 如果要在集群外部访问，可以通过端口转发实现（只适合临时测试用）：

`kubectl port-forward service/test-k8s 8888:8080`

> 如果你用 minikube，也可以这样`minikube service test-k8s`

直接弹出页面访问



### 对外暴露服务

上面我们是通过端口转发的方式可以在外面访问到集群里的服务，如果想要直接把集群服务暴露出来，我们可以使用`NodePort` 和 `Loadbalancer` 类型的 Service

将service.yaml修改为



```
apiVersion: v1
kind: Service
metadata:
  name: test-k8s
spec:
  selector:
    app: test-k8s
  # 默认 ClusterIP 集群内可访问，NodePort 节点可访问，LoadBalancer 负载均衡模式（需要负载均衡器才可用）
#  type: ClusterIP
  type: NodePort
  ports:
    - port: 8080        # 本 Service对外暴露 的端口
      targetPort: 8080  # 服务访问容器 的 容器端口
      nodePort: 31000   # 节点（如果是裸机，需要到对应的node节点）端口，范围固定 30000 ~ 32767
```



在minikube中 是进入dokcer中的这个minikube 容器 ，如果是裸机，需要到对应的node节点去运行该curl命令



http://localhost:31000/

  ![](NodePort.png)





##  StatefulSet

### 部署 StatefulSet 类型的 Mongodb



##### 1.创建 StatefulSet 配置文件mongo.yaml（service 和 StatefulSet 都在一个yaml里）

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
spec:
  serviceName: mongodb
  replicas: 3
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
        - name: mongo
          image: mongo:4.4
          # IfNotPresent 仅本地没有镜像时才远程拉，Always 永远都是从远程拉，Never 永远只用本地镜像，本地没有则报错
          imagePullPolicy: IfNotPresent
---
apiVersion: v1
kind: Service
metadata:
  name: mongodb
spec:
  selector:
    app: mongodb
  type: ClusterIP
  # HeadLess  不会分配集群的ip了 ，只能通过服务名字来访问
  clusterIP: None
  ports:
    - port: 27017
      targetPort: 27017
```

##### 启动 

```
PS G:\xrqProjects\test-k8s-main\yaml\statefulset> kubectl apply -f mongo.yaml
statefulset.apps/mongodb created
service/mongodb created
PS G:\xrqProjects\test-k8s-main\yaml\statefulset> kubectl get pod
NAME        READY   STATUS    RESTARTS   AGE
mongodb-0   1/1     Running   0          5m30s
mongodb-1   1/1     Running   0          4m43s
mongodb-2   1/1     Running   0          4m41s
PS G:\xrqProjects\test-k8s-main\yaml\statefulset> kubectl get statefulset
NAME      READY   AGE
mongodb   3/3     7m24s

```



### StatefulSet 特性

- Service 的 `CLUSTER-IP` 是空的，Pod 名字也是固定的。
- Pod 创建和销毁是有序的，创建是顺序的，销毁是逆序的。
- Pod 重建不会改变名字，除了IP，所以不要用IP直连



访问时，如果直接使用 Service 名字连接，会随机转发请求
要连接指定 Pod，可以这样`pod-name.service-name`
运行一个临时 Pod 连接数据测试下

kubectl run mongodb-client --rm --tty -i --restart='Never' --image docker.io/bitnami/mongodb:4.4.10-debian-10-r20 --command -- bash

```
I have no name!@mongodb-client:/$ mongo --host mongodb-0.mongodb
MongoDB shell version v4.4.10
connecting to: mongodb://mongodb-0.mongodb:27017/?compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("2904e96c-22e9-4a6e-a969-85467c83a68f") }
MongoDB server version: 4.4.24
---
The server generated these startup warnings when booting:
        2023-09-04T05:31:54.964+00:00: Using the XFS filesystem is strongly recommended with the WiredTiger storage engine. See http://dochub.mongodb.org/core/prodnotes-filesystem
        2023-09-04T05:31:55.667+00:00: Access control is not enabled for the database. Read and write access to data and configuration is unrestricted
        2023-09-04T05:31:55.667+00:00: /sys/kernel/mm/transparent_hugepage/enabled is 'always'. We suggest setting it to 'never'
---
>

```

