# 简介
无屏幕，无网线，串口玩树莓派

# 安装操作系统

## 下载[Manjaro-ARM](http://manjaro-arm.org/downloads.php)操作系统
### 本文以[Manjaro-ARM-minimal-rpi3-18.10](//rwthaachen.dl.osdn.jp/storage/g/m/ma/manjaro-arm/rpi3/minimal/18.10/Manjaro-ARM-minimal-rpi3-18.10.zip)为例  
### 校验.zip文件完整性
### 解压.zip文件，得到.img镜像文件

---

## 下载[Etcher](https://www.balena.io/etcher/)镜像烧录工具
![Etcher](resource\Etcher.png "Etcher")

---

## 烧录镜像到SD卡

---

## 把SD卡插入树莓派，串口连接树莓派&电脑
>**注意**：使用电源供电时不要连**VCC** <--->  **5V**  
**Pin14**(TXD)(外侧右数4) <---> **RXD**  
**Pin15**(RXD)(外侧右数5) <---> **TXD**  
**GND**(Pin15对面)(内侧侧右数5) <---> **GND**  

![raspberry-pi-pinout](resource\raspberry-pi-pinout.png "raspberry-pi-pinout")
### 查看COM号  
![COMnumber](resource\COMnumber.png "COMnumber")
### 配置[PuTTY](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html)  
![PuTTYConfiguration](resource\PuTTYConfiguration.png "PuTTYConfiguration")
### 点击**Open**，然后接通树莓派电源，第一次启动会自动调整分区大小然后重启，耐心等待启动信息蹦完
### 使用用户名：**root**，密码：**root** (不显示) 登录

---

# 初始调校

## 熄灭指示灯
nano  /usr/local/sbin/ledoff  

    sudo echo none > /sys/class/leds/ACT/trigger
    sudo echo 0 > /sys/class/leds/PWR/brightness

chmod +x  /usr/local/sbin/ledoff  
ledoff  
### 开机自动熄灭
nano /usr/lib/systemd/system/rc-local.service

    [Unit]
    Description=/etc/rc.local Compatibility
    ConditionPathExists=/etc/rc.local
    [Service]
    Type=forking
    ExecStart=/etc/rc.local
    TimeoutSec=0
    StandardOutput=tty
    RemainAfterExit=yes
    SysVStartPriority=99
    [Install]
    WantedBy=multi-user.target

nano /etc/rc.local

    #!/bin/bash
    #
    # /etc/rc.local: Local multi-user startup script.
    #

    echo "ledoff .................................................................."
    echo none > /sys/class/leds/ACT/trigger
    echo 0 > /sys/class/leds/PWR/brightness

    echo "block bluetooth ........................................................."
    rfkill block 0 1


    exit 0

chmod +x /etc/rc.local
systemctl enable rc-local.service

---

## 恢复指示灯
nano  /usr/local/sbin/ledon  

    sudo echo heartbeat > /sys/class/leds/ACT/trigger
    sudo echo 1 > /sys/class/leds/PWR/brightness

chmod +x  /usr/local/sbin/ledon  
ledon  

---

