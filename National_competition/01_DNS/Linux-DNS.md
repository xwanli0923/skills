# DNS服务：创建 DNS 服务器，实现企业域名访问。
1. 设置所有 linux 服务器的时区设为“上海”，本地时间调整为实
际时间。在防火墙中开启相应服务端口，设置服务开机自启动。
```bash
# ALL-LINUX-NODE 代表 linux1 - linux7 
$ ssh root@ALL-LINUX-NODE
$ rpm -qa | grep ^bash-com
$ yum -y install bash-completion
$ exit
$ ssh root@ALL-LINUX-NODE
$ timedatectl list-timezones | grep -i shanghai
Asia/Shanghai
$ timedatectl set-timezone Asia/Shanghai
# 查看其他有正确时间的主机 假如时间为 2022年4月11日23:30
$ yum install chrony
$ systemctl enable chronyd --now
$ yum -y install vim

$ ssh root@linux1
[root@linux1 ~]$ timedatectl set-ntp no
[root@linux1 ~]$ timedatectl set-time "2022-04-11 23:30"
[root@linux1 ~]$ hwclock -w
[root@linux1 ~]$ timedatectl set-ntp yes
```

2. 利用 chrony 配置 linux1 为其他 linux 主机提供 NTP 服务。
```bash
[root@linux1 ~]$ vim /etc/chrony.conf
  3 server 10.10.20.101 iburst
 23 allow 10.10.20.0/24
  26 local stratum 5
[root@linux1 ~]$ systemctl restart chronyd
[root@linux1 ~]$ chronyc -n sources
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^? 10.10.20.101                  0   6   377     -     +0ns[   +0ns] +/-    0ns
[root@linux1 ~]$ grep ^ntp /etc/services
ntp             123/tcp
ntp             123/udp                         # Network Time Protocol

[root@linux1 ~]$ firewall-cmd --add-port=123/udp
success
[root@linux1 ~]$ firewall-cmd --runtime-to-permanent
# ssh root@linux2-7
$  vim /etc/chrony.conf
  3 server 10.10.20.101 iburst
  
$ systemctl restart chronyd
$ chronyc -n sources
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* 10.10.20.101                  5   6    37    26    +72ns[ +115us] +/-  125us

$ chronyc tracking
Reference ID    : 0A0A1465 (linux1.skills.com)
Stratum         : 6
Ref time (UTC)  : Mon Apr 11 15:54:03 2022
System time     : 0.000000058 seconds slow of NTP time
Last offset     : +0.000115164 seconds
RMS offset      : 0.000115164 seconds
Frequency       : 0.424 ppm slow
Residual freq   : +0.000 ppm
Skew            : 0.588 ppm
Root delay      : 0.000246139 seconds
Root dispersion : 0.000020844 seconds
Update interval : 65.1 seconds
Leap status     : Normal
```
3. 利用 bind9 软件，配置 linux1 为主 DNS 服务器，采用 rndc 技
术提供不间断的 DNS 服务；配置 linux2 为备用 DNS 服务器。为所有
linux 主机提供冗余 DNS 正反向解析服务。
```bash
[root@linux1 ~]$ yum -y install bind bind-utils
[root@linux1 ~]$ vim /etc/named.conf 
 10 options {
 11         listen-on port 53 { any; };  # 修改
 12         listen-on-v6 port 53 { any; }; # 修改
 13         directory       "/var/named";
 14         dump-file       "/var/named/data/cache_dump.db";
 15         statistics-file "/var/named/data/named_stats.txt";
 16         memstatistics-file "/var/named/data/named_mem_stats.txt";
 17         secroots-file   "/var/named/data/named.secroots";
 18         recursing-file  "/var/named/data/named.recursing";
 19         allow-query     { any; }; # 修改
 31         recursion yes;
 32         allow-new-zones yes;  # 允许rndc添加新zone


[root@linux1 ~]$ vim /etc/named.rfc1912.zones 
# 追加新的内容
zone "skills.com" IN {
        type master;
        file "skills.com";
        allow-update { 10.10.20.101; };
        allow-transfer { 10.10.20.102; localhost; };
};

zone "20.10.10.in-addr.arpa" IN {
        type master;
        file "20.10.10.in-addr.arpa";
        allow-update { 10.10.20.101; };
        allow-transfer { 10.10.20.102; localhost; };
};

[root@linux1 ~]$ cd /var/named/
[root@linux1 named]$ cat named.localhost > skills.com 
[root@linux1 named]$ vim skills.com 
$TTL 1D
@       IN SOA  skills.com. root.skills.com. (
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
        NS      linux1.skills.com.
        NS      linux2.skills.com.
;       A       127.0.0.1
;       AAAA    ::1
linux1 IN A 10.10.20.101
linux2 IN A 10.10.20.102
linux3 IN A 10.10.20.103
linux4 IN A 10.10.20.104
linux5 IN A 10.10.20.105
linux6 IN A 10.10.20.106
linux7 IN A 10.10.20.107



[root@linux1 named]$ cat named.loopback > 20.10.10.in-addr.arpa 
$TTL 1D
@       IN SOA  skills.com root.skills.com. (
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
        NS      linux1.skills.com.
        NS      linux1.skills.com.
linux1 A 10.10.20.101
linux2 A 10.10.20.102
101  PTR linux1.skills.com.
102  PTR linux2.skills.com.
103  PTR linux3.skills.com.
104  PTR linux4.skills.com.
105  PTR linux5.skills.com.
106  PTR linux6.skills.com.
107  PTR linux7.skills.com.

[root@linux1 ~]$ named-checkconf 
[root@linux1 ~]$ named-checkzone  "skills.com" /var/named/skills.com 
zone skills.com/IN: loaded serial 0
OK
[root@linux1 ~]$ chown named:named /var/named/skills.com /var/named/20.10.10.in-addr.arpa 
[root@linux1 ~]$ ls -l /var/named/
total 24
-rw-r--r--. 1 named named  626 Apr 12 10:02 20.10.10.in-addr.arpa
drwxrwx---. 2 named named    6 Oct 12 01:02 data
drwxrwx---. 2 named named    6 Oct 12 01:02 dynamic
-rw-r-----. 1 root  named 2253 Oct 12 01:02 named.ca
-rw-r-----. 1 root  named  152 Oct 12 01:02 named.empty
-rw-r-----. 1 root  named  152 Oct 12 01:02 named.localhost
-rw-r-----. 1 root  named  168 Oct 12 01:02 named.loopback
-rw-r--r--. 1 named named  573 Apr 12 10:01 skills.com
drwxrwx---. 2 named named    6 Oct 12 01:02 slaves

# 配置rndc
[root@linux1 ~]$ rndc-confgen 
# Start of rndc.conf
key "rndc-key" {
    algorithm hmac-md5;
    secret "r/UVE/+cI2peXk0NuUoC+g==";
};

options {
    default-key "rndc-key";
    default-server 127.0.0.1;
    default-port 953;
};
# End of rndc.conf

# Use with the following in named.conf, adjusting the allow list as needed:
# key "rndc-key" {
#     algorithm hmac-md5;
#     secret "r/UVE/+cI2peXk0NuUoC+g==";
# };
# 
# controls {
#     inet 127.0.0.1 port 953
#         allow { 127.0.0.1; } keys { "rndc-key"; };
# };
# End of named.conf

[root@linux1 ~]$ vim /etc/rndc.conf
# Start of rndc.conf
key "rndc-key" {
    algorithm hmac-md5;
    secret "r/UVE/+cI2peXk0NuUoC+g==";
};

options {
    default-key "rndc-key";
    default-server 127.0.0.1;
    default-port 953;
};
# End of rndc.conf

[root@linux1 ~]$ vim /etc/rndc.key
[root@linux1 ~]$ cat /etc/rndc.key
key "rndc-key" {
    algorithm hmac-md5;
    secret "r/UVE/+cI2peXk0NuUoC+g==";
};

[root@linux1 ~]$ vim /etc/named.conf 
# 追加
 60 include "/etc/rndc.key";
 61 
 62 controls {
 63         inet 127.0.0.1 port 953
 64                 allow { 127.0.0.1; } keys { "rndc-key"; };
 65 };
 66 # End of named.conf

[root@linux1 ~]$ named-checkconf /etc/named.conf 
[root@linux1 ~]$ systemctl enable named --now
[root@linux1 ~]$ systemctl is-enabled named
enabled
[root@linux1 ~]$ systemctl is-active named
active

[root@linux1 ~]$ rndc status
WARNING: key file (/etc/rndc.key) exists, but using default configuration file (/etc/rndc.conf)
version: BIND 9.11.26-RedHat-9.11.26-6.el8 (Extended Support Version) <id:3ff8620>
running on linux1.skills.com: Linux x86_64 4.18.0-348.el8.0.2.x86_64 #1 SMP Sun Nov 14 00:51:12 UTC 2021
boot time: Tue, 12 Apr 2022 02:22:04 GMT
last configured: Tue, 12 Apr 2022 02:22:05 GMT
configuration file: /etc/named.conf
CPUs found: 1
worker threads: 1
UDP listeners per interface: 1
number of zones: 105 (97 automatic)
debug level: 0
xfers running: 0
xfers deferred: 0
soa queries in progress: 0
query logging is OFF
recursive clients: 0/900/1000
tcp clients: 3/150
TCP high-water: 3
server is up and running

[root@linux1 ~]$ firewall-cmd --permanent --add-port=53/tcp --add-port=53/udp --add-port=953/tcp --add-port=953/udp
success
[root@linux1 ~]$ firewall-cmd --reload 

[root@linux1 ~]$ nmcli connection show
NAME    UUID                                  TYPE      DEVICE 
ens160  b35a3016-263c-426b-b66e-d3de0f38df98  ethernet  ens160 

[root@linux1 ~]$ nmcli connection modify ens160 ipv4.addresses "10.10.20.101/24" ipv4.gateway "10.10.20.1" ipv4.dns "10.10.20.101 10.10.20.102" ipv4.method manual

[root@linux1 ~]$ nmcli connection reload 
[root@linux1 ~]$ nmcli connection up ens160 

# 测试
[root@linux1 ~]$ nslookup linux2
Server:        10.10.20.101
Address:    10.10.20.101#53

Name:    linux2.skills.com
Address: 10.10.20.102

[root@linux1 ~]$ nslookup linux3.skills.com
Server:        10.10.20.101
Address:    10.10.20.101#53

Name:    linux3.skills.com
Address: 10.10.20.103

[root@linux1 ~]$ nslookup 10.10.20.104
104.20.10.10.in-addr.arpa    name = linux4.skills.com.
# 建议所有机器配置静态IP以及设置DNS
[root@linux2 ~]$ nmcli connection modify ens160 ipv4.addresses "10.10.20.102/24" ipv4.gateway "10.10.20.1" ipv4.dns "10.10.20.101 10.10.20.102" ipv4.method manual
[root@linux2 ~]$ nmcli connection reload 
[root@linux2 ~]$ nmcli connection up ens160

[root@linux3 ~]$ nmcli connection modify ens160 ipv4.addresses "10.10.20.103/24" ipv4.gateway "10.10.20.1" ipv4.dns "10.10.20.101 10.10.20.102" ipv4.method manual
[root@linux3 ~]$ nmcli connection reload 
[root@linux3 ~]$ nmcli connection up ens160

[root@linux4 ~]$ nmcli connection modify ens160 ipv4.addresses "10.10.20.104/24" ipv4.gateway "10.10.20.1" ipv4.dns "10.10.20.101 10.10.20.102" ipv4.method manual
[root@linux4 ~]$ nmcli connection reload 
[root@linux4 ~]$ nmcli connection up ens160

[root@linux5 ~]$ nmcli connection modify ens160 ipv4.addresses "10.10.20.105/24" ipv4.gateway "10.10.20.1" ipv4.dns "10.10.20.101 10.10.20.102" ipv4.method manual
[root@linux5 ~]$ nmcli connection reload 
[root@linux5 ~]$ nmcli connection up ens160

[root@linux6 ~]$ nmcli connection modify ens160 ipv4.addresses "10.10.20.106/24" ipv4.gateway "10.10.20.1" ipv4.dns "10.10.20.101 10.10.20.102" ipv4.method manual
[root@linux6 ~]$ nmcli connection reload 
[root@linux6 ~]$ nmcli connection up ens160

[root@linux7 ~]$ nmcli connection modify ens160 ipv4.addresses "10.10.20.107/24" ipv4.gateway "10.10.20.1" ipv4.dns "10.10.20.101 10.10.20.102" ipv4.method manual
[root@linux7 ~]$ nmcli connection reload 
[root@linux7 ~]$ nmcli connection up ens160

### 也可以编写网卡配置文件手动更改

# 配置从服务器
[root@linux2 ~]$ yum -y install bind bind-utils
[root@linux2 ~]$ vim /etc/named.conf 
10 options {
 11         listen-on port 53 { any; };
 12         listen-on-v6 port 53 { any; };
 13         directory       "/var/named";
 14         dump-file       "/var/named/data/cache_dump.db";
 15         statistics-file "/var/named/data/named_stats.txt";
 16         memstatistics-file "/var/named/data/named_mem_stats.txt";
 17         secroots-file   "/var/named/data/named.secroots";
 18         recursing-file  "/var/named/data/named.recursing";
 19         allow-query     { any; };
 31         recursion yes;
 32         allow-new-zones yes; # 允许rndc添加新zone


46 [root@linux2 ~]$ vim /etc/named.rfc1912.zones 
# 追加
 47 zone "skills.com" IN {
 48         type slave;
 49         file "slaves/skills.com";
 50         masters { 10.10.20.101; };
 51 };
 52 
 53 zone "20.10.10.in-addr.arpa" IN {
 54         type slave;
 55         file "slaves/20.10.10.in-addr.arpa";
 56         masters { 10.10.20.101; };
 57 };
[root@linux2 ~]$ systemctl restart named
[root@linux2 ~]$ ls /var/named/slaves/
20.10.10.in-addr.arpa  skills.com
[root@linux2 ~]$ firewall-cmd --permanent --add-port=53/tcp --add-port=53/udp 
[root@linux2 ~]$ firewall-cmd --reload 
[root@linux2 ~]$ dig @linux2.skills.com -t A linux4.skills.com

; <<>> DiG 9.11.26-RedHat-9.11.26-6.el8 <<>> @linux2.skills.com -t A linux4.skills.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 20166
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 4be1569353e95986844f09146254f0b7cfee2ad6044a96b9 (good)
;; QUESTION SECTION:
;linux4.skills.com.        IN    A

;; ANSWER SECTION:
linux4.skills.com.    86400    IN    A    10.10.20.104

;; AUTHORITY SECTION:
skills.com.        86400    IN    NS    linux1.skills.com.
skills.com.        86400    IN    NS    linux2.skills.com.

;; ADDITIONAL SECTION:
linux1.skills.com.    86400    IN    A    10.10.20.101
linux2.skills.com.    86400    IN    A    10.10.20.102

;; Query time: 0 msec
;; SERVER: 10.10.20.102#53(10.10.20.102)
;; WHEN: Tue Apr 12 11:23:35 CST 2022
;; MSG SIZE  rcvd: 164
```
4. 所有 linux 主机 root 用户使用完全合格域名免密码 ssh 登录到
其他 linux 主机。
```bash
[root@linux1 ~]$ ssh-keygen 
[root@linux1 ~]$ for i in {1..7};do ssh-copy-id  root@linux$i.skills.com;done
[root@linux1 ~]$ for i in {1..7};do scp /root/.ssh/authorized_keys /root/.ssh/id_rsa* root@linux$i.skills.com:/root/.ssh/;done
```
5. 配置 linux1 为 CA 服务器,为所有 linux 主机颁发证书，不允许
修改/etc/pki/tls/openssl.conf。CA 证书有效期 20 年，CA 颁发证书有
效期均为 10 年，证书信息：国家=“CN”，省=“Beijing”，城市=“Beijing”，
组织=“Skills”，组织单位=“System”，公用名=skills.com。证书路径均
为/etc/ssl/skills.crt，私钥路径均为/etc/ssl/skills.key，chrome 浏览器访
问 https 网站时，不出现证书警告提示信息。
```bash
[root@linux1 ~]$ cd /etc/ssl

[root@linux1 ~]$ openssl genrsa -out ca-server.key -des3 2048
Generating RSA private key, 2048 bit long modulus (2 primes)
............+++++
........................................................................................................+++++
e is 65537 (0x010001)
Enter pass phrase for ca-server.key:
Verifying - Enter pass phrase for ca-server.key:

[root@linux1 ssl]$ openssl req -new -x509 -key ca-server.key -days 7300 -out ca-server.crt
Enter pass phrase for ca-server.key:
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:Beijing
Locality Name (eg, city) [Default City]:Beijing
Organization Name (eg, company) [Default Company Ltd]:Skills
Organizational Unit Name (eg, section) []:System
Common Name (eg, your name or your server\'s hostname) []:skills.com
Email Address []:

[root@linux1 ssl]$ grep ^dir /etc/pki/tls/openssl.cnf 
dir        = /etc/pki/CA        # Where everything is kept
dir        = /etc/pki/CA        # TSA root directory

[root@linux1 ssl]$ grep -n '\$dir' /etc/pki/tls/openssl.cnf 
62:certs        = $dir/certs        # Where the issued certs are kept
64:database    = $dir/index.txt    # database index file.
67:new_certs_dir    = $dir/newcerts        # default place for new certs.
69:certificate    = $dir/cacert.pem     # The CA certificate
70:serial        = $dir/serial         # The current serial number

74:private_key    = $dir/private/cakey.pem# The private key


[root@linux1 ssl]$ mkdir /etc/pki/CA
[root@linux1 ssl]$ touch /etc/pki/CA/index.txt
[root@linux1 ssl]$ mkdir /etc/pki/CA/newcerts
[root@linux1 ssl]$ echo 00 > /etc/pki/CA/serial
[root@linux1 ssl]$ openssl x509 -in ca-server.crt -out /etc/pki/CA/cacert.pem -outform PEM
[root@linux1 ssl]$ mkdir /etc/pki/CA/private
[root@linux1 ssl]$ openssl rsa -in ca-server.key -out /etc/pki/CA/private/cakey.pem

# 生成csr证书
[root@linux1 ssl]$ openssl genrsa -out skills.key 2048
[root@linux1 ssl]$ openssl req -new -key skills.key -out skills.csr
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:Beijing
Locality Name (eg, city) [Default City]:Beijing
Organization Name (eg, company) [Default Company Ltd]:Skills
Organizational Unit Name (eg, section) []:System
Common Name (eg, your name or your server\'s hostname) []:skills.com
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:

[root@linux1 ssl]$ cp -v skills.* /etc/pki/CA/
[root@linux1 ssl]$ cd /etc/pki/CA/
[root@linux1 CA]$ mkdir /etc/pki/CA/certs
[root@linux1 CA]$ openssl ca -in skills.csr -out certs/skills.crt -days 3650
Using configuration from /etc/pki/tls/openssl.cnf
Check that the request matches the signature
Signature ok
Certificate Details:
        Serial Number: 1 (0x1)
        Validity
            Not Before: Apr 12 09:25:20 2022 GMT
            Not After : Apr  9 09:25:20 2032 GMT
        Subject:
            countryName               = CN
            stateOrProvinceName       = Beijing
            organizationName          = Skills
            organizationalUnitName    = System
            commonName                = skills.com
        X509v3 extensions:
            X509v3 Basic Constraints: 
                CA:FALSE
            Netscape Comment: 
                OpenSSL Generated Certificate
            X509v3 Subject Key Identifier: 
                3B:49:B4:B5:60:DF:DC:F8:73:13:DB:31:1D:47:22:6A:5E:09:51:C6
            X509v3 Authority Key Identifier: 
                keyid:DD:83:3A:0D:84:14:05:B4:C6:A9:5E:6F:EC:8F:04:97:9F:ED:C6:1A

Certificate is to be certified until Apr  9 09:25:20 2032 GMT (3650 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated

# 如果签错，可以临时修改 index.txt.attr
# unique_subject = no
# 重新签后恢复yes
```

---
> 如果执行`rndc reload` 提示如下内容
> `WARNING: key file (rndc.key) exists, but using default configuration file (rndc.conf)`
**解决方案：**
```bash
$ mv /etc/rndc.conf{,.bak}
$ vim /etc/named.conf
62 # controls {
63 #       inet 127.0.0.1 port 953
64 #               allow { 127.0.0.1; } keys { "rndc-key"; };
65 # };
$ systemctl restart named　
```






