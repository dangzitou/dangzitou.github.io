---
title: "Windows 11 下安装 WSL 和 Ubuntu 24.04 全记录"
date: 2025-06-13 10:00:00 +0800
categories: [教程]
tags: [Windows11, WSL, Ubuntu, 环境搭建]
---

## 前言

最近想在 Windows 11 上体验 Linux 开发环境，但是虚拟机和宿主机的互通太局限，况且图形化界面对于开发者来说不是非常有必要，于是选择了安装 WSL（Windows Subsystem for Linux），并安装了 Ubuntu 24.04。本文记录一下详细步骤和遇到的坑，希望能帮到后来的朋友。

---

## 一、首先要在Windows11开启 WSL 功能

1. 按 `Win + S`，搜索“启用或关闭 Windows 功能”并打开。
2. 勾选：
    - “适用于 Linux 的 Windows 子系统”
    - “虚拟机平台（Virtual Machine Platform）”
    ![开启WSL功能](/assets/post_imgs/2025-06-13-windows11-install-wsl-ubuntu24-04/image.png)
3. 确认后重启电脑。

---

## 二、安装 WSL

1. 打开 PowerShell（管理员），输入：
    ![打开PowerShell](/assets/post_imgs/2025-06-13-windows11-install-wsl-ubuntu24-04/image-1.png)
    ```powershell
    wsl --install
    ```

2. 如果已经有 WSL1，升级到 WSL2：

    ```powershell
    wsl --set-default-version 2
    ```

---

## 三、安装 Ubuntu
### 方法一（推荐）：
1. 打开 Microsoft Store，搜索“Ubuntu”。
2. 找到并安装“Ubuntu”。
    ![alt text](/assets/post_imgs/2025-06-13-windows11-install-wsl-ubuntu24-04/image-4.png)
### 方法二：
- 由于主包的MsStore故障了，所以不得已采用第二种方法
![alt text](/assets/post_imgs/2025-06-13-windows11-install-wsl-ubuntu24-04/image-2.png)
1. 进入Ubuntu官网下载WSL版本（网址：https://ubuntu.com/desktop/wsl）
![alt text](/assets/post_imgs/2025-06-13-windows11-install-wsl-ubuntu24-04/image-3.png)
2. 下载解压后得到一个`ubuntu-24.04.2-wsl-amd64`文件
3. 修改文件名，添加后缀`.tar`，便于后续解压安装
4. 回到`PowerShell`，导入`Ubuntu`系统镜像，输入命令：
    ```
    wsl --import <发行版名称> <安装目录> <镜像文件路径>
    ```
    例如我的`ubuntu-24.04.2-wsl-amd64.tar`目录在`E:\Edge Download\ubuntu-24.04.2-wsl-amd64`，我希望WSL虚拟硬盘地址放在`E:\Ubuntu-24.04`我就输入：
    ```
    wsl --import Ubuntu-24.04 "D:\WSL\Ubuntu-24.04" "D:\Edge Download\ubuntu-24.04.2-wsl-amd64.tar"
    ```
    注意路径要用`""`括起来
    
    出现这样的页面即为安装成功
    ![alt text](/assets/post_imgs/2025-06-13-windows11-install-wsl-ubuntu24-04/image-5.png)
## 四、启动Ubuntu
### 1. **方法一**：开始菜单启动（简单快捷）
- 按下键盘上的 Win 键（或点击屏幕左下角的开始菜单）。
- 直接在搜索框输入 “Ubuntu” 或你安装时命名的发行版名称（如“Ubuntu-24.04”）。
- 出现“Ubuntu”应用后，点击它即可打开 Ubuntu 终端。
    ![alt text](/assets/post_imgs/2025-06-13-windows11-install-wsl-ubuntu24-04/image-6.png)
### 2. **方法二**：使用 Windows Terminal（个人推荐，自定义化程度高）
- 打开 Windows Terminal（`Win + x`再点击`终端`或直接搜索`终端`）。
- 在下拉菜单中选择“Ubuntu”或你导入的 WSL 发行版名称。
- 点击后会自动打开对应的 Linux 终端。
    ![alt text](/assets/post_imgs/2025-06-13-windows11-install-wsl-ubuntu24-04/image-7.png)
