# 排错指南 - Pod

本文档介绍 Pod 的异常状态，可能原因和解决办法。

排查 Pod 异常的常用命令如下：

+ 查看 Pod 状态：`kubectl get pods <pod-name> -n <namespace> -o wide`
+ 查看 Pod 的 yaml 配置：`kubectl get pods <pod-name> -n <namespace> -o yaml`
+ 查看 Pod 的事件：`kubectl describe pods <pod-name> -n <namespace>`
+ 查看 Pod 容器日志：`kubectl logs -n <namespace> <pod-name> [-c <container-name>]`

## Pod 一直处于 Pending 状态

Pending 状态说明 Pod 还没有调度到某个 Node 上面。可以通过 `kubectl describe pods <pod-name> -n <namespace>` 命令查看到 Pod 的事件，进而判断为什么没有调度。如：

    ``` bash
    $ kubectl describe pod mypod
    ...
    Events:
    Type     Reason            Age                From               Message
    ----     ------            ----               ----               -------
    Warning  FailedScheduling  12s (x6 over 27s)  default-scheduler  0/4 nodes are available: 2 Insufficient cpu.
    ```

可能的原因和解决方案：

1. 集群的 kube-scheduler 服务都挂掉了：

    可以在各节点运行 `ps -elf | grep kube-scheduler` 命令来验证。如果挂掉，可以使用 `systectl start kube-scheduler` 命令启动服务；

1. 节点没有打标签或打的标签值不匹配：

    如 Pod 指定了 nodeSelector、nodeAffinity、podAffinity 或 AntiAffinity 等标签选择器，但没有节点打对应的标签或打的标值不匹配。
    
    可以使用 `kubectl describe pods` 查看 Pod 的定义，使用 `kubectl get node --show-labels` 查看各 Node 的 lables 列表，使用 `kubectl label node lable_name=lable_value` 给 Node 打标签；

1. 节点资源不足：

    集群内所有的 Node 都不满足该 Pod request 的 CPU、Memory、GPU 或者临时存储空间等资源，或者 Node 上的 Pod 数量达到了上限(kubelet 的 --max-pods 参数指定)。
    
    可以通过 `kubectl describe node` 来查看各节点资源分配情况。然后通过删除不用的 Pod 来释放资源，或增加新的 Node 来增加资源。

    **案例**：
    
    1. 创建的 Pod 一直处于 Pending 状态，kubectl describe pods 显示 `No nodes are available that match all of the predicates: Insufficient pods (3).`

    原因：

    1. kubelet 默认最多能运行 110 个 Pods：

        ``` txt
        $ kubectl describe node m7-devops-128071|grep -A 7 'Capacity'
        Capacity:
        cpu:     40
        memory:  264040352Ki
        pods:    110
        Allocatable:
        cpu:     40
        memory:  263937952Ki
        pods:    110
        ```

    当所有 kubelet 运行的 Pods 数达到 110 后，新创建的 Pod 就会因为资源不足而无法调度，提示 Insufficient pods；

    解决方案：

    1. kubelet 的 --max-pods 选项可以指定运行的最大 Pod 数目，通过调大该参数，如 110 → 240， 可以解决该问题。
    1. 注意：因为 flanneld 配置的本节点 Pod 网段是 /24，所以一个 Node 最多运行 254 个 Pod（flannel、docker0 占用 2 个），--max-pods 不能超过该值；

