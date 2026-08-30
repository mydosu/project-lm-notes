# 有线网络配置和接入 and DDNS-go动态绑域名

- ### 有线网络配置和接入
开发板删除桌面环境后，有线网络网卡配置和自动连接脚本也会被一并删除,所以需要重新配置
一般Ubuntu有线网名称为eth0

1. 创建一个配置文件让 eth0 自动获取 IP
```bash
sudo bash -c 'cat > /etc/systemd/network/20-wired.network << EOF
[Match]
Name=eth0

[Network]
DHCP=yes
EOF'
```
2. 重启网络服务
```bash
sudo systemctl restart systemd-networkd
```
注:本镜像通过systemd-networkd管理网络
3. 检查网卡状态
```bash
sudo ip a show eth0
```
输出Be like:
![[Pasted image 20260807000423.png|586]]
可见配置成功
- #### DDNS-go动态绑域名
1. 安装ddns-go
使用一键安装脚本
```bash
curl -O https://raw.githubusercontent.com/k08255-lxm/ddns-go-installer/main/install.sh && sudo bash install.sh
```
2. 通过网页配置域名
在浏览器里访问 `http://OrangePi的内网IP地址:9876`
设置用户名和密码
选择 DNS 服务商：比如阿里云、腾讯云(DNSPod)、Cloudflare 等
填写 API 密钥：去DNS 服务商后台获取 API Token 或 AccessKey，填进去
![[Pasted image 20260807002103.png|587]]

![[Pasted image 20260807002131.png|554]]
3. 设置开机自启并检查运行状态
```bash
sudo systemctl enable ddns-go
sudo systemctl start ddns-go
sudo systemctl status ddns-go
```
输出be like:
![[Pasted image 20260807002346.png|574]]
则配置成功