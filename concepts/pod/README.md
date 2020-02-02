# Pod

## Introduction
Atomic unit, *it always runs on 1 node*. Only containers with the same lifecycle are in the same pod. 
- computing: 1 or N containers, but the number of containers within 1 pod is fixed
- storage: all containers in 1 pod have the same mount point, can access the same shared volume
- networking: all the containers in 1 pod use a unique IP address (by the `pause` container), communicate with one another using `localhost`
- scalability: increase or decrease the number of pods

## Properities

### RestartPolicy

- Always：只要退出就重启
- OnFailure：失败退出时重启
- Never：只要退出就再不重启

### Resources

k8s中资源的设置在pod中，由于 Pod 可以由多个 Container 组成，所以 CPU 和内存资源的限额，是要配置在每个 Container 的定义上的。这样，Pod 整体的资源配置，就由这些 Container 的配置值累加得到。

- CPU：k8s里为 CPU 设置的单位是“CPU 的个数”，比如500m，指的就是 500 millicpu，也就是 0.5 个 CPU 的意思。这样，这个 Pod 就会被分配到 1 个 CPU 一半的计算能力。
 - cpuset：可以通过设置 cpuset 把容器绑定到某个 CPU 的核上，而不是像 cpushare 那样共享 CPU 的计算能力。这种情况下，由于操作系统在 CPU 之间进行上下文切换的次数大大减少，容器里应用的性能会得到大幅提升。cpuset 方式是生产环境里部署在线应用类型的 Pod 时非常常用的一种方式。设置cpuset只需要将 Pod 的 CPU 资源的 requests 和 limits 设置为同一个相等的整数值即可。
- Memory：内存资源来说，它的单位自然就是 bytes。Kubernetes 支持你使用 Ei、Pi、Ti、Gi、Mi、Ki（或者 E、P、T、G、M、K），其中1Mi=1024\*1024；1M=1000\*1000。

#### requests vs. limits

- requests是在调度的时候使用的资源值，也就是kube-scheduler 只会按照 requests 的值进行计算。
- limits是真正设置 Cgroups 限制的值。

k8s认为容器化作业在提交时所设置的资源边界，并不一定是调度系统所必须严格遵守的，因为大多数作业使用到的资源其实远小于它所请求的资源限额。基于这种假设，Borg 在作业被提交后，会主动减小它的资源限额配置，以便容纳更多的作业、提升资源利用率。而当作业资源使用量增加到一定阈值时，Borg 会还原作业原始的资源限额，防止出现异常情况。而 Kubernetes 的 requests+limits 的做法，其实就是上述思路的一个简化版。用户在提交 Pod 时，可以声明一个相对较小的 requests 值供调度器使用，而 Kubernetes 真正设置给容器 Cgroups 的，则是相对较大的 limits 值，所以requests永远小于limits。

### Storage
pod-level storage which will be deleted when pod is destroyed.  
- emptyDir
- hostPath
- configMap
- secret

### Network
- Pod内的不通container间通过pause容器实现共享

- hostPort: expose 1 containerPort on the host

      ports: 
      - containerPort: 8080
        hostPort: 8081

- hostNetwork: expose all the containerPorts on the host

      hostNetwork: true

### Health Check
- livenessProbe：会一直检测，当应用不健康时（尤其是运行一段时间后坏了），则需要重启该pod。
  - exec:
  - tcpSocket:
  - httpGet:
  - initialDelaySeconds (s):
  - timeoutSeconds (s): 
- readinessProbe：优先于liveness，它会一直检测应用是否处于服务正常状态，当应用不健康时，不把pod标注为ready。
- Check Modes
  - CMD：
  - HTTP：
  - TCP：

### initContainer
只有当所有的initContainer都运行完之后，才会初始化containers
- `vim dpl.yaml`

    spec:
      initContainers:
      containers:

