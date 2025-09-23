---
title: "vsftpd 上传文件路径问题排查记录"
date: 2025-09-23 22:00:00 +0800
categories: [Linux, FTP]
tags: [vsftpd, FTP, Linux, 权限, 脚本]
---

## 问题背景

在配置 **vsftpd** 作为 FTP 服务时，我写了一个自动化上传的脚本，脚本逻辑大致是：

1. 打包本地文件为 `/tmp/www.dzt.tar.gz`
2. 登录 FTP 服务器
3. 切换到用户 `user2` 的目录
4. 上传备份文件

脚本内容:
```shell
#!/bin/bash
FTP_HOST="$1"
FTP_USER="$2"
FTP_PWD="$3"

echo "FTP主机: $FTP_HOST"
echo "FTP用户: $FTP_USER"

echo ">>> 修改ftp用户user2的目录权限为同用户组可读可写..."
sudo chmod 770 /srv/ftp/user2
echo ">>> 权限设置完成"

TAR_FILE="/tmp/www.dzt.tar.gz"
tar -czf "$TAR_FILE" -C /var www

if [[ ! -f "$TAR_FILE" ]]; then
    echo "错误: 打包失败，未生成 $TAR_FILE" >&2
    exit 1
fi
echo ">>> 打包成功: $TAR_FILE"

REMOTE_FILE=$(basename "$TAR_FILE")

echo ">>> 正在登录ftp服务器并上传文件..."

FTP_SCRIPT=$(mktemp)
cat > "$FTP_SCRIPT" << EOF
open $FTP_HOST
user $FTP_USER $FTP_PWD
binary
cd /srv/ftp/user2
put ${TAR_FILE} ${REMOTE_FILE}
ls
quit
EOF

# 执行ftp命令并捕获输出
FTP_OUTPUT=$(ftp -n < "$FTP_SCRIPT" 2>&1)
UPLOAD_SUCCESS=$?

rm -f "$FTP_SCRIPT"

echo "$FTP_OUTPUT" | grep -q "$REMOTE_FILE"

if [[ $? -eq 0 && $UPLOAD_SUCCESS -eq 0 ]]; then
    echo ">>> 上传成功！"
    echo ">>> 备份文件: $TAR_FILE 已上传至 FTP 服务器 /srv/ftp/user2/$REMOTE_FILE"
else
    echo "错误: FTP 上传失败！" >&2
    rm -f "$TAR_FILE"
    exit 1
fi

echo "=== 脚本执行完毕 ==="
```

但是，执行脚本的时候，到ftp上传文件这一步，却总是出现如下报错：
```sh
Could not create file.
```

---

## 排查过程

按理来说，执行了`sudo chmod 770 /srv/ftp/user2`就应该user2目录的所有者（用户user2）可以：
- 进入该目录（执行权限）
- 查看内容（读）
- 创建/删除文件（写）

和 user2 同组的用户 也可以进入、读取、写入该目录。

在页面输入：

```shell
groups user1
groups user2
```
返回：
```shell
user1 : user1 users group1
user2 : user2 users group1
```
说明确实是在同一个用户组下，这没毛病。但是为啥还是`Could not create file.`
于是尝试手动进入`/srv/ftp/user2`目录下看看能否创建文件，发现还是无权限。
于是回去查看命令行，发现自己在创建`/srv/ftp/user2`目录时执行了一行：
```shell
sudo chown user2:user2 /srv/ftp/user2
```
于是执行
```shell
sudo ls -la /srv/ftp/user2
```
发现：
```shell
total 8
drwxrwx--- 2 user2 user2 4096 Sep 23 21:03 .
drwxr-xr-x 3 root  ftp   4096 Sep 23 21:03 ..
```
终于找到问题。通过`drwxrwx--- 2 user2 user2 4096 Sep 23 21:03 .`可知`user2`文件夹目前所属的用户组是`user2`，相当于当时创建了一个新用户组。

## 解决过程
将`/srv/ftp/user2`文件夹用户组改成当时创建的group1即可。执行：
```shell
sudo chgrp group1 /srv/ftp/user2
```
重新执行脚本，显示：
```shell
FTP主机: localhost
FTP用户: user1
>>> 修改ftp用户user2的目录权限为同用户组可读可写...
>>> 权限设置完成
>>> 打包成功: /tmp/www.dzt.tar.gz
>>> 正在登录ftp服务器并上传文件...
>>> 上传成功！
>>> 备份文件: /tmp/www.dzt.tar.gz 已上传至 FTP 服务器 /srv/ftp/user2/www.dzt.tar.gz
=== 脚本执行完毕 ===
```
ftp登录一下user2查看`/srv/ftp/user2`目录下，找到打包好的文件，成功。

## 总结

1. **用户组与目录权限**  
   - 虽然目录的权限看似允许写入，但如果目录所属的用户组不正确，即使在同一组的用户也无法写入。  
   - 一定要确认 `chown` 和 `chgrp` 都设置正确，并配合 `chmod` 给予所需权限。

2. **FTP 上传路径与登录根目录**  
   - FTP 用户可能被 `chroot` 限制到某个目录，脚本中使用的相对路径可能和预期不一致。  
   - 最稳妥的方法是使用绝对路径，例如 `cd /srv/ftp/user2`，确保上传到正确目录。

3. **脚本与 FTP 命令分离**  
   - 不要在 FTP 命令脚本里写 Bash 条件判断。FTP 脚本只负责 FTP 指令，上传成功与否的判断应该在 Bash 外层完成。  
   - 可以通过 `ftp -n < script` 捕获输出，然后用 `grep` 或 `$?` 判断远程文件是否存在。

4. **调试方法**  
   - 上传前后用 `ls -la` 查看目录权限和文件情况。  
   - 手动 FTP 登录测试，验证目录可访问性和写入性，比直接依赖脚本报错更可靠。
