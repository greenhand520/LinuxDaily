
## 	安装
### 与 Windows 并存

在 Windows 中压缩出需要的空间，安装界面中手动分区。第一个分区作为EFI分区，最小300M，挂载点为 `/boot/efi`，fat32格式；第二个分区为根分区，挂载点为`/`，ext4 或者 btrfs 格式；第三个分区为 swap 分区，格式为 linuxswap，这个分区不是必须的。当然想单独分个区挂载 `/home` 也是可以的。

不建议手动调整某 Windows 分区来作为根分区安装系统，这个调节不能精细控制分区大小，对于强迫症很来说挺难受的。

安装过程中，开始释放文件后就可以断开网络了，否则大概卡在93%。试了修改源，还是不行，故还是断网直接。

### 安装在移动硬盘中
借助 [Ventory](https://www.ventoy.net/cn/) 可实现启动移动硬盘中的系统。这里在制作 Ventoy 启动盘时，需要在 ventoy[分区2](https://www.ventoy.net/cn/doc_disk_layout_gpt.html)之后预留一段空间来安装系统。将 ISO 文件复制到 ventoy[分区1](https://www.ventoy.net/cn/doc_disk_layout_gpt.html)中启动镜像，安装系统选择预留的一段空间，手动分区。我全部用作根分区，没有留交换分区也可以正常运行。根分区文件系统考虑到移动设备意外断电的可能性较大，选择 `ext4` 文件系统，不过我选择 `btrfs` 文件系统。分区结束后写入硬盘会提示缺少 EFI 引导分区，忽略即可。

回到 ventoy 分区1中，新建[ventoy文件夹](https://www.ventoy.net/cn/plugin_entry.html)，使用 ventoy 的[自定义菜单插件](https://www.ventoy.net/cn/plugin_grubmenu.html)来引导刚才装的系统。

参考已有的 grub 文件和 ventoy 文档，`ventoy_grub.cfg` 内容如下：

```shell
menuentry 'Manjaro Linux (on WDB)' --class manjaro --class gnu-linux --class gnu --class os {
	insmod gzio
	insmod part_gpt
	insmod ext2
	search --no-floppy --fs-uuid --set=root e9adc5fe-e98e-4c15-9747-360d57a80418
	linux	/boot/vmlinuz-5.15-x86_64 root=UUID=e9adc5fe-e98e-4c15-9747-360d57a80418 rw  udev.log_priority=3 
	initrd	/boot/intel-ucode.img /boot/initramfs-5.15-x86_64.img
}

menuentry '<-- Return to previous menu [Esc]' --class=vtoyret VTOY_RET {
    echo 'Return ...'
}
```

**注：**

1、`menuentry` 是在ventoy界面按`（Fn+）F6`后显示的菜单。

2、`set root` 设置根分区在移动存储设备的第三个分区上。

3、`/boot/vmlinuz-5.15-x86_64` 和 `/boot/initramfs-5.15-x86_64.img` 等文件需要查看被引导系统的 `/boot` 文件夹做相应修改。

4、`root=UUID=0313b3f1-f838-431e-b8bf-3b9c6ee7c410` 中的UUID需要查看`/etc/fstab` 文件，找到其中 `<type>` 为 `/` 的UUID，或者安装好后别重启，启动 GParted查看。

5、如果安装在 **btrfs** 文件系统中，则 `linux` 和 `initrd` 后的路径前面要添加 `/@`，如：

```
linux	/@/boot/vmlinuz-5.15-x86_64
```

6、如果不需要高级选项`submenu`里面的内容可删除。

7、Windows中可使用DiskGenius来将ext4文件格式中需要查看的文件复制出来。

相关：[用grub2制作多重引导的WinPE&Linux启动U盘](https://oscarcx.com/tech/usb-boot-winpe-and-linux-via-grub2.html#more)

### 系统迁移

使用 [partclone](https://partclone.org/) 来完整克隆分区数据。预先使用 GParted 新建 EFI 分区和根分区，在 Live 系统或另一个 Linux 系统中，安装 partclone，根据[文档](https://partclone.org/usage/partclone.php)，使用下面命令进行克隆根分区。

```shell
yay -S partclone
sudo partclone.btrfs -s /dev/nvme0n1p2 -o /dev/sdb2 -b
```

**注**：

1. 目标分区要比被源分区完整空间大，如果空间没有源分区空间大，可用 GParted 缩小源分区，克隆结束后再恢复；
2. 克隆分区会把 UUID 一起克隆，故克隆前删除源分区的快照；
3. 如果源硬盘上有被激活的 linux-swap 分区，在目标硬盘上记得创建对应的分区；
4. 克隆后，若要进行挂载，建议拔掉源分区硬盘，插上新分区硬盘，否则出现2个 UUID 一致的分区导致目标分区无法被挂载。

### 引导修复

如果EFI分区损坏

1、可以在 Live 系统或者另一个 Linux 中通过 GParted 新建一个 EFI 分区，并通过下列操作恢复：

查看要引导系统的分区和EFI分区的编号

```shell
fdisk -l
```

找到你的系统所在分区和EFI分区并挂载引导系统的分区

```shell
sudo mount /dev/nvme0n1p2 /mnt/manjaro
```

挂载 EFI 分区到系统分区的 `/boot/efi`目录

```shell
sudo mount /dev/nvme0n1p1 /mnt/manjaro/boot/efi
```

chroot 到硬盘系统分区

```shell
sudo chroot /mnt/manjaro
```

安装 grub

```shell
grub-install --target=x86_64-efi /dev/nvme0n1p1  #target 默认是x86_64-efi
grub-mkconfig -o /boot/grub/grub.cfg
update-grub
```

参考：http://iamcheyan.com/app/blog/index.php/archives/873/

**注：**

1. 针对使用 btrfs 文件系统的 Linux , 系统分区需要添加 @ ，如 `/mnt/manjaro/@`；
2. `/dev/nvme0n1p1` 为系统的 EFI 分区，`/dev/nvme0n1p2` 为系统根分区。

2、也可以通过自定义 Ventoy 的 grub 菜单，实现引导无法启动的 Linux 启动，再进入系统后做上面相似操作。

`ventoy_grub.cfg` 添加菜单举例，btrfs 文件系统如下：

```shell
menuentry 'Manjaro Linux' --class manjaro --class gnu-linux --class gnu --class os {
	insmod gzio
	insmod part_gpt
	insmod btrfs
	search --no-floppy --fs-uuid --set=root ad0e1db0-84e2-4267-8c4f-300dd4415c44
	linux	/@/boot/vmlinuz-5.15-x86_64 root=UUID=ad0e1db0-84e2-4267-8c4f-300dd4415c44 rw rootflags=subvol=@ 
	initrd	/@/boot/intel-ucode.img /@/boot/initramfs-5.15-x86_64.img
}
```

ext4 文件系统如下:

```shell
menuentry 'Manjaro Linux' --class manjaro --class gnu-linux --class gnu --class os {
	insmod gzio
	insmod part_gpt
	insmod ext2
	search --no-floppy --fs-uuid --set=root e9adc5fe-e98e-4c15-9747-360d57a80418
	linux	/boot/vmlinuz-5.15-x86_64 root=UUID=e9adc5fe-e98e-4c15-9747-360d57a80418 rw  udev.log_priority=3 
	initrd	/boot/intel-ucode.img /boot/initramfs-5.15-x86_64.img
}
```

**注：**

1、`rootflags=subvol=@` 是 btrfs 文件系统需要添加的，ext4 的话不需要。

2、UUID 为根分区 UUID。

进入系统后，后上面类似的操作。

查看要引导系统的分区和EFI分区的编号

```shell
fdisk -l
```

如果没有 EFI 分区就用 GParted 创建一个，确保 EFI 分区挂载到`/boot/efi`，没有的话就命令挂载下

```shell
sudo mount /dev/nvme0n1p1 /boot/efi
```

安装grub

```shell
sudo grub-install --target=x86_64-efi /dev/nvme0n1p1
sudo grub-mkconfig -o /boot/grub/grub.cfg
sudo update-grub
```

3、针对使用 partclone 迁移系统后的引导修复

拔掉原来系统所在硬盘，在 Live 系统或者另一个 Linux 中进行如下操作

挂载目标硬盘

```shell
sudo mount /dev/nvme0n1p2 /mnt/manjaro
```

记录目标硬盘各分区的 UUID

```shell
lsblk -f
```

修改新系统的 fstab 文件，将里面各分区 UUID 改为目标硬盘对应分区的 UUID

```shell
sudo nano /etc/fstab
```

补全挂载，并进行修复

```shell
sudo mount /dev/nvme0n1p2 /mnt/manjaro
# sudo mount /dev/nvme0n1p1 /mnt/manjaro/@/boot/efi

sudo mount --rbind /dev /mnt/manjaro/@/dev
sudo mount --rbind /dev/pts /mnt/manjaro/@/dev/pts
sudo mount --rbind /proc /mnt/manjaro/@/proc
sudo mount --rbind /sys /mnt/manjaro/@/sys
# sudo mount --rbind /run /mnt/manjaro/@/run

sudo chroot /mnt/manjaro/@

mount /dev/nvme0n1p2 /
mount /dev/nvme0n1p1 /boot/efi

grub-install --target=x86_64-efi /dev/nvme0n1p1
grub-mkconfig -o /boot/grub/grub.cfg
update-grub
```

**注：**

1.  `/dev/nvme0n1p1 ` 为新系统的 EFI 分区，`/dev/nvme0n1p2` 为新系统的根分区。
2. 插在硬盘盒里可能在启动的时候出现无法找到分区的问题，需要查到电脑上。

### 引导 Windows

grub 无法直接引导 Windows 启动，需要 Windows 引导分区 EFI 存在，grub添加如下内容，其中 `DD5A-104A` 为 EFI 分区UUID。

```shell
menuentry 'Windows Boot Manager (on /dev/sdc3)' --class windows --class os $menuentry_id_option 'osprober-efi-DD5A-104A' {
	insmod part_gpt
	insmod fat
	set root='$vtoydev,gpt3'
	if [ x$feature_platform_search_hint = xy ]; then
	  search --no-floppy --fs-uuid --set=root --hint-bios=$vtoydev,gpt3 --hint-efi=$vtoydev,gpt3 --hint-baremetal=ahci2,gpt3  DD5A-104A
	else
	  search --no-floppy --fs-uuid --set=root DD5A-104A
	fi
	chainloader /efi/Microsoft/Boot/bootmgfw.efi
}
```

我的 `ventoy_grub.cfg` 内容，仅供参考。

```shell
menuentry 'Manjaro Linux (on intel600p)' --class manjaro --class gnu-linux --class gnu --class os {
	insmod gzio
	insmod part_gpt
	insmod btrfs
	search --no-floppy --fs-uuid --set=root bbc4893d-3d4d-428f-a18f-b7a1c46fe7f1
	linux	/@/boot/vmlinuz-5.15-x86_64 root=UUID=bbc4893d-3d4d-428f-a18f-b7a1c46fe7f1 rw rootflags=subvol=@ 
	initrd	/@/boot/intel-ucode.img /@/boot/initramfs-5.15-x86_64.img
}

menuentry 'Manjaro To Go (on WDB)' --class manjaro --class gnu-linux --class gnu --class os {
	insmod gzio
	insmod part_gpt
	insmod ext2
	search --no-floppy --fs-uuid --set=root e9adc5fe-e98e-4c15-9747-360d57a80418
	linux	/boot/vmlinuz-5.15-x86_64 root=UUID=e9adc5fe-e98e-4c15-9747-360d57a80418 rw udev.log_priority=3 
	initrd	/boot/intel-ucode.img /boot/initramfs-5.15-x86_64.img
}

menuentry 'Windows11 To Go (on WDB)' --class windows --class os $menuentry_id_option 'osprober-efi-EB19-40F1' {
	insmod part_gpt
	insmod fat
	set root=($vtoydev,gpt3)
	if [ x$feature_platform_search_hint = xy ]; then
	  search --no-floppy --fs-uuid --set=root --hint-bios=$vtoydev,gpt3 --hint-efi=$vtoydev,gpt3 --hint-baremetal=ahci2,gpt3  EB19-40F1
	else
	  search --no-floppy --fs-uuid --set=root EB19-40F1
	fi
	chainloader /efi/Microsoft/Boot/bootmgfw.efi
}

menuentry '<-- Return to previous menu [Esc]' --class=vtoyret VTOY_RET {
    echo 'Return ...'
}
```

### 重新安装内核

内核损坏，无法启动系统，需在另一个 Linux 系统中重新安装内核，步骤和修复引导相似。

```shell
sudo mount /dev/nvme0n1p2 /mnt/manjaro
# sudo mount /dev/nvme0n1p1 /mnt/manjaro/@/boot/efi

sudo mount --rbind /dev /mnt/manjaro/@/dev
sudo mount --rbind /dev/pts /mnt/manjaro/@/dev/pts
sudo mount --rbind /proc /mnt/manjaro/@/proc
sudo mount --rbind /sys /mnt/manjaro/@/sys
# sudo mount --rbind /run /mnt/manjaro/@/run

sudo chroot /mnt/manjaro/@

mount /dev/nvme0n1p2 /
mount /dev/nvme0n1p1 /boot/efi

# 多了个安装内核的步骤
pacman -S linux

# 更新引导
grub-install --target=x86_64-efi /dev/nvme0n1p1
grub-mkconfig -o /boot/grub/grub.cfg
update-grub
```

**注**：

1. 安装内核时可能会有下面提示

   ```
   error: could not determine cachedir mount point /var/cache/pacman/pkg
   error: failed to commit transaction (not enough free disk space)
   ```

   编辑 `/etc/pacman.conf` 文件，注释掉大概39行的 `CheckSpace` 即可，后面可再恢复。

2. 更多内核安装操作，参考：https://zhuanlan.zhihu.com/p/373372136

### 修改 SWAP

修改后需要将 `etc/fstab`和 `/etc/default/grub` 这2个文件中原 交换分区的 UUID 换成新的。要不然启动会报

`ERROR:resume:hibernation device'UUID=xxxx' not found`这样的错误。

```properties
# etc/fstab
UUID=3f48c68c-4717-438e-aff6-26ac8f75fcde swap           swap    defaults,noatime 0 2

# /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="quiet resume=UUID=3f48c68c-4717-438e-aff6-26ac8f75fcde"
```

将上面2和文件中出现的旧 swap 分区 UUID 替换成新的后，执行

```shell
sudo update-grub
```



## 初始化

### 包管理

pacman 更换源

```shell
sudo pacman-mirrors -i -c China -m rank
sudo pacman -Syy
```

安装 yay

```shell
sudo pacman -S yay
```

从 github 克隆出错 `curl 92 HTTP/2 stream 0 was not closed cleanly`

```shell
git config --global http.version HTTP/1.1
```

### nano 文本编辑器

默认好像已经安装了

```shell
sudo pacman -S nano nano-syntax-highlighting
```

添加默认显示行号

```shell
nano ~/.nanorc
# 添加下面内容
set linenumbers
# 备份root用户的配置文件并链接到用户的nano配置文件
sudo mv /root/.nanorc /root/.nanorc.bak                                                      
sudo ln -s ~/.nanorc /root/.nanorc
```

**使用**

**1、光标控制**

移动光标：使用用方向键移动。

选择文字：按住鼠标左键拖到。

**2、复制、剪贴和粘贴**

复制一整行：Alt+6

剪贴一整行：Ctrl+K

粘贴：Ctrl+U

如果需要复制／剪贴多行或者一行中的一部分，先将光标移动到需要复制／剪贴的文本的开头，按 Ctrl+6（或者Alt+A）做标记，然后移动光标到 待复制／剪贴的文本末尾。这时选定的文本会反白，用 Alt+6 来复制，Ctrl+K 来剪贴。若在选择文本过程中要取消，只需要再按一次 Ctrl+6。

**3、搜索**

按 Ctrl+W，然后输入你要搜索的关键字，回车确定。这将会定位到第一个匹配的文本，接着可以用 Alt+W 来定位到下一个匹配的文本。

**4、翻页**

用 Ctrl+Y 到上一页，Ctrl+V 到下一页

**5、退出**

按 Ctrl+X，如果你修改了文件，下面会询问你是否需要保存修改。输入 Y 确认保存，输入 N 不保存，按 Ctrl+C 取消返回。

如果输入了 Y，下一步会让你输入想要保存的文件名。如果不需要修改文件名直接回车就行；若想要保存成别的名字（也就是另存为）则输入新名称然后确 定。这个时候也可用 Ctrl+C 来取消返回。

### 终端

#### 透明终端

会替换原来的，谨慎考虑

```shell
yay -S gnome-terminal-transparency
```

#### 安装  oh-my-zsh

```shell
git clone https://github.com/ohmyzsh/ohmyzsh.git ~/.oh-my-zsh 
cp ~/.zshrc ~/.zshrc.bak
cp ~/.oh-my-zsh/templates/zshrc.zsh-template ~/.zshrc
```

默认主题是 `robbyrussell`，如果觉得主题太多你可以选择使用随机模式，来由系统随机选择

```properties
ZSH_THEME="random"
```

#### 插件

修改 `.zshrc` 配置文件
将 plugins 修改为如下（将下载的插件名称添加进去）：

```properties
plugins=(git zsh-syntax-highlighting zsh-autosuggestions)
```

使修改生效

```shell
source ~/.zshrc
```

1、sudo

默认就装好的，需要自己开启。偶尔输入某个命令，提示没有权限，需要加sudo，这个时候按两下 ESC，就会在命令行头部加上 sudo

2、z
默认就装好的，需要自己开启。`cd` 命令进入 `~/user/github/Youthink` 文件夹，下一次再想进入 `Yourhink` 文件夹的时候,直接 `z youthink` 即可，或者只输入 `youthink` 的一部分 `youth` 都行。还有一个`autojump`的插件和`z`功能差不多，`autojump`需要单独装，如果 z 插件历史记录太多，并且有一些不是自己想要的，可以删除 `z -x` 不要的路径

3、zsh-syntax-highlighting

作用 平常用的ls、cd 等命令输入正确会绿色高亮显示，输入错误会显示其他的颜色。

```
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ~/.oh-my-zsh/plugins/zsh-syntax-highlighting
```

4、git

默认已开启。可以使用各种 `git` 命令缩写。比如：

```
git add --all ===> gaa
git commit -m ===> gcmsg
```

查看所有 `git` 命令缩写

```shell
cat ~/.oh-my-zsh/plugins/git/git.plugin.zsh
```

或者筛选对应的命令，如和 `config` 有关的命令

```shell
alias | grep config
```

5、zsh-autosuggestions

```shell
git clone https://github.com/zsh-users/zsh-autosuggestions ~/.oh-my-zsh/plugins/zsh-autosuggestions
```

如果感觉 `→` 补全不方便，还可以自定义补全的快捷键，比如我设置的逗号补全

```
bindkey ',' autosuggest-accept
```

在 `.zshrc` 文件添加这句话即可。

6、git-open

在终端里打开当前项目的远程仓库地址，每次改完本地代码，当你想用浏览器访问远程仓库的时候，就很方便。

支持打开的远程仓库

- github.com
- gist.github.com
- gitlab.com
- 自定义域名的 GitLab
- bitbucket.org
- Atlassian Bitbucket Server (formerly Atlassian Stash)
- Visual Studio Team Services
- Team Foundation Server (on-premises)

```shell
git clone https://github.com/paulirish/git-open.git ~/.oh-my-zsh/plugins/git-open
```

7、alias

如名。默认已经安装了，可在 `~/.zshrc` 中添加如下简化操作

```
alias cp="cp -i"
alias yi="yay -S"
alias yr="yay -R"
alias ys="yay -Ss"
alias yu="yay -Syu"
```

#### 更新 oh-my-zsh

修改 自动升级本身没有提示你

```properties
disable_update_prompt=true
```

禁用自动升级, 修改 ~/.zshrc

```properties
disable_auto_update=true
```

手动更新，运行

```shell
upgrade_oh_my_zsh
```

我的 `.zshrc` 文件部分内容

```properties
USE_POWERLINE="true"

export ZSH=$HOME/.oh-my-zsh

ZSH_THEME="random"

plugins=(git zsh-syntax-highlighting zsh-autosuggestions z sudo)

source $ZSH/oh-my-zsh.sh

# bindkey ',' autosuggest-accept

alias cp="cp -i"
alias yi="yay -S"
alias yr="yay -R"
alias ys="yay -Ss"
alias yu="yay -Syu"
alias ws="whereis"
```

#### 终端走代理

```shell
yay -S proxychains-ng
sudo nano /etc/proxychains.conf
# 注释掉60行的proxy_dns
# 最后的内容修改如下，其中9981为clash设置的端口
[ProxyList]
# add proxy here ...
# meanwile
# defaults set to "tor"
socks4 	127.0.0.1 9981
socks5  127.0.0.1 9981
http  127.0.0.1 9981
```

使用

`proxychains`后跟要走代理的命令，如：

```shell
proxychains yay -S xxxx
```

### 输入法 - fcitx5

参考：https://wiki.archlinux.org/title/Fcitx5

fcitx5 全家桶（fcitx5，fcitx5-qt，fcitx5-gtk，fcitx5-configtool）。

```shell
sudo pacman -S --noconfirm fcitx5-im
```

然后安装中文输入法和 zhwiki 词库。另字库可在设置中添加搜狗输入法的词库文件。

```shell
sudo pacman -S --noconfirm fcitx5-chinese-addons
# 中文维基百科词库
yay -S --noconfirm fcitx5-pinyin-zhwiki
# 汉语成语词库
fcitx5-pinyin-chinese-idiom
# 萌娘百科词库
fcitx5-pinyin-moegirl
```

激活输入法就是 `Ctrl` + `Space`，输入法切换就是熟悉的 `Ctrl` + `Shift`，在中文输入法下可以用 `Left Shift` 临时切到英文。使用`Ctrl + .`切换全角与半角。

修改环境变量 `/etc/environment`，添加如下内容：

```properties
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
export XMODIFIERS=@im=fcitx
export INPUT_METHOD=fcitx
export SDL_IM_MODULE=fcitx
```

记得将 fcitx5 添加到自启，然后注销或者重启即可。

```shell
cp /usr/share/applications/org.fcitx.Fcitx5.desktop ~/.config/autostart
chmod +x ~/.config/autostart/org.fcitx.Fcitx5.desktop
```

#### 皮肤

几个觉得不错的黑色主题

```shell
sudo pacman -S --noconfirm fcitx5-nord
yay -S fcitx5-skin-adwaita-dark
yay -S fcitx5-skin-fluentdark-git
```

参考：https://wiki.archlinux.org/title/Fcitx5

#### 特殊符号及 emoji

```shelL
# 单独的软件输入emoji
yay -S gnome-characters
# fcitx扩展输入emoji，安装后使用快捷键 ctl + alt + . 唤出
yay -S fcitx5-im-emoji-picker-git
```

可在设置中绑定快捷键快速调出

如果需要在 fcitx5 中打出特殊符号，将下面内容导入 fcitx5 的快速输入中即可。根据情况自行删减。快速输入的批量编辑中粘贴或者保存到 csv 文件中导入。

```properties
0	⓪
1	①
2	②
3	③
4	④
5	⑤
6	⑥
7	⑦
8	⑧
9	⑨
10	⑩
11	⑪
12	⑫
13	⑬
14	⑭
15	⑮
16	⑯
17	⑰
18	⑱
19	⑲
20	⑳
21	㉑
22	㉒
23	㉓
23	㉔
25	㉕
26	㉖
27	㉗
28	㉘
29	㉙
30	㉚
z	←
s	↑
y	→
x	↓
zy	↔
sx	↕
zs	↖
ys	↗
yx	↘
zx	↙
yf1	𝅗𝅥
yf	𝅘𝅥
yf2	𝅘𝅥𝅮
yf3	𝅘𝅥𝅯
gh	√
3gh	∛
4gh	∜
zby	∝
wqd	∞
j	∠
pyx	∥
jj	∩
bj	∪
jf	∫
2jf	∬
3jf	∭
sy	∴
yw	∵
xsy	∽
qdy	≌
czy	⊥
ydy	≈
!=	≠
bhy	⊂
<=	≤
>=	≥
0cf	⁰
1cf	¹
2cf	²
3cf	³
4cf	⁴
5cf	⁵
6cf	⁶
7cf	⁷
8cf	⁸
9cf	⁹
```

### clash

```shell
yay -S clash
```

#### 1、配置

启动 `clash` 生成默认配置文件 `config.yaml` 和 `Country.mmdb`，保存位置为`~/.config/clash/`

退出 clash 后下载所需的配置文件 `config.yaml` 和 `Country.mmdb`

```shell
sudo wget -O config.yaml ${你的订阅链接}
wget -O Country.mmdb https://www.sub-speeder.com/client-download/Country.mmdb
```

用下载的配置文件 `config.yaml` 和 `Country.mmdb替换默认的配置文件`

修改部分配置如下：

```yaml
port: 7980
socks-port: 7981
allow-lan: true
```

#### 2、修改系统代理

```shell
nano ~/.profile #在最后写入下面内容
```

```shell
export http_proxy="http://127.0.0.1:7890" 
export https_proxy="http://127.0.0.1:7890" 
export http_proxy="socks5://127.0.0.1:7891" 
export https_proxy="socks5://127.0.0.1:7891"
no_proxy="localhost,127.0.0.1,localaddress,.localdomain.com"
```

然后刷新

```shell
source ~/.profile
```

或者在设置中修改。

#### 3、控制 UI

在`~/.config/clash/config.yaml`中设置好ui地址和访问密码，密码也可以不设置

```yaml
external-controller: 127.0.0.1:37523
```

访问路径为：外部控制地址/ui，填入ip、端口、密码即可访问，密码没设旧留空。如：

```
https://clash.razord.top/#/proxies?host=127.0.0.1&port=37523&secret=${PWD}
# 或者进入页面输入密码
https://clash.razord.top/#/proxies?host=127.0.0.1&port=37523
```

#### 4、设置开机自启动

clash 服务

```shell
sudo nano /etc/systemd/system/clash.service
```

添加内容如下

```properties
[Unit]
Description=Clash daemon
After=network.target

[Service]
Type=simple
Restart=always
User=root
ExecStart=/usr/bin/clash -d /home/$USER/.config/clash

[Install]
WantedBy=multi-user.target
```

启用并启动服务

```shell
sudo systemctl daemon-reload # 重新加载systemctl daemon
sudo systemctl enable clash
sudo systemctl start clash
```

相关命令

```shell
# 启动Clash
sudo systemctl start clash
# 重启Clash
sudo systemctl restart clash
# 查看Clash运行状态
sudo systemctl status clash
```

然后打开浏览器验证是否能访问谷歌

#### 5、图形界面

```shell
yay -S clash-for-windows-chinese	
# 或者安装这个，rust编写的
yay -S clash-verge-bin
```

设置如下：

1、主页，打开混合配置，添加配置并在系统设置中修改手动代理的端口**都为9981**

```properties
mixin: true 
mixed-port: 9981
```

2、配置中导入订阅的地址

3、设置中修改系统代理

> Https Proxy: 127.0.0.1 9981
> Http Proxy: 127.0.0.1 9981
> Socks Host: 127.0.0.1 9981
> Ignore Hosts: localhost, 127.0.0.0/8, ::1

4、安装 [Proxy Switche](https://extensions.gnome.org/extension/771/proxy-switcher/)，可在顶栏直接切换代理

### Edge 浏览器

```shell
yay -S microsoft-edge-dev-bin
yay -S microsoft-edge-stable-bin
yay -S microsoft-edge-beta-bin
```

设置中文显示

```shell
sudo nano /opt/microsoft/msedge/microsoft-edge
# 首行添加
 export LANGUAGE=ZH-CN.UTF-8
```

### gnome

#### gnome 扩展

在火狐浏览器中安装 [GNOME Shell integration](https://addons.mozilla.org/en-US/firefox/addon/gnome-shell-integration/?utm_source=addons.mozilla.org&utm_medium=referral&utm_content=search) 浏览器扩展来下载安装安装 Gnome Shell 扩展。

扩展推荐：

[ArcMenu](https://extensions.gnome.org/extension/3628/arcmenu/) 程序菜单

[AppIndicator and KStatusNotifierItem Support](https://extensions.gnome.org/extension/615/appindicator-support/) 应用托盘图标显示在顶栏，我遇到它导致内存泄漏问题

[Blur my Shell](https://extensions.gnome.org/extension/3193/blur-my-shell/) 界面模糊

[Custom Hot Corners - Extended](https://extensions.gnome.org/extension/4167/custom-hot-corners-extended/) 屏幕四角鼠标手势控制

[Dash to Dock](https://extensions.gnome.org/extension/307/dash-to-dock/) Dock

[ddterm](https://extensions.gnome.org/extension/3780/ddterm/) 下拉式终端，建议绑定快捷键

[Desktop Icons NG (DING)](https://extensions.gnome.org/extension/2087/desktop-icons-ng-ding/) 桌面右击菜单添加更过选项，可在桌面添加快捷方式

[GSConnect](https://extensions.gnome.org/extension/1319/gsconnect/) KDEConnect 在 gnome shell 上的实现，与 gnome 结合的非常好

重置GSConnect

```shell
gnome-extensions uninstall gsconnect@andyholmes.github.io
rm -rf ~/.local/share/gnome-shell/extensions/gsconnect@andyholmes.github.io
rm -rf ~/.cache/gsconnect
rm -rf ~/.config/gsconnect
dconf reset -f /org/gnome/shell/extensions/gsconnect/
```

[Lock Keys](https://extensions.gnome.org/extension/36/lock-keys/) 大小写及数字健启用提示

[lunar-calendar](https://extensions.gnome.org/extension/675/lunar-calendar/) 农历支持，需要先安装 `lunar-date` ，设置系统语言为英文后乱码，解决

```shell
cp /usr/share/locale/zh_CN/LC_MESSAGES/lunar-date.mo /usr/share/locale/en_US/LC_MESSAGES/lunar-date.mo
```

注销后恢复

[NoAnnoyance v2](https://extensions.gnome.org/extension/2182/noannoyance/) 阻止“窗口已就绪”提示，直接聚焦到将相应窗口

[Proxy Switche](https://extensions.gnome.org/extension/771/proxy-switcher/) 顶栏直接切换代理

[Sound Input & Output Device Chooser](https://extensions.gnome.org/extension/906/sound-output-device-chooser/) 见名知义

[Media Label and Controls (Mpris Label)](https://extensions.gnome.org/extension/4928/mpris-label/) 顶栏添加媒体控制组件 

[Unite](https://extensions.gnome.org/extension/1287/unite/) 对窗口顶部面板进行了一些布局调整，并删除了窗口装饰，我用来去除JB-IDE的顶栏的，但是最大最小化按钮会被移动到顶栏，所以我配合`Custom Hot Corners - Extended`用鼠标手势来实现最大最小化。

[Vitals](https://extensions.gnome.org/extension/1460/vitals/) 网速 CPU RAM 硬盘等使用指示

[RunCat](https://extensions.gnome.org/extension/2986/runcat/) 一个显示 CPU 占有率的小猫，占有率越高小猫跑的越快

[Reboot to UEFI](https://extensions.gnome.org/extension/5105/reboottouefi/) 重启进入 BIOS 界面

[Caffeine](https://extensions.gnome.org/extension/517/caffeine/) 禁用屏幕保护程序和自动挂起

[Input Method Panel](https://extensions.gnome.org/extension/261/kimpanel/) 在 wayland 环境下需要安装，输入法面板使用 KDE 的 kimpanel 协议

[Bluetooth Quick Connect](https://extensions.gnome.org/extension/1401/bluetooth-quick-connect/) 允许配对的蓝牙设备通过 GNOME 系统菜单连接和断开连接，显示电池状态等

[Media Label and Controls (Mpris Label)](https://extensions.gnome.org/extension/4928/mpris-label/) 在面板中显示一个标签，其中包含来自 MPRIS 兼容播放器的歌曲/标题/专辑/艺术家信息。还可以控制播放器，提高/降低其音量，自定义标签，以及更多。此扩展适用于 Spotify, Vlc, Rhythmbox, Firefox, Chromium 和(可能)任何 MPRIS 兼容播放器。

#### gnome 配置

dconf && gsettings

dconf：是一套基于键的配置系统, 十分高效, 相当于 Windows 下的注册表，图形化编辑工具 `dconf-editor`

gsettings：是 GNOME-DE 下的高级 API, 是命令行工具/前端, 用来简化对 dconf 的操作

gnome 的软件基本都可以用 gsettings 进行配置。

比如下面的触摸板配置

```shell
gsettings set org.gnome.desktop.peripherals.touchpad tap-to-click true
gsettings set org.gnome.desktop.peripherals.touchpad speed 0.57
gsettings set org.gnome.desktop.peripherals.touchpad disable-while-typing false
```

分别对应:

- 轻击模拟鼠标点击, 默认为false
- 调整触摸板速度, 默认为0
- 打字时禁用触摸板, 默认为true

#### 保存加载配置

导出当前的 dconf 数据到某个文件:

```bash
dconf dump / > dconf.settings
```

加载/导入某个  dconf 文件到当前系统:

```bash
cat dconf.settings | dconf load -f /
```

背景图像默认位置是 `/home/mdmbct/.config/background` 可以预先复制一个名为 background 不带扩展名的图片文件过去

## 设置

### 与Windows时间同步

Windows 与 Linux 看待硬件时间的方式不同。Windows 把电脑的硬件时钟（RTC）看成是本地时间，即 RTC = Local Time，Windows 会直接显示硬件时间；而 Linux 则是把电脑的硬件时钟看成 UTC 时间，即 RTC = UTC，那么 Linux 显示的时间就是硬件时间加上时区。

所以，大概有一个种思路。一是让 Windows 认为硬件时钟是 UTC 时间，二是让 Linux 认为硬件时钟是本地时间。

解决：

1、修改 Windows 硬件时钟为 UTC 时间

以管理员身份打开 「PowerShell」，输入以下命令：

```powershell
Reg add HKLM\SYSTEM\CurrentControlSet\Control\TimeZoneInformation /v RealTimeIsUniversal /t REG_DWORD /d 1
```

2、修改 Linux 硬件时钟为本地时间

```shell
 timedatectl set-local-rtc 1 --adjust-system-clock
 shwclock -w # 将系统时间写入硬件
```

> command not found: hwclock

```shell
sudo pacman -S util-linux
```

或者https://blog.csdn.net/qq_36737934/article/details/90233406

### 修改主目录分类文件夹名为英文

方法一

```shell
sudo pacman -S xdg-user-dirs-gtk
export LANG=en_US
xdg-user-dirs-gtk-update
```

然后会有个窗口提示语言更改，更新名称即可，此时home下的文件夹名已变为英文。
接着需要将语言改回中文，执行：

```shell
export LANG=zh_CN.UTF-8
reboot
```

重启电脑后如果提示语言更改，选择`保留旧的名称`即可。

旧文件夹中如果有文件就会保存，记得移动文件后删除。

方法二

编辑  `~/.config/user-dirs.dirs`

```bash
nano ~/.config/user-dirs.dirs 
```

将文件中相应部分修改为以下内容：

```bash
XDG_DESKTOP_DIR="$HOME/Desktop"
XDG_DOWNLOAD_DIR="$HOME/Downloads"
XDG_TEMPLATES_DIR="$HOME/Templates"
XDG_PUBLICSHARE_DIR="$HOME/Public"
XDG_DOCUMENTS_DIR="$HOME/Documents"
XDG_MUSIC_DIR="$HOME/Music"
XDG_PICTURES_DIR="$HOME/Pictures"
XDG_VIDEOS_DIR="$HOME/Videos"
```

手动修改 home 用户目录下的目录为以上相应目录名。

**方法三**

系统设置 - 区域和语言 - 语言 - 选择英语，注销即可。

### NVIDIA显卡驱动安装

Manjaro 自带 `mhwd` 安装驱动很方便。

打开`Manjaro-setting-manager-硬件设定`，点击`Auto Install Proprietary Driver`按钮会自动安装闭源驱动。

或者手动选择安装的驱动。

或者通过命令

```shell
sudo mhwd -a pci nonfree 0300 
```

自动安装闭源驱动。

### 双显卡切换-OptimusManager 

Optimus Manager Github：https://github.com/Askannz/optimus-manager

Optimus Manager qt Github： https://github.com/Shatur/optimus-manager-qt

参考：https://wiki.archlinux.org/title/Hybrid_graphics

https://wiki.archlinux.org/title/PRIME

https://wiki.archlinux.org/title/NVIDIA_Optimus

```shell
yay -S gdm-prime # for gnome、gdm，替换原来的gdm
yay -S optimus-manager
yay -S optimus-manager-qt # 图形化设置界面
```

Gnome 默认情况下使用 Wayland，与 Optimus-Manager 兼容性不理想。要强制使用 Xorg，需要编辑文件`/etc/gdm3/custom.conf`，h或者文件 `/etc/gdm/custom.conf` 然后删除行`＃Waylandenable = false`前的`＃`。

KDE设置参考[这个](https://github.com/Askannz/optimus-manager#important--manjaro-kde-users)。

更多安装注意事项，参阅 [Github Readme Installation](https://github.com/Askannz/optimus-manager#installation) 

**电源配置**

参考：[power-management-options](https://github.com/Askannz/optimus-manager/wiki/A-guide--to-power-management-options) 

电源管理配置的原因，参见[这里](https://github.com/Askannz/optimus-manager/wiki/A-guide--to-power-management-options#a-guide--to-power-management-options-in-optimus-manager)，如果一直插电使用，可以忽略。

```shell
sudo nano /etc/optimus-manager/optimus-manager.conf 
```

### 字体补全

#### emoji支持 

```shell
yay -S noto-color-emoji-fontconfig
```

已安装的字体可以通过`fc-list`配合`grep`来查找。如果你确定安装了某个字体但是没找到，可以用`fc-cache -f -v`刷新字体缓存。

如：

```shell
fc-list | grep -i "emoji"
```

> /usr/share/fonts/noto/NotoColorEmoji.ttf: Noto Color Emoji:style=Regular

#### 微软字体

参考：https://wiki.archlinux.org/title/Microsoft_fonts

```shell
yay -S ttf-ms-win10-auto
```

或者提取Windows分区或者镜像中的字体

```shell
7z e Win10_1709_English_x64.iso sources/install.wim
7z e install.wim 1/Windows/{Fonts/"*".{ttf,ttc},System32/Licenses/neutral/"*"/"*"/license.rtf} -ofonts/
7z e install.wim Windows/{Fonts/"*".{ttf,ttc},System32/Licenses/neutral/"*"/"*"/license.rtf} -ofonts/ # Windows 7
```

然后选择部分字体复制到 `/usr/share/fonts/ms` 中。

```shell
sudo cp ./fonts/{arialbd.ttf arialbi.ttf ariali.ttf arial.ttf ariblk.ttf \
courbd.ttf courbi.ttf couri.ttf cour.ttf msjh.ttc msjhbd.ttc msyhbd.ttc \
msyhsb.ttc msyhsl.ttc msyh.ttc simsun.ttc symbol.ttf tahomabd.ttf tahoma.ttf timesbd.ttf \
timesbi.ttf timesi.ttf times.ttf wingding.ttf} /usr/share/fonts/ms
```

更新字体缓存

```shell
fc-cache -f -v
```

### 无法查看安卓设备文件

直接在文件管理器中通过MTP查看Android设备文件，需要安装以下插件：

如果文件管理器使用GVFS（GNOME Files, Xfce 的 Thunar），安装 `gvfs-mtp` 提供MTP支持或者是安装 `gvfs-gphoto2` 提供PTP支持。

如果文件管理器使用KIO（KDE 的 Dolphin），安装 `kio-mtp`即可，自带PTP支持。

### 无法添加 Samba

安装 `gvfs-smb`

```shell
yay -S gvfs-smb
```

### 直接双击打开 desktop 文件

新建文件 `/usr/bin/run-desktop` 并给予运行权限，内容如下：

```python
#!/usr/bin/python

from gi.repository import Gio
import sys 

def main(myname, desktop, *uris):
    launcher = Gio.DesktopAppInfo.new_from_filename(desktop)
    launcher.launch_uris(uris, None)

if __name__ == "__main__":
    main(*sys.argv)
```

新建文件 `~/.local/share/applications/run-desktop.dektop`，内容如下：

```properties
[Desktop Entry]
Version=1.0
Name=run-desktop
Exec=run-desktop %U
MimeType=application/x-desktop
Terminal=false
Type=Application
Comment=直接打开desktop文件
Categories=System;FileTools
Icon=~/Pictures/icon/smart_launcher.png
```

在 `~/.local/share/applications/mimeapps.list` 中添加如下内容：

```properties
[Default Applications]
# ....
application/x-desktop=run-desktop.desktop
```

### 插耳机有杂音

安装 `alsa-utils`

```shell
yay -S alsa-utils
```

控制台输入 `alsamixer`，按`（Fn +）F6`，选择第二个（HDA INTEL PCH），使用左右方向键选择到 `auto mute`，使用上下方向键设置为`disable`，如果和还有杂音，就把 `Loopback Mixing` 也设置为 `disable ` 。按 `ESC` 退出后，最后控制台输入  `sudo alsactl store` 保存。

### gnome 设置补充

```shell
yay -S system-config-printer gnome-user-share gnome-remote-desktop rygel openssh power-profiles-daemon
```

`gnome-user-share`：WebDav 协议文件共享

`gnome-remote-desktop`：ms-rd 协议的远程桌面

`rygel`：媒体文件分享

`openssh`：远程登陆

`power-profiles-daemon`：电源模式

### 添加删除软件设置补充

#### 程序图标右击无“显示详情”

```shell
yay -S pamac-gnome-integration
```

#### 添加 snap 和 flatpak 支持

```shell
yay -S libpamac-flatpak-plugin libpamac-snap-plugin
```

安装后在设置中可启用，一般 AUR 的软件够了。

### 开关机无日志输出

```shell
sudo nano /etc/default/grub
```

修改如下：

```properties
# 添加注释
# GRUB_TIMEOUT_STYLE=hidden
# GRUB_CMDLINE_LINUX_DEFAULT="quiet resume=UUID=1bad731c-6a6e-4ac0-9f96-7710de9441b7 udev.log_priority=3"
# 删掉quite
GRUB_CMDLINE_LINUX_DEFAULT="resume=UUID=1bad731c-6a6e-4ac0-9f96-7710de9441b7 udev.log_priority=3"
```

然后

```shell
sudo update-grub
```

### qt软件与gnome桌面风格不统一

```shell
yay -S adwaita-qt5 adwaita-qt6 qt5ct qt6ct
```

设置qt应用程序主题环境变量

```shell
sudo nano /etc/envirnment
# 将qt5ct或者qt6ct添加到环境变量
QT_QPA_PLATFORMTHEME=qt6ct
```

然后在`qt5 settings`或者`qt6 settings`程序中选择`style`为`Adwaita`。

### 挂载 NTFS 并读写

安装驱动

```shell
yay -S ntfs-3g
```

查看分区

```shell
sudo fdisk --list

# 显示的 NTFS 文件系统硬盘
Device         Start        End    Sectors   Size Type
/dev/nvme1n1p1  2048 2000408575 2000406528 953.9G Microsoft basic data
```

打开 `/etc/fstab`

```shell
sudo nano /etc/fstab
```

添加以下内容

```properties
/dev/nvme1n1p1 /mnt/Data ntfs-3g user,auto,rw,dev,exec,suid
```

该命令共有6个参数，以空格分割，其中：

`/dev/nvme1n1p1`表示你要挂载的分区，根据你查看分区的结果填写。

`/mnt/Data`表示挂载点，根据你自身需求填写。

`ntfs-3g`表示待挂载分区使用的文件系统。分为以下几种情况：

NTFS：填写ntfs-3g或ntfs（在Ubuntu 20.04中ntfs是链接到ntfs-3g的）。FAT32或FAT16或FAT：填写vfat.自动检测文件系统：填写auto。

`auto和 noauto`： 这是控制设备是否自动挂载的选项。auto是默认选择的选项，这样，设备会在启动或者你使用mount -a命令时按照fstab的内容自动挂载。如果你不希望这样，就使用noauto选项，如果这样的话，你就只能明确地通过手工来挂载设备。

`user和 nouser`：这是一个非常有用的选项，user选项允许普通用户也能挂载设备，而nouser则只允许root用户挂载。nouser是默认选项，这也是让很多 Linux新手头疼的东西，因为他们发现没有办法正常挂载光驱，Windows分区等。如果你作为普通身份用户遇到类似问题，或者别的其他问题，就请把 user属性增加到fstab中。

`exec和 noexec`： exec允许你执行对应分区中的可执行二进制程序，同理，noexec的作用刚好相反。如果你拥有一个分区，分区上有一些可执行程序，而恰好你又不愿意，或者不能在你的系统中执行他们，就可以使用noexec属性。这种情况多发生于挂载Windows分区时。exec是默认选项，理由很简单，如果 noexec变成了你/根分区的默认选项的话

`rw和ro`：让该分区以可擦写或者是只读的型态挂载上来，如果你想要分享的数据是不给用户随意变更的， 这里也能够配置为只读。则不论在此文件系统的文件是否配置 w 权限，都无法写入！

`sync和 async`：对于该文件系统的输入输出应该以什么方式完成。sync的意思就是同步完成，通俗点讲，就是当你拷贝一个东西到设备或者分区中时，所有的写入变化将在你输入cp命令后立即生效，这个东西应该立马就开始往设备或者分区里面拷贝了。而如果是async，也就是输入输出异步完成的话，当你拷贝一个东西到设备或者分区中时，可能在你敲击cp命令后很久，实际的写入操作才会执行，换句话说，就是进行了缓冲处理。有时候这种机制蛮不错的，因为sync会影响你系统的运行速度，但是这也会带来一些问题。想一想，当你希望将一个文件拷贝到u盘上时，你执行了cp 命令，却忘记执行umount命令（它会强行将缓冲区内容写入），那么你拷贝的文件实际上并没有在u盘上面。如果你是使用的mv命令，而你又很快将u盘拔出……恭喜你，文件会从这个星球上消失的。因此，虽然async是默认属性，但是对于u盘，移动硬盘这种可移动存储设备，最好还是让他们使用sync选项。

`suid和nosuid`：该文件系统是否允许 SUID 的存在？如果不是运行文件放置目录，也可以配置为 nosuid 来取消这个功能！

`defaults`：同时具有 rw, suid, dev, exec, auto, nouser, async 等参数。 基本上，默认情况使用 defaults 配置即可！

也可以在 `gnome-disk-utility`这个软件中手动配置。

### A problem has occurred and the system can‘t recover'

1. 按 **ctrl+alt+f2** (或者 **ctrl+alt+fn+f2** )可以进入命令行界面
2. 输入 **su root**，切换到root用户

```shell
yay -S --overwrite "*" gnome gdm
# 等待安装完成，重启
reboot
```

### 手机音频通过蓝牙流转至  Linux

```shell
# 安装pulseaudio
yay -S pulseaudio-bluetooth
# 配置
sudo nano /etc/pulse/system.pa
# 添加下面指令

### Adding bluetooth audio streaming on Linux ###
load-module module-bluetooth-policy
load-module module-bluetooth-discover

# 重启音频服务
pulseaudio --kill
pulseaudio --start
# 我这里重启服务后声音没有恢复，故重启系统了
# 接下来就可以在手机上连接linux设备，会发现是一个音频设备
```

### x11 窗口环境下非整数缩放

```shell
yay -S gnome-control-center-x11-scaling
```

然后`Alt+F2`，重启桌面，再次打开`gnome-setting`，就会发现分数缩放出来了

## 软件

### 人脸识别 - Howdy

参考：https://wiki.archlinux.org/title/Howdy

​		    https://github.com/boltgolt/howdy

1、安装

```shell
yay -S howdy
```

打开 [v4l-utils](https://archlinux.org/packages/?name=v4l-utils)，找到想用作人脸识别的摄像头，记住其文件名，我的是`/dev/video0`。

编辑 `/lib/security/howdy/config.ini` 文件，也可以 root 用户权限使用 `howdy config` 命令来编辑。向下浏览找到

```properties
# The path of the device to capture frames from
# Should be set automatically by an installer if your distro has one
device_path = null
```

将 `device_path =` 后面的内容改为找到的文件路径，例如我的是 `/dev/video0`，那么就是 `device_path = /dev/video0`。修改完成后使用 `Ctrl + X` 保存并退出。

2、改完后运行测试

```shell
sudo howdy test
```

如果前面没有设置错，这时候就会弹出一个框播放摄像头，可能是黑白色的。

测试没问题就可以添加人脸了

```shell
sudo howdy add
```

中间会让你输入标签，可根据不同的人输入不同的标签。比如张三的人脸输入 ZhangSan 。还可以多添加几个人脸，戴不戴眼睛，远近一点等等。

3、最后再将howdy应用到你想要实现人脸的地方这一步需要修改pam文件：

为 `sudo` 启用 Howdy 验证，修改 `/etc/pam.d/sudo`；

为如 [GDM](https://wiki.archlinux.org/title/GDM) 和 [SDDM](https://wiki.archlinux.org/title/SDDM) 的本地图形登录启用 Howdy 验证，修改 `/etc/pam.d/system-local-login`；

使用的是 [LightDM](https://wiki.archlinux.org/title/LightDM) ，如xfce，修改 `/etc/pam.d/lightdm`

为 Gnome 图形化界面开启验证，修改 `/etc/pam.d/polkit-1`

4、重启电脑即可。

**注意：**

1、重启后，打开基于 chrome 的浏览器，可能会提示

> 您登陆计算机时，登陆密钥环未被解锁

根据相关 [issues](https://github.com/boltgolt/howdy/issues/461) 暂时无解，搜索到的使用 `seahorse` 或者删除 `~/.local/share/keyrings` 后设置空密码都会导致edge在登陆和闪退间循环。

2、终端里有 GStreamer warnings

> ```
> [ WARN:0] global /build/opencv/src/opencv-4.1.1/modules/videoio/src/cap_gstreamer.cpp (1756) handleMessage OpenCV | GStreamer warning: Embedded video playback halted; module source reported: Could not read from resource.
> [ WARN:0] global /build/opencv/src/opencv-4.1.1/modules/videoio/src/cap_gstreamer.cpp (886) open OpenCV | GStreamer warning: unable to start pipeline
> [ WARN:0] global /build/opencv/src/opencv-4.1.1/modules/videoio/src/cap_gstreamer.cpp (480) isPipelinePlaying OpenCV | GStreamer warning: GStreamer: pipeline have not been created
> ...
> ```

环境变量中添加 `OPENCV_LOG_LEVEL=ERROR` 但是，据wiki说明可能会掩饰一些问题，wiki中提到在[b04ffe5](https://github.com/boltgolt/howdy/commit/b04ffe5bd83683949db53bcaf2b91559e30d8e4c)中提交中解决，但是 `whereis howdy` 后中找到提交中的文件发现与源码并不一样，故备份文件 `/lib/security/howdy/recorders/video_capture.py` 并把源码中的文件拿过来替换，再补上缺失的文件 `/lib/security/howdy/i18n.py`。

### 包管理

#### deb包安装 - debtap

```shell
yay -S debtap
```

使用

升级debtap

```shell
sudo debtap -u
```

安装具体包

```shell
sudo debtap ${包路径}
```

操作完成后会在deb包同级目录生成`×.tar.xz`文件，直接用`pacman`安装即可

```shell
sudo pacman -U ${新包路径}
```

不过为啥不看看AUR里有没有呢？👀

#### 主题安装 - ocs url

用于浏览器直接安装主题

```shell
yay -S ocs-url
```

### 社交通讯

#### QQ

```shell
yay -S icalingua++
# 官方qq
yay -S linuxqq
```

#### 微信

活久见了，2024年3月~~没🐴~~张小🐉终于出了原生 linux 微信。

```shell
yay -S wechat-universal-bwrap
```

#### 腾讯会议

```shell
yay -S wemeet-bin
```

#### Telegram

```shell
yay -S telegram-desktop
```

设置中文

添加 [Telegram 电报简体中文群组](https://t.me/tele_zh_cn)

### 编程开发

#### gnome 终端

替换 gnome-terminal。

```shell
yay -S gnome-console
```

#### vscode

```shell
yay -S visual-studio-code-bin
```

占用“在文件中文件打开”，终端输入

```shell
xdg-mime query default inode/directory
# 输出
code.desktop
# 然后改回 nautilus
xdg-mime default org.gnome.Nautilus.desktop inode/directory
```

再次查看

```shell
xdg-mime query default inode/directory       
# 输出
org.gnome.Nautilus.desktop
```

以上问题也会引起打开 `nautilus` 卡死。

#### gedit

```shell
sudo pacman -S gedit
```

自动保存

默认状态下 gedit 是没有开启自动保存功能的。查询一下 `auto-save ` 键的值是 `false`

```shell
gsettings get org.gnome.gedit.preferences.editor auto-save
```

设置自动保存

```shell
gsettings set org.gnome.gedit.preferences.editor auto-save true
```

默认自动保存时间为10分钟，这里设置为1分钟，设置的值只能为整数。

```shell
gsettings set org.gnome.gedit.preferences.editor auto-save-interval 1
```

查看 gedit 其他设置键

```shell
gsettings list-keys org.gnome.gedit.preferences.editor
```

#### notepadqq

```shell
yay -S notepadqq
```

#### 超大文本编辑 - WindEdit

去 [Github Releases](https://github.com/kingToolbox/WindEdit/releases) 直接下载最新版，运行文件夹中的 WindEdit 文件即可。

#### Api测试 - ApiPost

[官网](https://www.apipost.cn/download.html)下载压缩包，解压到想要的存放位置。

在`~/.local/share/applications`下新建desktop文件，内容如下：

```properties
[Desktop Entry]
Name=ApiPost
Comment=API 文档、调试、Mock、测试一体化协作平台
Exec=~/Apps/apipost/apipost6
Terminal=false
Icon=~/Apps/apipost/resources/app/icon/linux 图标 (主程序1024)/icon_xxhdpi.png
Type=Application
Categories=Development;
StartupNotify=false
StartupWMClass=ApiPost
```

如果可以执行的话，就会在程序菜单中看到启动图标了。

或者直接安装

```shell
yay -S apipost-bin
```

同样功能的 postman 

```shell
yay -S postman-bin
```

#### 远程连接 - WindTerm

QT 开发的、性能高、占用低、集成 sftp，更多查看 [Github](https://github.com/kingToolbox/WindTerm)

去 [Github Releases](https://github.com/kingToolbox/WindTerm/releases) 直接下载最新版，运行。

打开文件夹中的desktop文件，修改其中的 `Exec` 值为 `WindTerm` 路径，并复制到 `~/.local/share/applications`，修改组和其他的访问权限为**只读**。或者直接安装

```shell
yay -S windterm-bin
```

### 网络下载

#### 迅雷

```shell
yay -S xunlei-bin
```

#### qbittorrent 增强版

**1、安装**

```shell
yay -S make qbittorrent-enhanced
```

qbittorrent-enhanced-nox 是无 GUI 版本，可使用WebUI打开，默认登陆网址：ip:8080

默认账号：admin 密码：adminadmin

注意设置默认保存位置不能为中文，否则可能出现乱码问题

```shell
yay -S qbittorrent-enhanced-nox
```

设置开机自启动

```shell
sudo nano /etc/systemd/system/qb-nox.service
```

添加如下内容

```properties
[Unit]
Description=qbittorrent-nox-daemon
After=network.target

[Service]
Type=forking
User=$USER
ExecStart=qbittorrent-nox -d
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

然后

```shell
sudo systemctl daemon-reload
sudo systemctl enable qb-nox
sudo systemctl start qb-nox
```

**2、trackerslist**

trackerslist.com

[网站](https://trackerslist.com/#/zh)整合了全网热门 Tracker，经过筛选过滤，最终得到了一个优质的 Tracker 列表方便大家使用。

github trackerslist 项目

这个项目 [ngosang/trackerslist](https://github.com/ngosang/trackerslist) 收集了公共可用的 Tracker 服务器，每天都会测试更新。其 `trackers_best` 列表中整理了 20 个工作状态最好的 Tracker 服务器，并且提供 IP 地址版本：[trackers_best_ip](https://raw.githubusercontent.com/ngosang/trackerslist/master/trackers_best_ip.txt)。

newTrackon

[newTrackon](https://newtrackon.com/) 是一项用于监视现有开放和公共 Tracker 服务器的状态和运行状况的服务。在 newTrackon 网站上，可以查看全世界网友提交信息的 Tracker 服务器的运行状态、所在国家、IP 地址等信息。在首页列表上只有一个 Tracker 服务器位于中国，地址是 `udp://tracker.sbsub.com:2710/announce`。

ACGTracker

[ACGTracker](https://acgtracker.com/) 网站介绍自己是一个「免费、开放、中立、供任何个人和组织使用」的 Tracker 服务器，应该是国人维护的，但不清楚位于国内还是国外。Tracker 地址是`http://open.acgtracker.com:1096/announce`。

**3、问题**

添加了很多 tracker 之后，只有第一个工作 => `工具-选项-高级`中，勾选`总是向所有同级的 Tracker 汇报`

#### Aria2

```shell
yay -S aria2
```

1、创建`~/.local/aria2/aria2.conf`，内容如下：

```properties
dir=/home/$USER/Downloads
# on-download-complete=/conf/on-complete.sh
log=/home/$USER/.cache/aria2/download.log
log-level=error
console-log-level=error
## 保存会话相关
# 从会话文件中读取下载任务
input-file=/home/$USER/.cache/aria2/aria2.session
# 在Aria2退出时保存`错误/未完成`的下载任务到会话文件
save-session=/home/$USER/.cache/aria2/aria2.session
# 定时保存会话, 0为退出时才保存, 需1.16.1以上版本, 默认:0
save-session-interval=30
 
## 缓存
# 启用磁盘缓存, 0为禁用缓存, 需1.16以上版本, 默认:16M
disk-cache=5M
# 文件预分配方式, 能有效降低磁盘碎片, 默认:prealloc
file-allocation=trunc
# 断点续传
continue=true


## 下载连接相关 ##
# 最大同时下载任务数, 运行时可修改, 默认:5
max-concurrent-downloads=5
# 同一服务器连接数, 添加时可指定, 默认:1
max-connection-per-server=16
# 最小文件分片大小, 添加时可指定, 取值范围1M -1024M, 默认:20M
# 假定size=10M, 文件为20MiB 则使用两个来源下载; 文件为15MiB 则使用一个来源下载
min-split-size=10M
# 单个任务最大线程数, 添加时可指定, 默认:5
split=16
# 整体下载速度限制, 运行时可修改, 默认:0
#max-overall-download-limit=0
# 单个任务下载速度限制, 默认:0
#max-download-limit=0
# 整体上传速度限制, 运行时可修改, 默认:0
max-overall-upload-limit=1024kb
# 单个任务上传速度限制, 默认:0
max-upload-limit=128kb
# 禁用IPv6, 默认:false
disable-ipv6=true
# 禁用https证书检查
check-certificate=false
# 运行覆盖已存在文件
allow-overwrite=false
# 自动重命名
auto-file-renaming=true

## RPC相关设置
# 启用RPC, 默认:false
enable-rpc=true
# 允许所有来源, 默认:false
rpc-allow-origin-all=true
# 允许非外部访问, 默认:false
rpc-listen-all=false
# 事件轮询方式, 取值:[epoll, kqueue, port, poll, select], 不同系统默认值不同
#event-poll=select
# RPC监听端口, 端口被占用时可以修改, 默认:6800
rpc-listen-port=6800


## BT/PT下载相关
# 当下载的是一个种子(以.torrent结尾)时, 自动开始BT任务, 默认:true
#follow-torrent=true
# BT监听端口, 当端口被屏蔽时使用, 默认:6881-6999
listen-port=51413
# 单个种子最大连接数, 默认:55
#bt-max-peers=55
# 打开DHT功能, PT需要禁用, 默认:true
enable-dht=true
# 打开IPv6 DHT功能, PT需要禁用
enable-dht6=false
# DHT网络监听端口, 默认:6881-6999
#dht-listen-port=6881-6999
# 本地节点查找, PT需要禁用, 默认:false
bt-enable-lpd=true
# 种子交换, PT需要禁用, 默认:true
enable-peer-exchange=true
# 每个种子限速, 对少种的PT很有用, 默认:50K
#bt-request-peer-speed-limit=50K
# 客户端伪装, PT需要
peer-id-prefix=-UT341-
user-agent=uTorrent/341(109279400)(30888)
# 当种子的分享率达到这个数时, 自动停止做种, 0为一直做种, 默认:1.0
seed-ratio=1.0
# 强制保存会话, 话即使任务已经完成, 默认:false
# 较新的版本开启后会在任务完成后依然保留.aria2文件
#force-save=false
# BT校验相关, 默认:true
#bt-hash-check-seed=true
# 继续之前的BT任务时, 无需再次校验, 默认:false
bt-seed-unverified=true
# 保存磁力链接元数据为种子文件(.torrent文件), 默认:false
bt-save-metadata=true

bt-load-saved-metadata=true
# 删除未选择的文件
bt-remove-unselected-file=true
# 仅下载种子文件
bt-metadata-only=true
# 通过网上的种子文件下载，种子保存在内存
follow-torrent=mem
# 保存上传的种子文件
rpc-save-upload-metadata=false
# 开启分离仅做种任务选项
bt-detach-seed-only=true
bt-tracker=http://open.acgtracker.com:1096/announce,https://cdn.jsdelivr.net/gh/ngosang/trackerslist/trackers_best.txt,https://trackerslist.com/all.txt,https://newtrackon.com/api/stable?include_ipv6_only_trackers=0,https://cdn.jsdelivr.net/gh/DeSireFire/animeTrackerList/AT_all.txt,https://cdn.jsdelivr.net/gh/ngosang/trackerslist@master/trackers_all.txt,http://open.acgtracker.com:1096/announce,http://www.zone-torrent.net:80/announce.php,http://torrentzilla.org/announce.php,http://kinorun.com/announce.php,udp://tracker.edkj.club:6969/announce,udp://114.216.182.218:6969/announce,http://bt.3dmgame.com:2710/announce,http://tracker.ali213.net:8080/announce,http://open.tracker.ink:6969/announce,http://171.105.76.31:6969/announce,http://tracker.skyts.cn:6969/announce,http://tracker.skyts.net:6969/announce,http://171.105.76.169:6969/announce,http://bt.ali213.net:8080/announce,udp://tracker1.bt.moack.co.kr:80/announce,udp://156.234.201.18:80/announce,http://torrents-nn.cn:2710/announce,http://fxtt.ru:80/announce,udp://52.58.128.163:6969/announce,udp://tracker.filemail.com:6969/announce,http://www.megatorrents.kg:80/announce.php,udp://5.79.236.102:6969/announce,udp://tracker.tiny-vps.com:6969/announce,https://tracker.tamersunion.org:443/announce,http://tracker.torrentleech.org:2710/announce,https://tracker.lilithraws.cf:443/announce,udp://opentrackr.org:1337/announce,http://masters-tb.com/announce.php,http://btx.anifilm.tv:80/announce.php,http://60-fps.org:80/bt/announce.php,https://tr.burnabyhighstar.com:443/announce,https://tracker.nitrix.me:443/announce,http://xtremewrestlingtorrents.net:80/announce.php,udp://tracker.opentrackr.org:1337/announce,http://tracker.gcvchp.com:2710/announce,http://www.xwt-classics.net/announce.php,udp://185.134.22.3:6969/announce,https://tr.highstar.shop:443/announce,http://data-bg.net/announce.php,https://t.btcland.xyz:443/announce,https://tr.torland.ga:443/announce,https://tracker.foreverpirates.co:443/announce,http://www.tvnihon.com:6969/announce,http://xbtrutor.com:2710/announce,http://datascene.net/announce.php,http://54.93.194.194:2710/announce,http://tracker.gbitt.info:80/announce,http://tracker.minglong.org:8080/announce,http://torrent.mp3quran.net:80/announce.php,https://tr.doogh.club:443/announce,udp://93.158.213.92:1337/announce,https://tr.fuckbitcoin.xyz:443/announce,http://www.all4nothin.net/announce.php,http://www.thegeeks.bz:3210/announce,http://torrentsmd.com:8080/announce,udp://185.8.156.2:6969/announce,udp://engplus.ru:6969/announce,http://tracker.nucozer-tracker.ml:2710/announce,https://1337.abcvg.info:443/announce,https://tracker.nanoha.org:443/announce,https://tracker.kuroy.me:443/announce,http://mediaclub.tv/announce,udp://explodie.org:6969/announce,udp://184.105.151.166:6969/announce,udp://209.141.59.16:6969/announce,udp://tracker.theoks.net:6969/announce,http://torrent.unix-ag.uni-kl.de:80/announce,http://tracker.etree.org:6969/announce,http://alltorrents.net:80/bt/announce.php,http://www.theplace.bz:2910/announce,http://bt.firebit.org:2710/announce,udp://p4p.arenabg.com:1337/announce,http://tracker.trancetraffic.com:80/announce.php,udp://213.108.129.160:6969/announce,udp://91.216.110.52:451/announce,udp://www.torrent.eu.org:451/announce,http://baibako.tv/announce,udp://45.14.225.8:1337/announce,http://deviloid.net:6969/announce,udp://qg.lorzl.gq:233/announce,udp://bt.oiyo.tk:6969/announce,udp://tracker.0x.tf:6969/announce,http://all4nothin.net/announce.php,http://music-torrent.net:2710/announce,udp://tracker.open-internet.nl:6969/announce,udp://tracker.monitorit4.me:6969/announce,http://freerainbowtables.com:6969/announce,udp://tracker.altrosky.nl:6969/announce,udp://93.104.214.40:6969/announce,http://www.freerainbowtables.com:6969/announce,udp://37.59.48.81:6969/announce,http://concen.org:6969/announce,http://tracker2.dler.org:80/announce,http://tracker3.dler.org:2710/announce,udp://6ahddutb1ucc3cp.ru:6969/announce,http://www.thetradersden.org/forums/tracker/announce.php,udp://admin.videoenpoche.info:6969/announce,http://62.210.217.207:1337/announce,udp://tracker.zerobytes.xyz:1337/announce,http://torrents.hikarinokiseki.com:6969/announce,http://bt.zlofenix.org:81/announce,udp://tracker-udp.gbitt.info:80/announce,udp://89.36.216.8:6969/announce,https://torrent.ubuntu.com/announce,http://mvgroup.org:2710/announce,http://www.tribalmixes.com/announce.php,http://tracker.gigatorrents.ws:2710/announce,http://tracker.tricitytorrents.com:2710/announce,udp://173.212.223.237:6969/announce,udp://62.109.31.95:6969/announce,udp://tracker.jordan.im:6969/announce,udp://tracker-de.ololosh.space:6969/announce,udp://tracker.ololosh.space:6969/announce,udp://tracker.srv00.com:6969/announce,http://mvgforumtracker.mvgroup.org:80/tracker.php/announce,udp://anidex.moe:6969/announce,udp://tracker0.ufibox.com:6969/announce,http://tracker.tambovnet.org:80/announce.php,udp://fe.dealclub.de:6969/announce,udp://148.251.53.72:6969/announce,http://www.mvgroup.org:2710/announce,http://irrenhaus.dyndns.dk:80/announce.php,udp://199.217.118.72:6969/announce,udp://144.76.35.202:6969/announce,udp://89.234.156.205:451/announce,http://51.79.71.167:80/announce,http://t1.pow7.com:80/announce,http://pow7.com:80/announce,http://atrack.pow7.com:80/announce,http://t2.pow7.com:80/announce,http://bt-tracker.gamexp.ru:2710/announce,http://tehconnection.eu:2790/announce,udp://bubu.mapfactor.com:6969/announce,http://tracker.tasvideos.org:6969/announce,udp://vibe.sleepyinternetfun.xyz:1738/announce,udp://149.28.47.87:1738/announce,udp://retracker.netbynet.ru:2710/announce,udp://212.1.226.176:2710/announce,udp://tracker.sbsub.com:2710/announce,udp://opentor.org:2710/announce,http://www.thevault.bz:2810/announce,http://t.acg.rip:6699/announce,http://share.camoe.cn:8080/announce,http://51.81.200.170:6699/announce,udp://open.stealth.si:80/announce,udp://46.4.101.245:80/announce,http://big-boss-tracker.net/announce.php,udp://208.83.20.20:6969/announce,http://mixfiend.com/announce.php,http://torrent.arjlover.net:2710/announce,udp://88.80.28.7:6969/announce,udp://tracker.ddunlimited.net:6969/announce,udp://mail.realliferpg.de:6969/announce,udp://195.201.94.195:6969/announce,udp://52.70.94.249:6969/announce,http://blackz.ro/announce.php,http://159.69.65.157:6969/announce,http://tracker.files.fm:6969/announce,udp://198.100.149.66:6969/announce,udp://movies.zsw.ca:6969/announce,udp://cutiegirl.ru:6969/announce,udp://tracker.lelux.fi:6969/announce,udp://exodus.desync.com:6969/announce,http://217.30.10.83:6969/announce,http://tracker.theempire.bz:3110/announce,http://95.217.161.135:80/announce,http://tracker.kali.org:6969/announce,http://51.38.230.101:80/announce,udp://104.244.72.77:1337/announce,udp://tracker.leech.ie:1337/announce,udp://tracker.openbittorrent.com:80/announce,udp://tracker.dler.org:6969/announce,udp://tracker.dler.com:6969/announce,http://siambit.com/announce.php,http://torrent.resonatingmedia.com:6969/announce,http://bithq.org/announce.php,udp://tracker.pomf.se:80/announce,udp://65.108.63.133:80/announce,udp://sugoi.pomf.se:80/announce,http://tracker.loadbt.com:6969/announce,http://65.21.48.148:6969/announce,http://t.nyaatracker.com:80/announce,http://45.154.253.7:80/announce,udp://tracker.coppersurfer.tk:80/announce,udp://109.72.83.209:80/announce,http://tracker.pussytorrents.org:3000/announce,udp://152.231.114.171:1337/announce,http://163.172.209.40:80/announce,udp://tracker2.dler.com:80/announce,http://bttracker.debian.org:6969/announce,udp://213.108.105.23:6969/announce,http://bt.edwardk.info:4040/announce,http://www.worldboxingvideoarchive.com/announce.php,udp://176.123.8.121:3391/announce,udp://inferno.demonoid.is:3391/announce,http://openbittorrent.com:80/announce,http://198.251.84.144:80/announce,http://tracker.xdvdz.com:2710/announce,http://tracker.pow7.com:80/announce,http://nyaa.tracker.wf:7777/announce,http://tracker.linkomanija.org:2710/announce,udp://tracker.torrent.eu.org:451/announce,udp://51.15.2.221:6969/announce,udp://code2chicken.nl:6969/announce,http://tr.bangumi.moe:6969/announce,http://tracker.frozen-layer.net:6969/announce.php,http://tracker.frozen-layer.com:6969/announce,udp://tracker.fatkhoala.org:13720/announce,udp://tracker.tallpenguin.org:15730/announce,udp://193.106.31.107:13710/announce

```

2、创建`/etc/systemd/system/aria2c.service`，内容如下：

```properties
[Unit]
Description= Aria2c Service
After=network.target
 
[Service]
Type=simple
# 不能是root，否则下载的文件无权操作
User=$USER 
# 注意：下面一行指出了配置文档的路径，非常必要
ExecStart=/usr/bin/aria2c --conf-path=/etc/aria2/aria2.conf  --enable-rpc --rpc-listen-all
 
[Install]
WantedBy=multi-user.target
```

3、创建需要文件

```shell
mkdir -p ~/.cache/aria2
touch ~/.cache/aria2/download.log
touch ~/.cache/aria2/aria2.session
```

4、启用服务，设置开机自启

```shell
sudo systemctl daemon-reload
sudo systemctl enable aria2c
sudo systemctl start aria2c
# 查看状态
sudo systemctl status aria2c
```

输出大概如下：

```
● aria2c.service - Aria2c Service
     Loaded: loaded (/etc/systemd/system/aria2c.service; enabled; vendor preset: disabled)
     Active: active (running) since Fri 2022-04-29 16:01:36 CST; 18s ago
   Main PID: 121770 (aria2c)
      Tasks: 1 (limit: 18931)
     Memory: 8.6M
        CPU: 32ms
     CGroup: /system.slice/aria2c.service
             └─121770 /usr/bin/aria2c --conf-path=/etc/aria2/aria2.conf --enable-rpc --rpc-listen-all

4月 29 16:01:36 $USER-manjaro systemd[1]: Started Aria2c Service.
4月 29 16:01:36 $USER-manjaro aria2c[121770]: 04/29 16:01:36 [WARN] Neither --rpc-secret nor a combination of --rpc-user >
4月 29 16:01:36 $USER-manjaro aria2c[121770]: 04/29 16:01:36 [NOTICE] IPv4 RPC：正在监听 TCP 端口 6800
```

4、安装Aria2浏览器扩展

Edge：[Aria2 for Edge](https://microsoftedge.microsoft.com/addons/detail/aria2-for-edge/jjfgljkjddpcpfapejfkelkbjbehagbh)

Chrome：[Aria2 for Chrome](https://chrome.google.com/webstore/detail/aria2-for-chrome/mpkodccbngfoacfalldjimigbofkhgjn)

火狐：[Aria2 下载器集成组件](https://addons.mozilla.org/zh-CN/firefox/addon/aria2-integration/?utm_source=addons.mozilla.org&utm_medium=referral&utm_content=search)

#### Samba 共享

```shell
sudo pacman -S samba nautilus-share manjaro-settings-samba
reboot
```

修改配置文件

```shell
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.bak
```

修改内容如下，内容中的注释需删除

```properties
[global]
   workgroup = WORKGROUP
   dns proxy = no
   log file = /var/log/samba/%m.log
   max log size = 100
   client min protocol = SMB2
   server role = standalone server
   passdb backend = tdbsam
   obey pam restrictions = yes
   unix password sync = yes
   passwd program = /usr/bin/passwd %u
   passwd chat = *New*UNIX*password* %n\n *ReType*new*UNIX*password* %n\n *passwd:*all*authentication*tokens*updated*successfully*
   pam password change = yes
   map to guest = Bad Password
   usershare allow guests = yes
   # 解决“net usershare”返回错误 255
   usershare owner only = false 
   name resolve order = lmhosts bcast host wins
   security = user
   guest account = nobody
   usershare path = /var/lib/samba/usershare
   usershare max shares = 100
   #usershare owner only = yes
   force create mode = 0070
   force directory mode = 0070
   load printers = no
   printing = bsd
   printcap name = /dev/null
   disable spoolss = yes
   show add printer wizard = no

[printers]
   comment = All Printers
   browseable = no
   path = /var/spool/samba
   printable = yes
   guest ok = no
   read only = yes
   create mask = 0700

[print$]
   comment = Printer Drivers
   path = /var/lib/samba/printers
   browseable = yes
   read only = yes
   guest ok = no
   
# 分享目录，可自定义名字   
[share] 
	# 分享实际目录
	path = /mnt/MyFiles 
	# 共享的目录是否让所有人可见
	browseable = yes 
	# 是否只读
	read only = no 
	# 客户端上传文件的默认权限
	create mask = 0700 
	# 客户端创建目录的默认权限
	directory mask = 0700 
	# 是否可写
	writable = yes 
	# 是否允许匿名(guest)访问,等同于public
	guest ok = no
	# 有效用户
	valid user = $USER
```

通过命令

```shell
testparm
```

检查`sam.conf`是否有语法错误。

将系统用户添加到samba用户组，并设置密码

```shell
sudo smbpasswd -a $USER
```

重启smb，nmb服务

```shell
sudo systemctl restart smb nmb
```

如果重启失败，则执行testparm或者通过`status`查看原因。

查看所有的samba用户

```shell
sudo pdbedit -L
```

查看对应ip上的samba服务器，例如查看本机samba

```shell
sudo smbclient -L 127.0.0.1
```

验证samba是否可以访问

```shell
smbclient //127.0.0.1/share
```

#### 无线热点 - linux wifi hotspots

图形化配置无线热点

GitHub：https://github.com/lakinduakash/linux-wifi-hotspot

```shell
yay -S linux-wifi-hotspot 
```

#### secoclient

```shell
# 创建所需文件夹
sudo mkdir /etc/init.d
# 安装依赖
yay -S coreutils-arch
wget http://www.corem.com.cn/sites/default/files/tools/secoclient/secoclient-linux-64-7.0.2.26.run
chmod 755 secoclient-linux-64-7.0.2.26.run
sudo ./secoclient-linux-64-7.0.2.26.run
```

### 文档办公

#### xmind 

需要付费

```shell
yay -S xmind 
```

#### minmaster

```shell
yay -Ss mindmaster_cn
```

#### draw.io 

```shell
yay -S drawio-desktop
```

#### typora 免费版

```shell
yay -S typora-free
```

#### 桌面便利贴 - xpad

类似于 sticky

```shell
yay -S xpad
```

#### Obsidian

```shell
yay -S obsidian
```

#### WPS

```shell
yay -S wps-office-cn
yay -S wps-office-mui-zh-cn
# 缺失的字体
yay -S wps-office-fonts
yay -S ttf-wps-fonts
```

#### Libreoffice

```shell
yay -S libreoffice-still libreoffice-still-zh-cn
```

#### 打印

```shell
yay -S manjaro-printer
yay -S hplip
sudo systemctl enable --now cups.servic
sudo systemctl enable --now cups.socket
sudo systemctl enable --now cups.path
```

#### Master PDF Editor

```shell
yay -S masterpdfeditor
```

激活：`5.8.52-1` 成功

```shell
sudo perl -pi -e 's/(\xE8...\xFF)\x88(..\xBF\x30)/$1\xFE$2/g' /opt/master-pdf-editor-5/masterpdfeditor5
```

#### pandoc

```shell
yay -S pandoc
```

#### 电子书 - foliate

轻量级电子书阅读器，官网：https://johnfactotum.github.io/foliate/

```shell
yay -S tracker foliate
```

另外还有功能很全的 calibre，或者一个不错的阅读器 koodo-reader-bin

```shell
yay -S calibre
```

#### chm阅读器 - kchmviewer

```shell
yay -S kchmviewer
```

#### 电子书阅读及管理 - calibre

支持EPUB、MOBI、AZW3、DOCX、FB2、HTMLZ、LIT、LRF、PDB、PDF、PMIZ、RB、RTF、SNB、TCR、TXT、TXTZ、ZIP格式的电子书

```shell
yay -S calibre
```

### 图片处理

#### 截图 - 火焰截图

```shell
yay -S flameshot
```

建议在设置中将命令`flameshot gui`绑定快捷键

如果启动不了，需要安装下面两个可选依赖。

```shell
yay -S grim
yay -S xdg-desktop-portal
```

#### 画图 - drawing

```shell
yay -S drawing
```

#### 图片PS - GIMP

```shell
yay -S gimp
```

#### 图片查看 - gthumb

```shell
yay -S gthumb
```

### 视频音乐

#### ~~音乐播放 - listen1~~

国内多平台音乐播放，体验不是很好，每次启动都是播放歌单第一首歌，经常出现歌曲无法播放的情况

```shell
yay -S patch listen1-desktop-appimage
```

#### spotify

spotify去广告版，免费、曲库全，可以和手机联动，去广告参考[这里](https://github.com/Nuzair46/SpotX-Linux)

```shell
yay -S spotify-adblock
```

其他音乐平台歌单导入 spotify，参考这里：[yyrcd](https://yyrcd.com/n2s/)

#### 音乐下载 - 洛雪音乐

```shell
yay -S lx-music
```

#### 视频转码 - handbrake

```shell
yay -S handbrake
```

#### ~~视频格式转换 - ciano~~

```shell
yay -S ciano
```

#### 视频播放 - mpv

```shell
yay -S mpv 
```

参考：https://wiki.archlinux.org/title/Mpv

https://mpv.io/manual/stable/

https://vcb-s.com/archives/7594

https://hooke007.github.io/unofficial/mpv_start.html

```shell
cp -r /usr/share/doc/mpv/ ~/.config/
```

mpv.conf

图形化mpv配置文件生成-[mpvconfigurator](https://github.com/haasnhoff/mpvconfigurator)

```properties
save-position-on-quit
osc = no
border = no
# 自动加载包含媒体文件名的所有字幕
sub-auto = fuzzy
# 程序关闭时记住播放的位置，从而在下一次播放时继续上次的位置播放
save-position-on-quit
# 不使用默认的键盘和鼠标操作方案
no-input-default-bindings

# 预设的高质量渲染
profile = gpu-hq 
# 开启色彩管理
icc-profile-auto 
# 字幕渲染到视频源分辨率并随视频一起缩放并进行色彩管理
blend-subtitles = video 
```

input.conf

```properties
MBTN_LEFT     ignore
MBTN_LEFT_DBL cycle fullscreen
MBTN_RIGHT    cycle pause
MBTN_BACK     playlist-prev
MBTN_FORWARD  playlist-next

WHEEL_UP      add volume 2
WHEEL_DOWN    add volume -2
WHEEL_LEFT    seek 2
WHEEL_RIGHT   seek -2

# 退出
ESC quit
# 暂停/播放
SPACE cycle pause
# 全屏/窗口
ENTER cycle fullscreen
# 显示播放进度
\ show-progress
# 显示控制台
` script-binding console/enable
# 显示状态信息
/ script-binding stats/display-stats
# 显示/隐藏状态信息
? script-binding stats/display-stats-toggle
# 音量加1
UP    add volume 1
# 音量减1
DOWN  add volume -1
# 音量加10
Shift+UP    add volume  10
# 音量减10
Shift+DOWN  add volume -10
# 后退1s
LEFT  seek -1
# 前进1s
RIGHT seek  1
# 后退1min
Shift+LEFT  seek -60 exact
# 前进1min
Shift+RIGHT seek 60 exact
# 音频延迟100ms
Ctrl+UP add audio-delay -0.1
# 音频提前100ms
Ctrl+DOWN add audio-delay +0.1
# 音频延迟1000ms
Ctrl+Shift+UP add audio-delay -1
# 音频提前1000ms
Ctrl+Shift+DOWN add audio-delay +1
# 字幕延迟100ms
Ctrl+LEFT   add sub-delay -0.1
# 字幕提前100ms
Ctrl+RIGHT  add sub-delay 0.1
# 字幕延迟1000ms
Ctrl+Shift+LEFT add sub-delay  -1
# 字幕提前1000ms
Ctrl+Shift+RIGHT add sub-delay  1
# 重置播放速度
BS set speed 1.0
# 播放速度加了10%
PGUP add speed 0.1
# 播放速度减了10%
PGDWN add speed -0.1
# 播放速度x2
Shift+PGUP multiply speed 2.0
# 播放速度/2
Shift+PGDWN multiply speed 0.5
# 播放上一个项目
, playlist-prev
# 播放下一个项目
. playlist-next
# 播放上一桢
[ frame-back-step
# 播放下一桢
] frame-step
# 播放上一章
{ add chapter -1
# 播放下一章
} add chapter 1
# 对比度减1
1 add contrast -1
# 对比度加1
2 add contrast 1
# 亮度减1
3 add brightness -1
# 亮度加1
4 add brightness 1
# 灰度减1
5 add gamma -1
# 灰度加1
6 add gamma 1
# 饱和度减1
7 add saturation -1
# 饱和度加1
8 add saturation 1
# 色相减1
9 add hue -1
# 色相加1
0 add hue 1
# 重置色彩均衡器
- set contrast 0; set brightness 0; set gamma 0; set saturation 0; set hue 0
# 截图
= screenshot video
# 截图包括字幕
+ screenshot
# 切换音频
a cycle audio
# 静音/取消静音
A cycle mute
# 切换字幕
s cycle sub
# 显示/隐藏字幕
S cycle sub-visibility
# 显示轨道信息（视频、音频、字幕）
l show_text ${track-list}
# 显示播放列表
L show_text ${playlist}

POWER quit
PLAY cycle pause
PAUSE cycle pause
PLAYPAUSE cycle pause
PLAYONLY set pause no
PAUSEONLY set pause yes
STOP quit
FORWARD seek 60
REWIND seek -60
# 鼠标前进
NEXT playlist-next
# 鼠标后退
PREV playlist-prev
VOLUME_UP add volume 2
VOLUME_DOWN add volume -2
MUTE cycle mute
```

其他配置

```shell
git clone https://github.com/Mackel123/mpv-scripts-lazy.git
cp /home/$USER/.config/mpv/mpv.conf /home/$USER/.config/mpv/mpv.conf.bak
cp /home/$USER/.config/mpv/input.conf /home/$USER/.config/mpv/input.conf.bak
cp ./mpv-scripts-lazy/* /home/$USER/.config/mpv
```

结合上面的配置修改 `input.conf` 配置文件

更多配置参考：

[跨平台播放器 mpv 配置入门 | VCB-Studio](https://vcb-s.com/archives/7594)

#### 录屏 - obs studio

```shell
yay -S obs-studio
```

N卡可以安装 NVFBC

```shell
git clone https://github.com/keylase/nvidia-patch.git
cd nvidia-patch
sudo ./patch-fbc.sh
```

  安装 obs-nvfbc   

```shell
yay -S obs-nvfbc       
```

   需要切换到独显模式才能在 obs 里看到 NvFBCSource，且 CPU 占有率和之前没啥区别，但是看 [LTT 的视频](https://www.bilibili.com/video/BV17U4y1H7w1)，CPU 占有率会低很多。

键盘输入显示 - screenke

可以将键盘的键入显示在显示屏上，对于视频的制作相当有用。

```shell
yay -S screenke
```

#### 录屏 gif - peek

```shell
yay -S peek 
```

#### 视频剪辑 - 达芬奇

免费版不支持mp3 aac mp4，不推荐。

首先确保已经安装显卡闭源驱动。

参考： https://wiki.archlinux.org/title/DaVinci_Resolve

下载 [davinci-resolve-checker](https://github.com/Ashark/davinci-resolve-checker) 运行检查是否能够运行达芬奇，逐个安装缺失的包。

```shell
yay -S nvidia
yay -S opencl-nvidia
yay -S cuda
```

最后可能还是启动不了，在终端运行`/opt/resolve/bin/resolve`输出：

> /opt/resolve/bin/resolve: error while loading shared libraries: libcrypt.so.1: cannot open shared object file: No such file or directory

```shell
yay -S libxcrypt-compat
```

#### 视频剪辑 - shotcut

```shell
yay -S shotcut
```

### 文件管理

#### alist

一个支持多种存储，支持网页浏览和 WebDAV 的文件列表程序。

参考：[官方文档](https://alist.nn.ci/zh/)

```shell
yay -S alist-bin
```

#### 百度网盘

```shell
yay -S baidunetdisk-bin
```

#### 阿里云盘 - 小白羊

```shell
yay -S aliyunpan-odomu-bin
```

#### 坚果云

```shell
yay -S nutstore
```

删除 lightapp ，用 electron 实行的所谓的 lightapp。

```shell
rm -R ~/.nutstore/apps/lightapp
rm ~/.local/share/applications/nutstore-lightapp.desktop 
```

#### nautilus 扩展

```shell
yay -S nautilus-admin
yay -S nautilus-copy-path 
# 文件hash计算
yay -S gtkhash gtkhash-nautilus
yay -S nautilus-image-converter
yay -S python2 nautilus-renamer
# 文件(夹)对比 选中一个文件，右击稍后比较，再选中另外一个文件右击比较
yay -S meld nautilus-compare
# 设置文件（夹）颜色、图标
yay -S nautilus-folder-icons
yay -S folder-color-nautilus
# samba分享 有bug
yay -S nautilus-share 
# 使用JetBrainsIDE打开
jetbrains-nautilus-git
# 使用VSCode打开
yay -S pkg-config nautilus-code
# 右击新建文件
nautilus-empty-file
# 嵌入到nautilus界面中的终端
nautilus-terminal
```

让`jetbrains-nautilus-git`识别手动安装的 JetbrainsIDE：

1. 新建 `~/.local/share/JetBrains/Toolbox/apps/`文件夹；
2. 在其中新建需要右击打开的 IDE 文件夹，比如 `CLion`、`Idea`；
3. 在每个 IDE 文件夹下新建 `ch-0` 文件夹；
4. 为手动安装的 IDE 的安装路创建链接，并将其重命名为路径下 `product-info.json` 中 `buildNumber`的值，最后剪切到刚才创建的 `ch-0` 文件夹。  

如为 CLion 所在路径 `~/Apps/JetBrains/CLion`创建软链接，重命名为 `212.5284.40`，然后将软链接文件`212.5284.40`剪切到`~/.local/share/JetBrains/Toolbox/apps/CLion/ch-0`文件夹。

#### 压缩解压缩

```shell
yay -S unarchiver
```

解决解压文件乱码，支持所有压缩文件，使用

```shell
unar archive.zip
```

### 工具

#### 任务管理器 - sysmontask

类似 Windows 的界面，可以把父子进程信息放一起统计显示

Github：https://github.com/KrispyCamel4u/SysMonTask

```shell
yay -S sysmontask
```

每次启动都有弹窗，解决：

打开文件 `/usr/lib/python3.10/site-packages/sysmontask/sysmontask.py` ，搜索 `show only for the first time`，注释掉下面的代码：

```python
 if (self.settings.get_int("version-int")!=VERSION_INT):
     dialog=whatsnew_notice_dialog(self.Window,self)
     dialog.run()
     dialog.destroy()
     self.settings.set_int("version-int",VERSION_INT
```

另外一个不错的 `gnome-usage`，也把父子进程信息放一起统计显示，但是功能了个很简单，只能看。

#### 终端任务管理器 - htop

```shell
yay -S htop
```

#### 磁盘使用情况 - Baobab

```shell
yay -S baobab
```

#### 磁盘信息 - smartctl

```shell
yay -S smartmontools
```

使用

```shell
# 查看有哪些硬盘
smartctl --scan
# 查看nvme硬盘信息
sudo smartctl /dev/nvme0
# 查看另一块sata硬盘信息
sudo smartctl /dev/sda
```

#### 快速搜索 - FSearch

```shell
yay -S fsearch
```

建议在设置中将命令`fsearch --display=`绑定快捷键

添加被排除搜索的文件埃夹

```shell
# 回收站
/home/$USER/.local/share/Trash 
# 主题
/home/$USER/.local/share/themes 
/usr/share/themes
/usr/share/app-info/icons
/usr/share/icons
/home/$USER/.local/share/icons
${挂载点}/System Volume Information
${挂载点}/$RECYCLE.BIN
```

另外一个搜索速度很快的软件 - Catfish

```shell
yay -S catfish
```

#### 系统备份 - Timeshift

```shell
yay -S timeshift
```

#### 垃圾清理 - BlechBit

```shell
yay -S bleachbit
```

1、默认会以当前用户启动项部分垃圾文件没有权限删除，做以下修改。

新建`/usr/bin/blechbit_su`，内容如下：

```bash
#!/bin/bash

app_command='bleachbit --gui'

if [ "$(id -u)" -eq 0 ]; then
	# user is admin
	${app_command}
else
	# user is not admin
	if echo $- | grep "i" >/dev/null 2>&1; then
		# script is running in interactive mode
		su - -c "${app_command}"
	else
		# script is running in non-interactive mode
		if [ "$XDG_SESSION_TYPE" = "wayland" ] ; then
			xhost +SI:localuser:root
			pkexec ${app_command}
			xhost -SI:localuser:root
			xhost
		elif command -v pkexec >/dev/null 2>&1; then
			pkexec ${app_command}
		elif command -v sudo >/dev/null 2>&1; then
			x-terminal-emulator -e "sudo ${app_command}"
		elif command -v su >/dev/null 2>&1; then
			x-terminal-emulator -e "su - -c '${app_command}'"
		else
			x-terminal-emulator -e "echo 'Command must be run as root user: ${app_command}'"
		fi
	fi
fi
```

新建`/usr/share/applications/org.bleachbit.BleachBit.root.desktop`，内容如下：

```properties
[Desktop Entry]
Version=1.1
Type=Application
Name=BleachBit(Root)
Comment=Free space and maintain privacy
Comment[af]=Maak ruimte en behou privaatheid
Comment[zh_CN]=释放空间并保护隐私
GenericName=Unnecessary file cleaner
GenericName[zh_CN]=不必要文件清理器
Exec=su_bleachbit
Terminal=false
Icon=bleachbit
Categories=System;FileTools;GTK;
Keywords=cache;clean;free;performance;privacy;
StartupNotify=true
X-GNOME-UsesNotifications=true
```

2、添加被清理的文件（夹）

**适当选择，仅供参考。**

```shell
/var/cache/pkgfile
/var/log/pacman.log
/var/tmp/pamac-build-$USER
/var/log
# microsoft-edge-dev换成其他浏览器路径应该也适用
/home/$USER/.cache/microsoft-edge-dev/Default/Cache
/home/$USER/.config/listen1/Cache
/home/$USER/.config/Code/Cache
/home/$USER/.cache/yay
# 回收站
/home/$USER/.local/share/Trash 
```

#### 剪贴板记录 - CopyQ

```shell
yay -S copyq
```

建议在设置中将命令`copyq toggle`绑定快捷键

#### 硬盘操作 - GParted

```shell
yay -S gparted
```

####  阻止自动挂起 - caffein ng

```shell
yay -S caffeine-ng
```

和`caffeine gnome-shell`扩展功能一致，不过可以用的时候再打开，就不用一直在顶栏显示了。

建议在软件设置中设置自动激活，这样打开就不用再点击激活了，下面设置启动发现有 fcitx5 进程，就自动激活。

```shell
mkdir ~/.config/caffeine & echo "fcitx5" > ~/.config/caffeine/whitelist.txt
```

#### dos2unix

**dos2unix**也可轻松将一个windows下的文本文件转化为Unix兼容的格式

```shell
yay -S dos2unix
```

#### 16进制编辑器 - GNOME Hex Editor

```shell
yay -S ghex
```

其他类似软件还有 `bless`，`Okteta`，`wxHexEditor`，`Hexedit`

参考：https://zhuanlan.zhihu.com/p/59119723

#### 系统信息 - fastfatch

比`neofetch`输出平滑、显示的信息多

```shell
yay -S fastfetch
```

让输出花里胡哨些，可安装`lolcat`

```shell
fastfetch | lolcat
```

#### APK预览 - apk preview

```shell
yay -S apk-preview
```

#### 取色器 

```shell
yay -S gcolor3
```

#### NVIDIA 显卡监控

```shell
yay -S nvidia-system-monitor-qt
```

#### 字体查看与安装 - gnome font viewer

```shell
yay -S gnome-font-viewer
```

#### 电源管理 - TLP

参考：https://wiki.archlinux.org/title/TLP

https://linrunner.de/tlp/settings/#

https://linrunner.de/tlp

```shell
yay -S tlp tlp-rdw tlpui smartmontools
sudo systemctl enable tlp.service
sudo systemctl enable NetworkManager-dispatcher.service
sudo systemctl mask systemd-rfkill.service systemd-rfkill.socket
sudo systemctl start tlp.service # 或者sudo tlp start
```

使用下面命令检查运行状态

```shell
tlp-stat -s
```

检查输出结果是否类似

> +++ TLP Status
> State          = enabled
> RDW state      = enabled
> Last run       = 03:59:30 PM,   2234 sec(s) ago
> Mode           = AC
> Power source   = AC

#### 命令帮助 - tldr

```shell
yay -S tldr
```

使用

```shell
tldr cp                                                                                                             
  cp

  Copy files and directories.
  More information: https://www.gnu.org/software/coreutils/cp.

  - Copy a file to another location:
    cp path/to/source_file.ext path/to/target_file.ext

  - Copy a file into another directory, keeping the filename:
    cp path/to/source_file.ext path/to/target_parent_directory

  - Recursively copy a directory's contents to another location (if the destination exists, the directory is copied inside it):
    cp -r path/to/source_directory path/to/target_directory
    
```

#### uTools

```shell
yay -S utools
# 添加自启
cp /usr/share/applications/utools.desktop /home/mdmbct/.config/autostart
```

插件推荐

```
超级剪贴板1.4.0 > ALT+V
书签与历史记录 
易翻翻译 > ALT+T 
JSON编辑器 
解散文件夹 
Ctool 
网页快开 > 必应（ALT+B）谷歌（ALT+G）
关闭进程
讯飞ocr > ALT+R
备忘快贴
maven&gradle
计算稿纸
编码小助手
程序员手册
特殊符号
```

类似utools的一个软件 [ulauncher](https://ulauncher.io/)

```
yay -S ulauncher
```

### 连接

#### 远程桌面 - remmina

```shell
yay -S remmina freerdp libvncserver spice-gtk
```

`freerdp libvncserver spice-gtk` 是插件。

#### 远程连接 - todesk

官网下载地址：http://www.hellodesk.cn/linux.html

安装

```shell
sudo pacman -U todesk_4.1.0_x86_64.pkg.tar.zst
# 或者
yay -S todesk-bin
```

如打开无法显示中文，安装字体`noto-fonts-cjk` 

```shell
yay -S noto-fonts-cjk
```

如不能正常使用，请执行以下命令初始化

```shell
sudo systemctl stop todeskd.service
sudo mv /opt/todesk/config/todeskd.conf /opt/todesk/config/todeskd.conf.bak
sudo systemctl start todeskd.service
```

#### Teamviewer

```shell
yay -S teamviewer
```

设置守护进程开机自启

```shell
sudo systemctl daemon-reload
sudo systemctl enable teamviewerd.service
sudo systemctl start teamviewerd.service
# 查看是否启动成功
sudo systemctl status teamviewerd.service
```

#### Android 投屏 - scrcpy

```shell
# 会自动安装Android-tools包
yay -S scrcpy
# scrcpy的图形界面
yay -S guiscrcpy
```

安装好后打卡手机开发者模式，允许adb调试，在终端运行`scrcpy`

参考：https://github.com/Genymobile/scrcpy

​			https://github.com/srevinsaju/guiscrcpy

### Steam

```shell
yay -S steam-manjaro
```

问题：failed to load steamui.so

```shell
 mv ~/.local/share/Steam{,.old}
```

## 虚拟化

### KVM
参考：https://wiki.archlinux.org/title/KVM

https://wiki.archlinux.org/title/Kernel_module

https://wiki.archlinux.org/title/QEMU

https://wiki.archlinux.org/title/Libvirt

#### 安装

##### 检查

**1、是否支持硬件虚拟化**

```shell
LC_ALL=C lscpu | grep Virtualization
```

如果有一下输出，则支持。

> Virtualization:                  VT-x

**2、内核是否支持**

```shell
zgrep CONFIG_KVM /proc/config.gz       
```

后面显示**y**或**m**，证明内核支持虚拟化。

> CONFIG_KVM_GUEST=y
> CONFIG_KVM_MMIO=y
> CONFIG_KVM_ASYNC_PF=y
> CONFIG_KVM_VFIO=y
> CONFIG_KVM_GENERIC_DIRTYLOG_READ_PROTECT=y
> CONFIG_KVM_COMPAT=y
> CONFIG_KVM_XFER_TO_GUEST_WORK=y
> CONFIG_KVM=m
> CONFIG_KVM_INTEL=m
> CONFIG_KVM_AMD=m
> CONFIG_KVM_AMD_SEV=y
> CONFIG_KVM_XEN=y
> CONFIG_KVM_MMU_AUDIT=y

##### 安装所需包

通过 `qemu` 来使用 kvm，再通过 `libvirt` 来管理 qemu 虚拟机。

- **qemu-base** 提供 qemu-img 等命令；
- **edk2-ovmf** uefi支持
- **libvirt** 提供管理虚拟机、存储、网络的功能；
- **ebtables** 桥接网络管理，用于 default NAT 网络；
- **dnsmasq** 提供 DHCP DNS 服务，用于 default NAT 网络；
- **bridge-utils** 桥接网络管理，用于桥接网络；
- **openbsd-netcat** 用于通过 SSH 管理。

先安装这些所需包：

```shell
yay -S qemu-base edk2-ovmf bridge-utils ebtables dnsmasq openbsd-netcat virt-manager
```

其他可选包

- qemu-arch-extra 其它架构支持
- qemu-block-gluster glusterfs block 支持
- qemu-block-iscsi iSCSI block 支持
- qemu-block-rbd RBD block 支持
- ed2k-ovmf-macos macos支持

##### 设置 libvirtd 服务

```shell
sudo systemctl enable libvirtd
sudo systemctl start libvirtd
```

#### 桥接网络

默认是创建一个名为 virbr0 的虚拟桥接网络，如没有创建，参考：https://wiki.archlinux.org/title/Network_bridge

#### 显卡直通(未完成)

参考：https://eonun.com/posts/11be7fa2/

https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF

https://wiki.archlinux.org/title/Kernel_parameters#GRUB

1、开启 iommu
 IOMMU(i/o memory management unit)。iommu 有两大功能：控制设备 DMA 地址映射到机器物理地址（dmar），中断重映射（intremap）（可选）
 确认内核是否支持 iommu

```shell
cat /proc/cmdline | grep iommu
```

有输出则正常。如果没有输出，将 `intel_iommu=on` 添加到 grub 启动文件当中，编辑grub文件 ：

- 对于 Intel CPU(VT-d)，使用 `intel_iommu=on`。
- 对于 AMD CPU(AMD-Vi)，使用 `amd_iommu=on`。

```shell
sudo nano /etc/default/grub
```

`GRUB_CMDLINE_LINUX_DEFAULT` 后面添加 `intel_iommu=on`

```properties
GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on rd.driver.pre=vfio-pci kvm.ignore_msrs=1 resume=UUID=xxxx udev.log_priority=3"
```

然后重新生成 `grub.cfg` 文件：

```shell
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

重启后查看是否开启 iommu

```shell
sudo dmesg | grep -e DMAR -e IOMMU 
```

#### 创建虚拟机

参考：https://blog.csdn.net/allway2/article/details/103125982

https://dausruddin.com/how-to-enable-clipboard-and-folder-sharing-in-qemu-kvm-on-windows-guest/

Virtio 半虚拟化驱动程序可提高机器性能，减少 I/O 延迟并将吞吐量提高到接近裸机水平。大多数 Linux 发行版都包含 virtio 驱动程序作为标准配置，但是 Windows 平台却没有，需要手动安装。

**virtio-win** 

官方下载地址：https://fedorapeople.org/groups/virt/virtio-win

最新稳定版下载地址：https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso

新版不支持 win7，最后一个支持 win7的是 [0.1.173-4](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/archive-virtio/virtio-win-0.1.173-4/virtio-win-0.1.173.iso)

**机器配置**

- 磁盘总线：Virtio
- NIC型号：Virtio 以太网
- 视频型号：Virtio

选择好 Win 镜像后，需要添加一个 SATA CDROM 内容为 `virtio-win.iso` 文件，安装 Win 时浏览加载 Virtio 驱动后才能看到设置的 Virtio 磁盘。

进入系统后需要运行 virtio-win.iso 根目录的 `virtio-win-gt-x64.msi` 文件

自适应分辨率、与宿主机共享剪贴板、文件直接拖动，虚拟机安装 [spice-guest-tools](https://www.spice-space.org/download/windows/spice-guest-tools/)，点击下载[最新版](https://www.spice-space.org/download/windows/spice-guest-tools/spice-guest-tools-latest.exe )，下载，重启生效

文件共享，使用 SMB

#### 问题

1、Can't start KVM guest: "network 'default' is not active"

出现错误可列出网络检查

```shell
sudo virsh net-list --all
```

若没有任何网络，可手动定义网络；或有网络但 ifconfig 中没有，且启动错误也可手动添加

```shell
sudo virsh net-define /etc/libvirt/qemu/networks/default.xml
```

标记为自动启动

```shell
sudo virsh net-autostart default
```

需用重启系统生效

2、报错：device {"driver":"virtio-vga-gl","id":"video0","max_outputs":1,"bus":"pcie.0","addr":"0x1"}: opengl is not available'

解决： find the option in virt-manager. (futur reader : Display Spice > Listen type: None > Check OpenGL box)

参考：https://bbs.archlinux.org/viewtopic.php?pid=2018445#p2018445

### VirtualBox

```shell
sudo uname -r # 查看内核版本
5.15.57-2-MANJARO  # 内核版本
yay -S virtualbox # 安装virtualbox，切记选择相同的内核版本
```

在 [Index of http://download.virtualbox.org/virtualbox](http://download.virtualbox.org/virtualbox/) 下载拓展包，查看virtualbox版本， 选择expack结尾的。

#### 从实体硬盘启动

先创建一个 vmdk 文件来指向实体硬盘。

```shell
sudo VBoxManage internalcommands createrawvmdk -filename ./ManjaroToGo.vmdk  -rawdisk /dev/sdb
```

修改文件所属用户和用户组，可以照着已有的 virtualbox 虚拟磁盘文件修改

```shell
# $USER获取当前用户名
sudo chown $USER ./ManjaroToGo.vmdk 
# chown username:groupname filename
sudo chown $USER:$USER ./ManjaroToGo.vmdk 
```

将当前用户添加到 disk 组，注销重新登陆就可以在 VirtualBox 中添加文件了。

```shell
sudo gpasswd -a $USER disk
# 从disk组中删除
sudo gpasswd -d $USER disk
```

#### 问题

1、Kernel driver not installed 

> Kernel driver not installed (rc=-1908)
>
> The VirtualBox Linux kernel driver is either not loaded or not set up correctly. Please try setting it up again by executing
>
> '/sbin/vboxconfig'
>
> as root.
>
> If your system has EFI Secure Boot enabled you may also need to sign the kernel modules (vboxdrv, vboxnetflt, vboxnetadp, vboxpci) before you can load them. Please see your Linux system's documentation for more information.
>
> where: suplibOsInit what: 3 VERR_VM_DRIVER_NOT_INSTALLED (-1908) - The support driver is not installed. On linux, open returned ENOENT

运行下面命令解决

```shell
sudo modprobe vboxdrv
```

### Genymotion

```shell
yay -S genymotion
```

[官网](https://www.genymotion.com/)注册个帐号，软件登陆选择个人使用即可。

推荐 Android8.0或者9.0，因为有可用的ARM指令翻译包，下载地址： https://pan.baidu.com/s/1ViSS90z5hYXYats0Eifrow  提取码:tl5b

下载后直接拖进去，重启即可。

OpenGApps for ANdroid 9.0 下载地址：https://www.androidsage.com/2018/08/08/download-gapps-for-android-9-pie/

### Waydroid

参考：https://wiki.archlinuxcn.org/wiki/Waydroid

https://zhuanlan.zhihu.com/p/603603346

1、安装

```shell
# 我使用的是linux6.6.10，较低版本的linux安装binder请参考archwiki
yay -S binder_linux-dkms
# 等待安装完成
yay -S waydroid
# 剪切板同步
yay -S python-pyclip
```

2、初始化

在初始化 Waydroid 之后，如果映像不可用，将会自动下载最新的 Android 镜像

```shell
waydroid init
```

初始化支持 GApps 的 Waydroid：

```shell
waydroid init -s GAPPS -f
```

说明：下载可能会耗时很久；你也可以自行下载之后将文件放到 /usr/share/waydroid-extra/images/，使用到的地址如下：

```
https://sourceforge.net/projects/waydroid/files/images/system/lineage/waydroid_x86_64
https://sourceforge.net/projects/waydroid/files/images/vendor/waydroid_x86_64
```

接下来启动`waydroid-container.service`。

```shell
systemctl enable --now waydroid-container
```

Waydroid 现在应该能正常工作了。

2、使用

waydroid 只支持 wayland ，确保使用 wayland 而不是 x11，使用下面命令查询使用的窗口系统

```shell
echo $XDG_SESSION_TYPE
```

确保`waydroid-container.service` 正在运行，然后执行：

```shell
waydroid session start
```

Waydroid 会话现在已处于活动状态，这里有一些与 Waydroid 交互的实用命令：

启动 GUI：

```shell
waydroid show-full-ui
```

启动 shell：

```shell
waydroid shell
```

安装应用程序：

```
waydroid app install $path_to_apk
```

4、网络



### VMware workstation pro

```shell
yay -S vmware-workstation 
```

Pro16激活码

> ZF3R0-FHED2-M80TY-8QYGC-NPKYF
> YF390-0HF8P-M81RQ-2DXQE-M2UT6
> ZF71R-DMX85-08DQY-8YMNC-PPHV8

### Wine
参考：[Wine ArcWiki](https://wiki.archlinux.org/title/Wine_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))

#### 安装

```shell
yay -S wine wine-geock wine-mono wine-installer winetricks zenity 
```

其中 `wine-gecko ` 和 `wine-mono` 分别用于运行依赖于Internet Explorer和.NET的程序，

`wine-installer` 补全默认不会安装的快捷方式，但是其中的记事本快捷方式用不到，看起来难受，故删除。

```shell
sudo rm /usr/share/applications/wine-notepad.desktop
```

#### 补全字体

```shell
cd ~/.wine/drive_c/windows/Fonts && for i in /usr/share/fonts/**/*.{ttf,otf,ttc}; do ln -s "$i" ; done
```

部分中文字体可能因为dpi过小无法显示，运行 `winecfg`，依次点击 "显示 -> 屏幕分辨率 -> 125dpi" 。

#### 启用平滑字体

提高Wine渲染效果的一个好方法是使用cleartype平滑字体。 若要启用“次像素平滑(ClearType) RGB”:

终端导入注册表信息：

```shell
cat << EOF > /tmp/fontsmoothing
REnano4

[HKEY_CURRENT_USER\Control Panel\Desktop]
"FontSmoothing"="2"
"FontSmoothingOrientation"=dword:00000001
"FontSmoothingType"=dword:00000002
"FontSmoothingGamma"=dword:00000578
EOF
WINE=${WINE:-wine} WINEPREFIX=${WINEPREFIX:-$HOME/.wine} $WINE renano /tmp/fontsmoothing 2> /dev/null
rm /tmp/fontsmoothing
```

`renano` 备份 `HKEY_LOCAL_MACHINE\Software\Microsoft\Windows NT\CurrentVersion\FontLink\SystemLink`，内容如下：

```properties
Windows Registry Editor Version 5.00

[HKEY_LOCAL_MACHINE\Software\Microsoft\Windows NT\CurrentVersion\FontLink\SystemLink]
"Lucida Sans Unicode"=hex(7):4d,00,53,00,47,00,4f,00,54,00,48,00,49,00,43,00,\
  2e,00,54,00,54,00,43,00,2c,00,4d,00,53,00,20,00,55,00,49,00,20,00,47,00,6f,\
  00,74,00,68,00,69,00,63,00,00,00,4d,00,49,00,4e,00,47,00,4c,00,49,00,55,00,\
  2e,00,54,00,54,00,43,00,2c,00,50,00,4d,00,69,00,6e,00,67,00,4c,00,69,00,55,\
  00,00,00,53,00,49,00,4d,00,53,00,55,00,4e,00,2e,00,54,00,54,00,43,00,2c,00,\
  53,00,69,00,6d,00,53,00,75,00,6e,00,00,00,47,00,55,00,4c,00,49,00,4d,00,2e,\
  00,54,00,54,00,43,00,2c,00,47,00,75,00,6c,00,69,00,6d,00,00,00,00,00
"Microsoft Sans Serif"=hex(7):4d,00,53,00,47,00,4f,00,54,00,48,00,49,00,43,00,\
  2e,00,54,00,54,00,43,00,2c,00,4d,00,53,00,20,00,55,00,49,00,20,00,47,00,6f,\
  00,74,00,68,00,69,00,63,00,00,00,4d,00,49,00,4e,00,47,00,4c,00,49,00,55,00,\
  2e,00,54,00,54,00,43,00,2c,00,50,00,4d,00,69,00,6e,00,67,00,4c,00,69,00,55,\
  00,00,00,53,00,49,00,4d,00,53,00,55,00,4e,00,2e,00,54,00,54,00,43,00,2c,00,\
  53,00,69,00,6d,00,53,00,75,00,6e,00,00,00,47,00,55,00,4c,00,49,00,4d,00,2e,\
  00,54,00,54,00,43,00,2c,00,47,00,75,00,6c,00,69,00,6d,00,00,00,00,00
"Tahoma"=hex(7):4d,00,53,00,47,00,4f,00,54,00,48,00,49,00,43,00,2e,00,54,00,54,\
  00,43,00,2c,00,4d,00,53,00,20,00,55,00,49,00,20,00,47,00,6f,00,74,00,68,00,\
  69,00,63,00,00,00,4d,00,49,00,4e,00,47,00,4c,00,49,00,55,00,2e,00,54,00,54,\
  00,43,00,2c,00,50,00,4d,00,69,00,6e,00,67,00,4c,00,69,00,55,00,00,00,53,00,\
  49,00,4d,00,53,00,55,00,4e,00,2e,00,54,00,54,00,43,00,2c,00,53,00,69,00,6d,\
  00,53,00,75,00,6e,00,00,00,47,00,55,00,4c,00,49,00,4d,00,2e,00,54,00,54,00,\
  43,00,2c,00,47,00,75,00,6c,00,69,00,6d,00,00,00,00,00
```

`renano` 备份 `HKEY_LOCAL_MACHINE\Software\Microsoft\Windows NT\CurrentVersion\FontSubstitutes` ，我的备份内容为：

```shell
Windows Registry Editor Version 5.00

[HKEY_LOCAL_MACHINE\Software\Microsoft\Windows NT\CurrentVersion\FontSubstitutes]
"Arial Baltic,186"="Arial,186"
"Arial CE,238"="Arial,238"
"Arial CYR,204"="Arial,204"
"Arial Greek,161"="Arial,161"
"Arial TUR,162"="Arial,162"
"Courier New Baltic,186"="Courier New,186"
"Courier New CE,238"="Courier New,238"
"Courier New CYR,204"="Courier New,204"
"Courier New Greek,161"="Courier New,161"
"Courier New TUR,162"="Courier New,162"
"Helv"="MS Sans Serif"
"Helvetica"="Arial"
"MS Shell Dlg"="SimSun"
"MS Shell Dlg 2"="Tahoma"
"Times"="Times New Roman"
"Times New Roman Baltic,186"="Times New Roman,186"
"Times New Roman CE,238"="Times New Roman,238"
"Times New Roman CYR,204"="Times New Roman,204"
"Times New Roman Greek,161"="Times New Roman,161"
"Times New Roman TUR,162"="Times New Roman,162"
"Tms Rmn"="NSimSun"
```

设置部分字体为微软雅黑，在此之前确保安装了微软雅黑字体到Linux系统中。

```shell
cat << EOF > /tmp/msyh-link
REnano4
 
[HKEY_LOCAL_MACHINE\Software\Microsoft\Windows NT\CurrentVersion\FontLink\SystemLink]
"Lucida Sans Unicode"="msyh.ttc"
"Microsoft Sans Serif"="msyh.ttc"
"MS Sans Serif"="msyh.ttc"
"Tahoma"="msyh.ttc"
"Tahoma Bold"="msyhbd.ttc"
"msyh"="msyh.ttc"
"Arial"="msyh.ttc"
"Arial Black"="msyh.ttc"
EOF
WINE=${WINE:-wine} WINEPREFIX=${WINEPREFIX:-$HOME/.wine} $WINE renano /tmp/msyh-link 2> /dev/null
rm /tmp/msyh-link
```

```shell
cat << EOF > /tmp/FontSubstitutes
REnano4

[HKEY_LOCAL_MACHINE\Software\Microsoft\Windows NT\CurrentVersion\FontSubstitutes]
"Arial"="Arial"
"Courier New"="Courier New"
"FixedSys"="msyh"
"System"="msyh"
"MS Shell Dlg"="msyh"
"MS Shell Dlg 2"="msyh"
"Tms Rmn"="msyh"
EOF
WINE=${WINE:-wine} WINEPREFIX=${WINEPREFIX:-$HOME/.wine} $WINE renano /tmp/FontSubstitutes 2> /dev/null
rm /tmp/FontSubstitutes
```

#### 3D 性能设置

参考：[一个软件，管理全平台游戏，协助显卡级调整，榨干电脑性能](https://www.bilibili.com/video/BV1EE411p78Y)

​			[Lutris Doc](https://github.com/lutris/docs)

1、启用 CSMT 。开启 CSMT 后，wine 会使用一个单独的线程调用 OpenGL ，从而显著提升性能。3.2 之后默认是启用的，如果使用 staging 版，打开 `Configure Wine(winecfg)`，选择 `Staging` 标签并启用 `CSMT` 。其他版本终端导入注册表启用。

```shell
cat << EOF > /tmp/csmt
REnano4

[HKEY_CURRENT_USER\Software\Wine\Direct3D]
"csmt"=dword:00000001
EOF
WINE=${WINE:-wine} WINEPREFIX=${WINEPREFIX:-$HOME/.wine} $WINE renano /tmp/csmt 2> /dev/null
rm /tmp/csmt
```

但是 csmt 可能会让某些应用性能降低，将上面注册表中的 `00000001` 改为 `00000000` 导入。

2、强制使用 OpenGL 运行游戏。某些游戏可能具有 OpenGL 模式，该模式可能比其默认 DirectX 性能模式更好。虽然不同的应用程序启用 OpenGL 渲染的步骤是不同的，但是许多游戏接受 `-opengl` 参数。故，

```shell
 wine /path/to/3d_game.exe -opengl
```

3、安装驱动及 vulkan 支持库

NVIDA 显卡，安装需要的驱动和 vulkan 支持库，这里使用 Manjaro 自带的驱动安装命令安装驱动即可。

```shell
sudo pacman -S --needed nvidia-dkms nvidia-utils lib32-nvidia-utils nvidia-settings vulkan-icd-loader lib32-vulkan-icd-loader
```

`nvidia-settings` 找不到，可忽略。

Intel 显卡，vulkan api及32位游戏支持

```shell
sudo pacman -S --needed lib32-mesa vulkan-intel lib32-vulkan-intel vulkan-icd-loader lib32-vulkan-icd-loader
```

4、安装 DirectX 到 vulkan 的兼容层 DXVK 。DXVK 可以提高性能，在某些情况下，甚至可以更好地兼容，性能会比在 Windows 平台更好。**但是DXVK 覆盖了 DirectX 10 和 11 的 DLL，这可能被某些在线游戏的反作弊系统视为作弊行为，并可能导致封禁帐户。并且 DXVK 并不代表兼容所有游戏**

```shell
yay -S dxvk-bin
```

使用下面命令激活 wine 的 DXVK

```shell
WINEPREFIX=~/.wine setup_dxvk install
```

卸载 DXVK

```
WINEPREFIX=~/.wine setup_dxvk uninstall
```

下载[链接](https://github.com/doitsujin/dxvk/files/4506558/fr-041_debris.zip)中文件，运行 `DXVK_HUD=full wine fr-041_debris.exe` 测试 DXVK 是否可以被加载。

并不是所有游戏都支持 DXVK，比如 [PVZ](https://github.com/HenryJk/PvZWidescreen) ，反而导致 CPU 占有率变高很多（1x% -> 6x%）。参考 [issue](https://github.com/doitsujin/dxvk/issues/1585)，但是不适用于 PVZ，在 PVZ 中关闭 3D 加速使 CPU 占有率降低，但还是比没装 DXVK 高很多（1x% -> 3x%）。在 lutris 中打开占有率又会恢复正常。

5、为了更好运行 Windows 游戏，补全 wine 依赖

```shell
sudo pacman -S giflib lib32-giflib libpng lib32-libpng libldap lib32-libldap gnutls lib32-gnutls \
mpg123 lib32-mpg123 openal lib32-openal v4l-utils lib32-v4l-utils libpulse lib32-libpulse libgpg-error \
lib32-libgpg-error alsa-plugins lib32-alsa-plugins alsa-lib lib32-alsa-lib libjpeg-turbo lib32-libjpeg-turbo \
sqlite lib32-sqlite libxcomposite lib32-libxcomposite libxinerama lib32-libgcrypt libgcrypt lib32-libxinerama \
ncurses lib32-ncurses opencl-icd-loader lib32-opencl-icd-loader libxslt lib32-libxslt libva lib32-libva gtk3 \
lib32-gtk3 gst-plugins-base-libs lib32-gst-plugins-base-libs vulkan-icd-loader lib32-vulkan-icd-loader
```

6、安装 gamemode [lutris](https://lutris.net/)

```shell
yay -S gamemode lib32-gamemode lutris
```

其中  `lib32-gamemode` 一般用不到，可忽略，`lutris` 多平台游戏管理。

7、lutris 优化

打开 lutris，依次 `首选项 -> 全局设置 -> 环境变量`，添加以下环境变量

```
# key value

LD_PRELOAD /usr/lib/libgamemodeauto.so.0

# 下面环境变量建议针对游戏单独添加
# For NVIDIA GPU
__GL_THREADED_OPTIMIZATION 1
__GL_SHADER_CACHE 1
__GL_SHADER_DISK_CACHE_PATH=/home/$USER/.cache/lutris_gpu_cache
# For AMD GPU
mesa_glthread true
```

**进一步提升性能**，可在 lutris中 使用带有 tkg 版的 wine（默认启用ESync），并打开 ESync 。

使用 ESync 需要设置下内核，终端输入

```shell
ulimit -Hn
```

没有 ulimit 的话，安装一下。如输出结果 `>= 524288`，则无需继续设置。

如果不是，编辑 `/etc/systemd/system.conf`，搜索 `DefaultLimitNOFILE` ，使其结果为 `1024:524288`，并去掉前面的 `#` ，如下：

```properties
DefaultLimitNOFILE=1024:524288
```

编辑 `/etc/systemd/user.conf`，搜索 `DefaultLimitNOFILE` ，使其结果为 `524288`，并去掉前面的 `#` ，如下：

```properties
DefaultLimitNOFILE=524288
```

硬件加速 CUDA 和 OpenCL，直接使用 AUR 安装即可。

安装 CUDA ，参考达芬奇的安装。

```shell
yay -S nvidia cuda
```

安装 OpenCL，首先必须要安装 `opencl-headers`，不过一般已经安装过了，对于 AMD 显卡，安装 `opencl-mesa` 和 `lib32-opencl-mesa`，对于 NVIDIA 显卡，需要安装 `opencl-nvidia` 和 `lib32-opencl-nvidia`。

## 美化

### grub 主题

`vimx` ` stylish` `dark-matter`

更新命令

```shell
sudo update-grub
```

### gnome 主题

[gnome-look](https://www.gnome-look.org/) 下载主题（gtk-theme）和图标，手动复制到对应位置，安装后需要注销后才有效果

```shell
# gnomeshell theme 、gtk3/4 theme
sudo cp -r 解压后的主题包 ~/.local/share/themes/
# 图标、鼠标
sudo cp -r 解压后的图标包 ~/.local/share/icons/
```

#### 不错的主题

[Orchis](https://www.gnome-look.org/p/1357889)	[Sweet](https://www.gnome-look.org/p/1253385)	[Colloid-Dark-Nord](https://www.gnome-look.org/p/1661959)	[Flat-Remix-Darkest](https://www.gnome-look.org/p/1214931)	[Nordic](https://www.gnome-look.org/p/1267246)	[Kripton](https://www.gnome-look.org/p/1365372)

#### 不错的鼠标

[Volantes](https://www.gnome-look.org/p/1356095)	[Capitaine](https://www.gnome-look.org/p/1148692)	[Vimix](https://www.gnome-look.org/p/1358330)	[Sweet](https://www.gnome-look.org/p/1393084)	[Future](https://www.gnome-look.org/p/1457141)	[Qogir](https://www.gnome-look.org/p/1366182)	[Breeze Serie](https://www.gnome-look.org/p/999991)

#### 不错的图标

[Papirus](https://www.gnome-look.org/p/1166289)	[Tela](https://www.gnome-look.org/p/1279924)	[Reversal](https://www.gnome-look.org/p/1340791)	[Kora](https://www.gnome-look.org/p/1256209)

### Plymouth

参考：[Plymouth ArchWiki](https://wiki.archlinux.org/title/Plymouth_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))

https://www.cnblogs.com/guochaoxxl/p/14173124.html

1、安装Plymouth

```shell
yay -S plymouth gdm-plymouth
```

注意：使用N卡切换工具`OptimusManager` 需要`gdm-prime`这与`gdm-plymouth`冲突，无法同时使用。可安装`gdm-plymouth-prime`兼顾两者的功能。

```shell
yay -S gdm-plymouth-prime
```

2、设置钩子

编辑文件`/etc/mkinitcpio.conf`

```shell
sudo gedit /etc/mkinitcpio.conf
# 添加plymouth
HOOKS="base udev plymouth ... "
```

3、为引导程序设置`quiet splash`参数

```shell
sudo gedit /etc/default/grub
# 更改GRUB_CMDLINE_LINUX_DEFAULT为quiet splash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash vt.global_cursor_default=0 ..."
```

然后在终端输入

```shell
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

重新生成`grub.cfg`引导配置文件。

几个不错的：

https://www.gnome-look.org/p/1466107

https://www.gnome-look.org/p/1495885



### KDE 美化

#### 路径说明

```shell
~/.local/share/plasma/desktoptheme # 存放plasma主题
~/.local/share/plasma/look-and-feel/ # 存放全局主题
~/.local/share/plasma/plasmoids/ # 存放插件
~/.local/share/color-schemes # 存颜色
```

以上目录如果没有就自行创建。

## 相关文章

[ArchLinux 安装](https://archlinuxstudio.github.io/) 

[酷安版manjaro教程](https://www.kdocs.cn/l/cjavaV5iPwa0) （含 KDE 美化）

