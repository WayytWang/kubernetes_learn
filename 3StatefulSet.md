---
typora-root-url: pic
---

- 为什么需要StatefulSet?
  - Deployment管理Pod时，它认为所有的Pod都是一样的，没有顺序。需要时就创建一个Pod，不需要时就任意删除一个Pod
  - 但是实际应用中，Pod之间往往是有依赖关系的，比如主从关系、主备关系。
  - 另外还有数据存储类的应用，它的多个实例（Pod）会在本地磁盘上保存一份数据。而这些实例被杀掉之后，即便重建出来，实例与数据之间的对应关系也已经丢失了，导致应用失败。
  - 这种实例之间有不对等关系的，被称为“有状态应用”。
  - 管理这些“有状态应用”就需要StatefulSet。

# StatefulSet

- StatefulSet把真实世界里的应用状态，抽象成了两种情况：
  - 拓扑状态
    - 多个实例（Pod）间必须按照顺序启动，比如A要先于B启动，不仅第一次启动时，要先A后B，那当A、B被删除后，重新创建时也需要按照顺序。
    - 并且新创建的A、B的网络标识也需要和之前一样，这样原先的访问者才能使用同样的方式访问到新创建的Pod。
  - 存储状态
    - 多个实例绑定了不同的存储数据，对于这些实例来说，PodA被重新创建后，访问的数据和重新创建之前应该是同一份。



## Headless Service

- 在StatefulSet原理描述前，先铺垫一个实用概念Headless Service

- Service是kubernetes项目中将Pod暴露给外界访问的一种机制，比如一个Deployment有3个Pod，那么通过定义一个Service，用户就能通过这个Service访问到某一个具体的Pod

- 有两种访问方式：

  - Virtual IP方式：当访问10.0.23.1这个Service的IP地址时，其实是一个VIP，它会把请求转发到该Service所代理的某一个Pod上，具体原理见之后的Service章节。
  - DNS方式，DNS方式又分两种处理方法：
    - Normal Service：访问my-svc.my-namespace.svc.cluster.local ，就解析到了my-svc这个Service的VIP，后面的流程就和VIP方式一样了。
    - Headless Service：访问my-svc.my-namespace.svc.cluster.local直接解析到Pod的IP地址，不再是VIP了。

- Headless Service

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: nginx
    labels:
      app: nginx
  spec:
    ports:
    - port: 80
      name: web
    clusterIP: None
    selector:
      app: nginx
  ```

  - 关键是clusterIP=None，证明不需要VIP。

  - 这个Headless Service所代理的所有IP地址，都会绑定在这样格式的DNS记录：

    `<pod-name>.<svc-name>.<namespace>.svc.cluster.local`

    - 这个DNS是kubernetes为Pod分配的唯一”可解析身份“

## 验证拓扑状态

- 首先启动一个StatefulSet：

  ```yaml
  apiVersion: apps/v1
  kind: StatefulSet
  metadata:
    name: web
  spec:
    serviceName: "nginx"
    replicas: 2
    selector:
      matchLabels:
        app: nginx
    template:
      metadata:
        labels:
          app: nginx
      spec:
        containers:
        - name: nginx
          image: nginx:1.9.1
          ports:
          - containerPort: 80
            name: web
  ```

  - serviceName表示和哪个Service关联

- 创建Service和StatefulSet

    ![](/apply-service-statefulset.png)

- 查看创建成功的StatefulSet的Events

    ![](/stateful-event.png)

  - StatefulSet给它所管理的Pod名称做了编号：`<statefulset name>-<ordinal index>`
  - 编号从0开始，创建顺序是按照编号顺序进行的，在web-0进入Ready状态时，web-1会一直处于Pending状态
    - 所以Pod要有探针很重要

- 查看Pod的hostname

    ![](/pod-hostname.png)

  - 每个Pod的hostname都等于Pod自己的名字

- 启动临时Pod查看kubernetes中web-0、web-1DNS的情况

  `kubectl run -i --tty --image busybox:1.28.4 dns-test --restart=Never --rm /bin/sh`

   ![](/nslookup-statefulSet-pod.png)

  - Pod的DNS拼接规则：`<pod-name>.<svc-name>.<namespace>.svc.cluster.local`

- 删除两个Pod后，重新创建时，会按照既定的顺序创建

  - `kubectl delete pod -l app=nginx`

  - 使用`kubectl get pod -w -l app=nginx`,-w参数表示watch，实时查看状态。

      ![](/watch-statefulSet-delete.png)
     
     - 两个Pod删除后，按照原顺序重新创建了两个Pod，名字保持一致

- 结论：当触发了滚动更新时，更新会按照一定的编号顺序。但是如果只是删除了某一个Pod，就只重建当前的Pod就够了。
  - 疑问：这里顺序不就乱了吗？



## PV与PVC

- 在描述存储状态之前，先简单介绍下PV与PVC的概念
- 在Pod的spec.volumes中，可以定义一个具体类型的Volume，比如hostPath
- 但是很有可能，定义者不知道有哪些Volume可以使用，或者没有关键配置信息（存储相关信息较为机密）
- kubernetes引入了一组概念：Persistent Volume Claim(PVC)和Persistent Volume(PV)的API对象。
- PVC像是接口，使用者直接在Pod中使用PVC。
- PV像是实现，由存储相关的人员提供。
- PVC需要绑定PV才能使用。

## 验证存储状态