1. 创建 Pod 基础容器 (sanbox) 失败：

    现象：

    1. 创建的 Pod 一直处于 Pending 状态；

        ``` bash
        [root@scriptk8c ~]# k get pods
        NAME                 READY     STATUS    RESTARTS   AGE
        dnsuutils-ds-7q6dr   0/1       Pending   0          3s
        ```

    1. kubectl describe pod 显示创建 sandbox pod 失败：

        ``` bash
        [root@scriptk8c ~]# k describe pods dnsuutils-ds-7q6dr|tail
        Tolerations:     node.alpha.kubernetes.io/notReady:NoExecute
                        node.alpha.kubernetes.io/unreachable:NoExecute
                        node.kubernetes.io/disk-pressure:NoSchedule
                        node.kubernetes.io/memory-pressure:NoSchedule
        Events:
        Type     Reason                  Age               From                     Message
        ----     ------                  ----              ----                     -------
        Normal   SuccessfulMountVolume   2m                kubelet, 172.27.129.237  MountVolume.SetUp succeeded for volume "default-token-jhdrm"
        Warning  FailedCreatePodSandBox  25s (x8 over 2m)  kubelet, 172.27.129.237  Failed create pod sandbox.
        Warning  FailedSync              25s (x8 over 2m)  kubelet, 172.27.129.237  Error syncing pod
        ```

    原因：

    1. kubelet 基础容器配置参数：--pod-infra-container-image=docker02:35000/rhel7/pod-infrastructure:latest
    1. 节点本地不存在这个 image，所以 kubelet 去 registry docker02:35000 拉取该镜像；
    1. registry 需要登录认证，而 kubelet 没有提供认证信息，所以获取 pod-infra-container-image 失败，进而导致创建 sandbox 失败；

    解决方案：

    1. 从其他节点导出 pod-infra-container-image ，再导入到该节点，这样 kubelet 就不会去 registry 拉取镜像了；

        ``` bash
        $ docker save registry.access.redhat.com/rhel7/pod-infrastructure -o rhel.tar
        $ docker load -i rhel.tar
        ```

    1. 或者让 kublet 使用节点的 docker 认证信息去 registry 拉取镜像：
        1. 在节点 root 账户下，执行 docker login docker02:35000 命令，输入账号、密码登录；
        1. 修改 kubelet 配置文件，添加环境变量: HOME=root，重启 kubelet:

            ``` bash
            [root@scriptk8c ~]# grep HOME /mnt/disk01/opt/k8s/etc/kubernetes/kubelet
            HOME=/root
            
            [root@scriptk8c ~]# systemctl restart kubelet
            [root@scriptk8c ~]#
            ```

    注意：方案 2 适用于 k8s 1.8 版本，1.11 支持更多的认证方式；

    参考：

    1. https://v1-8.docs.kubernetes.io/docs/concepts/containers/images/
    1. https://kubernetes.io/docs/concepts/containers/images/#configuring-nodes-to-authenticate-to-a-private-repository

1. Pod 使用 HostNetwork，对应的端口(HostPort)在节点上已被占用：

    **案例**：
    
    1. 集群中已经部署一个 node_exporter 的 Daemonset，它使用了 hostNetwork 和 hostPID ，端口为 9100。则再部署一个使用相同端口的 daemonset 时，一直不创建 pod（describe 时所有 field 均为 0）：

        ``` bash
        [root@m7-devops-128123 log]# kubectl get ds |grep -E "NAME|node-exporter"
        NAME                               DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
        my-prom-prometheus-node-exporter   0         0         0         0            0           <none>          17m
        ```

    1. 查看所有 kube-xxx 组件的日志，均没有相关的异常日志。

    原因：

    1. Daemonset 由 DaemonSet Controller （非 scheduler ）负责调度。如果 DS Controller 发现没有 Node 满足调度条件，如 Node 资源不足、** HostPort 端口冲突**等，就不会为 Node 创建 POD；
    1. 本问题原因在于 Node_Exporter 使用了 hostNewtork，部署监听相同端口的 Daemonset 时出现 HostPort 端口冲突，故 K8S 不再创建 POD；

    解决方案：

    1. 换个 HostPort (Service 端口也需要更新) 或通常使用 Service 对外开放服务端口；

## Pod 一直处于 Waiting 或 ContainerCreating 状态

可能的原因和解决方案:

1. 创建 Pod 的基础容器 (sandbox) 失败，例如节点本地没有 pod-infrastructure 镜像，但是从 kubelet 拉取失败（如认证问题）：

    使用命令 `docker images |grep pod-infra` 确认节点 kubelet 参数 `--pod-infra-container-image` 对应的镜像，如 `registry.access.redhat.com/rhel7/pod-infrastructure:latest` 是否存在。如果不存在则手动拉取到节点（可能需要先登录 registry）。

