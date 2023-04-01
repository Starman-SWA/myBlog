---
title: 记一次几乎踩了所有坑的Ubuntu双系统安装
date: 2021-01-15 01:20:37
tags: Linux
copyright: true
categories:
- Linux
---


# 前言

我的破本本用虚拟机跑Linux实在是顶不住，于是我昨天晚上一时兴起，想要给笔记本装 Windows 10 + Ubuntu 20.04 LTS 双系统。我的笔记本是UEFI引导，128GB SSD + 1TB机械硬盘，分区表格式都是GPT，Windows 10装在SSD上，计划在机械硬盘上分出128GB装Ubuntu（至于为什么才分这么点，显然是因为1T的硬盘几乎快被游戏填满了，逃）。

要么是我电脑的硬件兼容性太差，要么是我运气不佳，这次安装我居然步步踩坑，从昨天晚上折腾到今天晚上才把所有问题解决。**这篇文章记录了我踩坑的全过程，仅供参考，切勿当作教程使用。**

# 一、安装过程

首先进行分区，我参考网上一篇教程，在SSD上划分出200M作为EFI引导分区，然后在机械硬盘上划分出128GiB空间作为Ubuntu主分区（注意：划分过程需要耐心等待，印象中花费了超过半个小时）。为了方便随后的安装过程中根据分区大小和位置找分区，我把分区结果拍了下来。

{% asset_img img1.jpg %}

然后就可以使用U盘启动Ubuntu安装程序了：我从官网上下载了Ubuntu 20.04 LTS镜像，刻录到U盘当中。进UEFI BIOS（以下简称BIOS）把安全启动关闭，把U盘启动优先级调为最高，进入Ubuntu安装向导。

## 坑1. 安装向导卡住

此时，第一个坑来了。在选择”正常安装“和”最小安装“的界面，我点击继续后卡住不动了。

<img src="img2.jpg" width=480 height=360>

解决方法是：在进入安装向导之前的grub菜单处按e编辑启动选项，将`quiet splash ---`改为`quiet splash nomodeset`，即增加nomodeset启动选项。nomodeset的含义如下：

> The newest kernels have moved the video mode setting into the kernel.   So all the programming of the hardware specific clock rates and  registers on the video card happen in the kernel rather than in the X  driver when the X server starts.. This makes it possible to have high  resolution nice looking splash (boot) screens and flicker free  transitions from boot splash to login screen. Unfortunately, on some  cards this doesnt work properly and you end up with a **black screen**. Adding the nomodeset parameter instructs the kernel to not load video drivers and use BIOS modes instead until X is loaded. 

大意为，在新版的kernels当中，加载视频驱动的工作从原本的在加载图形化服务时进行，改为在加载kernel的时候就进行了，而部分显卡不兼容这一设定，可以用nomodeset选项改回原来的模式。

继续进行安装向导，安装方式选择“其他”，手动进行分区。我将128GiB的空间划分为swap、/和/home三个分区，其中swap分区的大小参照Ubuntu官网划分为11GB，剩下的按照1:3划分给/和/home。swap分区作为虚拟内存的一部分，Ubuntu官网的推荐值为：如果使用休眠功能，最小值不小于RAM，否则不小于sqrt(RAM)；其最大值不应超过RAM的两倍。此外，在分区界面，我将启动盘设置为SSD上分出来的那200M。分区完成，进行后续的设置后就完成安装了（由于还未设置代理，安装过程中我skip了需要下载的内容）。

## 坑2. 开机后直接进入Windows 10

但是当我兴冲冲地重启电脑后发现：根本没有出现操作系统选择列表，开机后同原本一样直接进入Windows 10，第二个坑来了。

在Windows 10自带的磁盘管理页面中，我能看到机械硬盘上128GiB的空间被划为三个部分，同时SSD上那200M由空闲状态变为EFI系统文件状态，因此分区没有问题。会不会是引导的问题？我使用启动盘中的Try Ubuntu功能进入Ubuntu试用版，使用以下指令安装并运行了引导修复工具：

```sh
sudo add-apt-repository ppa:yannubuntu/boot-repair -y
sudo apt-get update
sudo apt-get install boot-repair -y
```

