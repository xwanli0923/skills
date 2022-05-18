# APACHE2服务
为了搭建快速、可靠的网页服务器，请采用 Apache 服
务器，实现对企业网站的安全有效访问。
1.配置 linux2 为 apache2 服务器，安装 apache2，http 访问时自动
跳转到 https。
```bash
[root@linux2 ~]# yum -y install httpd httpd-tools mod_ssl
# 复制证书
[root@linux1 ~]# scp /etc/pki/CA/certs/skills.crt root@linux2:/etc/pki/tls/certs/
[root@linux1 ~]# scp /etc/pki/CA/skills.key  root@linux2:/etc/pki/tls/private/
# 修改ssl.conf
[root@linux2 ~]# vim /etc/httpd/conf.d/ssl.conf 
# 替换关键词
:%s/localhost/skills

# 创建站点
[root@linux2 ~]# cat /etc/httpd/conf.d/skills.com.conf
<VirtualHost *:80>
        RewriteEngine on
        ServerName linux2.skills.com
        ServerAlias www.skills.com
        Redirect permanent / https://www.skills.com/
</VirtualHost>

[root@linux2 ~]# firewall-cmd --permanent --add-port=80/tcp --add-port=443/tcp
success
[root@linux2 ~]# firewall-cmd --reload 
```
2.使用 skills.com 或 any.skills.com（any 代表任意网址前缀，用
linux2.skills.com 和 web.skills.com 测 试 ）访问时，自动跳转到
[www.skills.com。](http://www.skills.com./)
```bash
# 增加DNS别名
[root@linux1 ~]# cat /var/named/skills.com 
skills.com. IN A 10.10.20.102
www IN A 10.10.20.102
web.skills.com. IN CNAME linux2.skills.com.
```
3.客户端访问时，必需有 SSL 证书。

4.关闭不安全的服务器信息，在任何页面不会出现系统和 WEB
服务器版本信息
```bash
[root@linux2 ~]# rpm -ql httpd | grep default
/usr/share/doc/httpd/httpd-default.conf
[root@linux2 ~]# mkdir /etc/httpd/conf/extra
[root@linux2 ~]# cp /usr/share/doc/httpd/httpd-default.conf /etc/httpd/conf/extra/
[root@linux2 ~]# echo "Include conf/extra/httpd-default.conf" >> /etc/httpd/conf/httpd.conf 
[root@linux2 ~]# curl -I localhost
HTTP/1.1 301 Moved Permanently
Date: Tue, 12 Apr 2022 09:11:07 GMT
Server: Apache/2.4.37 (rocky) OpenSSL/1.1.1k
Location: https://www.skills.com/
Content-Type: text/html; charset=iso-8859-1

[root@linux2 ~]# vim /etc/httpd/conf/extra/httpd-default.conf 
 55 ServerTokens Prod
 65 ServerSignature On

[root@linux2 ~]# systemctl restart httpd
[root@linux2 ~]# curl -I localhost
HTTP/1.1 301 Moved Permanently
Date: Tue, 12 Apr 2022 09:11:49 GMT
Server: Apache
Location: https://www.skills.com/
Content-Type: text/html; charset=iso-8859-1
```

