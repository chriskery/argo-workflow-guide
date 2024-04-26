# argo-workflow-guide

[Argo Workflows](https://argoproj.github.io/argo-workflows/) 是一个[云原生](https://en.wikipedia.org/wiki/Cloud-native_computing)的通用的工作流引擎。本教程主要介绍如何用其完成持续集成（Continous Integration, CI）任务。

## 基本概念

对任何工具的基本概念有一致的认识和理解，是我们学习以及与他人交流的基础。
以下是本文涉及到的概念：

- WorkflowTemplate，工作流模板
- Workflow，工作流

为方便读者理解，下面就几个同类工具做对比：

| Argo Workflow    | Jenkins  |
| ---------------- | -------- |
| WorkflowTemplate | Pipeline |
| Workflow         | Build    |

## 安装

通过如下命令安装 argo：

```shell
kubectl create namespace argo
kubectl apply -n argo -f https://github.com/argoproj/argo-workflows/releases/latest/download/install.yaml
```

如果你的环境访问 GitHub 时有网络问题，可以使用下面的命令来安装：

```shell
docker run -it --rm -v $HOME/.kube/:/root/.kube --network host --pull always ghcr.io/linuxsuren/argo-workflows-guide:master
```

Argo 组件：
上面的安装命令会安装两个 Deploy，分别对应 Argo 的两个组件：

- Workflow Controller：负责运行 Workflows
- Argo Server：提供 UI 和 API

## 安装 Argo CLI
访问如下链接查看 Argo Cli 安装方法详情：[https://github.com/argoproj/argo-workflows/releases/](https://github.com/argoproj/argo-workflows/releases/)
  For Mac：

```shell
# Download the binary
curl -sLO https://github.com/argoproj/argo-workflows/releases/download/v3.5.6/argo-darwin-amd64.gz

# Unzip
gunzip argo-darwin-amd64.gz

# Make binary executable
chmod +x argo-darwin-amd64

# Move binary to path
mv ./argo-darwin-amd64 /usr/local/bin/argo

# Test installation
argo version
```

## 设置访问方式
默认情况下argo-server采用客户端认证，这要求客户端使用**Kubernetes bearer token 进行身份认证。**对于学习、体验等场景，我们可以通过下面的命令直接设置绕过登录：
```shell
kubectl patch deployment \
  argo-server \
  --namespace argo \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/args", "value": [
  "server",
  "--auth-mode=server"
]}]'
````

我们可以用下面的方式或者其他方式来设置 Argo Workflows 的访问端口：

```shell
kubectl -n argo port-forward deploy/argo-server --address 0.0.0.0 2746:2746
# 或者设置为 NodePort
kubectl -n argo patch svc argo-server --type='json' -p '[{"op":"replace", "path":"/spec/type", "value":"NodePort"}, {"op":"add", "path":"/spec/ports/0/nodePort","value":31517}]'
```

需要注意的是，这里默认的配置下，服务器设置了自签名的证书提供 HTTPS 服务，因此，确保你使用 https:// 协议进行访问。
例如，地址为：https://10.121.218.242:2746/

## Workflow 概念

#### Workflow 被定义为 Kubernetes 里面的资源。每一个 Workflow 由一个或多个模版组成（称为 Template），这些模版中的某一个称为 Entrypoint（即 Workflow 的运行起点）。

```shell
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  name: hello
spec:
  serviceAccountName: argo # this is the service account that the workflow will run with
  entrypoint: main # the first template to run in the workflows
  templates:
  - name: main
    container: # this is a container template
      image: docker/whalesay # this image prints "hello world" to the console
      command: ["cowsay"]
```

在上面的示例中，我们定义了一个 Workflow，他有一个模版，叫作 main，entrypoint 为 main 模版。

## Template

### 基本概念

Template 可分为两种大类：work 和**orchestration。**

- **work 定义了一系列需要完成的工作，work 类型包括：**
  - Container
  - Container Set
  - Data
  - Resource
  - Script
- **orchestrates：对 work 进行编排，包括：**
  - DAG
  - Steps
  - Suspend

**比如我们定义了三个 work-work1、work2、work3，我们想要串行执行三个 work，则需要 DAG 对 work 进行编排。**

### Work 类型

- **Container：不同的 Container 运行在不同的 Pod 中**
- **ContainerSet：在同一个 Pod 中运行多个 Container**
- **Data：允许你从某个存储（比如 S3）中获取数据。**
- Resource： 允许你创建一个 K8S 资源并等待某个条件满足，一般用于需要跟其他的 Kubernetes 系统进行协作的场景。
- **script：其实也是 Container，不过启动命令使用脚本的方式提供。**

  ### Orchestrates

  ##### DAG：

  ```shell
  apiVersion: argoproj.io/v1alpha1
  kind: Workflow
  metadata:
  generateName: dag-
  spec:
  entrypoint: main
  templates:
    - name: main
      dag:
        tasks:
          - name: a
            template: whalesay
          - name: b
            template: whalesay
            dependencies:
              - a
    - name: whalesay
      container:
        image: docker/whalesay
        command: [ cowsay ]
        args: [ "hello world" ]
  ```

  这里定义了两个 templates，一个是 main、一个是 whalesay，main 作为 entrypoint。main 里面包含两个 tasks，a 和 b，都使用 whalesay template，但 b 依赖 a，也就是说，在 a 成功运行完成之前，b 不会运行。运行完成后查看执行结果：

  ```shell
  argo -n argo get @latest
  STEP          TEMPLATE  PODNAME              DURATION  MESSAGE
  ✔ dag-shxn5  main
  ├─✔ a        whalesay       dag-shxn5-289972251  6s
  └─✔ b        whalesay       dag-shxn5-306749870
  ```

DAG Loops
Argo Flow 支持运行大量的并行的作业，DAG 定义了一个`withItems`属性，可以用来进行任务的循环。

```shell
  dag:
        tasks:
          - name: print-message
            template: whalesay
            arguments:
              parameters:
                - name: message
                  value: "{{item}}"
            withItems:
              - "hello world"
              - "goodbye world"
```

在这个例子中，print-message 将会运行 2 次。{{item}}市 Template Tags 的写法，在运行时会被解析成 "hello world" 和 "goodbye world"。DAG 默认情况会并行执行 Item，因此两个任务会同时被启动。
DAG Loop 提供了另外一个叫做`withSequence`，可以直接用来定义任务数量，而无需一行行写 items：

```shell
 dag:
        tasks:
          - name: print-message
            template: whalesay
            arguments:
              parameters:
                - name: message
                  value: "{{item}}"
            withSequence:
              count: 5
等价于：
            withItems:
              - "0"
              - "1"
              - "2"
              - "3"
              - "4"
```

**Template Tags（模版变量）**
**Template Tags 其实就是在 Template 中定义了一系列的变量，这些变量的值在运行时进行传入。（类似 Helm Template 变量，渲染时通过 values.yaml 进行传值）**
**不同的 Work 类型可以定义不同的 Template Tags，但也有一些变量是跟类型无关的，如下所示：**

```shell
    - name: main
      container:
        image: docker/whalesay
        command: [ cowsay ]
        args: [ "hello {{workflow.name}}" ]
```

{{workflow.name}} 引用了当前 Workflow 的 name。

##### Step：

Exit Handler：

```shell
      dag:
        tasks:
          - name: a
            template: whalesay
            onExit: tidy-up
```

上面的意思是 a 运行完成以后执行 task tidy-up，如果我们想要在某些任务结束后执行另外一个 task，我们可以使用 Exit Handler。

## Workflow Template

**Workflow templates** (区别于 template) 允许我们定义可重用的 Template 模版：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: hello
spec:
  entrypoint: main
  templates:
    - name: main
      container:
        image: docker/whalesay
        command: [cowsay]
```

## 简单示例

下面是一个非常简单的 Argo Workflow 示例：

1. 通过 Argo cli 提交：

```go
argo submit -n argo --watch https://raw.githubusercontent.com/argoproj/argo-workflows/main/examples/hello-world.yaml
```

--watch flag 会监听 workflow 的运行状态，监测 workflow 是否成功运行。当 workflow 运行结束后，针对 workflow 的监听会停止。

```shell
Name:                hello-world-vkg5j
Namespace:           argo
ServiceAccount:      unset (will run with the default ServiceAccount)
Status:              Pending
Created:             Wed Apr 24 20:19:19 +0800 (now)
Progress:
Name:                hello-world-vkg5j
Namespace:           argo
ServiceAccount:      unset (will run with the default ServiceAccount)
Status:              Running
Created:             Wed Apr 24 20:19:19 +0800 (now)
Started:             Wed Apr 24 20:19:19 +0800 (now)
Duration:            0 seconds
Progress:            0/1
```

argo cli 相关命令：

```shell
1.argo list -n argo //列出workflow list
。@latest is an alias for the latest workflow:
2.argo get -n argo @latest/--name xx //查看workflow详情
3.argo logs -n argo @latest/--name xx //查看workflow日志
```

2. 通过 kubectl 提交：

```shell
cat <<EOF | kubectl apply -n default -f -
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
generateName: hello-world-
labels:
 workflows.argoproj.io/archive-strategy: "false"
annotations:
 workflows.argoproj.io/description: |
   This is a simple hello world example.
spec:
entrypoint: whalesay
templates:
- name: whalesay
 container:
   image: docker/whalesay:latest
   command: [cowsay]
   args: ["hello world"]
EOF
```

执行成功后，就可以在下面的地址访问到刚刚创建的工作流模板：
https://10.121.218.242:2746/workflow-templates/default
选择其中一个模板，点击 SUBMIT 按钮，并设置对应的参数后即可触发工作流的执行。
在 Workflows 的详情页面中，我们做如下的操作：

- RESUBMIT，使用相同的模板以及参数触发一次新的执行

  ## Input 和 Outputs

  输入输出的一种类型叫做**parameter**

  ### 输入：作为输入的 parameter 大多时候为常见的字符串。

  ```shell
    - name: main
      inputs:
        parameters:
          - name: message
            value: "hello"
      container:
        image: docker/whalesay
        command: [ cowsay ]
        args: [ "{{inputs.parameters.message}}" ]
  ```

  我们通过"{{inputs.parameters.message}}"引用了 parameters 中定义的 message 的值"hello"。
  除此之外我们也可以在使用 argo 命令提交 workflow 的时候修改 parameters 的值：
  :::info
  argo submit --watch input-parameters-workflow.yaml -p message='Welcome to Argo!'
  :::

  ### 输出：作为输出的 parameters 大多时候引用源为文件。

  ```shell
  - name: whalesay
    container:
      image: docker/whalesay
      command: [sh, -c]
      args: ["echo -n hello world > /tmp/hello_world.txt"]
    outputs:
      parameters:
      - name: hello-param
        valueFrom:
          path: /tmp/hello_world.txt
  ```

  这里定义了一个输出参数 hello-param，他的内容是 container 结束时候的/tmp/hello_world.txt 的内容，即"hello world"。
  在 DAG 或者 Step 中，我们可以使用**template tag 将**某个任务的输出作为其他任务的输入 paramter：

  ```shell
  dag:
        tasks:
          - name: generate-parameter
            template: whalesay
          - name: consume-parameter
            template: print-message
            dependencies:
              - generate-parameter
            arguments:
              parameters:
                - name: message
                  value: "{{tasks.generate-parameter.outputs.parameters.hello-param}}"
  ```

## CronWorkflow

CronWorkflow 使用 cron 进行调度：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: CronWorkflow
metadata:
  name: hello-cron
spec:
  schedule: "* * * * *"
  workflowSpec:
    entrypoint: main
    templates:
      - name: main
        container:
          image: docker/whalesay
```

### Artifact

Argo 将存储在对象存储(S3, MinIO, GCS, etc.).里面的文件叫做一个 Artifact。我们可以引用某个 Artifact 作为输入，也可以指定输出到 Artifact 中。Argo 称存储 Artifact 的对象存储为**artifact repository。**

- An **input artifact** is a file downloaded from storage (i.e. S3) and mounted as a volume within the container.
- An **output artifact** is a file created in the container that is uploaded to storage.

输出 Artifact：

```yaml
- name: save-message
  container:
    image: docker/whalesay
    command: [sh, -c]
    args: ["cowsay hello world > /tmp/hello_world.txt"]
  outputs:
    artifacts:
      - name: hello-art
        path: /tmp/hello_world.txt
```

当容器结束的时候，会存储容器内的/tmp/hello_world.txt 到**artifact repository 中，文件名为 hello-art。Artifact 支持存储目录。**

## Webhook

所有主流 Git 仓库都是支持 webhook 的，借助 webhook 可以当代码发生变化后实时地触发工作流的执行。

1. It only allows you to create workflows from a WorkflowTemplate , so is more secure.
2. It allows you to parse the HTTP payload and use it as parameters.
3. It allows you to integrate with other systems without you having to change those systems.
4. Webhooks also support GitHub and GitLab, so you can trigger workflow from git actions.

Argo Workflows 利用 WorkflowEventBinding 将收到的 webhook 请求与 WorkflowTemplate 做关联。请参考下面的例子：

```shell
cat <<EOF kubectl apply -n default -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: submit-workflow-template
rules:
  - apiGroups:
      - argoproj.io
    resources:
      - workfloweventbindings
    verbs:
      - list
  - apiGroups:
      - argoproj.io
    resources:
      - workflowtemplates
    verbs:
      - get
  - apiGroups:
      - argoproj.io
    resources:
      - workflows
    verbs:
      - create
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: github.com
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: github.com
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: submit-workflow-template
subjects:
  - kind: ServiceAccount
    name: github.com
    namespace: default
---
apiVersion: v1
stringData:
  github.com: |                              # 这里对应 ServiceAcccount 名称
    type: github                          # 固定的几个类型
    secret: "argo-workflow-secret"    # webhook 中配置的 Secret Token
kind: Secret
metadata:
  name: argo-workflows-webhook-clients
type: Opaque
---
apiVersion: argoproj.io/v1alpha1
kind: WorkflowEventBinding
metadata:
  name: pull-request-binding
spec:
  event:
    # 通过 webhook 的 payload 对请求进行过滤，并联动触发对应的工作流模板
    selector: payload.project.name == "gogit" && payload.object_attributes.state == "opened"
  submit:
    workflowTemplateRef:
      name: gogit       # 关联工作流模板
    arguments:
      parameters:
      - name: branch
        valueFrom:
          # 从 webhook 的 payload 中提取值作为参数
          event: payload.object_attributes.source_branch
EOF
```

然后，在代码仓库中添加如下的 webhook 地址（其中，default 是 WorkflowEventBinding 所在的命名空间）：

```
https://argo-workflow-ip:port/api/v1/events/default/
```

上面的 Secret 名称 argo-workflows-webhook-clients 是固定的，所在命名空间也就是 webhook 地址中的 default。支持的 Git Provider 名称也是固定的几个：

- bitbucket
- bitbucketserver
- github
- gitlab

  ## Argo Event

  [Argo Event](https://github.com/argoproj/argo-events) 是另外一种使得代码更新后自动触发流水线的方式。你需要单独[安装](https://argoproj.github.io/argo-events/quick_start/)。

- Events Controller-Manager 用作 Events 控制器，为一个 Deployment
- EventSource 接受消息（来自 webhook 或其他）

  - 每个 EventSource 资源对应一个无状态服务（Deployment）

- EventBus 为消息总线

  - 为一个有状态服务（Statefulsets）

- Sensor 用于获取消息并触发动作（流水线）

  - 每个 Sensor 资源对应一个无状态服务（Deployment）

以下是默认的消息总线，无需做如何配置，创建后会自动创建一个有状态服务：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: EventBus
metadata:
  name: default
  namespace: default
spec:
  nats:
    native:
      # Optional, defaults to 3. If it is < 3, set it to 3, that is the minimal requirement.
      replicas: 3
      # Optional, authen strategy, "none" or "token", defaults to "none"
      auth: none
```

下面的资源会自动创建 EventSource 的 Deployment 和 Service（可以手动修改服务为 NodePort）：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: EventSource
metadata:
  name: default
  namespace: default
spec:
  service:
    ports:
      - port: 12000
        targetPort: 12000
  gitlab:
    pr:
      # Project namespace paths or IDs
      projects:
        - "cmp/al-cloud"
      webhook:
        endpoint: /push # 固定值
        port: "12000" # 固定值
        method: POST
        url: http://172.11.0.6:32168 # Sensor 的访问地址
      accessToken:
        key: gitlab-token
        name: gitlab-secret
      events:
        - PushEvents
        - MergeRequestsEvents
        - TagPushEvents
        - NoteEvents
      secretToken:
        key: webhook-secret
        name: gitlab-secret
      enableSSLVerification: false
      gitlabBaseURL: http://10.121.218.82:6080
      deleteHookOnFinish: true
```

下面的资源会自动创建 Sensor 的 Deployment：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata:
  name: default
  namespace: default
spec:
  dependencies:
    - name: pr
      eventSourceName: default
      eventName: pr
      filters:
        data:
          - path: body.object_attributes.state
            type: string
            value:
              - opened
  triggers:
    - template:
        name: argo-workflow-trigger
        argoWorkflow:
          operation: submit
          source:
            resource:
              apiVersion: argoproj.io/v1alpha1
              kind: Workflow
              metadata:
                generateName: default-
              spec:
                arguments:
                  parameters:
                    - name: branch
                    - name: pr
                workflowTemplateRef:
                  name: default
          parameters:
            - src:
                dependencyName: pr
                dataKey: body.object_attributes.source_branch
              dest: spec.arguments.parameters.0.value
            - src:
                dependencyName: pr
                dataKey: body.object_attributes.iid
              dest: spec.arguments.parameters.1.value
```

下面是 Gitlab 创建 release 分支或推送到 master 分支时的写法：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata:
  name: al-cloud-release
  namespace: default
spec:
  dependencies:
    - name: push
      eventSourceName: default
      eventName: al-cloud-push
      filters:
        exprLogicalOperator: "or"
        data:
          - path: body.object_kind
            type: string
            value:
              - push
          - path: body.before
            type: string
            value:
              - "0000000000000000000000000000000000000000"
        exprs:
          - expr: ref =~ "refs/heads/release-"
            fields:
              - name: ref
                path: body.ref
          - expr: ref == "refs/heads/master"
            fields:
              - name: ref
                path: body.ref

  triggers:
    - template:
        name: trigger
        argoWorkflow:
          operation: submit
          source:
            resource:
              apiVersion: argoproj.io/v1alpha1
              kind: Workflow
              metadata:
                generateName: al-cloud-push-
              spec:
                arguments:
                  parameters:
                    - name: branch
                workflowTemplateRef:
                  name: pr-al-cloud
          parameters:
            - src:
                dependencyName: push
                dataKey: body.ref
              dest: spec.arguments.parameters.0.value
```

## 引用已有模板中的任务

Argo Workflows 允许以三种方式引用已有的任务：

- 当前工作流
- 当前命名空间中的工作流模板
- 全局（Cluster 级别）的工作流模板

这相当于 Java、Golang 等编程语言中的引用方式，分别可以引用：当前源文件、当前包、其他包下的函数。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: hello-world
spec:
  templates:
    - name: hd
      dag:
        tasks:
          - name: hd
            templateRef: # 表示引用其他模板中的任务
              name: hook # 模板名称
              template: hook # 模板中的任务名称
              clusterScope: true # 为 true 是从全局（Cluster）中查找模板，为 false 时从当前命名空间中查找
            arguments:
              parameters: # 给所引用的任务传递参数
                - name: pr
                  value: 1
```

### 小结

我们可以将公用的模板作为模板库，供工作流调用，这样就可以使得工作流变得简单。

## 认证模型

Argo workflows 支持三种认证模型：

- server
  - 采用服务端的 ServiceAccount，UI 节目无需登录认证，可作为体验、测试等场景使用
- client
  - 客户端需要提供 Token 等认证信息
  - 从 v3.0+ 开始作为 Argo workflows 的默认认证方式
- sso
  - 后端有对应的 ServiceAccount 选择机制，包括有：优先级、表达式等匹配不同的用户、用户组权限

其中，sso 和 client 可以组合使用，分别为：UI、webhook、SDK Client 等提供认证。

## SSO

为了保证 Argo workflows 同时支持 [SSO(Single Sign-On)](https://argoproj.github.io/argo-workflows/argo-server-sso/)以及 webhook 的执行，需要设置认证模式为：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argo-server
  namespace: argo
spec:
  template:
    spec:
      containers:
        - args:
            - server
            - --auth-mode=sso # UI 登录后所有操作使用的权限，参考后面的配置
            - --auth-mode=client # webhook 触发时采用的权限模式
          name: argo-server
```

下面以 [Dex](https://github.com/devops-ws/dex-guide) 为例（需要有：read_user、openid 的授权），给出配置 SSO 信息：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: workflow-controller-configmap
  namespace: argo
data:
  sso: |
    issuer: https://10.121.218.184:31392/api/dex                    # Dex 服务地址
    clientId:
      name: argo-workflows-sso
      key: client-id
    clientSecret:
      name: argo-workflows-sso
      key: client-secret
    redirectUrl: https://10.121.218.184:30298/oauth2/callback       # 这里 Argo workflows 的地址必须是浏览器可访问的
    insecureSkipVerify: true
    scopes:
    - groups                        # 用组作为权限划分
    - email
    rbac:
      enabled: true                 # 启用 RBAC 权限认证，下面需要提供对应的配置
```

创建上面所需要的 Secret：

```shell
cat <<EOF | kubectl apply -f argo -f
apiVersion: v1
data:
  # 下面的 client-id、client-secret 可以向 oauth 服务提供者拿到
  client-id: YXJnby13b3JrZmxvd3Mtc3Nv
  client-secret: cmljaw==
kind: Secret
metadata:
  name: argo-workflows-sso
type: Opaque
EOF
```

为 SSO 登录的用户提供只读权限：

```shell
cat <<EOF | kubectl apply -n argo -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: user-default-login
  annotations:
    workflows.argoproj.io/rbac-rule: "'dev' in groups"        # dev 用户组登录后会使用该账号
    workflows.argoproj.io/rbac-rule-precedence: "10"          # 多条规则匹配的情况下，选择数字大的
EOF

cat <<EOF | kubectl apply -n argo -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    app.kubernetes.io/instance: argo-workflow
  name: argo-view-default-login-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: argo-aggregate-to-view                          # 内置的只读角色
subjects:
- kind: ServiceAccount
  name: user-default-login
  namespace: argo
EOF
```

### 小结

可以给 Argo workflows 配置任意兼容 OAuth 2 的提供商，例如：Dex、GitHub、Gitlab 公有云、Gitlab 社区版、Argo CD 等。
内置的角色包括（以下都是 ClusterRole）：

- argo-aggregate-to-view
- argo-aggregate-to-edit
- argo-aggregate-to-admin
- argo-cluster-role，没有 workfloweventbindings 的权限
- argo-server-cluster-role，包含所有需要的权限

  ## 插件机制

  Argo Workflows 内置了[几种类型的任务模板](#%E4%BB%BB%E5%8A%A1%E6%A8%A1%E6%9D%BF%E7%B1%BB%E5%9E%8B)，这些任务类型或是方便解决特定问题，或是可以解决通用问题。此外，我们还可以通过[执行器（Executor）插件](https://argoproj.github.io/argo-workflows/plugins/)扩展 Argo Workflows 的功能。
  执行器插件，会作为工作流 Pod 中 sidecar 的形式存在，通过 HTTP 提供服务。Argo Workflows 规定了 URI，以及 Request 和 Response。据此，我们可以看出来插件的几个特点：

- 插件可以用任何编程语言实现
- 执行插件任务时无需启动新的 Pod，减少了对 Pod 的消耗

该插件功能默认是未启用的，我们可以在控制器（Controller）中添加环境变量的方式启用插件功能。请参考如下配置：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workflow-controller
spec:
  template:
    spec:
      containers:
        - name: workflow-controller
          env:
            - name: ARGO_EXECUTOR_PLUGINS
              value: "true"
```

安装插件时，只需要添加一个 ConfigMap 即可。例如：

```yaml
apiVersion: v1
data:
  sidecar.automountServiceAccountToken: "false"
  sidecar.container: |
    args:
    - --provider
    - gitlab
    image: ghcr.io/linuxsuren/workflow-executor-gogit:master
    command:
    - workflow-executor-gogit
    name: gogit-executor-plugin
    ports:
    - containerPort: 3001
    resources:
      limits:
        cpu: 500m
        memory: 128Mi
      requests:
        cpu: 250m
        memory: 64Mi
    securityContext:
      allowPrivilegeEscalation: true
      runAsNonRoot: true
      runAsUser: 65534
kind: ConfigMap
metadata:
  labels:
    workflows.argoproj.io/configmap-type: ExecutorPlugin
  name: gogit-executor-plugin
  namespace: argo
```

我们可以把上面的 ConfigMap 添加到 Argo Workflows 控制器所在的命名空间中，也可以添加到执行工作流所在的命名空间中。另外，当存在多个同名的插件时，会以工作流所在命名空间的插件为主。
插件安装成功的话，你可以在控制器中查看到类似如下的日志输出：

```
level=info msg="Executor plugin added" name=gogit-executor-plugin
```

插件的使用方法如下：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: plugin
  namespace: default
spec:
  entrypoint: main
  hooks:
    exit:
      template: status
    all:
      template: status
      expression: "true"
  templates:
    - container:
        args:
          - search
          - kubectl
        command:
          - hd
        image: ghcr.io/linuxsuren/hd:v0.0.70
      name: main
    - name: status
      plugin:
        gogit-executor-plugin: # 下面支持任何格式给插件传递参数
          owner: linuxsuren
          repo: test
          pr: "3"
```

这里有[更多社区维护的插件](https://argoproj.github.io/argo-workflows/plugin-directory/)，有通过 Python、Golang、Rust 等语言实现的。
如果你想了解如何开发一个插件，可以继续往后阅读。下面介绍插件机制对 HTTP 的请求、响应的规定：

- Request payload 中可以解析到与当前工作流的信息，包括：名称、命名空间、插件参数
  - 我们可以参考 [ExecuteTemplateArgs](https://github.com/argoproj/argo-workflows/blob/774bf47ee678ef31d27669f7d309dee1dd84340c/pkg/plugins/executor/template_executor_plugin.go#L19) 来解析请求
- Response 需要告知任务执行的状态
  - 我们可以参考 [ExecuteTemplateReply](https://github.com/argoproj/argo-workflows/blob/774bf47ee678ef31d27669f7d309dee1dd84340c/pkg/plugins/executor/template_executor_plugin.go#L32) 作为 HTTP 响应的数据

## 归档

Argo Workflow 支持将工作流执行记录（Workflow）的信息存储到 PostgreSQL 或 MySQL 中，以达到更长久地保存执行记录但又不会影响到 Kubernetes 集群的性能。

这里，给出一个归档（ [Archive](https://argoproj.github.io/argo-workflows/workflow-archive/) ）数据到 PostgreSQL 的配置方法：

首先，安装 [PostgreSQL](https://www.postgresql.org/) 。这里采用 Helm Chart 的方式来安装：

```shell
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
cat > values.yaml <<EOF
auth:
enablePostgresUser: true
postgresPassword: "StrongPassword"
username: "root"
password: "root"
database: "app_db"
EOF
helm install postgresql-dev -f values.yaml bitnami/postgresql
```

Argo Workflows 会以 Secret 的方式读取数据库的用户名、密码，下面是创建 Secret 的命令：

```shell
kubectl create secret generic --from-literal=username=root --from-literal=password=root argo-postgres-config -n argo
```

然后，参考下面的 ConfigMap 启用工作流的归档功能：

```yaml
apiVersion: v1
data:
persistence: |
archive: true
postgresql:
  host: postgresql-dev.argocd.svc
  port: 5432
  database: app_db
  tableName: argo_workflows
  userNameSecret:
    name: argo-postgres-config
    key: username
  passwordSecret:
    name: argo-postgres-config
    key: password
kind: ConfigMap
metadata:
name: workflow-controller-configmap
namespace: argo
```

上面的配置步骤都完成，执行工作流后，我们可以在 UI 界面左侧菜单上看到归档的执行记录。也可以通过数据库命令行客户端连接数据库，查看数据的表记录信息：

```shell
export POSTGRES_PASSWORD=root
kubectl run postgresql-dev-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:14.1.0-debian-10-r80 --env="PGPASSWORD=$POSTGRES_PASSWORD" --command -- psql --host postgresql-dev.argocd.svc -U root -d app_db -p 5432
```

下面是一些 PostgreSQL 命令行客户端的参考：

```shell
\dt                                                   # 查看当前数据库中的表
select name,phase from argo_archived_workflows;       # 查看已归档的工作流执行记录
```

你会看到类似如下的输出：

```sql
app_db=> \dt
               List of relations
Schema |              Name              | Type  | Owner
--------+--------------------------------+-------+-------
public | argo_archived_workflows        | table | root
public | argo_archived_workflows_labels | table | root
public | argo_workflows                 | table | root
public | schema_history                 | table | root
(4 rows)
```

app_db=> select name,phase from argo_archived_workflows;
name | phase
--------------+-----------
plugin-pl6rx | Succeeded
plugin-8gs7c | Succeeded

````
## GC
Argo Workflows 有个工作流执行记录（Workflow）的清理机制，也就是 Garbage Collect(GC)。GC 机制可以避免有太多的执行记录， 防止 Kubernetes 的后端存储 Etcd 过载。
我们可以在 ConfigMap 中配置期望保留的工作执行记录数量，这里支持为不同状态的执行记录设定不同的保留数量。配置方法如下：
```yaml
apiVersion: v1
data:
  retentionPolicy: |
    completed: 3
    failed: 3
    errored: 3
kind: ConfigMap
metadata:
  name: workflow-controller-configmap
  namespace: argo
````

需要注意的是，这里的清理机制会将多余的 Workflow 资源从 Kubernetes 中删除。如果希望能更多历史记录的话，建议启用并配置好归档功能。
除了工作流有回收清理机制外，也可以针对 Pod 设置回收机制，参考配置如下：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: hello-world # Name of this Workflow
  namespace: default
spec:
  podGC:
    strategy: OnPodCompletion
```

清理策略的可选值包括：

- OnPodCompletion
- OnPodSuccess
- OnWorkflowCompletion
- OnWorkflowSuccess

建议 PodGC 与日志持久化配合使用，不然可能会由于 Pod 被删除后无法查看工作流日志。

## 可观测

Argo Workflows 支持通过 Prometheus 采集监控指标，包括：[预定义、自定义](https://argoproj.github.io/argo-workflows/metrics/)的指标，下面是添加自定义指标的示例：

```yaml
spec:
  metrics:
    prometheus:
      - name: exec_duration_gauge
        labels:
          - key: name
            value: "{{workflow.name}}" # 工作流名称
          - key: templatename
            value: "{{workflow.labels.workflows.argoproj.io/workflow-template}}" # 工作流模板名称
          - key: namespace
            value: "{{workflow.namespace}}" # 工作流所在命名空间
        help: Duration gauge by name
        gauge:
          value: "{{workflow.duration}}" # 工作流执行时长
      - counter:
          value: "1"
        help: "Total count of all the failed workflows"
        labels:
          - key: name
            value: "{{workflow.name}}"
          - key: namespace
            value: "{{workflow.namespace}}"
          - key: templatename
            value: "{{workflow.labels.workflows.argoproj.io/workflow-template}}"
        name: failed_count
        when: "{{workflow.status}} == Failed"
      - counter:
          value: "1"
        help: "Total count of all the successed workflows"
        labels:
          - key: name
            value: "{{workflow.name}}"
          - key: namespace
            value: "{{workflow.namespace}}"
          - key: templatename
            value: "{{workflow.labels.workflows.argoproj.io/workflow-template}}"
        name: successed_count
        when: "{{workflow.status}} == Succeeded"
      - counter:
          value: "1"
        help: "Total count of all the workflows"
        labels:
          - key: name
            value: "{{workflow.name}}"
          - key: namespace
            value: "{{workflow.namespace}}"
          - key: templatename
            value: "{{workflow.labels.workflows.argoproj.io/workflow-template}}"
        name: total_count
```

上面包含了工作流的成功、失败、总量的数据指标。

## 工作流默认配置

在实际场景下，我们往往需要配置不少的工作流模板，而这些模板中也通常会有一些通用的配置项，例如： 拉取私有镜像的凭据、Pod 回收策略、卷挂载等待。我们可以把这些公共配置加到 ConfigMap 中，请参考如下：

```yaml
apiVersion: v1
data:
  workflowDefaults: |
    spec:
      podGC:
        strategy: OnPodCompletion           # Pod 完成后即删除
      imagePullSecrets:
      - name: harbor-pull                   # 公共的私有镜像拉取凭据
      volumeClaimTemplates:                 # 默认的代码拉取卷位置
        - metadata:
            name: work
          spec:
            accessModes: ["ReadWriteOnce"]
            resources:
              requests:
                storage: 64Mi
kind: ConfigMap
metadata:
  name: workflow-controller-configmap
  namespace: argo
```

## Golang SDK

Argo Workflows 官方[维护了 Golang、Java、Python 语言](https://argoproj.github.io/argo-workflows/client-libraries/)的 SDK。下面以 Golang 为例，讲解 SDK 的使用方法。
在运行下面的示例前，有两点需要注意的：

- Argo Workflows Server 地址
- Token

你可以选择直接使用 argo-server 的 Service 地址，将端口 2746 转发到本地，或将 Service 修改为 NodePort，或者其他方法暴露端口。也可以执行下面的命令，再启动一个 Argo 服务：

```shell
argo server
```

第二个，就是用户认证的问题了。如果你对 Kubernetes 认证系统非常熟悉的话，可以跳过这一段，直接找一个 Token。为了让你对 Argo 的用户认证更加了解，我们为下面的测试代码创建一个新的 ServiceAccount。
我们需要分别创建：

- Role，规定可以对哪些资源有哪些操作权限

  ```shell
  kubectl create role demo --verb=get,list,update,create --resource=workflows.argoproj.io --resource=workflowtemplates.argoproj.io -n default
  ```

- ServiceAccount，代表一个用户

  ```shell
  kubectl create serviceaccount demo -n default
  ```

- RoleBinding，将用户和角色（Role）进行绑定

  ```shell
  kubectl create rolebinding demo --role=demo --serviceaccount=default:demo -n default
  ```

- Secret，关联一个 ServiceAccount，并自动生成 Token

  ```shell
  kubectl apply -n default -f - <<EOF
  apiVersion: v1
  kind: Secret
  metadata:
  name: demo.service-account-token
  annotations:
    kubernetes.io/service-account.name: demo
  type: kubernetes.io/service-account-token
  EOF
  ```

  上面的例子中，我们使用的是 Role 和 RoleBinding ，这样的角色只能允许访问所在命名空间（namespace）的资源。上面创建的用户，只能够访问 default 这命名空间下的 Workflow 和 WorkflowTemplate 。 如果想要创建一个全局的角色以及绑定，可以使用 ClusterRole 和 ClusterRoleBinding 。
  上面的用户创建完成后，我们就可以通过下面的命令拿到指定权限的 Token 了：

  ```shell
  kubectl get secret -n default demo.service-account-token -ojsonpath={.data.token}|base64 -d
  ```

  接下来，创建一个 Golang 工程，并将下面的示例代码拷贝到源文件 main.go 中。

  ```shell
  mkdir demo
  cd demo
  go mod init github.com/linuxsuren/demo
  go get github.com/argoproj/argo-workflows/v3@v3.4.4
  go mod tidy
  ```

  示例代码：

  ```
  package main
  ```

import (
"fmt"
"github.com/argoproj/argo-workflows/v3/pkg/apiclient"
"github.com/argoproj/argo-workflows/v3/pkg/apiclient/workflow"
"github.com/argoproj/argo-workflows/v3/pkg/apiclient/workflowtemplate"
"github.com/argoproj/argo-workflows/v3/pkg/apis/workflow/v1alpha1"
metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

/\*_
** Before run this demo, please create a parameterless WorkflowTemplate in namespace default.
** In this demo, we will print all the WorkflowTemplates in namespace default.
\*\* Then run a Workflow base on the first WorkflowTemplate.
_/

func main() {
opt := apiclient.Opts{
ArgoServerOpts: apiclient.ArgoServerOpts{
URL: "localhost:31808", // argo-server address
Path: "/",
Secure: true,
InsecureSkipVerify: true,
},
AuthSupplier: func() string {
return "Bearer your-token"
},
}
ctx, client, err := apiclient.NewClientFromOpts(opt) // the context will carry on auth
if err != nil {
panic(err)
}

```
wftClient, err := client.NewWorkflowTemplateServiceClient()
if err != nil {
    fmt.Println("failed to get the WorkflowTemplates client", err)
    return
}
defaultNamespace := "default"

fmt.Println("get the WorkflowTemplate list from", defaultNamespace)
wftList, err := wftClient.ListWorkflowTemplates(ctx, &workflowtemplate.WorkflowTemplateListRequest{
    Namespace: defaultNamespace,
})
if err != nil {
    fmt.Println("failed to list WorkflowTemplates", err)
    return
}
for _, wft := range wftList.Items {
    fmt.Println(wft.Namespace, wft.Name)
}

if wftList.Items.Len() > 0 {
    wft := wftList.Items[0]

    wfClient := client.NewWorkflowServiceClient()
    _, err := wfClient.CreateWorkflow(ctx, &workflow.WorkflowCreateRequest{
        Namespace: defaultNamespace,
        Workflow: &v1alpha1.Workflow{
            ObjectMeta: metav1.ObjectMeta{
                GenerateName: wft.Name,
            },
            Spec: v1alpha1.WorkflowSpec{
                WorkflowTemplateRef: &v1alpha1.WorkflowTemplateRef{
                    Name: wft.Name,
                },
            },
        },
    })
    if err != nil {
        fmt.Println("failed to create workflow", err)
    }
}
```

}

```
最后，执行命令：go run .
把上面的示例代码编译后，二进制文件大致在 60M+
## References

- [DevOps Practice Guide](https://github.com/LinuxSuRen/devops-practice-guide)
- [Argo CD Guide](https://github.com/LinuxSuRen/argo-cd-guide)
- [Argo Rollouts Guide](https://github.com/LinuxSuRen/argo-rollouts-guide)
- [更多场景下的模板样例](templates/README.md)
```