1. Pod yaml 定义中请求的 CPU、Memory 太小或者**单位不对**，不足以成功运行 Sandbox。

    常见的错误是：
    1. Pod yaml 定义 request 或 limit Memory 值是**不带单位**（如 Gi, Mi, Ki 等）的数值，如 4，这时只分配和限制使用 4 bytes。
    2. 内存单位 M 写成了小写 m，如 1024m，表示 1.024 Byte；
    
    `kubectl descirbe pod` 显示：

        Pod sandbox changed, it will be killed and re-created。
    
    kubelet 日志报错: 

        to start sandbox container for pod ... Error response from daemon: OCI runtime create failed: container_linux.go:348: starting container process caused "process_linux.go:301: running exec setns process for init caused \"signal: killed\"": unknown

1. 拉取镜像失败：
    + 配置了错误的镜像；
    + Kubelet 无法访问镜像仓库（国内环境访问 gcr.io 需要特殊处理）；
    + 拉取私有镜像的 imagePullSecret 没有配置或配置有误；
    + 镜像太大，拉取超时（可以适当调整 kubelet 的 --image-pull-progress-deadline 和 --runtime-request-timeout 选项）；

## Pod 一直处于 ImagePullBackOff 状态

可能的原因和解决方案：

1. http 类型的 registry，但是没有添加到 dockerd 的 insecure-registry=172.27.129.211:35000 配置参数中：

    dockerd 默认从 https 类型的 registry 拉取镜像，如果使用 https 类型的 registry，则必须将它添加到 insecure-registry 参数中，然后重启或 reload dockerd 生效。

1. https 类型的 registry，但是使用自签名的 ca 证书，dockerd 不识别：

    将 registry 加入到 insecure-registry 参数中，然后重启或 reload dockerd 生效。或者将它的 ca 证书放置到 `/etc/docker/certs.d/<registry:port>/ca.crt` 位置；

1. registry 需要认证，但是 Pod 没有配置 imagePullSecret，配置的 Secret 不存在或有误：

    首先创建一个 docker-registry 类型的 Secret：

    ``` bash
    $ kubectl create secret docker-registry my-secret --docker-server=DOCKER_REGISTRY_SERVER --docker-username=DOCKER_USER --docker-password=DOCKER_PASSWORD --docker-email=DOCKER_EMAIL
    ```

    然后在容器中引用这个 Secret:

    ``` bash
    spec:
      containers:
      - name: private-reg-container
        image: <your-private-image>
      imagePullSecrets:
      - name: my-secret
    ```

1. 镜像文件损坏，需要重新 push 镜像文件

    kubectl describe pod 报错：
        
        Failed to pull image "docker02:35000/env/release/3.2.0/prophet/app/executor-service-kube.tar:release-3.2.0-8": rpc error: code = Unknown desc = error pulling image configuration: unknown blob

1. kubelet 的 --registry-qps、--registry-burst 值太小（默认分别为 5，10)，并发拉取镜像时被限制，kubectl describe pod 报错：
    
        Failed to pull image "172.27.129.211:35000/metricbeat-prophet:6.0.1": pull QPS exceeded，这时，可以调大这两个参数，然后重启 kubelet 解决。

    --registry-qps 太小导致的并发限制案例：

    ``` bash
    [root@m7-power-k8s01 ~]# kubectl get pods -n env30 -o wide alert-engine-alert-engine-54fd454f64-clbd7
    NAME                                               READY     STATUS                  RESTARTS   AGE       IP               NODE
    alert-engine-alert-engine-54fd454f64-clbd7         0/3       ImagePullBackOff        0          59m       172.30.208.111   m7-power-k8s03

    [root@m7-power-k8s01 ~]# kubectl describe pods -n env30 alert-engine-alert-engine-54fd454f64-clbd7 | tail -3
    Warning  Failed                  44m (x2 over 46m)   kubelet, m7-power-k8s03  Failed to pull image "172.27.129.211:35000/filebeat-with-module:6.0.0": pull QPS exceeded.
    Normal   BackOff                 5m (x83 over 51m)   kubelet, m7-power-k8s03  Back-off pulling image "172.27.129.211:35000/filebeat-with-module:6.0.0"
    Normal   Pulling                 36s (x13 over 51m)  kubelet, m7-power-k8s03  pulling image "172.27.129.211:35000/filebeat-with-module:6.0.0"
    ```

