
### 0.前言简述
描述：使用Github-Action或者Aliyun镜像服务同步镜像到个人DockerHub或者私有镜像仓库中

**作者主页:** https://www.weiyigeek.top
**作者博客:** https://blog.weiyigeek.top
**作者公众号:** 全栈工程师修炼指南

---

### 1.使用Github Action优雅的同步国外镜像到个人DockerHub中
描述: 由于国内上网环境的原因，在部署某些云原生应用时，通常会遇到镜像无法直接拉取，例如 `k8s.io、gcr.io、quay.io` 等国外仓库中的镜像，在最开始的做法是使用他人同步到dockerHub仓库中的此版本镜像，或者是采用国外的vps虚拟主机使用`docker pull/docker tag/docker push`命令的方式复制到dockerHub仓库，但是对于作者来说这两种都不是最优解，因为有可能他人没有同步到你所需要的版本或者说你根本就没有VPS，此时应该怎么办呢。

虽然前面作者写了一篇【如何使用Aliyun容器镜像服务对海外gcr、quay仓库镜像进行镜像拉取构建?】的文章地址：
https://mp.weixin.qq.com/s/oQ82YWYRnSIUp-RXLdNS8A 

但是作者仍然觉得不够优雅，并且不能批量的同步，此处作者在使用Github-Action构建项目时，突发奇想为何不用Github Action+Skopeo工具来同步镜像呢，说做就做，遂有了此篇文章。


Github项目地址(欢迎大家Fork): https://github.com/WeiyiGeek/action-sync-images/


**操作流程:**
Step 1.登录Gitub，点击右上角`+`,然后创建一个名为`action-sync-images`的Github仓库。