修复完成后弹出以下窗口：

<img src="img3.jpg" width=480 height=360>

按照第一句话，我应该将在BIOS当中把Ubuntu引导文件加入到启动项当中，但是我的BIOS并没有添加自定义启动项的功能。按照第二句话，我应该把Ubuntu启动项设置到Windows前面，但是我并没有在BIOS的启动选项当中看到Ubuntu和Windows，只有一个笼统的“操作系统的启动管理员”。因此我按照第三句话，在Windows命令行中将启动路径设置为Ubuntu的引导文件，没有效果。

我翻遍了BIOS，尝试了包括清除安全启动key和设置密码之内的多种方法都没有找到添加启动项的地方，正当我不知道应该如何继续时，我发现了启动项“操作系统的启动管理员”左边的小箭头。。。原来这个选项是可以细分的，按Enter果然弹出了Windows和Ubuntu两个启动项。这可能是我整个安装过程中犯过的最蠢的错误：对自己电脑的BIOS操作不熟悉。我将Ubuntu启动项设置到Windows前面之后，果然不再进入Windows了，第二个坑解决。

<img src="img4.jpg" width=480 height=360>

## 坑3. 开机后进入grub命令行

第二个坑一解决，第三个坑随之而来：开机后进入的是grub命令行界面而不是grub菜单界面。我依然怀疑是引导出了问题，因此插入U盘，选择擦除旧系统再安装Ubuntu的选项重装系统。由于对引导缺乏认识，我也不清楚之前的引导是否正确安装，因此这次我选择让安装程序自动擦除旧系统后自动分区并安装。结果还是一样，依旧是开机后进入grub命令行。此时我再次查看Windows 10的磁盘管理，发现swap、/、/home的分区大小几乎没有改变，但是在机械硬盘上增加了一个EFI引导分区，且SSD上原来分配的200M也还在。

在grub命令行输入以下指令进入grub菜单：

```shell
ls
# 此时会显示磁盘列表
ls 磁盘编号/boot/grub # 对所有磁盘编号尝试此指令，直至提示找到文件
set root=找到文件的磁盘编号
set prefix=找到文件的磁盘编号/boot/grub
insmod normal
normal
```

此时终于能进入grub菜单，选择Ubuntu系统。

## 坑4. Ubuntu系统只有鼠标

紧接着第四个坑又来了：Ubuntu系统进入后，只有鼠标和背景，没有图标，无法进行任何操作。

经过尝试，解决方法是使用安全模式进入Ubuntu，修改/etc/default/grub文件中的`quiet splash`改为`quiet splash nomodeset`，然后执行`sudo update-grub`指令更新grub。居然还是同样的问题，只不过这个指令如果在grub菜单中按e修改是一次性的，只有修改/etc/default/grub文件才能永久生效。

踩完了四个坑，终于能正常进入Ubuntu了。不过，一进入系统，我就明显感觉屏幕非常暗。尝试笔记本的亮度调节功能键，无效；进入Ubuntu设置，居然找不到调节亮度的地方！我决定先配置代理，能够浏览Google之后再来解决亮度的问题。



顺利进入Ubuntu后，我使用`df -lh`和`fdisk -l`指令查看分区情况，发现/boot/efi被挂载在Windows 10的EFI引导分区上。此时，SSD上我划分出来的200M是闲置的，于是我使用Windows 10的DiskGenius软件将200M合并回Windows 10的C盘。至此，Ubuntu系统可以算是顺利安装完成。

# 二、配置代理

我使用的代理工具是Clash。从Github仓库下载Clash的Release版本，解压后获得可执行文件。我存放在/opt/clash文件夹当中。第一次运行Clash会自动生成配置文件config.yaml和IP库文件Country.mmdb，文件存放路径为`~/.config/clash`。我将自己的config.yaml和Country.mmdb覆盖`~/.config/clash`目录中的文件，Clash即可正常运行。可在浏览器中输入clash.razord.top进入Clash的Web界面。

## 1. GNOME代理

进入系统设置-网络-网络代理，模式设置为手动，填入相应的IP和端口号即可。此时，Firefox可以访问Google。

