---
title: 使用 SSH连接Ubuntu on Windows (WSL)
date: 2022-07-07
author: hb0730
authorLink: https://blog.hb0730.me
categoires: [ 'Other' ]
---

* 首先是卸载重装一遍ssh服务，这里不是很确定是不是自带ssh服务有没有问题。

  ```shell
  sudo apt-get remove openssh-server
  sudo apt-get install openssh-server
  ```

* 编辑[sshd_config](https://ubuntu.com/server/docs/service-openssh)文件

  ```shell
  sudo vim /etc/ssh/sshd_config
  ```

  **PasswordAuthentication yes # 允许用户名密码方式登录**

* 修改完重启ssh服务

  ```shell
  sudo service ssh restart
  ```

* 查看ssh服务

  ```shell
  sudo service ssh status
  
  
  * sshd is running
  ```

  



##### 可以使用SSH客户端连接WSL



---

相关文章:

* [Windows WSL](https://docs.microsoft.com/zh-cn/windows/wsl/)
* [Windows Server Docs](https://docs.microsoft.com/zh-cn/windows/wsl/)

### 