- 如果希望以后每次打开终端默认进入Ubuntu，可以设置一下默认配置文件为Ubuntu（如下图所示）
![alt text](/assets/post_imgs/2025-06-13-windows11-install-wsl-ubuntu24-04/image-10.png)
![alt text](/assets/post_imgs/2025-06-13-windows11-install-wsl-ubuntu24-04/image-11.png)
### 3. **方法三**：命令行启动
- 按 Win + R，输入 wsl 回车，默认会进入你设置的默认 Linux 发行版（比如 Ubuntu）。
- 如果你有多个 WSL 发行版，输入如下命令启动指定的版本：
    ```
    wsl -d Ubuntu-24.04
    ```
    这里的 `Ubuntu-24.04` 是你导入时设置的名称，请根据实际情况替换。
    ![alt text](/assets/post_imgs/2025-06-13-windows11-install-wsl-ubuntu24-04/image-8.png)



---
## 五、常见问题及解决办法

### 1. 安装速度慢/下载失败
- 建议切换到国内源后再更新系统，或使用科学上网工具。

### 2. wsl: 检测到 localhost 代理配置，但未镜像到 WSL。NAT 模式下的 WSL 不支持 localhost 代理。
![alt text](/assets/post_imgs/2025-06-13-windows11-install-wsl-ubuntu24-04/image-9.png)
这不是报错，而是一个“警告”或“提示”

它告诉你：检测到 Windows 系统有为 localhost（127.0.0.1）设置代理，但 WSL 在 NAT 网络模式下（就是WSL2默认的模式）不支持直接使用 localhost 代理。

啥意思？

就是你使用了科学上网工具，代理服务一般监听在 Windows 的本地 127.0.0.1:端口 上。
但 WSL2 是一个虚拟网络环境，WSL2 里的 127.0.0.1 指向的是 Linux 子系统自己，而不是 Windows 的 127.0.0.1，所以不能直接访问 Windows 上的 localhost 代理。

如果你不想看到这个提示，直接把科学上网工具退出了就好。如果想让WSL能够使用代理上网，可以看下一条。

### 3. 如何能让WSL能够使用代理上网？

这一点对于要经常性访问外网下载东西的开发者来说很重要。
#### 1. 开启科学上网工具的局域网连接，设置科学上网工具代理端口，开启HTTP(S)端口，将端口设置一下（可以设置自己喜欢的端口），这个要根据自己的工具设置，我这里用的是Clash Verge；
![alt text](/assets/post_imgs/2025-06-13-windows11-install-wsl-ubuntu24-04/image-12.png)
#### 2. 回到Windows Terminal并进入Ubuntu，输入以下指令,进入.bashrc文件：
```
nano ~/.bashrc
```
#### 3. 用鼠标滚轮或方向键滑动到文件末尾，添加如下代码（这里的`7899`设置成你自己刚刚在代理软件设置的HTTP端口）：
```
WSL_HOST_IP=$(ip route | grep -m 1 default | awk '{print $3}')
export http_proxy="http://$WSL_HOST_IP:7899"
export https_proxy="http://$WSL_HOST_IP:7899"
```
#### 4.设置防火墙入站规则，避免流量被拦截。
- 按 Win + S 搜索“防火墙”，点进“高级安全Windows Defender 防火墙”。
![alt text](/assets/post_imgs/2025-06-13-windows11-install-wsl-ubuntu24-04/image-13.png)
- 选择左侧“入站规则”，右侧点“新建规则”。
- 选择“端口”，点“下一步”。
- 选择“TCP”，指定“特定本地端口”，输入：7899（你的代理端口）
- 点“下一步”，选择“允许连接”，继续。
- 适用情况全部勾选（域、专用、公用），点“下一步”。
- 填个名字，比如“`Clash Verge 7899 for WSL`”，点“完成”。
![alt text](/assets/post_imgs/2025-06-13-windows11-install-wsl-ubuntu24-04/image-15.png)
#### 5.完成后在Ubuntu输入`curl -I google.com`验证，成功
![alt text](/assets/post_imgs/2025-06-13-windows11-install-wsl-ubuntu24-04/image-14.png)
---


## 六、总结

WSL 让我们在 Windows 下也能顺畅体验 Linux，开发更高效！如果你也遇到安装问题，欢迎留言交流。

---