## 2. 终端代理和开机启动

在系统设置当中配置代理只能让GNOME应用使用，而终端程序还要单独配置。配置方法十分简单，只需设置两个环境变量：

```shell
export http_proxy=http://127.0.0.1:7890
export https_proxy=http://127.0.0.1:7890
```

此时在非root账户当中可以使用`wget google.com`指令获取谷歌主页，然而在root账户中还不行，因为进入root账户时默认不会保存环境变量的值，解决方法是在/etc/sudoers中加入：

```bash
Defaults env_keep += "http_proxy https_proxy no_proxy"
```

再来配置开机启动，开机启动首先需要以下shell脚本：

```shell
#!/bin/bash
gsettings set org.gnome.system.proxy mode 'manual' # ubuntu网络模式配置为手动
cd /opt/clash  # 切换到Clash的目录
./clash -d /opt/clash &  # 在后台执行clash客户端
echo "Clash started!"  # 启动成功的提示
```

-d后面的路径设置为存放config.yaml和Country.mmdb的路径，&表示后台运行。

将上述脚本连同设置两个环境变量的指令一同添加到~/.profile文件当中，即可实现登陆当前账户时自动运行Clash并设置系统代理。由于/etc/sudoers文件也进行了修改，root账户也能够走代理。

# 三、无法设置亮度

再回过头来解决亮度的问题，我先后尝试过以下几种方法：

## 1. 修改grub

在/etc/default/grub中设置以下参数并更新grub：

```shell
GRUB_CMDLINE_LINUX="acpi_backlight=vendor"
```

失败。

## 2. 使用亮度调节工具

我先后尝试了brightness-controller和brightness-controller-simple两款工具，均失败。

## 3. 修改亮度文件

与亮度有关的文件存放在/sys/class/backlight文件夹中，然而，我的这个文件夹是空的，该方法失败。

## 4. 更新驱动

一开始我使用这条指令更新驱动：

```shell
sudo ubuntu-drivers autoinstall
```

然而不仅没有效果，因为这条指令为我安装了新版本的内核，使系统多内核共存，我还无法进入系统。我只能手动选择内核版本，设置为旧版系统启动，然后删除新版本。几个小时后，我阴差阳错地在“应用-软件和更新-附加驱动“当中看到了Nvidia显卡的驱动选择列表。默认选择的是开源驱动，我将其改为最新版的专用的、tested的驱动。

驱动更新后，重启电脑，居然又进不去系统了，提示"unable to bind the codec"。解决方法为：将/etc/default/grub文件中的quite、slash、nomodeset三个参数都删除。quiet splash的含义如下：

> The splash (which eventually ends up in your /boot/grub/grub.cfg ) causes the splash screen to be shown.
>
> At the same time you want the boot process to be quiet, as otherwise all kinds of messages would disrupt that splash screen.
>
> Although specified in GRUB these are kernel parameters influencing the loading of the kernel or its modules, not something that changes GRUB  behaviour. The significant part from GRUB_CMDLINE_LINUX_DEFAULT is CMDLINE_LINUX.

光看说明我也不太明白，不过三个参数均删除之后确实可以进入系统，而且由于更新了驱动，亮度调节的滑块也出现了。Fn功能键也能正常调节亮度。

# 四、扬声器无声

我写这篇博客写到一半时，突然想听歌，于是又发现了另外一个问题：扬声器之前有声音的，现在没有了。好家伙，连写博客都不能让我好好写。我运行自带的alsamixer工具发现，Headphone音量被设置为0，只需设置成100即可恢复正常。我怀疑是显卡驱动更新的时候更改的。

# 五、无法挂起

在挂起状态按任意键唤醒之后，屏幕左上角会显示一个光标，然后卡在这个页面。尝试网上的教程安装laptop-mode-tools也无法解决，最后发现将显卡驱动降级即可。我室友的Surface Book也有同样的问题，但是它的核显支持的驱动较少，无法用这个方法解决。

# 六、总结

这次安装踩的坑是真的多，不过我的第一反应是想用这篇博客记录下这个过程，而不是放弃。可能，这就是Linux的魅力吧。

生命不止，折腾不息。