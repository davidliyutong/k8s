# Storage

## Etcd-based

### ConfigMap

We can consider ConfigMap as a PV-dir which contains a set of *variables* or files.  

- variables (key-value): if cm is mounted, key is displayed as file in the dir, value as content of the file
- file: if cm is mounted, file is displayed as file in the dir

#### CMD

- create from yaml

```shell
kubectl apply -f 10_cm1-pod-env.yaml 
kubectl exec cm1-pod-env -- env # display the env variables
```

- create from a host dir

```shell 
kubectl create configmap cm2 --from-file=./configs
```

- create from a host file

```shell
kubectl create configmap cm3 --from-file=./configs/db.conf
```

- create from a key-value

```shell
kubectl create configmap cm4 --from-literal=key5=value5 
```

- pass all key-value pairs of CM to ENV

```shell
kubectl apply -f 12_cm1-pod2-env.yaml
kubectl logs cm1-pod2-env
```

- pass each key-value pair of  CM to ENV

```shell
kubectl create -f 14_cm1-pod3-env.yaml
kubectl logs cm1-pod3-env
```

- mount ConfigMap as volume：key-->文件名，value-->文件的内容

```shell
kubectl create -f 16_cm1-pod4-vol.yaml
kubectl exec cm1-pod4-vol -- ls /etc/config
```

### Secret

Secret 是用来保存和传递密码、密钥、认证凭证这些敏感信息的对象。使用 Secret 的好处是可以避免把敏感信息明文写在配置文件里。在 K8s 集群中配置和使用服务不可避免的要用到各种敏感信息实现登录、认证等功能，例如访问 AWS 存储的用户名密码。为了避免将类似的敏感信息明文写在所有需要使用的配置文件中，可以将这些信息存入一个 Secret 对象，而在配置文件中通过 Secret 对象引用这些敏感信息。这种方式的好处包括：意图明确、避免重复、减少暴露机会。

创建 secret 时会用 BASE64 编码之后以与 ConfigMap 相同的方式存到 Etcd，当 mount 到一个 pod 时会先解密在挂载。

使用场景：docker-registry、generic、tls

#### 编码/解码

```shell
echo -n 'admin' | base64 # --> YWRtaW4=
echo -n 'redhat' | base64 # --> cmVkaGF0
echo 'YWRtaW4=' | base64 --decode # --> admin
echo 'cmVkaGF0' | base64 --decode # --> redhat
```

#### CMD

```shell
kubectl apply -f 20_secret1.yaml
kubectl get secret
kubectl apply -f 22_secret2-pod-env.yaml
kubectl logs secret2-pod-env # username, password decoded already
kubectl apply -f 24_secret3-pod-volume.yaml
kubectl exec secret3-pod-volume -- cat /xxx/username
```

## Volume-based

PV 和 PVC 使得 K8s 集群具备了存储的逻辑抽象能力，使得在配置 Pod 的逻辑里可以忽略对实际后台存储技术的配置，而把这项配置的工作交给 PV 的配置者，即集群的管理者。存储的 PV 和 PVC 的这种关系，跟计算的 Node 和 Pod 的关系是非常类似的：PV 和 Node 是资源的提供者，根据集群的基础设施变化而变化，由 K8s 集群管理员配置；而 PVC 和 Pod 是资源的使用者，根据业务服务的需求变化而变化，由 K8s 集群的使用者即服务的管理员来配置。

### Pod Volume
必须在定义pod的时候同时定义pod volume，其生命周期为pod的生命周期。

Volume is always hosted by each local node. 

#### emptyDir
Create a new empty dir for the volume

```shell
kubectl apply -f 30_vol1-pod-emptydir.yaml
kubectl exec vol1-emptydir -- ls /data
```

#### hostPath
Use an existing dir for the volume

```shell
kubectl create -f 32_vol2-pod-hostpath.yaml
kubectl exec vol2-pod-hostpath -- ls /data 
```

### Persistent Volume
#### Persistent Volume (PV)
对底层共享存储的抽象。

PV is independent from pods, has the life-cycle as the whole k8s cluster. PV is not hosted on a node, it belongs to the k8s cluster.

##### 访问模式
- (RWO) ReadWriteOnce – the volume can be mounted as read-write by a single node (单node的读写) 
- (ROM) ReadOnlyMany – the volume can be mounted read-only by many nodes (多node的只读) 
- (RWM) ReadWriteMany – the volume can be mounted as read-write by many nodes (多node的读写) 

##### 回收策略
- Retain 保留策略：允许人工处理保留的数据。（默认）
- Delete 删除策略：将删除pv和外部关联的存储资源，需要插件支持。
- Recycle 回收策略：将执行清除操作，之后可以被新的pvc使用，需要插件支持。

##### PV卷阶段状态
- Available – 资源尚未被claim使用
- Bound – 卷已经被绑定到claim了
- Released – claim被删除，卷处于释放状态，但未被集群回收。
- Failed – 卷自动回收失败

##### CMD

```shell
kubectl apply -f 40_pv1.yaml
```

#### PersistentVolumeClaim (PVC)
用户对于存储资源的申请。

PVC is used to create a PV which will be later declared and used in a pod.

```shell
kubectl apply -f 42_pvc1-pod.yaml # create a PVC which will bound to the PV1, and create a pod
kubuctl exec pvc1-pod -- ls /data
```

#### Storage Class

上面介绍的 PV 和 PVC 模式需要运维人员先创建好 PV，然后开发人员定义 PVC 进行一一 Bond。但是如果 PVC 请求成千上万，那么就需要创建成千上万的 PV，对于运维人员来说维护成本很高。K8s 提供一种自动创建 PV 的机制，叫 StorageClass，它的作用就是创建 PV 的模板。

具体来说，StorageClass 会定义一下两部分：

1. PV 的属性 ：比如存储的大小、类型等；
2. 创建这种 PV 需要用到的存储插件：比如 Ceph 等；

有了这两部分信息，K8s 就能根据用户提交的 PVC 找到对应的 StorageClass，然后 K8s 就会调用 StorageClass 声明的存储插件创建需要的 PV。

这里我们以NFS为例，要使用NFS，我们就需要一个nfs-client的自动装载程序，我们称之为Provisioner，这个程序会使用我们已经配置好的NFS服务器自动创建持久卷，也就是自动帮我们创建PV。

```shell
kubectl apply -f 50_sc1-hostpath.yaml # create a default storage class
kubectl apply -f 52_sc1-pvc-pod.yaml # create a PVC and a pod
kubectl exec -it storage-pvc-sc -- /bin/sh # access to the pod and test the storage
```

## Third-party Drivers

### Ceph

- [Ceph](ceph/README.md)


