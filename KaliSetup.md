## 初始化

### 修改源

```shell
sudo nano /etc/apt/sources.list
```

从下面几个中选一个添加进去

```
# 官方源
deb http://http.kali.org/kali kali-rolling main non-free contrib
deb-src http://http.kali.org/kali kali-rolling main non-free contrib
# 中科大源
deb http://mirrors.ustc.edu.cn/kali kali-rolling main non-free contrib
deb-src http://mirrors.ustc.edu.cn/kali kali-rolling main non-free contrib
# 阿里云源
deb http://mirrors.aliyun.com/kali kali-rolling main non-free contrib
deb-src http://mirrors.aliyun.com/kali kali-rolling main non-free contrib
# 清华大学源
deb http://mirrors.tuna.tsinghua.edu.cn/kali kali-rolling main contrib non-free
deb-src https://mirrors.tuna.tsinghua.edu.cn/kali kali-rolling main contrib non-free
# 浙大源
deb http://mirrors.zju.edu.cn/kali kali-rolling main contrib non-free
deb-src http://mirrors.zju.edu.cn/kali kali-rolling main contrib non-free
```

对于 Ubuntu 使用下面这些源

```
# 阿里云
deb http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
# 源码
deb-src http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse

# 清华
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic main restricted universe multiverse
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-security main restricted universe multiverse
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-updates main restricted universe multiverse
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
# 源码
deb-src http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic main restricted universe multiverse
deb-src http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-security main restricted universe multiverse
deb-src http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-updates main restricted universe multiverse
deb-src http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
```

  保存后更新

```shell
sudo apt update  
sudo apt upgrade 
sudo apt clean 
```

### 中文输入法

```shell
sudo apt install fcitx5 fcitx5-chinese-addons fcitx5-skin-nord
```

修改环境变量 `/etc/environment`，添加如下内容：

```properties
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
export XMODIFIERS=@im=fcitx
export INPUT_METHOD=fcitx
export SDL_IM_MODULE=fcitx
```

使用 im-config 工具可以配置首选输入法，在任意命令行输入：

```bash
im-config
```

根据弹出窗口的提示，将首选输入法设置为 Fcitx 5 即可。

记得将 fcitx5 添加到自启，然后注销或者重启即可。

### 修改语言为中文

```shell
sudo dpkg-reconfigure locales
```

修改语言

修改 [*] en_US.UTF-8 UTF-8 至 [ ] en_US.UTF-8 UTF-8

修改 [ ] zh_CN.UTF-8 UTF-8 至 [*] zh_CN.UTF-8 UTF-8

然后回车确定，接着设置默认语言 `zh_CN.UTF-8` 回车确定后重启。

## 软件

### 网络下载

#### clash

[github](https://github.com/Dreamacro/clash/releases) 下载，剩下的参考 ManjaroSetup
