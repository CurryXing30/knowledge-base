### 1. 安装依赖

1. 安装 WinFsp：https://winfsp.dev/rel/
2. 安装 SSHFS-Win：https://github.com/billziss-gh/sshfs-win/releases

### 2.使用 `net use`（带密码）方式：

##### 新建个bat

```
@echo off
REM 保存 SSH 密码（只需第一次执行）
cmdkey /add:sshfs.r /user:ubuntu /pass:=NWPU=2022531tc

REM 卸载旧挂载（防止冲突）
net use Z: /delete /yes

REM 挂载远程主目录
net use Z: \\sshfs.r\ubuntu@10.216.151.167!6000

exit
```

### 3.创建软连接（快捷访问）

可以在 Windows 中为挂载目录创建软链接，例如：（cmd中）

```
mklink /D D:\Linux_lax Z:\mnt\sda\ubuntu\lax
```