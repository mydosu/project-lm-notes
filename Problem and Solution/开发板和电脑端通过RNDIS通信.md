# 开发板和电脑端通过RNDIS通信

由于内核版本是 `6.18.41-current-sunxi64`，这个版本比较新。板子`g_ether`模块在编译时，没有包含 RNDIS 支持，所以使用 `configfs` 来手动配置一个RNDIS 复合设备。
```bash
# 1. 安装 dnsmasq
sudo apt update
sudo apt install dnsmasq -y

# 2. 创建网络配置文件
sudo tee /etc/systemd/network/10-usb0.network > /dev/null << 'EOF'
[Match]
Name=usb0

[Network]
Address=192.168.137.2/24 #让usb0接口在开机时自动拥有IP
EOF

# 3. 创建 dnsmasq 配置 即配置 DHCP 服务器（给电脑分配 IP）
sudo tee /etc/dnsmasq.d/usb0.conf > /dev/null << 'EOF'
interface=usb0  #只监听 usb0 接口，不影响其它网卡。
dhcp-range=192.168.137.200,192.168.6.200,255.255.255.0,24h
port=0  #关闭 DNS 功能，避免与系统自带的 systemd-resolved 抢端口53
EOF

# 4. 创建 RNDIS 启动脚本
sudo tee /usr/bin/rndis-gadget.sh > /dev/null << 'EOF'
#!/bin/bash
umount /sys/kernel/config 2>/dev/null
mount -t configfs none /sys/kernel/config
# 确保 configfs 挂载正确（解决重启后挂载点被占用的问题）

cd /sys/kernel/config/usb_gadget/
mkdir -p g1
cd g1

echo 0x045E > idVendor
echo 0x00DB > idProduct

echo 0x0100 > bcdDevice
echo 0x0200 > bcdUSB

mkdir -p strings/0x409
echo "0123456789abcdef" > strings/0x409/serialnumber
echo "Orange Pi" > strings/0x409/manufacturer
echo "RNDIS Gadget" > strings/0x409/product

mkdir -p configs/c.1
mkdir -p configs/c.1/strings/0x409
echo "Config 1: RNDIS" > configs/c.1/strings/0x409/configuration
echo 250 > configs/c.1/MaxPower

mkdir -p functions/rndis.usb0
ln -sf functions/rndis.usb0 configs/c.1/

# 自动检测 UDC 名称并绑定 2026.8.8
UDC_NAME=$(ls /sys/class/udc/ | head -n 1)
if [ -n "$UDC_NAME" ]; then
    echo $UDC_NAME > UDC
else
    echo "Error: No UDC found!"
    exit 1
fi
EOF

sudo chmod +x /usr/bin/rndis-gadget.sh

# 5. 配置 libcomposite 开机加载，使 /sys/kernel/config/usb_gadget/ 目录可用
echo "libcomposite" | sudo tee /etc/modules-load.d/libcomposite.conf

# 6. 创建 systemd 服务（开机自启）
sudo tee /etc/systemd/system/rndis-gadget.service > /dev/null << 'EOF'
[Unit]
Description=RNDIS USB Gadget
After=network.target

[Service]
ExecStart=/usr/bin/rndis-gadget.sh
Type=oneshot
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

# 7. 启用并启动服务
sudo systemctl daemon-reload
sudo systemctl enable rndis-gadget.service
sudo systemctl enable dnsmasq
sudo systemctl start rndis-gadget.service
sudo systemctl start dnsmasq

# 8. 重启验证
sudo reboot
```
orange pi服务状态查看
```bash
sudo systemctl status rndis-gadget.service
sudo systemctl status dnsmasq
```
输出be like:
![[Pasted image 20260807005920.png|670]]
注：Failed to set DNS configuration: Link lo is loopback device.为正常现象，我们禁用DNS防止影响正常上网
![[Pasted image 20260807010101.png]]
查看是否有ip
```bash
sudo ip a show usb0
```
![[Pasted image 20260807010248.png|677]]
在连接的电脑上运行cmd，输入（中间有用户安装驱动的关键过程，详见[[Windows RNDIS 兼容设备驱动安装]]）
```text
ping 192.168.137.2
```
![[Pasted image 20260807010413.png]]
知道配置成功