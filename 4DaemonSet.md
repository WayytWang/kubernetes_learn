DaemonSet的主要作用是在kubernetes集群里，运行一个Daemon Pod，这个Pod有如下三个特征：

- 这个Pod运行在kubernetes集群里的每一个节点（Node）上
- 每个节点上只有一个这样的Pod实例。
- 当有新的节点加入集群时，该Pod会自动地在新节点上被创建出来；当旧节点被删除后，它上面的Pod也相应地会被回收掉。



这种Daemon Pod的意义在于:

- 各种网络插件的Agent组件，都必须运行在每一个节点上，用来处理这个节点上的容器网络
- 各种存储插件的Agent组件，都必须运行在每一个节点上，用来在这个节点上挂载远程存储目录，操作容器的Volume目录
- 各种监控组件和日志组件，也必须运行在每一个节点上，负责这个节点上的监控信息和日志收集



更重要的是，DaemonSet开机运行时机，很多时候比整个kubernetes集群出现的时机都要早，这样会有一些问题，如：

- 假设DaemonSet是一个网络插件的Agent组件，这时候整个kubernetes集群还没有可用的容器网络，所有的Worker节点的状态都是NotReady（NetworkReady=false）。这种情况下，普通的Pod肯定不能运行在这个集群上。



但是DaemonSet控制的Pod可以



# DaemonSet

- 对应的yaml

  ```yaml
  apiVersion: apps/v1
  kind: DaemonSet
  metadata:
    name: fluentd-elasticsearch
    namespace: kube-system
    labels:
      k8s-app: fluentd-logging
  spec:
    selector:
      matchLabels:
        name: fluentd-elasticsearch
    template:
      metadata:
        labels:
          name: fluentd-elasticsearch
      spec:
        tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
        containers:
        - name: fluentd-elasticsearch
          image: k8s.gcr.io/fluentd-elasticsearch:1.20
          resources:
            limits:
              memory: 200Mi
            requests:
              cpu: 100m
              memory: 200Mi
          volumeMounts:
          - name: varlog
            mountPath: /var/log
          - name: varlibdockercontainers
            mountPath: /var/lib/docker/containers
            readOnly: true
        terminationGracePeriodSeconds: 30
        volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
  ```

  - Docker容器里的应用日志，默认会保存到在宿主机的/var/lib/docker/containers/{{.容器ID}}/{{.容器ID}}-json.log文件里
  - 这个DaemonSet的作用是，将宿主机中的Docker容器日志的和宿主机本身的日志挂载到Pod中，交由fluentd处理并转发到elasticsearch中

- DaemonSet是如何保证每个Node上有且只有一个被管理的Pod？

  - DaemonSet Controller会去从Etcd中获取所有的Node列表，然后一一检查是否有name=fluentd-elasticsearch标签的Pod在运行
  - 检查见过有三种情况
    - 无Pod则创建
    - 有Pod，数量大于1，则移除多余的Pod
    - 正好一个，完美

- 删除很简单，但如何在指定的Node上创建新Pod?

  - Pod.spec.affinity中有字段nodeAffinity
  
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: with-node-affinity
  spec:
    affinity:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: metadata.name
              operator: In
              values:
              - node-geektime
  ```
  
  - requireDuringSchedulingIgnoredDuringExecution:表示这个nodeAffinity必须在每次调度时候予以考虑。同时也意味可以设置某些情况下不考虑这个nodeAffinity。
  
  - operator的可以支持丰富的语法，In表示部分匹配。
  
  - 这个Pod将来只允许在“metadata.name”是“node-geektime”的节点上。
  
  - DaemonSet Controller会在创建Pod的时候，自动在这个Pod的API对象里加上nodeAffinity定义，节点名就是当前遍历到的Node名。
  
  - DaemonSet并不需要修改用户提交的YAML文件里的Pod模板，而是在向kubernetes发起请求之前，直接修改根据模板生成的Pod对象。
  
  - DaemonSet还会给这个Pod自动加上另外一个与调度相关的字段:"tolerations",这意味者，这个Pod会容忍某些Node的Taint。
  
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
    	name: with-toleration
    spec:
    	tolerations:
    	- key: node.kubernetes.io/unschedulable
    	  operator: Exists
    	  effect: NoSchedule
    ```
  
    - 这表示了所有被标记为“unschedulable”污点的Node，都被容忍允许调度。（调度过程见后面的调度部分）
  
    - DaemonSet能先于kubernetes集群部署，靠的就是这个字段
  
    - 假如当前DaemonSet管理的是一个网络插件的Agent Pod，那么就必须在这个DaemonSet的YAML文件里，给它的Pod模板加上一个能够容忍网络不可用的toleration：
  
      ```yaml
      template:
      	metadata:
      		labels:
      			name: network-plugin-agent
      		spec:
              	tolerations:
              	- key: node.kubernetes.io/network-unavailable
              	  operator: Exists
              	  effect: NoSchedule
      ```
  
      - 这就是为什么能先部署集群本身，在部署网络插件的原因。
  
  - 一般来说master节点上，kubernetes集群是不允许部署Pod的，因为Master节点默认有一个污点：`node-role.kubernetes.io/master`。如果想要在Master节点上部署Pod，只需带上相应的toleration即可。



## 实际create一个DaemonSet对象

- 创建
- 查看Pod
- 查看DaemonSet



## ControllerRevision

- DaemonSet也能做版本管理
  - 设置镜像版本：`kubectl set image ds/fluentd-elasticsearch fluentd-elasticsearch=k8s.gcr.io/fluentd-elasticsearch:v2.2.0 --record -n kube-system `
    - `--record`会记录本次运行的指令
  - 查看滚动更新的过程：`kubectl rollout status ds/fluentd-elasticsearch -n kube-system`
  - 查看历史版本：`kubectl rollout history daemonset fluentd-elasticsearch -n kube-system`
  - 可以回滚历史版本
  
- DeploymentSet是靠ReplicaSet实现的版本管理，一个版本对应一个ReplicaSet。

- DaemonSet靠的是ControllerRevision，专门用来记录某种Controller对象的版本。刚刚部署的DaemonSet也有对应的ControllerRevision。

- StatefulSet也是依靠ControllerRevision做版本管理。

  
