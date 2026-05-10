# 随身 WiFi (UZ801/MSM8916) 刷机全记录：Debian 与 OpenWrt

刷机过程工具已上传百度网盘：通过网盘分享的文件：高通410，uz801板子刷linux和op系统.7z
链接: https://pan.baidu.com/s/1gAqZ8hHzwsOAgtsvRGzQ3Q?pwd=kdyb 提取码: kdyb
--来自百度网盘超级会员v5的分享

## 第一部分：刷入 Debian 系统（基础 Fastboot 模式）

### 开启 ADB 调试

将随身 WiFi 插入电脑，连上它发出的 WIFI。
打开浏览器访问 192.168.100.1/usbdebug.html，提示成功即可打开 ADB。

### 进入 Fastboot 模式

打开包含 adb 所在的文件夹（如 platform-tools），在路径地址栏输入 cmd 并回车打开命令行。
输入 adb devices，确认返回了设备 ID，代表连接成功。
输入 adb reboot bootloader，设备会重启并进入 Fastboot 模式。
注意：打开 Windows 设备管理器，必须看到设备被识别为 Android 相关的 bootloader 设备。如果有黄色感叹号，需要手动强行更新为安卓驱动。
强行更新驱动（硬核救场）：用 Zadig 查看该设备的 VID 和 PID。然后用 VSCode 打开 android_winusb.inf，
找到 [Google.NTamd64] 和 [Google.NTx86] 这两个标签，在它们下方插入以下代码：
%SingleBootLoaderInterface% = USB_Install, USB\VID_xxxx&PID_xxxx
%SingleAdbInterface% = USB_Install, USB\VID_xxxx&PID_xxxx
（把 xxxx 换成你查到的实际值）。保存后，在设备管理器右键该设备 -> 更新驱动程序 -> 浏览我的电脑以查找 -> 强制选择你改好的 inf 文件即可成功装上驱动。）
[Google.NTx86] 是给 32 位系统看的，[Google.NTamd64] 是给 64 位系统看的

### 下载 OpenStick 基础包

前往 GitHub OpenStick 项目的 v1 发布页，下载以下三个文件：
base-generic.zip
boot-uz801.img
debian.zip

### 刷入底包与系统

解压 base-generic.zip，进入 base 目录，双击执行 flash.bat 脚本。（每次按回车前，可以新开一个 cmd 用 fastboot devices 命令确认设备处于正常连接状态）。
解压 debian.zip，进入解压出的 debian 目录。
把下载的 boot-uz801.img 重命名为 boot.img，并将其复制到 debian 目录中，覆盖原有的 boot.img。
双击执行 debian 目录下的 flash.bat 脚本，按提示回车完成系统刷入。

Debian 关机须知（防止变砖）
Debian 系统带有文件日志记录，不能直接拔插头断电。需要通过 SSH 连接（例如 ssh vv@192.168.1.200），输入 sudo poweroff 命令，等终端提示断开连接后，心里默数 5 到 10 秒，再物理拔出设备。

## 第二部分：刷入 OpenWrt 系统（高阶 9008 EDL 模式，无损保基带）

### 基础环境准备

Windows 系统需提前安装 Python 3 和 Git 工具，并确保添加到了系统环境变量。
下载 Zadig 驱动替换工具（单文件免安装版备用）。

### 安装开源 EDL 刷机环境

解压你下载好的 OpenWrt 固件包（带 flash.sh 和各个 img 的那个）。
进入固件包里的 edl 文件夹，在上方地址栏输入 cmd 回车。
运行命令：pip install . （注意最后有一个英文句号，代表当前目录）。
安装跑完后，输入 edl -h，如果出现一大堆命令参数说明，代表环境安装成功。
（Windows 特有坑点避让：如果在上述安装时报语法错误，是因为 Git 把文件识别成了软链接。解决办法是：把 edl 文件夹根目录下的 edl 无后缀文件，强制复制并覆盖到 edlclient 目录下的 edl.py 文件，然后重新运行 pip install . 即可）。

### 准备高通引导文件 (Loaders)

退回到 OpenWrt 固件包的主目录。
在空白处右键选择 Open Git Bash here。
为了防止 GitHub 超时，输入命令使用国内镜像源拉取：
git clone https://ghp.ci/https://github.com/bkerler/Loaders.git
下载完成后点开 Loaders 文件夹检查，里面必须直接是 amazon、huawei、lenovo 等各个厂商的文件夹，绝对不能出现嵌套（即不能套着一个 Loaders-main 文件夹，如果套了，把里面的东西全剪切到外面这一层来）。

