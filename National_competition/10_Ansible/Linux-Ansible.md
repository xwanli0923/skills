# Ansible 服务
为了提高了工作效率,由程序自动的、重复的执行任务，请采用 Ansible 服务，实现自动化运维。
1. 在 linux1 上安装 ansible，作为 ansible 的控制节点。linux2-linux7 作为 ansible 的受控节点。
* 从共享目录拷贝tar文件到本地
```bash
$ yum list installed platform-python
Installed Packages
platform-python.x86_64                                3.6.8-41.el8.rocky.0           
$ yum -y install ansible
```
2. 编写/root/my.yml 剧本，实现在 linux1 的/root 目录创建一个ansible.txt 文件，然后复制到所有受控节点的/root 目录。
```bash
$ vim /etc/ansible/hosts
linux[2:7].skills.com
$ ansible-doc file
$ vim /root/my.yml
```
```yaml
---
- hosts: all
  tasks:
  - name: Create new file
    file:
      path: /root/ansible.txt
      state: touch
```
```bash
$ ansible-playbook my.yml
```

