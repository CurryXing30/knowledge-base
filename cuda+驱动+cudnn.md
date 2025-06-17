## 🧽 1. 清理已有 CUDA / NVIDIA 驱动（**谨慎操作**）

### 查看现有安装（状态审查）：

```
dpkg -l | grep -i nvidia
dpkg -l | grep -i cuda
```

### 模拟卸载 NVIDIA 驱动和 CUDA 驱动：

```
sudo apt-get remove --purge 'nvidia*' 'cuda-drivers*' --simulate
sudo apt-get autoremove --simulate
```

### 确认无误后执行清理：

```
bash复制编辑sudo apt-get remove --purge 'nvidia*' 'cuda*' 'cuda-drivers*'
sudo apt-get autoremove -y    （-y要谨慎 一定要看前边模拟删除的东西是不是都是没用的 避免删除系统关键库）
sudo apt-get autoclean
sudo rm -rf /usr/local/cuda*                  （这个是必须的 清理干净）
sudo rm -rf /etc/apt/sources.list.d/cuda*		（这个可有可无 需要看下指定文件是什么避免误删）
sudo rm -rf /var/cuda-repo-*  					（这个可有可无 需要看下指定文件是什么避免误删）
```

### 检查 NVIDIA 驱动、CUDA是否还残留

```
dpkg -l | grep -i nvidia
dpkg -l | grep -i cuda
```

## 🖥️ 2. 桌面恢复（防止显卡卸载影响 GUI）可选项 桌面丢了再用

桌面丢失时恢复 X 环境（已确认有效）：

```
sudo apt install libx11-6 libgl1 libegl1-mesa -y
```

可选恢复完整桌面：

```
sudo apt install ubuntu-desktop -y
```

## 🧱3. 内核环境干净（看下内核环境）

```
sudo apt install linux-headers-$(uname -r) build-essential dkms 
```

## 🚫 4. 禁用 Nouveau 驱动（防冲突）

创建禁用文件：（如果已经存在 可以先cat下 看下状态）

```
sudo nano /etc/modprobe.d/blacklist-nouveau.conf（不一定用）
```

内容如下：

```
cat /etc/modprobe.d/blacklist-nouveau.conf （必用）
# 应包含以下内容
# blacklist nouveau
# options nouveau modeset=0
```

更新并重启：

```
sudo update-initramfs -u
sudo reboot
```

确认 Nouveau 未加载：

```
lsmod | grep nouveau  # 若无输出则表示未加载
```

## 📌5.**驱动用 Ubuntu 官方推荐**

安装推荐驱动：

```
sudo ubuntu-drivers autoinstall （安装最新的一般没啥问题 新版本会兼容旧版本）或者sudo ubuntu-drivers devices      （查看推荐驱动挑选版本）
```

然后：

```
sudo reboot
```

重启之后再运行：

```
nvidia-smi
```

如果能看到 NVIDIA 驱动界面即成功。

## 🔧 6.安装 CUDA 11.8 Toolkit（不装 520 驱动）

**确保不要安装 `cuda` 包，只安装 `cuda-toolkit-11-8`，否则会装回 520 驱动。**

```
cd ~/lkx/downloads
sudo dpkg -i cuda-repo-ubuntu2204-11-8-local_11.8.0-520.61.05-1_amd64.deb
sudo cp /var/cuda-repo-ubuntu2204-11-8-local/cuda-*-keyring.gpg /usr/share/keyrings/
sudo apt update
```

然后只装 toolkit：

```
sudo apt install cuda-toolkit-11-8
```

------

完成后验证：

```
nvcc -V            # 检查 CUDA 编译器
nvidia-smi         # 检查驱动工作
```

### ⚠️ 注意

不要运行：

```
sudo apt install cuda
```

否则会默认安装旧的驱动（520），与当前系统不兼容。

## 🧭 7.cudnn安装

```
cd ~/lkx/downloads
tar -xvf cudnn-linux-x86_64-8.9.5.30_cuda11-archive.tar.xz
```

这会解压出一个目录：

```
cudnn-linux-x86_64-8.9.5.30_cuda11-archive/
```

------

##### 拷贝文件到 CUDA 目录

进入该目录：

```
cd cudnn-linux-x86_64-8.9.5.30_cuda11-archive
```

执行如下命令将头文件与库文件拷贝到 CUDA：

```
sudo cp include/* /usr/local/cuda-11.8/include/
sudo cp lib/* /usr/local/cuda-11.8/lib64/
```

然后更新动态链接库缓存：

```
sudo ldconfig
```

------

##### 验证 cuDNN 安装是否成功

你可以使用如下命令验证：

```
ls /usr/local/cuda-11.8/include/cudnn*.h
ls /usr/local/cuda-11.8/lib64/libcudnn*
```

##### 用 PyTorch 检查 cuDNN 是否可用

Python 中运行：

```
import torch
print("CUDA 是否可用:", torch.cuda.is_available())
print("cuDNN 是否可用:", torch.backends.cudnn.is_available())
print("cuDNN 版本:", torch.backends.cudnn.version())
```

**预期结果：**

```
CUDA 是否可用: True
cuDNN 是否可用: True
cuDNN 版本: 8905
```

## ⚙️ 8. 多版本 CUDA 管理（可选）

查看软链接：

```
ls -l /usr/local/ | grep cuda
```

切换版本：

```
sudo update-alternatives --install /usr/local/cuda cuda /usr/local/cuda-11.8 110
sudo update-alternatives --config cuda
```

------

## 🔧 9. 修复 cuDNN `.so` 符号链接（不一定会遇到）

### 创建自动修复脚本：

```
nano fix_cudnn_symlinks.sh
```

内容如下：

```
#!/bin/bash
set -e
cd /usr/local/cuda-11.8/lib64

for f in libcudnn*so.8.9.5; do
    base=$(echo $f | sed 's/\.so\.8\.9\.5//')
    sudo ln -sf $f "${base}.so.8"
    sudo ln -sf "${base}.so.8" "${base}.so"
done

echo "✅ 已修复 cuDNN 符号链接。"
```

执行脚本：

```
chmod +x fix_cudnn_symlinks.sh
./fix_cudnn_symlinks.sh
```

刷新动态库：

```
sudo ldconfig
```