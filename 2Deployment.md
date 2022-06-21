---
typora-root-url: pic
---

- Pod这个看似复杂的API，实际上是对容器进一步的抽象和封装。Pod有很多字段和属性，使得kubernetes能操作它。
- 这些“操作”逻辑由控制器（Controller）完成。Deployment是一个最基本的控制器对象。



## Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

- spec.selector.matchLabels指定了要被操作的Pod
- spec.replicas=2确保了携带app: nginx标签的Pod的个数永远是2个



### 控制器模型

- kubernetes项目中通用的编排模式：控制循环（control loop）

  ```go
  for {
      实际状态 := 获取集群中对象X的实际状态
      期望状态 := 获取集群中对象X的期望状态
      if 实际状态 == 期望状态 {
          continue
      } else {
          执行编排动作，将实际状态调整为期望状态
      }
  }
  ```

  - 实际状态来自kubernetes集群本身
    - kubelet通过心跳向控制器汇报的容器状态和节点状态
    - 监控系统中保存的应用监控数据
    - 控制器主动收集（一般从Etcd）
  - 期望状态一般来自于用户提交的YAML文件，这些信息往往都保存在Etcd中
  - 编排动作通常被叫做调谐（Reconcile）。这个调谐过程被成为 Reconcile Loop或者Sync Loop

- 控制器本身负责定义被管理对象的期望状态，比如Deployment里的replicas=2

- 被控制对象的定义来自于一个”模板“，比如Deployment里的template字段，这字段内容和Pod对象的API定义丝毫不差。

- Deployment里的template字段在kubernetes项目中有一个专有名字PodTemplate（Pod模板），大多数控制器都是使用PodTemplate来统一定义它所要管理的Pod。

- 类似Deployment这样的一个控制器，实际上都是由控制器定义（包括期望状态）部分加上下半部分的被控制对象的模板组成的。

- 在所有API对象的Metadata里，都有一个字段叫做ownerReference，用于保存当前这个API对象的拥有者（Owner）信息。

  - Deployment控制的Pod的ownerReference是谁？



## ReplicaSet

- Deployment实现了水平扩展/收缩(horizontal scaling out/in)。当更新了Deployement的Pod模板时，就需要遵循一种”滚动更新“(rolling update)的方式来升级现有容器。

- 实现滚动更新的能力，依赖的是kubernetes项目中一个非常重要的概念（API对象）：ReplicaSet

  ```yaml
  apiVersion: apps/v1
  kind: ReplicaSet
  metadata:
    name: nginx-set
    labels:
      app: nginx
  spec:
    replicas: 3
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
          image: nginx:1.7.9
  ```

  - RelicaSet对象是Deployement的一个子集，Deployment控制器实际操纵的，是这样的ReplicaSet对象而不是Pod对象。

  - 当一个Deployment定义的副本数是3时，具体实现是Deployment控制一个ReplicaSet：

      ![](/deployment-replicaset.png)

  - ReplicaSet通过”控制器模式“保证系统中有app=nginx标签的Pod数量等于指定个数。这也是Deployment只允许容器的restartPolicy=Always的主要原因。



### 水平扩展/收缩(horizontal scaling out/in)

- 有了RelicaSet控制器，水平扩展/收缩就很简单。`kubectl scale deployment xxx --replicas=4`

   ![](/scale.png)

  



### 滚动更新

#### 创建一个Deployment

- 首先使用yaml文件创建一个Deployment

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: nginx-deployment
    labels:
      app: nginx
  spec:
    replicas: 3
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
          image: nginx:1.7.9
          ports:
          - containerPort: 80
  ```

- 查看创建的Deployment状态信息

   ![](/deployment-status.png)

  - READY：/前表示当前处于Running状态的Pod个数，/后表示用户期望的Pod副本个数
  - UP-TO-DATE：表示处于最新版本的Pod个数，最新版本指Pod的Spec部分和Pod模板里定义的完全一致
  - AVAILABLE：当前已经可用的Pod个数：既是Running状态，又是最新版本，并且已经处于Ready（健康检查正确）状态的Pod个数。

- 在创建的过程中，可通过`kubectl rollout status deployment/xxx`查看状态的变化

   ![](/rollout-status.png)

- Deployment创建的ReplicaSet也能看到

   ![](/get-replicaset.png)

  - DESIRED：表示期望的Pod数
  - CURRENT：表示当前处于Running状态的Pod个数

- ReplicaSet的Name由Deployment的名字和一串随机数组成，Pod的名字又由ReplicaSet名字加一串随机数组成，ReplicaSet的随机串会被加到Pod的标签中

   ![](/replicaSet-name.png)

- Deployment在ReplicaSet基础上添加了UP-TO-DATE，用于滚动更新



#### 触发滚动更新

- 使用`kubectl edit deployment/nginx-deployment`修改Pod中容器的镜像版本，将1.8的nginx改成1.9.1

- 可以使用`kubectl rollout status deployment/nginx-deployment`查看状态，`kubectl describe deployment nginx-deployment`看Events更直观

   ![](/scale-events.png)

  - 修改Pod定义后，创建了一个新的ReplicaSet（6db6bb7b98），和之前的ReplicaSet”此消彼长“。最终结果：

     ![](/replicaSet-after-edit.png)

- Deployment的spec中由RollingUpdateStrategy可以控制”此消彼长“的状态：

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: nginx-deployment
    labels:
      app: nginx
  spec:
  ...
    strategy:
      type: RollingUpdate
      rollingUpdate:
        maxSurge: 1
        maxUnavailable: 1
  ```

  - maxSurge指定的是除了DESIRED数量之外，在一次”滚动“中Deployment控制器可以创建多个新Pod
  - maxUnavailable指的是在一次”滚动“中Deployment控制器可以删除多少个旧Pod。除了可以指定具体个数还能指定百分比，如50%，表示最多可以删除50% * DESIRED 数量个Pod

#### 一些相关命令

- `kuectl set image deployment/nginx-deployment nginx=nginx:1.9.1`设置容器镜像

- `kubectl rollout undo deployment/nginx-deployment`回滚到Deployment的上一个版本

- 回滚Deployment历史版本

  ```shell
  # 查看历史版本
  kubectl rollout history deployment/nginx-deployment
  
  # 查看历史版本具体细节
  kubectl rollout history deployment/nginx-deployment --revision=2
  
  # 回滚指定版本
  kubectl rollout undo deployment/nginx-deployment --to-revision=2
  ```

- 不想每次修改Deployment，都立马生效（开销大）。可以累计几次修改之后统一生效

  ```shell
  # 暂停真正的更新
  kubectl rollout pause deployment/nginx-deployment
  
  # ... 多次 edit
  
  # 执行
  kubectl rollout resume deployment/nginx-deployment
  ```

  