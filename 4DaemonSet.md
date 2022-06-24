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

- 如何在指定的Node上创建新Pod?

  - Pod.spec.affinity中有字段nodeAffinity
  - 