## Pod 一直处于 ImageInspectError 状态

现象：

1. 启动 Pod 失败，kubectl get pods 显示的 STATUS 为 "ImageInspectError"

    ``` bash
    [root@m7-devops-128071 gitlab]# kubectl get pods --all-namespaces|grep Inspect
    fangrong                  pms-558b58dfbd-fpjzl                                              0/1       ImageInspectError   0          13h
    prophet-resource-automl   pas-08a7e677-5c4e-4e2d-992f-7d93cd0f05d5-automl-544ff774b7xgh88   0/1       ImageInspectError   0          13h
    prophet-resource-automl   pas-1ccab6a2-0415-4542-8ccf-348dd28451c4-automl-6c6bd6f4644sqpw   0/1       ImageInspectError   0          13h
    qatest312                 pms-b97bc97fc-5djfl                                               0/1       ImageInspectError   0          6h
    ```

1. kubectl describe pod 显示，ImageInspectError 的原因为 readlink /mnt/disk0/docker/data/overlay2: invalid argument：

    ``` bash
    [root@m7-devops-128071 gitlab]# kubectl describe pods -n fangrong pms-558b58dfbd-fpjzl|tail -2
    Warning  FailedSync             1h (x522 over 13h)  kubelet, m7-devops-128107  Error syncing pod
    Warning  InspectFailed          3m (x590 over 13h)  kubelet, m7-devops-128107  Failed to inspect image "docker02:35000/env/develop/prophet/app/pms.tar:develop-175": rpc error: code = Unknown desc = Error response from daemon: readlink /mnt/disk0/docker/data/overlay2: invalid argument
    ```

1. 打开 dockerd 的 debug 日志级别，查看对应的日志显示 在 readlink /mnt/disk0/docker/data/overlay2 出现了 os.PathError 错误：

    ``` bash
    13 06:00:04 m7-devops-128107 dockerd[34157]: time="2018-08-13T06:00:04.061634901+08:00" level=debug msg="FIXME: Got an API for which error does not match any expected type!!!: readlink /mnt/disk0/docker/data/overlay2: invalid argument" error_type="*os.PathError" module=api
    8月 13 06:00:04 m7-devops-128107 dockerd[34157]: time="2018-08-13T06:00:04.061670830+08:00" level=error msg="Handler for GET /v1.31/images/docker02:35000/grafana/grafana-enhanced:5.2.0/json returned error: readlink /mnt/disk0/docker/data/overlay2: invalid argument"
    8月 13 06:00:04 m7-devops-128107 dockerd[34157]: time="2018-08-13T06:00:04.061704408+08:00" level=debug msg="FIXME: Got an API for which error does not match any expected type!!!: readlink /mnt/disk0/docker/data/overlay2: invalid argument" error_type="*os.PathError" module=api
    ```

原因：

1. 节点上 docker 镜像文件损坏，当使用它启动容器后，容器文件系统错误，进而导致系统调用 readlink() 返回 os.PathError 错误；
1. dockerd 不能正确处理这个 Error https://github.com/allencloud/docker/blob/master/api/server/httputils/errors.go#L65，所以提示 FIXME: Got an API for which error does not match any expected type!!!
1. image 文件损坏可能与重启服务器导致的文件系统不完整有关，可以使用 fsck 命令修复文件系统；

验证节点上镜像文件损坏的步骤：

1. 在该节点上使用 image 起容器，结果启动失败：

    ``` bash
    [root@m7-devops-128107 ~]# docker run -it docker02:35000/env/develop/prophet/app/pms.tar:develop-175 sh
    docker: Error response from daemon: OCI runtime create failed: container_linux.go:348: starting container process caused "exec: \"/bin/bash\": stat /bin/bash: no such file or directory": unknown.
    [root@m7-devops-128107 ~]#
    ```

