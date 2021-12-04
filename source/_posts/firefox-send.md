---
title: 记一次在vps服务器上部署Firefox Send
date: 2020-04-19 03:45:36
tags:
-  linux
-  服务器
-  Firefox
-  网盘
categories:
- 日记
headimg: https://cdn.jsdelivr.net/gh/XiaoMengXinX/blog/img/post/firefox-send/1.webp
author: XiaoMengXin
---
简单，私密的文件分享服务。
<!-- more -->

## 什么是Firefox Send？

![](https://cdn.jsdelivr.net/gh/XiaoMengXinX/blog/img/post/firefox-send/2.png)

A file sharing experiment which allows you to send encrypted files to other users. 

（摘自Firefox官方github repo）

简单来说，Firefox Send是火狐旗下的**临时**网盘，它可以在**全平台**使用，**网页式**操作，**不需要附加组件**（而且**不限速**），可以在任何现代浏览器中使用。以超链接形式分享，可设置分享的文件**下载次数、时间和密码**，达到指定下载次数或时长后文件自动过期，并自动从 Send 服务器中删除，在某种程度上相当于**阅后即焚**。

更重要的是，Firefox Send是一款**开源软件**。（[GitHub地址](https://github.com/mozilla/send)）

这意味着任何用户都可以搭建属于自己的Firefox Send

## 教程

那么哪个男孩不想搭建自己的临时网盘呢？

### 依赖

Firefox Send基于nodejs，基本的搭建至少需要[Node.js 10.x](https://nodejs.org/) 以及从GitHub拉源码的git

（你也可以使用docker一键部署，具体见 https://github.com/mozilla/send ）

由于我的服务器是Ubuntu，那么直接使用官方的命令安装：

```shell
curl -sL https://deb.nodesource.com/setup_10.x | sudo bash -
sudo apt install -y nodejs git
```

CentOS可使用如下命令安装：

```shell
curl -sL https://rpm.nodesource.com/setup_10.x | bash -
yum install nodejs -y
```

### 安装

确保你已经安装好了如上依赖，执行：

```shell
git clone https://github.com/mozilla/send.git
```

从GitHub拉取源码，

然后执行：

```shell
npm install
```

即可完成开发环境部署。

你可以使用：

```shell
npm start
```

在 http://localhost:8080 上启动临时的调试服务器。

若测试没问题后，使用：

```shell
npm run build
```

编译生产环境。

这里遇到了一个坑：

build时提示：

> Browserslist: caniuse-lite is outdated. Please run next command "npm update caniuse-lite browserslis"

且执行npm update后依然报错

Google到的方法是：

> 直接删除该项目node_modules下面的caniuse-lite和browserslist这两个文件夹
>
> 然后执行 npm i caniuse-lite browserslist

尝试多次依然无效

最后在 https://github.com/postcss/autoprefixer/issues/1184 看到了正解：

> 在项目文件夹执行
>
> ```shell
> npx browserslist@latest --update-db
> ```

完美解决

编译好了生产环境，就可以用：

```shell
npm run prod
```

让你的网盘跑起来了。

**默认的端口为1443，可以在配置文件 server/config.js 中修改**

### 配置

#### 文件大小

Firefox Send默认的游客最大文件上传大小为1GB，这对于一些需求来说可能还远远不够。

你同样可以在 server/config.js 中修改这个数值

找到这一段：

```js
anon_max_file_size: {
    format: Number,
    default: 1024 * 1024 * 1024,
    env: 'ANON_MAX_FILE_SIZE'
  }
```

将 "default: 1024 * 1024 * 1024" 改成你想要的数值

例如改成游客最多上传5GB大小的文件

只需在最后一个1024后加上 " * 5" 即可

修改后的配置：

```javascript
anon_max_file_size: {
    format: Number,
    default: 1024 * 1024 * 1024 * 5,
    env: 'ANON_MAX_FILE_SIZE'
  }
```

#### 反向代理

Firefox Send的生产环境默认是运行在localhost:1443上的，

那么如何给它添加一个类似 https://send.firefox.com/ 的域名呢？

这就可以使用nginx反向代理实现了。

将你的域名解析到你的服务器，并在nginx绑定你的域名，

反向代理配置如下：

```
location /api/ws {
				proxy_redirect off;
				proxy_pass http://127.0.0.1:1443;
				proxy_http_version 1.1;
				proxy_set_header Upgrade $http_upgrade;
				proxy_set_header Connection "upgrade";
				proxy_set_header Host $http_host;
                }
location / {
			proxy_pass       http://127.0.0.1:1443;
			proxy_set_header Host $host;
			proxy_set_header X-Real-IP $remote_addr;
			proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            }
```

*注：进行反向代理的时候，还需要代理/api/ws这个路径，因为firefox-send文件上传使用的是websocket协议*

至此，你的临时网盘就搭建好了

## 后记

为了稳定的运行，我用screen开启了一个视窗，这样npm就可以一直在视窗内跑，退出终端也不会停止。

我自己搭建的网盘：https://send.onetext.xyz/

Firefox Send上传的文件都可以在 /tmp/send-\*\*\*\*\*\*\*\* 文件夹找到，

实测文件已经过加密，即使你有服务器的完全访问权限，也读取不了别人上传的文件，

所以其安全性还是相当高的。

最后附上几张页面截图

![](https://cdn.jsdelivr.net/gh/XiaoMengXinX/blog/img/post/firefox-send/3.png)

![](https://cdn.jsdelivr.net/gh/XiaoMengXinX/blog/img/post/firefox-send/4.png)

![](https://cdn.jsdelivr.net/gh/XiaoMengXinX/blog/img/post/firefox-send/5.png)

![](https://cdn.jsdelivr.net/gh/XiaoMengXinX/blog/img/post/firefox-send/6.png)