### 进入 9008 模式并替换底层驱动

开启 ADB 调试
将随身 WiFi 插入电脑，打开浏览器访问 192.168.100.1/usbdebug.html，提示成功即可打开 ADB。

打开包含 adb 所在的文件夹（如 platform-tools），在路径地址栏输入 cmd 并回车打开命令行。
输入 adb devices，确认返回了设备 ID，代表连接成功。
输入 adb reboot edl 进入 9008 底层刷机模式。

打开 Zadig 工具，点击顶部菜单 Options，勾选 List All Devices。
在下拉列表中找到并选中 Qualcomm HS-USB QDLoader 9008 设备（有时带有 COM 端口号）。
在绿色箭头右侧的 Driver 处，通过上下箭头选择 libusb-win32，然后点击 Replace Driver 按钮。
替换完成后，打开 Windows 设备管理器，该设备会显示为 QHSUSB_BULK，且前面没有黄色感叹号。

解决协议假死与暴力注入引导（最关键一步）
拔掉板子！按住板子上的复位小按键，重新插入电脑的 USB 口，这步是为了重置刚才可能卡死的 Sahara 握手协议。
在 OpenWrt 固件包主目录空白处，右键选择 Open Git Bash here。
依次执行以下三条命令（逐行复制并回车）：

第一条，从刚才下载的库里抓取通用的 8916 引导文件，放到当前目录并改名：
find Loaders -type f -name "8916.mbn" | head -n 1 | xargs -I {} cp {} my_loader.mbn

第二条，暴力修改固件包里那个不听话的 shell 脚本，把引导文件强塞进每一行刷机命令中：
sed -i 's/edl /edl --loader=my_loader.mbn /g' openwrt-msm89xx-msm8916-yiming-uz801v3-flash.sh

第三条，打印分区表，直接验证底层连通性：
edl printgpt --loader=my_loader.mbn

只要上面最后一条命令跑完，屏幕上哗啦啦滚出 aboot, boot, system 等分区列表，且最下方出现 Total disk size 字样，说明芯片底层通道已被完美打通，可以进行下一步！

### 一键无损刷入 OpenWrt

趁着通道打开，继续在 Git Bash 窗口中执行刷机脚本：
./openwrt-msm89xx-msm8916-yiming-uz801v3-flash.sh
运行后提示 Continue with flashing? 时，输入大写的 Y 并回车。
脚本会自动提取备份原机基带（存入 saved 文件夹），接着刷入系统，最后恢复基带。
在此过程中，如果出现红色的 Error 提示（如 payload to target size is too large，或者最后的 USBError Pipe error 断连），这都是正常的降级握手和重启现象，直接无视即可。
直到最后看到 Flash completed successfully，板子会自动切断连接并重启。

解决网络冲突与登入后台
这一步必须严格操作，防止子网冲突：拔掉电脑上现有的网线，并关闭电脑的 WiFi。确保电脑没有任何其他网络。
将刷好的随身 WiFi 从电脑拔下，默数 3 秒后重新插入 USB 口。
耐心等待 1 到 2 分钟，让系统生成密钥并启动，直到电脑的网络中心出现新的 RNDIS 虚拟网卡（UsbNcm Host Device），并且分配到了 IPv4 地址。
打开浏览器，访问默认网关 192.168.1.1。
输入默认账号 root，密码 password（或留空），即可成功登入 OpenWrt (LuCI) 后台。

修改静态 IP 并融入主网络
在 OpenWrt 后台顶部导航栏，点击 网络，再点击 接口。
找到 LAN 接口，点击右侧蓝色的 修改。
将 IPv4 地址改为与你家主路由器不冲突的地址，例如 192.168.1.200。
滑到页面最底部，点击 保存并应用。
此时浏览器页面会失去响应（因为 IP 已经改变）。直接将随身 WiFi 拔下，重新插入电脑，让电脑获取新的 200 网段 IP。
现在，你可以恢复电脑原本的 WiFi 和网线连接。以后在任何同一局域网下的设备，都可以直接浏览器访问 192.168.1.200 来管理你的随身 WiFi 软路由了。