1. 或者，导出 image 镜像失败，提示文件完整性校验出错：

    ``` bash
    [root@m7-devops-128107 ~]# docker save docker02:35000/env/develop/prophet/app/pms.tar:develop-175 -o pms.img
    Error response from daemon: file integrity checksum failed for "etc/anacrontab"
    ```

解决方案：

1. 删除节点上所有使用损坏 image 的容器，否则不能删除 image：

    ``` bash
    [root@m7-devops-128107 ~]# docker ps -a|grep pms.tar
    ecdad07d835b        docker02:35000/env/develop/prophet/app/pms.tar:develop-175                   "/bin/bash /opt/work…"    About a minute ago   Created                                               competent_meninsky
    f9c805e91ac7        docker02:35000/env/develop/prophet/app/pms.tar:develop-175                   "/bin/bash /opt/work…"    2 minutes ago        Created                                               hopeful_stonebraker
    63b6a12efc3e        docker02:35000/env/develop/prophet/app/pms.tar:develop-175                   "/bin/bash /opt/work…"    8 minutes ago        Created                                               confident_wright
    [root@m7-devops-128107 ~]# docker rm ecdad07d835b # 删除所有使用损坏的 pms.tar 镜像的容器
    ecdad07d835b
    [root@m7-devops-128107 ~]# docker rm f9c805e91ac7
    f9c805e91ac7
    [root@m7-devops-128107 ~]# docker rm 63b6a12efc3e
    63b6a12efc3
    ```

1. 删除节点上损坏的 image (必须要有 Deleted: sha256 开头的输出结果，才表明实际删除了 image layer 文件，否则需要删除使用它的容器后再删除 image)：

    ``` bash
    [root@m7-devops-128107 ~]# docker rmi docker02:35000/env/develop/prophet/app/pms.tar:develop-175
    ```

1. 重新拉取 image 文件：

    ``` bash
    [root@m7-devops-128107 ~]# docker pull docker02:35000/env/develop/prophet/app/pms.tar:develop-175
    ```

参考：

+ https://github.com/allencloud/docker/blob/master/api/server/httputils/errors.go#L65
+ https://github.com/kubernetes/kubernetes/issues/63612

## Pod 一直处于 CrashLoopBackOff 状态

CrashLoopBackOff 状态说明容器曾经启动了，但又异常退出了。此时 Pod 的 RestartCounts 通常是大于 0 的，可以先查看一下容器的日志：

``` bash
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> [-c container_name]
kubectl logs --previous <pod-name> -n <namespace> [-c container_name]
```

从 describe pod 的 State、Last State 里，以及容器日志可以发现一些容器退出的原因，比如：

+ 容器进程退出，如域名解析失败、连接数据库失败；
+ 健康检查失败退出
+ OOMKilled
+ 镜像文件损坏

``` bash
$ kubectl describe pod mypod
...
Containers:
  sh:
    Container ID:  docker://3f7a2ee0e7e0e16c22090a25f9b6e42b5c06ec049405bc34d3aa183060eb4906
    Image:         alpine
    Image ID:      docker-pullable://alpine@sha256:7b848083f93822dd21b0a2f14a110bd99f6efb4b838d499df6d04a49d0debf8b
    Port:          <none>
    Host Port:     <none>
    State:          Terminated
      Reason:       OOMKilled
      Exit Code:    2
    Last State:     Terminated
      Reason:       OOMKilled
      Exit Code:    2
    Ready:          False
    Restart Count:  3
    Limits:
      cpu:     1
      memory:  1G
    Requests:
      cpu:        100m
      memory:     500M
...
```

如果此时如果还未发现线索，还可以到容器内执行命令来进一步查看退出原因：

``` bash
$ kubectl exec cassandra -- cat /var/log/cassandra/system.log
```

如果还是没有线索，那就需要 SSH 登录该 Pod 所在的 Node 上，查看 Kubelet 或者 Docker 的日志进一步排查了：

``` bash
# Query Node
kubectl get pod <pod-name> -o wide

# SSH to Node
ssh <username>@<node-name>

# Query kubelet log
journalctl -u kubelet 

# Query docker log
journalctl -u docker
```

**案例一**：

