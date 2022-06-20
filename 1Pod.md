---
typora-root-url: pic
---

# Pod

- pod是k8s的原子调度单位

## 为什么需要Pod？

- 容器的本质是进程，而k8s更像是操作系统

- 有可能一个业务需要多个容器，而这些容器之间有“超亲密关系”：

  - 互相之间会发生直接的文件交换

  - 使用localhost或者Socket文件进行本地通信

  - 非常频繁的远程调用

  - 共享某些Linux Namespace

    ......

- 这些容器组需要部署在同一台机器，这就需要Pod

- Pod比容器更像“虚拟机”

### 为什么非要抽出一个Pod概念？

- 从k8s的调度层面，完全可以将这么一组“超亲密关系”容器放在同一个Node上。

- Pod只是一个逻辑概念

  - k8s真正处理的，还是宿主机操作系统的Cgroups，Namespace

- Pod其实是一组共享了某些资源的容器

  - 比如A、B两个容器共享同一个Network Namespace要如何操作？如何避免两个容器存在拓扑关系？

    - Infra容器
      - 在Pod中，这是第一个创建的容器
      - 占用极少的资源，使用一个特殊的镜像：k8s.gcr.io/pause
      - Infra容器“Hold”住Network Namespace后，其他用户容器就能加入Infra容器的Network Namespace了。
      - Pod的生命周期和Infra容器一致，而与其他容器无关。

  - A、B两个容器公用一个Volume

    - A、B属于同一个Pod，将Volume的定义设计在Pod层级即可。

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: two-containers
    spec:
      restartPolicy: Never
      volumes:
        - name: shared-data
          hostPath:
            path: /data
      containers:
        - name: nginx-container
          image: nginx
          volumeMounts:
            - name: shared-data
              mountPath: /usr/share/nginx/html
        - name: debian-container
          image: debian
          volumeMounts:
            - name: shared-data
              mountPath: /pod-data
          command: ["/bin/sh"]
          args: ["-c", "echo Hello from the debian container > /pod-data/index.html"]
    ```



## 典型例子

### WAR包与Web服务器

- Web服务器需要WAR包

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: javaweb-2
spec:
  volumes:
    - name: app-volume
      emptyDir: {}
  initContainers:
    - name: war
      image: xxxx/war:v2
      command: ["cp","/sample.war","/app"]
      volumeMounts:
        - mountPath: /app
          name: app-volume
  containers:
    - name: tomcat
      image: xxxx/tomcat:7.0
      command: ["sh","-c","/root/apache-tomcat-7.0.42-v2/bin/start.sh"]
      volumeMounts:
        - mountPath: /root/apache-tomcat-7.0.42-v2/webapps
          name: app-volume
      ports:
        - containerPort: 8080
          hostPort: 8001
```

- Init Container会先于spec.containers先启动，直到Init Container启动并退出了，用户容器才会启动
- 这里将war包拷贝到了app-volume中，tomcat容器启动时，会去app-volume中找到war包
- 这种启用辅助容器来完成一些独立与主容器之外的模式叫做**sidecar**



### 容器日志收集

- 在Pod层建立一个Volume
- 应用容器将日志输出在这个Volume
- 启动一个sidecar容器，从Volume中读取日志，将日志收集





## 深入解析Pod对象

- 凡是调度、网络、存储以及安全相关的属性，基本都是Pod级别的。这些描述的是“机器”这个整体，而不是运行的“程序”

### 机器相关属性

#### NodeSelector

- 是一个供用户将Pod与Node进行绑定的字段

  ```yaml
  apiVersion: v1
  kind: Pod
  ...
  spec:
    nodeSelector:
      disktype: ssd
  ```

  - Pod永远只能运行在携带“disktype: ssd”标签的节点上

#### NodeName

- 一旦Pod的这个字段被赋值，k8s会认为这个Pod已经经过了调度。所以在测试的时候，可以通过该字段“骗过”调度器

#### HostAliases

- 定义了Pod的hosts文件

  ```yaml
  apiVersion: v1
  kind: Pod
  ...
  spec:
   hostAliases:
   - ip: "10.1.2.3"
     hostnames:
     - "foo.remote"
     - "bar.remote"
  ```

- 在`/etc/hosts`文件中，就能看到YAML文件中配置的数据

- 在k8s项目中，如果要设置hosts文件，一定要通过这种方法。直接修改hosts文件，在Pod被删除重建后，kubelet会自动覆盖掉被修改的内容

### Linux Namespace相关属性

#### Pod中所有容器共享PID Namespace

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  shareProcessNamespace: true
  containers:
  - name: nginx
    image: nginx
  - name: shell
    image: busybox
    stdin: true
    tty: true
