---
title: 如何搭建自己的博客 - 基于 Hexo + Docker + Nginx + Git + Linux 
date: 2019-06-27 00:45:33
updated: 2019-06-27 00:45:33
tags:
- Hexo
- 博客
categories: Hexo
---
过去两年一直在 CSDN 上写博客，不过有广告、等待审核等一些不方便的地方，一直想搭建一个自己的博客。正好手里有一台闲置的腾讯云服务器，于是下定决心准别搭建一个个人博客。经过两天的折腾，终于初步搭建成功。下面分享下搭建过程。

# 准备材料
**注：本方案非 hexo + GitHub 的免费方案，需要拥有一台个人公网服务器。**
- 公网服务器一台
- GitHub 仓库
- 个人域名（非必须）


# 搭建思路
用个人公网服务器作为站点，使用 GitHub 作为文章的仓库。在服务器上通过 git pull 最新的博客仓库，并通过 Hexo 框架生成静态博客站点（这里可以使用 Docker 快速部署 Hexo 运行环境），再用 Nginx 反向代理博客静态页，最后将个人域名解析到博客地址即大功告成。

## 为什么使用 GitHub？
使用 GitHub 作为文章的存储仓库，这样可以在任意一台装有 Git 环境的电脑上编写博客，并同步到 GitHub 仓库中，然后在私有服务器中更新库并发布博客即可。

当然，使用其他平台仓库也可，或者你也可以手动上传博客内容到服务器上，如果你不怕麻烦的话。

## 为什么使用 Hexo？
看个人喜好。调研了一下 Jekyll 和 Hexo，据说 Jekyll 要重一些，而且官网有点丑。具体 Hexo 的特性及优势在这里就不赘述了。

## 为什么使用个人服务器？
GitHub Page + Hexo 是个免费的建站方案。不过有几点不太友好的地方：
- 大陆访问 GitHub 速度有时会很慢
- GitHub Page 有流量限制
- 百度爬虫对 GitHub 进行了屏蔽，不利于 SEO

个人使用的是腾讯云的服务器，而且新用户有很大力度的打折优惠。不差钱的朋友可以选择使用个人服务器，灵活度更高。

## 为什么使用 Docker？
其实 Docker 不是必需的。只要在服务器中装好 Hexo 环境即可。Hexo 依赖于 Node.js，然而我在服务器上安装 Node.js 时遇到一些问题（安装极为缓慢），索性用 Docker 来搭建 Hexo 环境。

同时，使用 Docker 使得搭建环境变得简单，也方便日后的迁移维护等。

## 为什么使用 Nginx？
高性能的 HTTP 和反向代理 Web 服务器，还有什么比这个更好的选择么？

## 有关个人域名
没有个人域名的话，访问博客就得使用 ip 了。

不过需要注意的是，域名解析到境内服务器需要进行 ICP 备案。

好了，接下里就让我们一步一步来按照这个思路操作。

# 服务器环境搭建
以下操作都基于 CentOS 7

## 安装 Git
这里简单说下 Git 安装过程

```
# 下载 git
wget https://mirrors.edge.kernel.org/pub/software/scm/git/git-2.22.0.tar.gz

# 解压
tar -zxvf git-2.22.0.tar.gz

# 配置
cd git-2.22.0 
./configure --prefix=/usr/local/git

# 安装 
make && make install

# 配置环境比那里
vi /etc/profile.d/git.sh

# 在文件中添加
export PATH=$PATH:/usr/local/git/bin

# 查看git版本 
git --version
```
到这里，Git 就已经安装成功了。

## GitHub 建库
首先，在服务器上生成一个 ssh key。

```
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

登录你的 [GitHub](https://github.com/)（没有的同学需要注册一个哈），配置公钥。

创建一个仓库，用于存放博客文件。

## Clone Git 库
在服务器上将仓库克隆下来。

```
cd /usr/blog

# clone 仓库。记得将地址更换成你自己的
git clone git@github.com:MartinMa94/hexo-blog.git
```

至此，有关 Git 的相关准备工作就结束了。

## 安装 Docker

```
# Docker 要求 CentOS 系统的内核版本高于 3.10 ，查看本页面的前提条件来验证你的 CentOS 版本是否支持 Docker。
# 通过 uname -r 命令查看你当前的内核版本
uname -r