1. Pod 启动失败，kubectl get pods 显示状态为 CrashLoopBackOff，kubectl describe pods 显示错误信息：read-only file system error

现象：

1. Pod 启动失败，kubectl get pods 显示状态为 CrashLoopBackOff：

    ``` bash
    [root@m7-devops-128071 gitlab]# kubectl get pods --all-namespaces -o wide|grep 128123|grep -v Running|grep pas
    prophet-resource-qa312    pas-7e9f06b0-6f42-4c40-b9a9-eb3471ee824b-predictor-7789d7bjqzdw   0/2       CrashLoopBackOff    5          3m        172.30.37.27     m7-devops-128123
    prophet-resource-qa312    pas-c7917262-fd06-4f98-86bf-96bf3813e5ec-predictor-765c577zp6jl   0/2       CrashLoopBackOff    4          3m        172.30.37.16     m7-devops-128123
    prophet-resource-qa312    pas-d12af6cb-3659-48b3-b2d7-f34fc9536c23-online-dataload-7455d7   0/1       ImagePullBackOff    0          14m       172.30.37.29     m7-devops-128123
    ```

1. kubectl describe pods 显示 mkdir 时返回 read-only file system error：

    ``` bash
    E0813 14:55:53.024865    3777 remote_runtime.go:209] StartContainer "d41d314a85af6eb8c7e5e" from runtime service failed: rpc error:
    code = Unknown desc = failed to start container "d41d314a85af6eb8c7e5ec38ff89": Error response from daemon: OCI runtime create failed:
    container_linux.go:348: starting container process caused
    "process_linux.go:402: container init caused \"rootfs_linux.go:58: mounting \\\"/mnt/disk0/k8s/kubelet/pods/ea9035f3-9ec5-11e8-af9c-0cc47adb93d8/volumes/kubernetes.io~empty-dir/log-entry--predictor--1\\\" to rootfs \\\"/mnt/disk0/docker/data/overlay2/962a78785442dc45bdf95e/merged\\\"
    at \\\"/mnt/disk0/docker/data/overlay2/962a78785442dc45b0f5/merged/collect/predictor/root/predictor/logs/predictor\\\" caused
    \\\"mkdir /mnt/disk0/docker/data/overlay2/962a78785442dc495e/merged/collect/predictor/root/predictor/logs/predictor:
    read-only file system\\\"\"": unknown
    ```

原因：

1. 节点本地 docker image 文件损坏，当使用它启动容器后，容器文件系统错误，进而导致被只读挂载(ro)；
1. 或者，docker pull image 时出错(提示 error pulling image configuration: unknown blob)，导致挂载时文件系统错误；

解决方案：

1. 删除节点上所有使用损坏 image 的容器，然后删除 image，再重新 pull image；
1. 确认 registry 中的 image 文件是否完整，不完整时重新 push image；

**案例二**:

现象：

1. Pod 启动失败，kubectl get pods 显示状态为 CrashLoopBackOff：
    
    ``` bash
    $ kubectl get pods -n metricbeat  -o wide |grep  m7-power-128050
    metricbeat-995dcffbd-6rppf            0/1       CrashLoopBackOff   1142       9d        172.30.168.11    m7-power-128050
    ```
1. kubectl describe pods 显示 docker 将 pod 的 kubelet 目录 mount 到容器目录中时提示 no such file or directory：

    ``` bash
    $ kubectl describe pods -n metricbeat metricbeat-995dcffbd-6rppf | tail -10
    Warning  Failed                 4d                    kubelet, m7-power-128050  Error: failed to start container "metricbeat": Error response from daemon: OCI runtime create failed: container_linux.go:348: starting container process caused "process_linux.go:402: container init caused \"rootfs_linux.go:58: mounting \\\"/mnt/disk2/k8s/kubelet/pods/54d426d3-cacd-11e8-971a-5e384b278319/volume-subpaths/config/metricbeat/0\\\" to rootfs \\\"/mnt/disk1/docker/data/overlay2/5753fb64509f490968802fe00a6a6e000b7f17f4839d62ba5ca1dc484c86ba22/merged\\\" at \\\"/mnt/disk1/docker/data/overlay2/5753fb64509f490968802fe00a6a6e000b7f17f4839d62ba5ca1dc484c86ba22/merged/etc/metricbeat.yml\\\" caused \\\"no such file or directory\\\"\"": unknown
    Warning  Failed                 4d                    kubelet, m7-power-128050  Error: failed to start container "metricbeat": Error response from daemon: OCI runtime create failed: container_linux.go:348: 
    Normal   Pulled                 43m (x1135 over 9d)   kubelet, m7-power-128050  Container image "docker.elastic.co/beats/metricbeat:6.4.1" already present on machine
    Warning  FailedSync             23m (x26664 over 4d)  kubelet, m7-power-128050  Error syncing pod
    Warning  BackOff                3m (x25616 over 4d)   kubelet, m7-power-128050  Back-off restarting failed container
    ```

