+++
title = 'Ansible'
date = 2019-07-17T23:02:42+08:00
draft = false
+++

# Ansible 

一篇搞定，快速学习

----
## 一、特点
- 模块化： 调用特定的模块，完成特定的内容
- 使用 Paramiko(Python对于ssh的实现)，PyYAML,Jinja2(模板语言) 三个关键模块
- 支持自定义模块，可以使用任何语言编写
- 基于Python实现
- 不是简单，基于Python和SSH,无agent，无需代理，不依赖PKI(无需SSL)
- 安全，基于OpenSSH
- 幂等性：一个任务执行一遍和N边结果是一样的（要注意代码逻辑）
- 支持playbook编排任务，YAML格式，编排任务，支持丰富的数据机构
- 较强大的role的多层解决方案
----

## 二、架构
```
用户/PLAYBOOK => INVENTORY/API/MODULES/PLUGINS => HOSTS/NETWORK
INVEMTORY: Ansible管理主机的清单 /etc/ansible/hosts
MODULES: Ansible执行命令的功能模块、多数为内置核心模块，也可以自定义
PLUGINS: 模块功能的补充，如连接类型插件、循环插件、变量插件、过滤插件等
API: 供第三方程序调用的应用程序接口
```
用户直接使用命令或者使用 Playbook 或者 调用API 去远程主机执行命令
----

## 三、注意事项
- 执行ansible的主机一般称为主控端，中控，master或堡垒机
- 主控段python需要在2.6或以上
- 被控端python版本小鱼2.4的需要安装 python-simplejson
- windows不能作为主控
----

## 四、安装

- epel源rpm包安装

`sudo yum -y install epel-release && sudo yum -y install ansible`

- [官方其他安装方式](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)

- 查看安装版本和命令
```text
ansible --version

ansible 2.9.27
  config file = /etc/ansible/ansible.cfg
  configured module search path = [u'/root/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/site-packages/ansible
  executable location = /usr/bin/ansible
  python version = 2.7.5 (default, Nov 16 2020, 22:23:17) [GCC 4.8.5 20150623 (Red Hat 4.8.5-44)]
```

`rpm -ql ansible` 可以查看ansible都下载了哪些文件。
也可以在终端输入ansible 后<kbd>Tab<kdb> 键查看输出的命令

----

## 五、基本使用
### 准备机器
| 节点名称 | 节点IP         | 角色           |
|----|--------------|--------------|
| k8s-master-001 | 10.241.12.3  | Ansible-Server |
| k8s-master-003 | 10.241.12.1  | 被控制端         |
| k8s-master-002 | 10.241.12.2  | 被控制端         |
| k8s-node-003 | 10.241.12.9  | 被控制端         |
| k8s-node-001 | 10.241.12.12 | 被控制端         |

### anisble管理主机的方式
- Ad-HC 即利用ansible命令，主要用于临时命令场景
- Ansible-playbook 主要用于长期规划好的，大型项目的使用，需要有前期的规划过程。



### 配置文件
- /etc/ansible/ansible.cfg #主配置文件，配置ansible工作
    ```
  [defaults]
  inventory = /etc/ansible/hosts #主机列表
  library = /usr/share/my_modules #库文件存放目录
  remote_tmp = $HOME/.ansible/tmp #临时 py 命令文件存放在远程主机目录
  local_tmp = $HOME/.ansible/tmp #本机的临时命令执行目录
  forks = 5 #默认并发数量
  sudo_user = root #默认sudo用户
  ask_sudo_pass = True #每次执行ansible 命令是否询问ssh密码
  ask_pass = True
  remote_port =22
  host_key_checking = False #检查对应服务器的host_key, 建议取消注释
  log_path = /var/log/ansible.log #日志文件 建议启用
  module_name = command #默认模块，可以修改为shell模块 
  ```
- /etc/ansible/hosts #主机清单
为了批量快捷的管理部分主机
    ```
  #举例子
  #[标签名]
  #ip1
  #ip2
  #ip3:222
  #将下面的配置写入 /etc/ansible/hosts
  [k8s-master]
  #k8s-master-003
  10.241.12.1
  #k8s-master-002
  10.241.12.2
  [k8s-slave]
  #k8s-node-003
  10.241.12.9
  #k8s-node-001
  10.241.12.12

  ```
- /etc/ansible/role #存放角色

### ansible-doc 命令文档
此命令用来显示模块的帮助文档
格式
```text
ansible-doc [OPTION] [MODULE...]
-l , --list #列出可用模块
-s , --snippet #显示指定模块的playbook片段
例子
ansible-doc -s ping
```
### ansible 命令
使用SSH协议实现对远程主机的配置。 所以主控端需要能基于密钥认证的方式联系哥哥被管理节点
```bash
#例如使用sshpass批量实现key的验证
ssh-keygen -f /root/.ssh/id_rsa -P ''
NET= 192.168.100
export SSHPASS=xxxx
for IP in {1..200};do
  sshpass -e ssh-copy-id $NET.$IP
done
```
#### 格式
```bash
ansible <host_pattern> [ -m module_name] [-a args]
```
#### 重要选项说明
```text
--version #显示版本
-m moudle #指定模块，默认为command
-v #详细过程 -vv -vvv 更加详细
--list-hosts #列出主机列表
-k, --ask-pass #提示输入ssh连接密码，默认key验证
-C, --check #检查，并不执行
-T, --timeout=TIMEOUT #执行远程命令的用户,默认10S
-u，--user=PEMOTE_USER #执行园长执行的用户
-b, --become #代替旧版的sudo 切换
--beconme-user=USERNAME #指定sudo的runas用户，默认为root
-K, --ask-becomne-pass #提示输入sudo时的口令
```

