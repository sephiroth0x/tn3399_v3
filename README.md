# TN3399_V3

## 规格参数

|   部件名称    |       芯片型号       |                           备注说明                           |
| :-----------: | :------------------: | :----------------------------------------------------------: |
|      CPU      |        rk3399        | Dual-core Cortex-A72 up to 1.8GHz;Quad-core Cortex-A53 up to 1.4GHz;Mali-T864 GPU |
|      RAM      |       K4B8G16        |               Dual-channel DDR3 1GB;板载 4 颗                |
|     Flash     | SanDisk SDINBDG4-16G |                           eMMC 5.1                           |
|      PMU      |        RK808D        |                                                              |
|   Ethernet    |       RTL8211E       |                      10/100/1000 Base-T                      |
|    WIFI+BT    |        AP6255        |               WIFI IEEE802.11 a/b/g/n/ac;BT4.2               |
|   SATA 3.0    |        JMS578        |                                                              |
|    USB 2.0    |        FE1.1s        |              板载 TYPE A 插座 x 2;接插件插座x5               |
|    USB 3.0    |       VL817-Q7       |                     板载 TYPE A 插座 x 2                     |
|     UART      |      SP3232EEN       |                                                              |
| HDMI 2.0+LVDS |  358775G + ALC5640   |                                                              |
|   Audio PA    |        NS4258        |                            5W x 2                            |

## 烧写调试

### 固件烧写

#### 烧写到 eMMC

>   相关工具：https://github.com/rockchip-linux/rkbin

1.  创建 udev 规则。

    可以避免无 sudo 权限时不能发送指令或烧写报错 “Creating Comm Object failed!”。

```shell
echo 'SUBSYSTEM=="usb", ATTR{idVendor}=="2207", MODE="0660", GROUP="plugdev"' | sudo tee /etc/udev/rules.d/51-android.rules
```

2.  进入烧写模式。

    连接 Micro-USB，长按 RECOVER 键并上电，如果为 Android 系统此时应为 Loader 模式，其他系统可能为 MaskRom 模式。

```shell
# 查看连接的设备
./rkbin/tools/upgrade_tool ld
# 或使用 rkdeveloptool
# 因 ./rkbin/tools/rkdeveloptool 这个不支持 ld 命令故需要重新编译 rkdeveloptool
git clone https://github.com/rockchip-linux/rkdeveloptool.git
sudo apt install -y libudev-dev libusb-dev dh-autoreconf libglib2.0-dev
cd rkdeveloptool && autoreconf -i && ./configure && make
sudo mv rkdeveloptool /usr/local/bin/
rkdeveloptool ld

# 应会有如下设备信息
DevNo=1 Vid=0x2207,Pid=0x330c,LocationID=301    Loader
# 或
DevNo=1 Vid=0x2207,Pid=0x330c,LocationID=301    MaskRom

# Loader 模式进入 MaskRom 模式方法
./rkbin/tools/upgrade_tool rd 3
# 或
rkdeveloptool rd 3
```

3.  烧写固件。

    烧写 u-boot 需要在 MaskRom 模式下进行，否则报错 “The device does not support this operation!”。

```shell
# u-boot
./rkbin/tools/upgrade_tool db ./RK3399MiniLoaderAll_V1.05.bin
# 或
rkdeveloptool db ./RK3399MiniLoaderAll_V1.05.bin

# system
./rkbin/tools/upgrade_tool wl 0x0 ./system.img
# 或
rkdeveloptool wl 0x0 ./system.img
```

#### 烧写到 SDCard

>   也可以使用图形化烧写工具 balena-etcher-electron
>
>   https://github.com/balena-io/etcher/releases

```shell
# sdX 为 sdcard 对应的块设备文件
sudo dd if=system.img of=/dev/sdX bs=4M oflag=sync status=noxfer
```

### 串口调试

>   测试了市面上常见的几款 CH340G、PL2303HX 方案的串口，均存在只能读不能写问题，建议使用 CP2104 方案的串口代替。

#### 通信软件

>   也可以使用 minicom 、putty、SecureCRT 等工具。

```shell
# 解决串口权限问题
sudo usermod -a -G dialout $USER
# 重新登录后生效
reboot

# 安装串口通信软件
sudo apt install -y python3-serial

# 打开串口
miniterm /dev/ttyUSB0 1500000

# 退出 miniterm 快捷键为 Ctrl + ]
```

## 系统配置

### 网络

```shell
# WiFi 配置
nmcli dev wifi connect "hotspot-name" password "password"
```

### 本地化

```shell
# 安装英文与简体中文语言包
sudo apt install -y language-pack-en language-pack-zh-hans
# 设置本地化
sudo dpkg-reconfigure locales

# 配置时区
sudo apt install -y tzdata
sudo dpkg-reconfigure tzdata
# 同步硬件时间
sudo hwclock -s
```

## 常见问题

Q：操作正确，能进 MaskRom 也能更新系统 ，但没法更新 u-boot。

A：连接串口并打开串口调试软件，开机后迅速在调试窗口按任意键，打断 uboot 启动（如果不能打断请更换 CP2104 方案的串口），执行如下命令破坏 uboot，重启后会自动进入 MaskRom 模式。

```shell
mmc dev 0
mmc erase 0 200
reset
```

---

Q：使用 apt 更新软件包时出现如下警告：

>   perl: warning: Setting locale failed.
>   perl: warning: Please check that your locale settings:
>           LANGUAGE = (unset),
>           LC_ALL = (unset),
>           LANG = "zh_CN.UTF-8"
>       are supported and installed on your system.
>   perl: warning: Falling back to the standard locale ("C").

A：参考 系统配置-本地化 一节安装相应语言包即可解决。

---