```

- Pod中有两个容器

  - nginx容器
  - 开启了tty和stdin的shell容器

- 运行YAML文件后，使用命令 `kubectl attach -it nginx -c shell`进入nginx Pod中的shell容器，并使用`ps ax`命令看到整个Pod内的进程

   ![](/shareProcessNamespace.png)

  

### Containers字段

#### ImagePullPolicy 镜像拉取策略

- Always
  - 默认值。每次创建Pod时，都会拉取一次镜像。当spec.containers.name为nginx或者nginx：latest时也会被认为时Always
- Never
  - 从不
- IfNotPresent
  - 当宿主机不存在时拉取

#### Lifecycle

- 容器状态发生变化时触发的一系列“钩子”

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: lifecycle-demo
  spec:
    containers:
      - name: lifecycle-demo-container
        image: nginx
        imagePullPolicy: IfNotPresent
        lifecycle:
          postStart:
            exec:
              command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]
          preStop:
            exec:
              command: ["/usr/sbin/nginx","-s","quit"]
  ```

- imagePullPolicy的值填写的IfNotPresent

   ![](/shareProcessNamespace.png)

  - 第二个事件Message可以看到，已经没有重新拉取镜像了

- postStart

  - 在容器的ENTRYPOINT执行后执行，但是不能严格保证顺序，因为执行postStart时，ENTRYPOINT可能还没完



- preStop
  - 在容器被杀死之前执行
  - 它会阻塞当前杀死容器流程，等到Hook执行完后，容器才会退出

### Status部分

- 除metadata和spec之外，还有status字段

#### pod.status.phase

- Pending
  - YAML文件提交给了k8s，API对象已经被创建并保存在Etcd中。但是Pod中的某些容器因为某种原因不能顺利被创建，比如调度不成功
- Running
  - Pod已经调度成功了，和一个具体节点绑定。它包含的容器都已经成功创建了，至少有一个正在运行中
- Succeeded
  - Pod中的所有容器都正常运行完毕，并且已经退出了。在一次性任务中比较常见
- Failed
  - Pod里至少有一个容器以不正常的状态退出，这个状态的出现，需要Debug查看这个容器的应用
- Unknown
  - 异常状态，Pod的状态不能持续地被kubelet汇报给kube-apiserver，可能是主从节点间的通信处了问题

#### pod.status.conditions

- 更细分的状态值
- 用来描述造成当前Status的具体原因
- 比如当前Pod状态是Pending，对应的Condition是Unschedulable表示是调度出现了问题
- Ready意味Pod不仅已经正常启动，还可以对外提供服务
  - Running和Ready有一定区别？






## projected volume

- 不是为了存放容器中的数据，也不是为了容器与宿主机之间的数据交换，而是为了让容器使用预先定义好的数据
- 分为四类
  - Secret
  - ConfigMap
  - Downward API
  - ServiceAccountToken

### secret

Secret对象可以帮Pod中要访问的加密数据存放到Etcd中。然后可以在Pod容器中挂载Volume的方式，访问到这些Secret里保存的信息了。更重要的是，一旦对应的Etcd里的数据被更新，这些Volume里的文件内容同样也会被更新，这是kubelet组件在定时维护这些Volume。

- 编写PodYaml文件，挂载secret

  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: test-projected-volume 
  spec:
    containers:
    - name: test-secret-volume
      image: busybox
      args:
      - sleep
      - "86400"
      volumeMounts:
      - name: mysql-cred
        mountPath: "/projected-volume"
        readOnly: true
    volumes:
    - name: mysql-cred
      projected:
        sources:
        - secret:
            name: user
        - secret:
            name: pass
  ```

  - PodYamk文件中的`user`、`pass`对应Secret对象，生成步骤如下：

     - 编写文件

         ![](/vim-pass-user.png)

     - kubectl create secret generic xxx --from-file=xxx

       ![](/create-secret-user.png)

       ![](/create-secret-pass.png)

     - kubectl get secrets查看

        ![](/getSecrets.png)

  - 启动Pod并进入Pod查看挂载情况

    ![](/apply-secretPod-file-exec.png)

  - 另外还能使用YAML文件创建secret对象

    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: mysecret
    type: Opaque
    data:
      user: YWRtaW4=
      pass: MWYyZDFlMmU2N2Rm
    ```

    - data中的值不应该用明文，需要base64编码

       ![](/secret-base64.png)

    - 创建secret对象

       ![](/apply-secret.png)

    - 使用Pod挂载`mysecret`Volume，启动Pod，并进入Pod查看挂载情况

      ![](/apply-secretPod-exec.png)




### configMap

configMap对象与secret对象类似，区别是configMap保存的不需要加密的、应用所需的配置信息。

- kubectl create configmap xxx --from-file=xxx创建

### Downward API

- Dowbward API让Pod里的容器能够直接获得这个Pod API对象本身的信息

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-downwardapi-volume
  labels:
    zone: us-est-coast
    cluster: test-cluster1
    rack: rack-22
