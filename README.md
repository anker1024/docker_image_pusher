# Docker Images Pusher

使用Github Action将国外的Docker镜像转存到阿里云私有仓库，供国内服务器使用，免费易用<br>
- 支持DockerHub, gcr.io, k8s.io, ghcr.io等任意仓库<br>
- 支持最大40GB的大型镜像<br>
- 使用阿里云的官方线路，速度快<br>
- **自动跳过内容未变化的镜像，避免重复推送**<br>
- **命名增强：自动前缀防重名，支持 `=>` 自定义**<br>

## 使用方式


### 配置阿里云
登录阿里云容器镜像服务<br>
https://cr.console.aliyun.com/<br>
启用个人实例，创建一个命名空间（**ALIYUN_NAME_SPACE**）
![](/doc/命名空间.png)

访问凭证–>获取环境变量<br>
用户名（**ALIYUN_REGISTRY_USER**)<br>
密码（**ALIYUN_REGISTRY_PASSWORD**)<br>
仓库地址（**ALIYUN_REGISTRY**）<br>

![](/doc/用户名密码.png)


### Fork本项目
Fork本项目<br>
#### 启动Action
进入您自己的项目，点击Action，启用Github Action功能<br>
#### 配置环境变量
进入Settings->Secret and variables->Actions->New Repository secret
![](doc/配置环境变量.png)
将上一步的**四个值**<br>
ALIYUN_NAME_SPACE,ALIYUN_REGISTRY_USER，ALIYUN_REGISTRY_PASSWORD，ALIYUN_REGISTRY<br>
配置成环境变量

### 添加镜像
打开images.txt文件，添加你想要的镜像。支持以下格式：

- 直接写镜像名，可带tag，不写tag默认latest<br>
  例如：`nginx:latest`、`gdy666/lucky:latest`
- 支持 `--platform=xxxxx` 参数指定镜像架构<br>
  例如：`--platform=linux/arm64 xiaoyaliu/alis`
- 支持私库地址<br>
  例如：`k8s.gcr.io/kube-state-metrics/kube-state-metrics:v2.0.0`
- 支持 `#` 开头作为注释<br>
  例如：`# xhofe/alist:latest`
- 支持自定义目标镜像名，使用 `=>` 分隔源镜像和自定义名称<br>
  例如：`xhofe/alist:latest => xhofe_alist:latest`

提交文件后，自动进入Github Action构建。

#### 自动命名规则
默认情况下，程序会把源镜像的命名空间作为前缀加到镜像名称前，避免不同命名空间下同名镜像冲突。

例如：
```
xhofe/alist
xiaoyaliu/alist
```
会分别转存为：
```
xhofe_alist
xiaoyaliu_alist
```

如果源镜像没有命名空间（例如 `nginx:latest`），则不会添加命名空间前缀，直接使用原镜像名。

#### 自定义命名
如果不想使用自动命名，或者希望完全指定目标镜像名，可以使用 `=>` 语法：
```
源镜像 => 目标镜像名:标签
```

例如：
```
xhofe/alist:latest => xhofe_alist:latest
xiaoyaliu/alist:latest => xiaoyaliu_alist:latest
k8s.gcr.io/kube-state-metrics/kube-state-metrics:v2.0.0 => kube-state-metrics:v2.0.0
```

指定自定义命名后，最终推送的镜像名就是 `ALIYUN_NAME_SPACE/目标镜像名:标签`，不再自动添加命名空间前缀。

如果同时使用了 `--platform`，平台前缀仍然会加在自定义名称前面，避免不同架构互相覆盖。

![](doc/images.png)

### 使用镜像
回到阿里云，镜像仓库，点击任意镜像，可查看镜像状态。(可以改成公开，拉取镜像免登录)
![](doc/开始使用.png)

在国内服务器pull镜像, 例如：<br>
```
docker pull registry.cn-hangzhou.aliyuncs.com/shrimp-images/alpine
```
registry.cn-hangzhou.aliyuncs.com 即 ALIYUN_REGISTRY(阿里云仓库地址)<br>
shrimp-images 即 ALIYUN_NAME_SPACE(阿里云命名空间)<br>
alpine 即 阿里云中显示的镜像名<br>

### 多架构
需要在images.txt中用 `--platform=xxxxx` 手动指定镜像架构。指定后的架构会以前缀的形式放在镜像名字前面。

例如：
```
--platform=linux/arm64 xiaoyaliu/alis
```
最终镜像名会包含平台前缀，例如：
```
linux_arm64_xiaoyaliu_alis
```

如果同时使用自定义命名：
```
--platform=linux/arm64 xiaoyaliu/alis:latest => xiaoyaliu_alis_arm64:latest
```
最终镜像名为：
```
linux_arm64_xiaoyaliu_alis_arm64:latest
```

![](doc/多架构.png)

### 镜像重名
程序默认会将命名空间作为前缀加在镜像名称前，因此不同命名空间下的同名镜像不会冲突。

例如：
```
xhofe/alist
xiaoyaliu/alist
```
会分别转存为：
```
xhofe_alist
xiaoyaliu_alist
```

如果希望完全自定义镜像名，可以使用上文的自定义命名语法：
```
xhofe/alist => my_alist
```

![](doc/镜像重名.png)

### 避免重复推送
每次运行 Action 时，程序都会先对比**源镜像**和**阿里云远端已存在镜像**的 config digest（镜像内容摘要）：

- 如果两者一致，说明镜像内容没有变化，**直接跳过拉取和推送**，不消耗磁盘、网络和 Action 时间。
- 如果远端不存在，或者 digest 不一致（说明上游已更新），才会执行拉取、打 tag、推送流程。

整个过程通过 `docker manifest inspect` 完成，**不需要拉取镜像**，只走 registry API，因此判断本身几乎不消耗时间和空间。

这让项目非常适合配置成定时执行（例如每天同步一次），即使上游镜像没有变化，也能零开销地确认同步状态。

### 定时执行
修改/.github/workflows/docker.yaml文件
添加 schedule即可定时执行(此处cron使用UTC时区)
![](doc/定时执行.png)


## 原作者
视频教程：https://www.bilibili.com/video/BV1Zn4y19743/<br>
作者：[技术爬爬虾](https://github.com/tech-shrimp/me)<br>
B站，抖音，Youtube全网同名<br>