原因：容器已经挂了，但是 kubelet 还不知晓，导致 kubelet 将 pod 目录挂载到容器目录时，容器目录不存在。

解决办法：删除 pod，然后自动重建。

## Pod 一直处于 Error 状态

通常处于 Error 状态说明 Pod 启动过程中发生了错误。常见的原因包括：

+ 依赖的 ConfigMap、Secret 或者 PV 等不存在
+ 请求的资源超过了管理员设置的限制，比如超过了 LimitRange 等
+ 违反集群的安全策略，比如违反了 PodSecurityPolicy 等
+ 容器无权操作集群内的资源，比如开启 RBAC 后，需要为 ServiceAccount 配置角色绑定

## Pod 一直处于 Terminating 或 Unknown 状态

正常情况下，如果删除了 Pod，经过一个 grace period（默认 30s）后，如果 Pod 还在 Running，则 kublet 会向 docker 发送 kill 命令，进而 docker 向 Pod 中的所有进程发送 SIGKILL 信号，强行删除 Pod。所以，如果节点工作正常，一般一个 grace period 后，Pod 会被清除。

如果节点失联 NotReady，默认 5min 后，node controller 开始驱逐它上面的 Pods，即将该 Node 上的 Pod 标记为 Terminating 状态，然后在其它节点上再起 Pod。从 v1.5 开始，node controller 不再从 etcd 中强行删除（force delete）失联 Node 上的 Pod 信息，而是等待节点恢复连接后，确认驱逐的 Pod 都已经 Terminating 后才删除这些 Pods。所以，这一段时间内，Pod 可能有多副本运行的情况。想要删除 NotReady 节点上的 Terminating 或 Unknown 状态 Pod 的方法：

+ 从集群中删除该 Node：kubectl delete node <node-name>
+ Node 恢复正常。Kubelet 会重新跟 kube-apiserver 通信确认这些 Pod 的期待状态，进而再删除这些 Pod。
+ 用户强制删除。用户可以执行 kubectl delete pods <pod> --grace-period=0 --force 强制删除 Pod。除非明确知道 Pod 的确处于停止状态（比如 Node 所在 VM 或物理机已经关机），否则不建议使用该方法。特别是 StatefulSet 管理的 Pod，强制删除容易导致脑裂或者数据丢失等问题。

处于 Terminating 状态的 Pod 在 Kubelet 恢复正常运行后一般会自动删除。但有时也会出现无法删除的情况，并且通过 kubectl delete pods <pod> --grace-period=0 --force 也无法强制删除。此时一般是由于 finalizers 导致的，通过 kubectl edit 将 finalizers 删除即可解决。

    ``` bash
    "finalizers": [
    "foregroundDeletion"
    ]
    ```

另一种导致删除的 Pod 一直处于 Terminating 状态的原因：Pod 业务容器和 Sandbox 都被正常删除，但是 kubelet 在 umount Pod 挂载的目录时一直失败，提示 device or resource busy，进而导致 Pod 的状态一直是 Terminating：

