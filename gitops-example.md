_我们将使用Argo Workflow构建一个CI/CD的PipeLine来展示 Workflow的使用，_
_包括以下几个步骤：_

1. 使用git-sync clone一个git仓库到volume中
2. 使用Kaniko/buildkit构建docker镜像
3. 推送docker镜像到镜像仓库

# 使用git-sync clone一个git仓库到volume中
1.1.首先我们需要创建一个secret，包含git ssh credentials用来从github中clone 仓库到volume中，执行以下命令：
```yaml
##create a secret for git
kubectl create secret generic git-creds \
     --from-file=ssh=$HOME/.ssh/id_rsa \
     --from-file=known_hosts=$HOME/.ssh/known_hosts 
```
1.2.然后我们创建一个Workflow Template，将上面创建的secret挂载到我们的容器中。
```yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: ci-test
spec:
  #mount secrets for the workflow
  volumes:
    - name: git-secret
      secret:
        defaultMode: 256
        secretName: git-creds # your-ssh-key
    - name: repo-root
      hostPath:
        path: /tmp/ci-test
        type: DirectoryOrCreate
  nodeSelector:
      argo-worflow: "true"
  #start the sequence
  templates:
      #this is the git fetch magic as an init container in the first step
      initContainers:
        - image: k8s.gcr.io/git-sync:v3.1.6
          args:
            - "--repo=git@github.com:<YOUR-ORG>/<YOUR-REPO>.git"
            - "--root=/workdir/root"
            - "--max-sync-failures=3"
            - "--timeout=200"
            - "--branch=main"
            - "--ssh"
            - "--one-time"
          name: git-data
          volumeMounts:
            - name: repo-root
              mountPath: /workdir ##
            - name: git-secret
              mountPath: /etc/git-secret
      securityContext:
        runAsUser: 0 # to allow read of ssh key
```
我们在volumes中声明了一个叫做git-secret的volume，引用了我们刚才创建的git-creds secrets。同时使用hostpath声明了挂载本机的/tmp/ci-test作为共享存储，我们整个Workflow的数据都会存在这个共享存储上。同时使用了NodeSelector指定该Workflow在固定节点运行（因为repo-root不是共享存储）。
volumes声明完成后，我们在templates.initcontainers里声明了一个init-containers，该container 使用了k8s.gcr.io/git-sync:v3.1.6镜像，负责将git repo拷贝到/workdir中，这样，我们就完成了使用使用git-sync clone一个git仓库到volume中的过程。
```yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: ci-test
spec:
  entrypoint: pipeline
  #mount secrets for the workflow
  volumes:
    - name: git-secret
      secret:
        defaultMode: 256
        secretName: git-creds # your-ssh-key
    - name: repo-root
      hostPath:
        path: /tmp/ci-test
        type: DirectoryOrCreate
  nodeSelector:
      argo-worflow: "true"
  hostNetwork: true
  #start the sequence
  templates:
    - name: pipeline
      dag:
        tasks:
          - name: git-sync
            template: git-sync
          - name: check-repo
            template: check-repo
            depends: git-sync
    - name: check-repo
      container:
        image: alpine:latest
        command: [sh, -c]
        args:
          [
            "echo getting message from volume; ls /workdir/root/argo-workflow-guide > ls.txt; cat ls.txt",
          ]
        volumeMounts:
          - name: repo-root
            mountPath: /workdir ##
          - name: git-secret
            mountPath: /etc/git-secret ##
    - name: git-sync
      container:
        image: docker.io/chriskery/git-sync:v3.1.6
        args:
          - "--repo=git@github.com:chriskery/argo-workflow-guide.git"
          - "--root=/workdir/root"
          - "--dest=argo-workflow-guide"
          - "--max-sync-failures=3"
          - "--timeout=200"
          - "--branch=main"
          - "--ssh"
          - "--one-time"
        command: ["/git-sync"]
        name: git-data
        volumeMounts:
          - name: repo-root
            mountPath: /workdir ##
          - name: git-secret
            mountPath: /etc/git-secret
      securityContext:
        runAsUser: 0 # to allow read of ssh key
```
我们定义了一个叫做pineline和一个叫做check-repo的template，entrypoint为pipeline。Workflow会先运行initContainers的步骤，然后再运行entrypoint的步骤。pipeline里调用了check-repo，因此当这个workflow运行后，我们能在check-repo的步骤看到相关的输出。完整的WorkflowTemplate如下：
```yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: ci-test
spec:
  entrypoint: pipeline
  #mount secrets for the workflow
  volumes:
    - name: git-secret
      secret:
        defaultMode: 256
        secretName: git-creds # your-ssh-key
    - name: repo-root
      hostPath:
        path: /tmp/ci-test
        type: DirectoryOrCreate
  nodeSelector:
      argo-worflow: "true"
  hostNetwork: true
  #start the sequence
  templates:
    - name: pipeline
      dag:
        tasks:
          - name: git-sync
            template: git-sync
          - name: check-repo
            template: check-repo
            depends: git-sync
    - name: check-repo
      container:
        image: alpine:latest
        command: [sh, -c]
        args:
          [
            "echo getting message from volume; ls /workdir/root/argo-workflow-guide > ls.txt; cat ls.txt",
          ]
        volumeMounts:
          - name: repo-root
            mountPath: /workdir ##
          - name: git-secret
            mountPath: /etc/git-secret ##
    - name: git-sync
      container:
        image: docker.io/chriskery/git-sync:v3.1.6
        args:
          - "--repo=git@github.com:chriskery/argo-workflow-guide.git"
          - "--root=/workdir/root"
          - "--dest=argo-workflow-guide"
          - "--max-sync-failures=3"
          - "--timeout=200"
          - "--branch=main"
          - "--ssh"
          - "--one-time"
        command: ["/git-sync"]
        name: git-data
        volumeMounts:
          - name: repo-root
            mountPath: /workdir ##
          - name: git-secret
            mountPath: /etc/git-secret
      securityContext:
        runAsUser: 0 # to allow read of ssh key
```
你现在可以通过Argo Workflow UI或Argo CLI提交这个Workflow Template。运行完成之后你应该能看到类似的输出：
![截屏2024-04-26 00.07.58.png](https://cdn.nlark.com/yuque/0/2024/png/12923067/1714061286022-b57623c7-29ab-4680-9cd7-d8774d456a81.png#averageHue=%23574a42&clientId=u7ebd007b-bae5-4&from=drop&id=SDRst&originHeight=1266&originWidth=2840&originalType=binary&ratio=2&rotation=0&showTitle=false&size=223708&status=done&style=none&taskId=u54c8538d-8798-4c4a-9c46-773147be01d&title=)
看看我们刚才clone下来的git仓库：
![截屏2024-04-26 00.07.18.png](https://cdn.nlark.com/yuque/0/2024/png/12923067/1714061242502-175d2a45-4b9c-499b-b196-4a021fceed75.png#averageHue=%23292d39&clientId=u7ebd007b-bae5-4&from=drop&id=e3So7&originHeight=216&originWidth=1124&originalType=binary&ratio=2&rotation=0&showTitle=false&size=74199&status=done&style=none&taskId=u81bde82c-3e3d-4baf-8732-ccd1cc9d334&title=)
现在有了数据，我们可以开始构建和推送我们的镜像了。
# 使用Kaniko/buildkit构建Docker镜像
> **Kaniko**: Kaniko is a tool to build container images from a Dockerfile, inside a container or Kubernetes cluster. [https://github.com/GoogleContainerTools/kaniko](https://github.com/GoogleContainerTools/kaniko)
> **buildkit**: buildKit is a toolkit for converting source code to build artifacts in an efficient, expressive and repeatable manner.
> [https://github.com/moby/buildkit](https://github.com/moby/buildkit)

kaniko和buildkit都可以在user space中完成镜像构建的功能，而无需跟主机上上的docker.socket进行通信。这里我们使用kaniko进行镜像构建。首先我们先创建一个secret将docker registry的credentials放在里面：
```yaml
#docker-secret:
kubectl create secret generic docker-secret --from-file=config.json=$HOME/.docker/config.json 

#config.json 类似于：
{"auths": {"https://index.docker.io/v1/": {"username":"$USERNAME","password":"$PASSWORD"},}}
#或者 
# echo "$USERNAME":"$PASSWORD" | base64 
#{"auths": {"https://index.docker.io/v1/": {"auth":"上一步的输出"}}
```
然后我们增加一个Task，进行镜像的构建与推送在上面的Workflow Template上增加以下的内容：
```yaml
  entrypoint: pipeline
  #mount secrets for the workflow
  volumes:
    - name: docker-secret
      secret:
        secretName: docker-secret
  #start the sequence
  templates:
    - name: pipeline
      dag:
        tasks:
          - name: git-sync
            template: git-sync
          - name: check-repo
            template: check-repo
            depends: git-sync
          - name: build-docker
            template: build-docker 
            depends: check-repo
    - name: build-docker
      container:
        #argo is gonna ask for a command - the debug version allows us to exec this way i think
        image: docker.io/chriskery/executor:v1.9.0
        imagePullPolicy: Always
        env:
          - name: DOCKER_USERNAME
            value: chriskery
          - name: DOCKER_PASSWORD
            value: 1584537639qq
        command: ["/kaniko/executor"]
        # (if we set cache true we should make an ECR container update)
        # i gave the full path to the docker file but maybe relative is fine
        args:
          [
            "--context=/workdir/root/argo-workflow-guide",
            "--destination=docker.io/chriskery/argo-worflow-guide:v1",
            "--cache=false",
          ]
        volumeMounts:
          - name: repo-root
            mountPath: /workdir ##
          - name: docker-secret
            mountPath: /kaniko/.docker
        resources:
          limits:
            cpu: 1
            memory: 5Gi
```
完整的示例如下：
```yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: ci-test
spec:
  entrypoint: pipeline
  #mount secrets for the workflow
  volumes:
    - name: git-secret
      secret:
        defaultMode: 256
        secretName: git-creds # your-ssh-key
    - name: repo-root
      hostPath:
        path: /tmp/ci-test
        type: DirectoryOrCreate
    - name: docker-secret
      secret:
        secretName: docker-secret
  nodeSelector:
      argo-worflow: "true"
  hostNetwork: true
  #start the sequence
  templates:
    - name: pipeline
      dag:
        tasks:
          - name: git-sync
            template: git-sync
          - name: check-repo
            template: check-repo
            depends: git-sync
          - name: build-docker
            template: build-docker 
            depends: check-repo
    - name: check-repo
      container:
        image: alpine:latest
        command: [sh, -c]
        args:
          [
            "echo getting message from volume; ls /workdir/root/argo-workflow-guide > ls.txt; cat ls.txt",
          ]
        volumeMounts:
          - name: repo-root
            mountPath: /workdir ##
          - name: git-secret
            mountPath: /etc/git-secret ##
    - name: build-docker
      container:
        #argo is gonna ask for a command - the debug version allows us to exec this way i think
        image: docker.io/chriskery/executor:v1.9.0
        imagePullPolicy: Always
        env:
          - name: DOCKER_USERNAME
            value: chriskery
          - name: DOCKER_PASSWORD
            value: 1584537639qq
        command: ["/kaniko/executor"]
        # (if we set cache true we should make an ECR container update)
        # i gave the full path to the docker file but maybe relative is fine
        args:
          [
            "--context=/workdir/root/argo-workflow-guide",
            "--destination=docker.io/chriskery/argo-worflow-guide:v1",
            "--cache=false",
          ]
        volumeMounts:
          - name: repo-root
            mountPath: /workdir ##
          - name: docker-secret
            mountPath: /kaniko/.docker
        resources:
          limits:
            cpu: 1
            memory: 5Gi
    - name: git-sync
      container:
        image: docker.io/chriskery/git-sync:v3.1.6
        args:
          - "--repo=git@github.com:chriskery/argo-workflow-guide.git"
          - "--root=/workdir/root"
          - "--dest=argo-workflow-guide"
          - "--max-sync-failures=3"
          - "--timeout=200"
          - "--branch=main"
          - "--ssh"
          - "--one-time"
        command: ["/git-sync"]
        name: git-data
        volumeMounts:
          - name: repo-root
            mountPath: /workdir ##
          - name: git-secret
            mountPath: /etc/git-secret
      securityContext:
        runAsUser: 0 # to allow read of ssh key
```
# 
