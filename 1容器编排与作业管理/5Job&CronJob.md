---
typora-root-url: pic
---

Deployment、StatefulSet、DaemonSet这三个编排概念编排的对象，都是“在线业务”，即Long Running Task。

但是有一类作业叫“离线业务”，或者叫做Batch Job（计算业务），这种业务在计算完成后就直接退出了，而Deployment这些会不断重启，所以不适用。

Borg项目中，对作业进行了分类处理，提出了LRS（Long Running Service）和Batch Jobs两种作业形态，对它们进行分别管理和混合调度。



对于Batch Jobs就需要用到一个API对象：Job

# Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
        - name: pi
          image: resouer/ubuntu-bc
          command: ["sh","-c","echo 'scale=10000; 4*a(1)' | bc -l "]
      restartPolicy: Never
  backoffLimit: 4
```

- 这个Job所控制的Pod做的事情就是计算π小数点后10000位的值，是一个一次性任务。

- Job控制的Pod不需要使用spec.selector

      ![](describe-job.png)

  - 这个Job管理的Pod自动加上了一个Label
    - controller-uid = <随机字符串>
  - 而Job对象本身，自动加上了Label对应的Selector
  - Job Controller之所以要使用这种携带UID的Lable，就是为了避免不同的Job对象所管理的Pod发生重合
  - Pod顺利完成计算后，会进入Completed状态
  - 使用`kubectl logs pi-lx29c`

- 如果这个离线作业失败了

  - 当restartPolicy=Never，那么离线作业失败后，Job Controller会不断地尝试创建一个新Pod。
    - 重新创建的次数由`spec.backoffLimit`决定。
    - 重新创建Pod的间隔是呈指数增加的。
  - 当restartPolicy=OnFailure，Job Controller会不断重启Pod里的容器
  - 当Pod运行结束后，会进入Completed状态，但是如果应某种原因结束不了，可以使用`spec.activeDeadlingSeconds`来限制最长运行时间（单位秒）。
    - 当超过这个限制时间后，Pod还没结束，Pod的终止原因会记录为：reason.DeadlineExceeded

- 离线任务之所以能被成为Batch Job，是因为它们可以并行执行。

  - Job的spec中还有两个参数：

    - `parallelism`：Job任意时间最多可以启动多少个Pod同时运行
    - `completions`：Job至少要完成的Pod数目

     ![](jobs-parallelism-completions.png)

    -				同时只有parellelism个Pod在Running，最终有completions个Pod进入Completed

## 常用的使用Job对象的方法

### 外部管理器 + Job模板

- 把Job的YAML文件定义为一个模板，由外部工具控制这些“模板”来生成Job

  ```yaml
  apiVersion: batch/v1
  kind: Job
  metadata:
    name: process-item-$ITEM
    labels:
      jobgroup: jobexample
  spec:
    template:
      metadata:
        name: jobexample
        labels:
          jobgroup: jobexample
      spec:
        containers:
        - name: c
          image: busybox
          command: ["sh", "-c", "echo Processing item $ITEM && sleep 5"]
        restartPolicy: Never
  ```

- $ITEM是一个“变量”，外部控制器创建Pod的时候替换掉即可。

- 这个模板所有的Job，都有jobgroup=jobexample

- 使用shell就能创建一批YAML文件：

  ```shell
  for i in apple banana cherry
  do
  	cat job-tmpl.yaml | sed "s/\$ITEM/$i/" > ./jobs/job-$i.yaml
  done	
  ```



### 拥有固定任务数目的并行Job

- 这种模式下，只关心最后是否有spec.completions个任务成功退出。并不关心并行度。



### 不指定完成数，只指定并行度

- 有的业务需求下，任务的总数是未知的，这种情况下Pod必须能够知道，自己什么时候可以退出。

# CronJob

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

- spec.jobTemplate表示了CronJob是一个Job对象的控制器

- CronJob创建和删除Job的依据是schedule字段的Cron表达式

  - *表示从0开始
  - /表示“每”
  - 1表示偏移量

- apply这个对象后能看到：

   ![](/get-cronjob.png)

- 有一些特殊的Job，当它还没执行完时，下一次调度就来了，就可以通过`spec.concurrencyPolicy`来处理：

  - concurrentPolicy=Allow：允许Job同时存在
  - concurrentPolicy=Forbid：不允许Job同时存在，该周期被跳过
  - concurrentPolicy=Replace：意味这新Job会替换旧的Job

- 而某一次Job创建失败了，这次创建就会标记成“miss”，在指定的时间窗口内，miss的次数达到100，CronJob就会停止再创建这个Job

- 这个时间窗口可以由`spec.startingDeadlineSeconds`字段指定，单位为秒。