![weiyigeek.top-创建Github仓库图](https://img.weiyigeek.top/2023/5/20230727092416.png)


Step 2.首先点击仓库里中的settins菜单，选择安全选项卡，点击Action，然后将会进入到 `Actions secrets and variables`，此时为了账号密码，我们需要提前设置我们Docker hub登录的账号密码到项目的secrets中（PS: fork了此项目的朋友可以自行将对应DocekrHub设置为自己的账号密码）。

![weiyigeek.top-创建action使用的secrets图](https://img.weiyigeek.top/2023/5/20230727094133.png)


Step 3.然后点击仓库里中的Action菜单，在选择一个 simple workflows 将会为我们创建一个新的工作流文件或者在项目根目录自行创建一个`.github/workflows/sync-images-dockerHub-example.yaml`目录文件。

![weiyigeek.top-快速创建 simple workflows 图](https://img.weiyigeek.top/2023/5/20230727092651.png)


Step 4.此处我们拉取kubernetes 最新的 V1.27.4 版本，使用kubeadm搭建集群此时我们要在Github Action中使用skopeo工具将`registry.k8s.io`仓库中的镜像同步到docker.io，执行下述shell命令，我们提前获取所需镜像并拼接拷贝命令，若需拷贝到自己的hub仓库请执行自行修改`DOCKER_HUBUSERURL`，此处我dockerhub用户名是`weiyigeek`。
```bash
K8SVERSION=1.27.4
DOCKER_HUBUSERURL=docker.io/weiyigeek
kubeadm config images list --kubernetes-version=${K8SVERSION} 2>/dev/null > K8sv1.27.4.txt
for i in `cat K8sv1.27.4.txt`;do
  echo skopeo copy --all docker://${i} docker://${DOCKER_HUBUSERURL}/${i##*/}
done

# 执行结果:
skopeo copy --all docker://registry.k8s.io/kube-apiserver:v1.27.4 docker://docker.io/weiyigeek/kube-apiserver:v1.27.4
skopeo copy --all docker://registry.k8s.io/kube-controller-manager:v1.27.4 docker://docker.io/weiyigeek/kube-controller-manager:v1.27.4
skopeo copy --all docker://registry.k8s.io/kube-scheduler:v1.27.4 docker://docker.io/weiyigeek/kube-scheduler:v1.27.4
skopeo copy --all docker://registry.k8s.io/kube-proxy:v1.27.4 docker://docker.io/weiyigeek/kube-proxy:v1.27.4
skopeo copy --all docker://registry.k8s.io/pause:3.9 docker://docker.io/weiyigeek/pause:3.9
skopeo copy --all docker://registry.k8s.io/etcd:3.5.7-0 docker://docker.io/weiyigeek/etcd:3.5.7-0
skopeo copy --all docker://registry.k8s.io/coredns/coredns:v1.10.1 docker://docker.io/weiyigeek/coredns:v1.10.1

```

Step 5.将上述执行结果放置在`Use Skopeo Tools Sync Image to Docker Hub`子步骤下，然后将下述工作流的脚本复制粘贴到`sync-images-dockerHub-example.yaml`文件中，然后点击`commit changes`进行提交即可，注意下面是使用skopeo工具进行同步，为啥要使用此工具可以参考作者的【如何使用Skopeo做一个优雅的镜像搬运工】此篇文章地址: https://mp.weixin.qq.com/s/_r9WLMAIbOFEzj7-OWPWDw。

```bash
# 工作流名称
name: Sync-Images-to-DockerHub-Example
# 工作流运行时显示名称
run-name: ${{ github.actor }} is Sync Images to DockerHub.
# 怎样触发工作流
on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# 工作流程任务（通常含有一个或多个步骤）
jobs:
  syncimages:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout Repos
      uses: actions/checkout@v3
      
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2.9.1

    - name: Login to Docker Hub
      uses: docker/login-action@v2.2.0
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}
        logout: false
    
    # 使用shell命令批量同步所需的镜像到dockerHub中
    - name: Use Skopeo Tools Sync Image to Docker Hub
      run: |
        #!/usr/bin/env bash
        skopeo copy --all docker://registry.k8s.io/kube-apiserver:v1.27.4 docker://docker.io/weiyigeek/kube-apiserver:v1.27.4
        skopeo copy --all docker://registry.k8s.io/kube-controller-manager:v1.27.4 docker://docker.io/weiyigeek/kube-controller-manager:v1.27.4
        skopeo copy --all docker://registry.k8s.io/kube-scheduler:v1.27.4 docker://docker.io/weiyigeek/kube-scheduler:v1.27.4
        skopeo copy --all docker://registry.k8s.io/kube-proxy:v1.27.4 docker://docker.io/weiyigeek/kube-proxy:v1.27.4
        skopeo copy --all docker://registry.k8s.io/pause:3.9 docker://docker.io/weiyigeek/pause:3.9
        skopeo copy --all docker://registry.k8s.io/etcd:3.5.7-0 docker://docker.io/weiyigeek/etcd:3.5.7-0
        skopeo copy --all docker://registry.k8s.io/coredns/coredns:v1.10.1 docker://docker.io/weiyigeek/coredns:v1.10.1
```

![weiyigeek.top-sync-images-dockerHub-example图](https://img.weiyigeek.top/2023/5/20230727103541.png)


Step 6.commit提交后将会触发工作流执行，此时我们回到仓库的action页面，点击如下图所示的，查看此工作流执行情况，是否有同步失败的情况。

![weiyigeek.top-查看工作流执行情况图](https://img.weiyigeek.top/2023/5/20230727103839.png)

Step 7.最后登录我的Docker Hub ( https://hub.docker.com/r/weiyigeek/ )验证是否已经同步过来, 可以从下图看到已经同步过来了。此后我们便可以使用 `docker pull` 命令或者是 `ctr image pull` 命令拉取镜像即可。

![weiyigeek.top-验证镜像同步图](https://img.weiyigeek.top/2023/5/20230727105454.png)


温馨提示: 默认`Docker Hub`我们创建的账号都是免费计划，虽然没有空间的大小限制，但是有下载次数以及下载速度的限制，所以有条件的尽量自行使用内部私有镜像仓库。

至此，使用Github Action + Skopeo 工具优雅的同步镜像到dockerHub中完毕!

<br/>

### 2.使用Aliyun容器镜像服务拉取同步

如何使用Aliyun容器镜像服务对海外gcr、quay仓库镜像进行镜像拉取构建?

参考文章：https://mp.weixin.qq.com/s/oQ82YWYRnSIUp-RXLdNS8A

```bash
k8s.gcr.io/sig-storage/nfs-subdir-external-provisioner  > registry.cn-hangzhou.aliyuncs.com/weiyigeek/nfs-subdir-external-provisioner:v4.0.2

gcr.io/kaniko-project/executor:latest ->  registry.cn-hangzhou.aliyuncs.com/weiyigeek/kaniko-executor:latest
```

---

<span style="color:red">温馨提示</span>：微信小程序【极客全栈修炼】上线了，可以直接在微信中浏览唯一极客技术博客（ https://blog.weiyigeek.top ）中的相关文章，涉及网络安全、系统运维、应用开发、物联网实战、全栈文章，希望和大家一起学习进步，欢迎浏览交流！  

<div align="center">
  <img src="https://www.weiyigeek.top/img/share.jpg" alt="个人主页站点-微信公众号-微信小程序【极客全栈修炼】" />
</div>