# 将 yum 包更新到最新。
yum update

# 卸载旧版本(如果安装过旧版本的话)
yum remove docker docker-common docker-selinux docker-engine

# 安装需要的软件包，yum-util 提供 yum-config-manager 功能，另外两个是d evicemapper 驱动依赖的
yum install -y yum-utils device-mapper-persistent-data lvm2

# 设置 yum 源
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# 查看 docker list
yum list docker-ce --showduplicates | sort -r

# 安装 docker
yum install -y docker-ce
# 或者指定版本
# yum install -y docker-ce-17.12.0.ce

# 启动并加入开机启动
systemctl start docker
systemctl enable docker

# 验证安装是否成功(有 client 和 service 两部分表示 docker 安装启动都成功了)
docker version

```

鉴于国内网络问题，后续拉取 Docker 镜像十分缓慢，我们可以需要配置加速器来解决，我使用的是网易的镜像地址：http://hub-mirror.c.163.com。
新版的 Docker 使用 `/etc/docker/daemon.json` 配置 Daemon。
在该配置文件中加入（没有该文件的话，请先建一个）：
```
vi /etc/docker/daemon.json

{
  "registry-mirrors": ["http://hub-mirror.c.163.com"]
}

# 最后，刷新配置并重启 Docker
systemctl daemon-reload
systemctl restart docker
```
至此，Docker 便安装成功了。

## 创建 Dockerfile
Dockerfile 是 Docker 镜像的描述文件，我们通过编写 Dockerfile 文件可以配置我们想要的 Docker 镜像。有关 Dockerfile 的详细内容这里我就不多做介绍了。

在服务器中的某个目录下，比如 `~/dockerfile/` 中，我们创建一个 Dockerfile 文件。


```
# 创建 Dockerfile 文件
vi ~/dockerfile/Dockerfile

# 文件内容如下
FROM node:latest
MAINTAINER Martin <514969951@qq.com>
RUN npm install
# install hexo
RUN npm install hexo-cli -g
# install hexo server
RUN npm install hexo-server
RUN npm install hexo-deployer-git
# set home dir
WORKDIR /usr/blog
```

下面我来简单解释下文件内容：
- FROM：基础镜像信息，我们需要一个 node 环境的镜像。
- MAINTAINER：维护者信息。
- RUM：镜像操作指令。我们这里安装有关 Hexo 的环境。
- WORKDIR：在构建镜像时，指定镜像的工作目录，之后的命令都是基于此工作目录，如果不存在，则会创建目录。这里我们将工作目录设为镜像中的 /user/blog。之后我们的博客文件也正是放在这个文件夹里。

然后执行 Dockerfile 文件，创建我们的 Hexo 镜像。

```
docker build -t hexo-image .
```

稍等片刻，我们名叫 hexo-image 的镜像就创建好了。

## 创建 Docker 容器

```
# 创建名叫 hexo-blog-1 的 docker 容器
docker run -itd -p 4000:4000 -v /usr/blog/hexo-blog/:/usr/blog --name hexo-blog-1 hexo-image
```

端口 4000 是 hexo server 默认的端口，我们将容器中端口 4000 映射到服务器上的 4000 端口，便于稍后测试用。同时，我们将服务器上的 `/usr/blog/hexo-blog` 目录与容器的工作目录 `/usr/blog` 关联。hexo-blog 即我之前 clone 下来的仓库名（这取决于你建库时的库名），这样我们在容器中操作 `/usr/blog` 目录就等于操作服务器上的 `/usr/blog/hexo-blog` 目录，而之后我们只需要 pull 最新的库就可以更新我们的博客内容了。

## 初始化 Hexo

进入 docker 容器
```
docker exec -it hexo-blog-1 /bin/bash
```

如果之前配置正确的话，我们现在应该进入到容器中的 /usr/blog 目录下

![](https://blog-1-1255473009.cos.ap-beijing.myqcloud.com/blogPic/2019-06/20190626230135.png)

接下来，我们初始化 Hexo

```
hexo init
```

这时候我们可以看到，目录中生成了一些文件，这些就是 Hexo 初始化后的文件了。

因为我们在镜像中安装了 hexo-server，所以我们直接可以通过启动 hexo-server 来查看博客的效果。如果你使用你的本地机器晚上上面一系列操作，你可以在浏览器中直接键入 http://localhost:4000 来查看博客效果。

![](https://blog-1-1255473009.cos.ap-beijing.myqcloud.com/blogPic/2019-06/20190626231736.png)

但现在我们的文件内容在服务器上，可以通过 `服务器ip:4000` 来访问。

如果访问不到看下是不是服务器防火墙没有打开 4000 端口，以及服务器运营商安全组的相关设置。

## 生成博客静态页
通过 hexo init 我们创建了博客的初始化内容，不过这些内容并不是静态页。而 hexo-server 是 Hexo 自带的服务模块，一般也不会用在生产上。我们需要将博客内容生成静态网站，通过 Nginx 这样的 Web 服务器来发布我们的静态页。

使用如下命令生成 Hexo 静态页。

```
hexo g
```

这时我们会看到目录中多了一个 public 文件夹，所有的静态页面都在这个文件夹里。

## Nginx 映射
有关 Nginx 的安装参考我的这篇文章 [Linux 系统安装 Nginx 的方法](https://blog.csdn.net/Colton_Null/article/details/78513268)。

修改 Nginx 的配置文件，添加博客页的代理。

在 Nginx 安装目录中的 conf 目录中的 nginx.conf 文件中，添加如下配置


```
server {
        listen       80;
        server_name  www.martinmyz.cn;

        location / {
            root   /usr/blog/hexo-blog/public;
            index  index.html;
        }
    }
