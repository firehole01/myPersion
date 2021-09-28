# k8s底座上安装catalog和skyline

##  基于k8s进行容器化部署

### 前置条件

拥有一套部署好的k8s集群，以及三台用于部署数据库的节点；

### foundationdb分布式部署

foundationDB建议部署2副本，部署参见文件《foundationdb-distributed-deploy.md》文档 。

### 镜像准备

如果在外网环境下，我们的catalog:v1和skyline:v1镜像已经推到开源镜像仓库；

可以使用docker search dongsishan 查询到catalog和skyline的两个镜像；

通过docker pull dongishan/catalog:v1 或者 docker pull dongishan/skyline-server:v1下载到镜像；

### catalog server部署（在k8s的master节点上执行）

#### 1.创建配置项configMap;

```
$  kubectl create configmap CFGNAME --from-literal=KEY=VALUE 
```

CFGNAME 为要创建的配置名称，可以设置为catalog-configmap 

KEY为fdb.cluster，VALUE 为foundationdb启动服务器上/etc/foundationdb/fdb.cluster文件中的内容； 

或者先从远端获取到配置文件，修改后进行创建；

```
$  wget https://raw.githubusercontent.com/firehole01/myPersion/main/catalog-configmap.yaml
```

修改文件中fdb.cluster的值：

```
$  vim  catalog-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
	name:  catalog-configmap
	namespace: default
data:
	fdb.cluster:  xxxxxx    #将值修改成/etc/foundationdb/fdb.cluster文件的内容

$  kubectl create -f catalog-configmap.yaml
```

查看当前所有的配置文件

```
$   kubectl get configmap --all-namespaces 

NAMESPACE     NAME                                 DATA   AGE  
default       catalog-configmap                       1      2s 

$   kubectl describe configmap catalog-configmap    #用于查看配置内容是否正确
```

#### 2. 创建工作负载deployment

下载yaml文件

```
$ wget https://raw.githubusercontent.com/firehole01/myPersion/main/catalog-deployment.yaml
```

修改该文件中的configmap名称以及image镜像地址

```
$ vim catalog_deployment.yaml

        spec:
            volumes:
              - name: vol-15927317327    #使用字母-数字的格式进行命名
                configMap:
                    name: catalog-configmap    # 前面创建的configMap的名称
                    items:
                      - key: fdb.cluster
                        path: fdb.cluster
            containers:
              - name: containers-catalog-demo     #给要启动的容器命名，中间不能出现下划线
                image: dongsishan/catalog:v1   
                           #如果镜像地址已经知道，这里要填完整的镜像路径，否则要保证各节点能找到该镜像
                volumeMounts:
                  - name: vol-15927317327     #前面spec.template.spec.volume.name设置的值
                    readOnly: true
                    mountPath: /etc/foundationdb
              
 保存退出；
 $  kubectl create deployment -f catalog_deployment.yaml
```

 查看工作负载

```
$ kubectl get depoyments  --all-namespaces
$ kubectl get pods -o wide
```

备注：

> 如果出现异常，可以通过下述命令进行删除
>
> ```
> $ kubectl delete depoyments/xxxxx    # xxxx代表创建的工作负载的名称
> ```

#### 3.创建服务Service

```
$   kubectl create service clusterip catalog --tcp 8082:8082
```

clusterip表示服务访问类型-集群内访问；

除此之外，还有loadbalancer(负载均衡)， nodeport(节点端口)；

对于catalog服务要选用集群内的访问；

--tcp port1:port2 是指容器端口设置为8082，访问端口设置为 8082



### skyline server部署（在k8s的master节点上执行）

#### 1.创建工作负载deployments

下载yaml文件：

```
$  wget https://raw.githubusercontent.com/firehole01/myPersion/main/skyline-deployment.yaml
```

修改yaml文件中的镜像仓库地址；

```
$ vim skyline_deployment.yaml

        spec:
            containers:
              - name: containers-skyline-demo     #给要启动的容器命名，中间不能出现下划线
                image: dongsishan/skyline-server:v1   
                           #如果镜像地址已经知道，这里要填完整的镜像路径，否则要保证各节点能找到该镜像
                env:
                  - name: CATALOG_HOST     #配置环境变量用于skyline找catalog通信
                    value: catalog
                    
保存退出；
$  kubectl create deployment -f skyline_deployment.yaml
```

#### 2.创建服务service

```
$   kubectl create service loadbalancer skyline --tcp 9001:9002
```

对于skyline服务要选用负载均衡访问方式；

--tcp port1:port2 是指两个端口分别为9001和9002