## 连接WiFi，并配置静态IP
>参考：https://wiki.archlinux.org/index.php/Systemd-networkd  
不得不用[**火星WiFi**](http://zkytech.com/)，否则辣鸡校网会强制掉线
### 生成密文密码
wpa_passphrase  2333 "123456789" > /etc/wpa_supplicant/wpa_supplicant-wlan0.conf  
### 删除明文密码 #psk="123456789"
nano  /etc/wpa_supplicant/wpa_supplicant-wlan0.conf  

    network={
        ssid="2333"
        psk=294fa263156b3e82ce42a7c71936dcfed54460d4596d1d0f32f3c9745dd36510
    }

### 配置静态IP
nano /etc/systemd/network/25-wireless.network

    [Match]
    Name=wlan0

    [Network]
    Address=192.168.191.101/24
    Gateway=192.168.191.1
    DNS=192.168.191.1

systemctl enable wpa_supplicant@wlan0.service  
systemctl enable systemd-networkd.service  
reboot  
### 检查连接
ip a  
ping baidu.com

---

## 设置时间 & NTP
pacman -S  [openntpd](https://wiki.archlinux.org/index.php/OpenNTPD)  

nano /etc/ntpd.conf  

    server 0.cn.pool.ntp.org
    server 1.cn.pool.ntp.org
    server 2.cn.pool.ntp.org
    server 3.cn.pool.ntp.org
    sensor *
    # constraints from "https://www.google.com"


ntpd -n  
timedatectl set-[timezone](https://wiki.archlinux.org/index.php/System_time_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)#%E6%97%B6%E5%8C%BA) Asia/Shanghai  
### 检查时间
timedatectl  

systemctl enable openntpd.service  
systemctl list-unit-files --state=enabled  | grep openntpd  
![openntpd](resource\openntpd.png "openntpd")

---

## 更新
### 添加源
nano /etc/pacman.d/mirrorlist  

    # tencent
    Server = https://mirrors.cloud.tencent.com/archlinuxarm/$arch/$repo
    Server = https://mirrors.cloud.tencent.com/archlinuxcn/$arch/$repo

    # tsinghua
    Server = https://mirrors.tuna.tsinghua.edu.cn/archlinuxarm/$arch/$repo
    Server = https://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch/$repo

    # ustc
    Server = http://mirrors.ustc.edu.cn/archlinuxarm/$arch/$repo
    Server = http://mirrors.ustc.edu.cn/archlinuxcn/$arch/$repo

pacman -Syy
### 使用aria2多线程下载 加速pacman
pacman -S [aria2](https://wiki.archlinux.org/index.php/Aria2)  
nano /etc/pacman.conf
> 將下面一行加在 [options] 段下

    XferCommand = /usr/bin/aria2c --allow-overwrite=true --continue=true --file-allocation=none --log-level=error --max-tries=2 --max-connection-per-server=2 --max-file-not-found=5 --min-split-size=5M --no-conf --remote-time=true --summary-interval=60 --timeout=5 --dir=/ --out %o %u

grep aria2c /etc/pacman.conf
![aria2](resource\aria2.png "aria2")
### 更新系统
pacman -Syu  
### 添加常用软件
pacman -S  [wget](https://wiki.archlinux.org/index.php/Wget) [git](https://wiki.archlinux.org/index.php/Git_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)) [npm](https://wiki.archlinux.org/index.php/Node.js) [gcc](https://wiki.archlinux.org/index.php/GNU_Compiler_Collection) [yaourt](https://wiki.archlinux.org/index.php/AUR_helpers_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))  [net-tools](https://wiki.archlinux.org/index.php/Network_configuration_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))

---

## [SCP](https://winscp.net/eng/download.php) & [SSH](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html)
### 允许root远程登录
nano /etc/ssh/sshd_config  

    PermitRootLogin yes
    PasswordAuthentication yes

systemctl restart sshd  

### 修改root密码
passwd root
### 修改普通用户用户名
>usermod  -l  新用户名  -d  /home/新用户名  -m  老用户名

usermod  -l bugwriter  -d  /home/bugwriter  -m  manjaro
### 修改普通用户密码
passwd bugwriter
### WinSCP配置如图
![WinSCP](resource\WinSCP.png "WinSCP")

---

## [node-red](https://github.com/node-red/node-red)
sudo npm install -g --unsafe-perm node-red  
node-red
### 查看控制台
http://192.168.191.101:1880 
### 添加启动项  
wget https://raw.githubusercontent.com/node-red/raspbian-deb-package/master/resources/nodered.service -O /lib/systemd/system/nodered.service  

wget https://raw.githubusercontent.com/node-red/raspbian-deb-package/master/resources/node-red-start -O /usr/bin/node-red-start  

wget https://raw.githubusercontent.com/node-red/raspbian-deb-package/master/resources/node-red-stop -O /usr/bin/node-red-stop  

chmod +x /usr/bin/node-red-st*  
nano /lib/systemd/system/nodered.service  

    User=root
    Group=root
    #WorkingDirectory=/home/pi

systemctl daemon-reload  
systemctl enable nodered.service  
systemctl list-unit-files | grep node  
![nodered](resource\nodered.png "nodered")
reboot
### 查看控制台
http://192.168.191.101:1880 

---

## VNC
pacman -S [xorg](https://wiki.archlinux.org/index.php/Xorg_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)) xorg-xinit [lxqt](https://wiki.archlinux.org/index.php/LXQt_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)) [tigervnc](https://wiki.archlinux.org/index.php/TigerVNC)  
### 设置默认桌面  
nano /etc/X11/xinit/xinitrc  

    #twm &
    #xclock -geometry 50x50-1+1 &
    #xterm -geometry 80x50+494+51 &
    #xterm -geometry 80x20+494-0 &
    #exec xterm -geometry 80x66+0+0 -name login

    exec startlxqt

### 退出root  
su bugwriter  
### 启动vncserver  
vncserver  
### 设置连接密码，如：1233321   (不显示)  
### 使用[**VNC Viewer**](https://www.realvnc.com/en/connect/download/viewer/)连接效果  
![VNCViewer](resource\VNCViewer.png "VNCViewer")
![VNCViewerRename](resource\VNCViewerRename.png "VNCViewerRename")
### 关闭vnc  
vncserver -kill :1  

---

## 克隆系统（约30分钟）
>参考：https://github.com/billw2/rpi-clone  

su  
root  
pacman -S [rsync](https://wiki.archlinux.org/index.php/Rsync_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))  
git clone https://github.com/billw2/rpi-clone.git  
cd rpi-clone  
cp rpi-clone /usr/local/sbin/sys-clone  
cp rpi-clone-setup /usr/local/sbin/sys-clone-setup  
### 使用USB读卡器，插入一张新SD卡到树莓派  
sys-clone sda -f  

---

## 备份系统
>参考：https://github.com/bup/bup


---

# 未完待续