spec:
  containers:
    - name: client-container
      image: k8s.gcr.io/busybox
      command: ["sh", "-c"]
      args:
      - while true; do
          if [[ -e /etc/podinfo/labels ]]; then
            echo -en '\n\n'; cat /etc/podinfo/labels; fi;
          sleep 5;
        done;
      volumeMounts:
        - name: podinfo
          mountPath: /etc/podinfo
          readOnly: false
  volumes:
    - name: podinfo
      projected:
        sources:
        - downwardAPI:
            items:
              - path: "labels"
                fieldRef:
                  fieldPath: metadata.labels
```

- pod.volumes中声明了downwardAPI的Volume，暴露了metadata.labels信息给容器，pod.metadata.labels中字段将自动挂载成为容器中的/etc/podinfo/labels文件
- kubectl logs xxx 查看Downward API的日志

​		![](/downwardAPI-log.png)

- 现在DownwardAPI支持的字段非常丰富

  ```
  1. 使用fieldRef可以声明使用:
  spec.nodeName - 宿主机名字
  status.hostIP - 宿主机IP
  metadata.name - Pod的名字
  metadata.namespace - Pod的Namespace
  status.podIP - Pod的IP
  spec.serviceAccountName - Pod的Service Account的名字
  metadata.uid - Pod的UID
  metadata.labels['<KEY>'] - 指定<KEY>的Label值
  metadata.annotations['<KEY>'] - 指定<KEY>的Annotation值
  metadata.labels - Pod的所有Label
  metadata.annotations - Pod的所有Annotation
  
  2. 使用resourceFieldRef可以声明使用:
  容器的CPU limit
  容器的CPU request
  容器的memory limit
  容器的memory request
  ```

- DownwardAPI能获取的信息，一定是Pod里的容器进程启动之前就能确定下来的信息。

### Service Account

- Service Account是k8s进行权限分配的对象。比如Service Account A可以只被允许对k8s API进行GET操作，Service Account B可以对k8s API所有的操作权限。

- Service Account是一个特殊的Secret对象，叫ServiceAccountToken。所有运行在k8s集群上的应用，都必须使用ServiceAccountToken中保存的授权信息，才能合法的访问API Server。
- 为方便使用，k8s提供了一个默认的Service Account。所有Pod都可以无声明的挂载它。
  - 注意：这里好像已经改掉了，确认过后回来修改




## 容器健康检查和恢复机制

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: test-liveness-exec
spec:
  containers:
    - name: liveness
      image: busybox
      args:
        - /bin/sh
        - -c
        - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
      livenessProbe:
        exec:
          command:
            - cat
            - /tmp/healthy
        initialDelaySeconds: 5
        periodSeconds: 5
```

- 容器启动后，会创建/tmp/healthy文件，30s后，会把这个文件删除。

- 还定义了一个livenessProble，它的类型是exec，在容器启动后，会执行 cat /tmp/healthy 命令。容器启动后5s(initialDelaySeconds)执行，每5s(periodSeconds)执行一次。

- 实际操作中，启动Pod后，describe能看到Events：

  ![](/livenessProbe-events.png)

  但是，看pod的状态依旧就Running

   ![](/livenessProbe-Pod-status.png)

​		不过能发现，Restarts字段变成了1，这就是Pod恢复机制

- livenessProble还支持定义为发起HTTP或者TCP请求的方式

### Pod恢复机制

- restartPolicy是Pod的Spec部分的一个标准字段，它的值默认是Always，任何时候发生了异常，它一定会被重新创建。
- 一旦一个Pod与Node绑定了，除非解除绑定（pod.spec.node字段被修改），否则不会离开这个节点。
- 值：
  - Always
    - 只要容器不在运行状态，就自定重启容器
  - OnFailure
    - 异常时自动重启容器
  - Never
    - 从来不重启容器

- 设计恢复机制的基本设计原理
  - 只要Pod的restartPolicy指定的策略允许重启异常的容器，那么这个Pod就会保持Running状态，并进行容器重启。
  - 对于包含多个容器的Pod，只有它里面的所有的容器都进入异常状态后，Pod才会进入Failed状态。（Ready字段表示正常容器的个数）

### PodPreset

- 实验版本1.23.5无PodPreset对象

- Pod预设置

```yaml
apiVersion: settings.k8s.io/v1alpha1
kind: PodPreset
metadata:
  name: allow-database
spec:
  selector:
    matchLabels:
      role: frontend
  env:
    - name: DB_PORT
      value: "6379"
  volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
    - name: cache-volume
      emptyDir: {}
```

- PodPreset中的spec.seletor.matchLabels决定了，会挑选出metadata中定义了role=frontend label的Pod加上预定义。如以下Pod：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: website
  labels:
    app: website
    role: frontend
spec:
  containers:
    - name: website
      image: nginx 
      ports:
        - containerPort: 80
```

