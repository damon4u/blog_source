---
title: Charles抓取HTTPS接口
date: 2018-10-22 17:41:50
tags: http
categories: http
---

日常开发过程中，可能会需要抓取线上https的接口，这里介绍使用 __Charles__ 来实现。

<!-- more -->

首先在电脑上安装 __Charles__ ，破解不破解都可以。我这里的版本是4.2.1。

之后需要在手机上安装SSL证书。
点击 __Charles__ 的 __Help__ -> __SSL Proxying__ -> __Install Charles Root Certificate on a Mobile Device or Remote Browser__ :
![](/images/charles1.png)

之后会弹出弹窗，提示如何安装证书：
首先配置手机网络使用Charles的HTTP代理。
点击已经连接的Wi-Fi网络边上的叹号按钮，打开手机的网络配置页面，拖到最下面选择配置代理按钮，选择手动，然后服务器填写电脑的ip，端口填写8888：
![](/images/charles2.jpeg)

配置好代理后，用手机浏览器访问 `chls.pro/ssl`，提示安装证书，然后根据提示安装即可。

安装完证书还不算完，新的ios系统需要单独信任证书配置：
选择手机的 __通用__ -> __关于本机__ -> __证书信任设置__ ，开启证书信任。

这样手机端就配置完成了。

回到电脑端Charles，选择 __Proxy__ -> __SSL Proxying Settings__ ，在这里添加对那些域名进行代理拦截，如果全部拦截，就配置为`*:443`即可：
![](/images/charles3.png)

至此，配置工作完成，应该可以抓取大部分https接口了。

不用的时候，记得断开手机代理。