举例使用ping模块 
```text
# ansible all -m ping 或者  ansible k8s-master#这里指定了hosts里写的标签组 -m ping
[WARNING]: Invalid characters were found in group names but not replaced, use -vvvv to see details
10.241.12.1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
10.241.12.12 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
10.241.12.2 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
10.241.12.9 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}

```
如果没有验证通过，需要 -k 来输入密码，很麻烦。所以建议使用秘钥。


### 选择特定主机
```text
[root@k8s-master-001 ~]# ansible all --list-hosts
[WARNING]: Invalid characters were found in group names but not replaced, use -vvvv to see details
  hosts (4):
    10.241.12.9
    10.241.12.12
    10.241.12.1
    10.241.12.2
#或者指定标签组
[root@k8s-master-001 ~]# ansible k8s-master --list-hosts
[WARNING]: Invalid characters were found in group names but not replaced, use -vvvv to see details
  hosts (2):
    10.241.12.1
    10.241.12.2
```
使用通配符指定主机
```text
ansible '*' --list-hosts
ansible 10.241.* --list-hosts
ansible 10.241.12.1:10.241.12.2 --list-hosts #俩个机器中的一个 ，或者关系
ansible 'k8s-master:&k8s-node' --list-hosts #与的关系，在k8s-master标签组里也在k8s-node里面
ansible 'k8s-master:!k8s-node' --list-hosts #非得关系，在k8s-master标签组里但不在 k8s-node里面
```
正则表达式选择主机
```text
ansible '~(k8s-master|k8s-node)' -m ping
```
### Ansible 执行命令的过程
1.加载 默认的/etc/ansible/ansible.cfg
2.加载自己对应的模块文件，如 command
3.通过ansible将模块或者命令生成对应的临时py文件，将文件传输至远程服务器对应执行用户的$HOME/.ansible/tmp/ansible-tmp-数字/xxx.py
4.赋予执行权限
5.执行并且返回结果
6.删除py文件，退出

### ansible执行的状态
在 /etc/ansible/ansible.cfg 的 [colers] 模块中有颜色的详细说明
```text
[colors]
#highlight = white
#verbose = blue
#warn = bright purple
#error = red
#debug = dark gray
#deprecate = purple
#skip = cyan
#unreachable = red
#ok = green
#changed = yellow
#diff_add = green
#diff_remove = red
#diff_lines = cyan
```
### ansible-galaxy # ansible 使用别人下载的工具
此工具会链接 https://galaxy.ansible.com 下载对应的roles

列出已经安装的galaxy
```text
ansible-galaxy list
```
下载对应的 roles
```text
#
ansible-galaxy install pogosoftware.etcd
#返回内容"
#- downloading role 'etcd', owned by pogosoftware
#- downloading role from https://github.com/pogosoftware/ansible-role-etcd/archive/v1.0.2.tar.gz
#- extracting pogosoftware.etcd to /root/.ansible/roles/pogosoftware.etcd
#- pogosoftware.etcd (v1.0.2) was installed successfully
```
查看里面的roles结构
```text
 tree  /root/.ansible/roles/pogosoftware.etcd
/root/.ansible/roles/pogosoftware.etcd
├── defaults
│   └── main.yml
├── handlers
│   └── main.yml
├── LICENSE
├── meta
│   └── main.yml
├── tasks
│   ├── etcd_user.yml
│   └── main.yml
└── templates
    └── etcd.service.j2

5 directories, 7 files
#可以 cp -a 一个文件名称然后再里面改自己的定制化需求
```

删除galaxy
```text
ansible-galaxy remove pogosoftware.etcd
```
----

### ansible-pull 
将ansible的命令推送至远程

### ansible-palybook (后面有详解)
此工具用于编写好的playbook任务
```text
cat hello.yml
- hosts: k8s-slave
  remote_user: root
  tasks:
    - name: hello world
      command: /usr/bin/wall hello world

#执行的时候记得打开 k8s-slave中的一个主机终端看效果
 ansible k8s-slave  --list-host
 会发现终端输出了 hello  world
```

### ansible-vault 加密yml文件
```text
ansible-vaut encrypt hello.yml #加密 会让输入加密口令
ansible-vaut decrypt hello.yml #解密
ansiblt-vault view hello.yml #查看
ansible-vault edit hello.yml #编辑加密文件
ansiblt-vault rekey hello.yml #修改口令
ansiblt-vault create new.yml #创建新文件

```

### ansible-console 可交互命令
支持tab
提示符号格式：
```text
执行用户@当前操作主机组（当期主机组的主机数量）[f:并发数]$
```
常用子命令：
- 设置并发数量：forks n 例如 forks 10
- 切换组： cd 主机组 例如 cd web
- 列出当前主机组例如 list
- 列出所有的内置命令： ？ 或者help

例子
```text
ansible-console
[WARNING]: Invalid characters were found in group names but not replaced, use -vvvv to see details
Welcome to the ansible console.
Type help or ? to list commands.

root@all (4)[f:5]$ cd k8s-master
root@k8s-master (2)[f:5]$ list
10.241.12.1
10.241.12.2
root@k8s-master (2)[f:5]$ yum name=lftp
10.241.12.1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "msg": "",
    "rc": 0,
    "results": [
        "lftp-4.4.8-12.el7_8.1.x86_64 providing lftp is already installed"
    ]
}
...
```

## 六、Ansible 常用模块
### Command 模块
```text

```