```

server_name 改成你自己的域名或者地址。注意，域名需要解析到你的服务器上。
location 中，root 为博客的目录，index.html 即为主页。

在 Nginx 的 sbin 目录中，执行

```
./nginx -s reload
```

重新加载配置，这时我们再通过访问配置好的域名或服务器地址，即可看到自己的博客了。

## 别忘了将博客内容 push 到 GitHub 中
最后，我们要把目录中的内容推送到 GitHub 仓库中。

这里建议大家将 /public 目录和 db.json 文件添加到忽略中，不需要 Git 跟踪上传。因为每次 hexo g 后，这两个文件都会自动更新。

到这里，我们的博客搭建就已经完成了。

# 如果发布博文？
在任何一台机器上，拉取 Git 库，在 `/source/_posts` 目录中按照 Hexo 规定的格式添加 Markdown 文件，并推送到 GitHub 中。在服务器中 pull 下来最新的内容。

```
git pull
```

通过 Docker 运行 Hexo 命令

```
hexo clean
hexo g
```

即生成了最新的博客静态页。

# 一键更新博客脚本
这一步是为了更加简便的操作，将上述的更新操作通过脚本来完成。之后我们每次发布文章，只需要执行一次脚本即可。

编写 shell 脚本

```
vi ~/myShell/update-blog.sh
```


```
#! /bin/bash

cd /usr/blog/hexo-blog
git pull
docker exec hexo-blog-1 hexo clean
docker exec hexo-blog-1 hexo g
```

设置脚本权限

```
chmod 777 update-blog.sh
```

想要更新博客的话，执行一下脚本就好啦

```
./update-blog.sh
```


# 后续工作（未完成）
身为一个程序员，总想着如何偷懒。虽然有了一键更新博客脚本，但每次 push 博文后，还得跑到服务器上去执行这个脚本，还是有些不方便。这里有两种解决方案：
1. 定时执行脚本。
2. 使用 GitHub 的 Webhooks 功能，在每次 push 后通知服务器执行脚本。

该功能正在研究，弄好后同步到本文中。

---

好了，Hexo 博客的初步搭建就介绍到这里，大家可以愉快的玩耍了。之后会逐步分享 Hexo 博客的配置与优化，大家可以到 [Hexo 官网](https://hexo.io/zh-cn/) 先研究下。

最后，祝大家建站成功！

---

本博客 GitHub 地址：[hexo-blog](https://github.com/MartinMa94/hexo-blog)