### Static Pod
静态pod不经apiserver，都是本地的pod通过kubelet直接启动。
Pod only exists on a node, managed by the local kubelet but node k8s master.  
It cannot be managed by the API server, so it cannot be managed by ReplicationController, Deployment or DaemonSet.
- launched by the config file */etc/kubelet.d/*
- launched by HTTP 

## CMD

- list pods
  - `kubectl get pods`: 
  - `keubctl get pods --show-all`
  - `kubectl get pods --watch`: 实时监控
- describe pods
  - `kubectl describe pods`
  - `kubectl describe pods POD_ID`
- launch a pod
  - `kubectl create -f POD.yml`
  - `kubectl apply -f POD.yml`
- delete pod
  - `kubectl delete pod POD_ID` 
  - `kubectl delete -f POD.yml`
- exec
  - `kubectl exec POD_ID -- CMD`: run a cmd in the 1st CT of the pod
  - `kubectl exec POD_ID -- curl localhost:8080`: internal access
  - `kubectl exec POD_ID -c CT_ID -- CMD `: run a cmd in the container CT_ID of the pod

### ConfigMap
- ENV: get 1 value from ConfigMap *Spec/Containers/env*

      name: APPLOGLEVEL     # environment variable name
      valueFrom: 
        configMapKeyRef: 
          name: cm-appvars
          key: apploglevel

- ENV: get all values from ConfigMap: *Spec/Containers/env*

      envFrom: 
        configMapRef: 
          name: cm-appvars

- volumeMount: *spec/containers/volumeMounts*

      name: serverxml       # volume name
      mountPath: /configfiles
      
      volumes: 
      - name: severxml
       configMap: 
         name: cm-appconfigfiles
         items: 
         - key: key-severxml
           path: sever.xml
         - key: key-loggingproperties
           path: logging.properties


### Downward API
- ENV: get pod info from field *Spec/Containers/env*

      name: MY_POD_NAME     # environment variable name
      valueFrom: 
        fieldRef: 
          fieldRef: metadata.name
  
  - metadata: fixed info about a pod 
    - metadata.name
    - metadata.namespace 
  - status: variable info about a pod
    - status.podIP

- ENV: get container info from resourceField *Spec/Containers/env*

      name: MY_CPU_REQUEST     # environment variable name
      valueFrom: 
        resourceFieldRef: 
          containerName: test-container
          resource: requests.cpu
  - requests.cpu
  - requests.memory
  - limits.cpu
  - limits.memory  
- volume: *volumes*

      - name: podinfo
        downwardAPI:
          items: 
            - path: "labels" # create a file called "labels"
              fieldRef: 
                fieldPath: metadata.labels      # all the labels of in the metadata


## Exercises
### 1 Pod with 1 Container
- `kubectl create -f pod1.yaml`
- `kubectl exec -it pod1 -- env`
- `kubectl exec -it pod1 -- /bin/sh`
- `kubectl describe pod pod1`: get IP address
- `ping POD1_IP`: can ping pod1

### 1 Pod with 2 Containers
- `kubectl create -f pod2.yaml`
- `kubectl exec -it pod2 -c ct-nginx -- /bin/bash`
  - `apt-get update`
  - `apt-get install curl`
  - `curl localhost`: get the hello message from the container
- `kubectl describe pod pod2`: get IP address
- `curl POD2_IP`: get the hello message from the node
- `kubectl exec -it pod2 -c ct-debian -- /bin/bash`
  - `echo Chanage message from pod2-ct-busybox > /data/index.html `
- `curl POD2_IP`: get the new message from the node

### Pod with resource limitation
- `kubectl apply -f pod3.yaml`
这个pod状态变为**OOMKilled**，因为它是内存不足所以显示Container被杀死


### Pod Liveness CMD Check
- `kubectl apply -f pod-liveness-cmd.yaml`
- `kubectl get pods`: 通过查看发现liveness-exec的RESTARTS在30秒后由于检测到不健康一直在重启

### Pod Liveness HTTP Check
- `kubectl apply -f pod-liveness-http.yaml`: `k8s.gcr.io/liveness`镜像会使`/healthz`服务时好时坏
- `kubectl get pods`
- `curl 192.168.2.19:8080/healthz`

### Pod NodeSelector
- `kubectl label nodes node01 disktype=ssd`
- `kubectl get nodes node01 --show-labels`
- `kubectl apply -f pod-nodeSelector.yaml`
- `kubectl get pod -o wide`

### InitContainer
- `kubectl apply -f pod-initcontainer.yaml`: the init CT creates the file 'testfile'
- `kubectl exec -it myapp-pod -- ls /storage/`

### Static Pod
- `mv pod-static.yaml /etc/kubernetes/manifests/`：kubelet就会自动启动static pod
- `kubectl get pod`
- `kubectl delete pod`
- `kubectl get pod`：看到有删除该pod，但是是骗人的
