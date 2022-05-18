
 # PXE服务由于企业新购一批服务器，需要安装 linux 操作系统，请采用 PXE 服务实现需求。
1. 配置 linux4 为 PXE 服务器，实现完全自动安装 Linux。
2. 安装 DHCP 服务，地址范围为 10.10.20.10-10.10.20.19，网关为10.10.20.254，DNS 为 10.10.20.101，域名为 skills.com。
```bash
$ yum -y install dhcp-server
$ cd /etc/dhcp/
$ cat /usr/share/doc/dhcp-server/dhcpd.conf.example > dhcpd.conf
  1 # A slightly different configuration for an internal subnet.
  2 subnet 10.10.20.0 netmask 255.255.255.0 {
  3   range 10.10.20.10 10.10.20.19;
  4   option domain-name-servers 10.10.20.101;
  5   option domain-name "skills.com";
  6   option routers 10.10.20.254;
  7   default-lease-time 600;
  8   max-lease-time 7200;
  9   next-server 10.10.20.104;
 10   filename "pxelinux.0";
 11 }
$ systemctl enable dhcpd --now
$ yum -y install syslinux-tftpboot
```
3. 安装 tftpd，为 PXE 客户端提供启动服务，TFTP 目录为默认值。
```bash
$ yum install tftp-server
$ systemctl enable tftp.service --now
$ cp /tftpboot/pxelinux.0 /var/lib/tftpboot/
```
4. 安装 apache2 服务，为 PXE 客户端提供软件包；挂载 linux 光 盘文件到/var/www/html/cdrom。
```bash
$ yum -y install httpd
# 由于练习环境同tomcat:80冲突，修改为8080
$ sed -i 's/Listen 80/Listen 8080/g' /etc/httpd/conf/httpd.conf
$ systemctl enable httpd --now
$ mkdir /var/www/html/cdrom
$ firewall-cmd --permanent --add-port=8080/tcp
$ firewall-cmd --reload 
$ vim /etc/httpd/conf.d/welcome.conf
9     Options +Indexes
$ systemctl restart httpd
```
* 向tftp中填充引导文件
```bash
$ cd /var/www/html/cdrom/isolinux/
$ rsync -uP *  /var/lib/tftpboot/
$  cd /var/lib/tftpboot/
$ mkdir pxelinux.cfg
$ cp isolinux.cfg pxelinux.cfg/default
$ chmod 644 pxelinux.cfg/default
$ vim pxelinux.cfg/default
 61 label linux
 62   menu label ^Install Linux by PXE
 63   kernel vmlinuz
 64     append initrd=initrd.img inst.repo=http://10.10.20.104:8080/cdrom inst.ks=http://10.10.20.104:8080/ks/ks.install quiet
```
* 准备应答文件
```bash
$ mkdir /var/www/html/ks
$ cp anaconda-ks.cfg  /var/www/html/ks/ks.install
#version=RHEL8
# Use graphical install
text
reboot

url --url=http://10.10.20.104:8080/cdrom

%packages
@^minimal-environment
kexec-tools

%end

# Keyboard layouts
keyboard --xlayouts='us'
# System language
lang en_US.UTF-8

# Network information
network  --bootproto=dhcp --device=ens160 --ipv6=auto --activate
network  --hostname=localhost.localdomain

# Run the Setup Agent on first boot
firstboot --disable

ignoredisk --only-use=nvme0n1
# Partition clearing information
clearpart --none --initlabel
# Disk partitioning information
part / --fstype="xfs" --ondisk=nvme0n1 --size=4096 --grow
part swap --fstype="swap" --ondisk=nvme0n1 --size=2048
part /boot --fstype="xfs" --ondisk=nvme0n1 --size=1024

# System timezone
timezone Asia/Shanghai --isUtc --nontp

# Agreed EULA
eula --agreed

# Root password
rootpw --iscrypted $6$jAbv0fN624V42580$Yw8/o88FI/holB29jBHcaie74Hey52jG5TY3mf53DjB8gQpQB5y6ZpGsVNm47IerptgPVtQO2vCQRXNmCub6O0
user --name=devops --password=Pass-1234

$ chmod 644 /var/www/html/ks/ks.install
$ firewall-cmd --permanent --add-port=69/udp --add-port=67/udp
$ firewall-cmd --reload
```
> 如何摧毁引导
> ```bash
> $ dd if=/dev/zero of=/dev/vda bs=1M count=4
> ```