``` bash
9月 22 20:04:07 ee-test kubelet[3583]: E0922 20:04:07.711666    3583 nestedpendingoperations.go:264] Operation for "\"kubernetes.io/configmap/c74b89dd-be53-11e8-a1f4-525400f721a0-filebeat-config\" (\"c74b89dd-be53-11e8-a1f4-525400f721a0\")" failed. No retries permitted until 2018-09-22 20:04:08.711632504 +0800 CST (durationBeforeRetry 1s). Error: error cleaning subPath mounts for volume "filebeat-config" (UniqueName: "kubernetes.io/configmap/c74b89dd-be53-11e8-a1f4-525400f721a0-filebeat-config") pod "c74b89dd-be53-11e8-a1f4-525400f721a0" (UID: "c74b89dd-be53-11e8-a1f4-525400f721a0") : error deleting /mnt/disk01/k8s/lib/kubelet/pods/c74b89dd-be53-11e8-a1f4-525400f721a0/volume-subpaths/filebeat-config/filebeat/1: remove /mnt/disk01/k8s/lib/kubelet/pods/c74b89dd-be53-11e8-a1f4-525400f721a0/volume-subpaths/filebeat-config/filebeat/1: device or resource busy
```

原因：
1. CentOS 3.10 内核以及 17.12.1 以前版本的 docker 在 umount 时可能会出现 device busy 情况；

解决方案：

1. 升级 docker-ce 到 17.12.1 及以上版本；
1. 升级操作系统内核到 4.4.x;

参考：

+ https://github.com/moby/moby/issues/22260
+ https://github.com/kubernetes/kubernetes/issues/65110

## Pod 状态长时间(小时级别)不更新，一直处于 Creating、Terminating 状态

现象：

1. 创建、删除或重建 Pod 后，等待很长时间(小时级别），kubectl get pods 显示 Pod 的 Status 一直是 Creating、Terminating 状态，进而导致 Service 长时间不可用；
1. 在 Pod 调度到的节点上，执行 docker ps |grep xxx 命令，可以看到 Creating 的 Pod 容器已实际 Running 一段时间了；

原因：

1. kubelet 周期（1s）向 docker 查询各容器的状态、配置参数、image 是否存在、image 版本等信息；
1. kubelet 将查询的结果发给 kube-apiserver，以更新 Pod 的状态，这个发送过程受多个 QPS 参数控制；
1. kubelet 的 QPS 配置参数：  --event-qps 、--kube-api-qps、--event-burst、--kube-api-burst 默认值分别为 5、5、10、10 ；
1. 当节点 Pod 数目过多(200+)，Pod 状态更新时间非常频繁时，kubelet 的默认 QPS 值成为瓶颈，导致 Pod 状态不能及时更新到 kube-apiserver。查看 Pod 的 Status时， 一直处于 Creating、Terminating 状态，进而导致 Service 长时间不可用；

解决办法：

1. 调大 kubelet 的 QPS 配置参数：

    ``` bash
    --event-qps=0
    --kube-api-qps=2000
    --kube-api-burst=4000
    --registry-qps=0
    ```

参考：

1. https://github.com/kubernetes/kubernetes/issues/39113#issuecomment-305919878

## Pod 行为异常

这里所说的行为异常是指 Pod 没有按预期的行为执行，比如没有运行 podSpec 里面设置的命令行参数。这一般是 podSpec yaml 文件内容有误，可以尝试使用 --validate 参数重建容器，比如：

``` bash
kubectl delete pod mypod
kubectl create --validate -f mypod.yaml
```

也可以查看创建后的 podSpec 是否是对的，比如

```bash
kubectl get pod mypod -o yaml
```

## 修改静态 Pod 的 Manifest 后未自动重建

Kubelet 使用 inotify 机制检测 /etc/kubernetes/manifests 目录（可通过 Kubelet 的 --pod-manifest-path 选项指定）中静态 Pod 的变化，并在文件发生变化后重新创建相应的 Pod。但有时也会发生修改静态 Pod 的 Manifest 后未自动创建新 Pod 的情景，此时一个简单的修复方法是重启 Kubelet。

## 参考文档

+ [Troubleshoot Applications](https://kubernetes.io/docs/tasks/debug-application-cluster/debug-application/)
+ [Pod 异常排错](https://feisky.gitbooks.io/kubernetes/zh/troubleshooting/pod.html)
