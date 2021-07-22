# ansible

## ansible主要组成部分

- ansible-playbook执行过程：
    - 将已有编排好的任务集写入ansible-playbook
    - 通过执行ansible-playbook命令分拆任务集至逐条ansible命令，按预定规则逐条执行
    
- ansible主要操作对象
    - hosts主机
    - networking网络设备
    
    
## 安装 

- yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm centos7环境下安装epel源
- yum install ansible
  
## 相关文件

````

配置文件
/etc/ansible/ansible.cfg 主配置文件，ansible工作特性
/etc/ansible/hosts 主机清单
/etc/ansible/roles/ 存放角色的目录

程序
/usr/bin/ansible 主程序，临时命令执行工具
/usr/bin/ansible-play-book 定制自动化任务，编排剧本工具 
/usr/bin/ansible-pull 远程执行的命令的工具
/usr/bin/ansible-vault 文件加密工具
/usr/bin/ansible-console 基于Console界面与用户交互的执行工具
````

## ansible配置文件

````

ansible 配置文件/etc/ansible/ansible.cfg(一般保持默认)
[defaults]
#inventory      = /etc/ansible/hosts  #主机列表配置文件
#library        = /usr/share/my_modules/ #库文件存放目录
#module_utils   = /usr/share/my_module_utils/ 
#remote_tmp     = ~/.ansible/tmp #临时py命令文件存放在远程主机目录
#local_tmp      = ~/.ansible/tmp #本机临时命令执行目录
#forks          = 5 #默认并发数（同时执行5个操作，eg五台主机五台的执行）
#poll_interval  = 15 
#sudo_user      = root #默认sudo用户
#ask_sudo_pass = True #每次执行ansible命令是否询问ssh密码
#ask_pass      = True
#transport      = smart
#remote_port    = 22
#module_lang    = C
#module_set_locale = False
#host_key_checking = False #检查对应服务的的host_key，建议取消注释
#log_path = /var/log/ansible.log #日志文件，建议取消注释

````

## ansible 系列命令

````
1、ansible-doc 显示模块帮助
ansible-doc [options][module]
-a  显示所有模块文档
-l，--list 列出可用模块
-s，--snippet 显示指定模块的playbook片段

2、ansible<host-pattern>[-m module_name] [-a args]
--version 显示版本
-m module 指定模块，默认为command
-v 详细过程 -vv -vvv 更详细
--list-host 显示主机列表，可简写--list
-k ，--ask-pass 提示输入ssh连接密码。默认key验证
-K， --ask-become-pass  提示输入sudo时的口令
-C，--check 检查不执行
-T --timeout=TIMEOUT 执行命令的超时时间，默认10s
-u --user=REMOTE——USER 执行远程执行的用户
-b， --become 代替旧版本的sudo切换

3、ansible的Host-pattern  匹配主机的列表
all：表示所有Inventory中的所有主机
*：通配符
ansible “*” -m ping
ansible 192.168.1.* -m ping
ansible “*srvs” -m ping
或关系 
ansible “webserver:dbserver” -m ping
ansible "webserver:dbserver" -m ping #执行在web组并且在dbserver组中的主机（忽略重复的）
与关系
ansible "webserver:&dbserver" -m ping
只执行在web组并且也在dbserver组中的主机
逻辑非
ansible 'webserver:!dbserver' -m ping  【注意此处只能使用单引号！】
综合逻辑
ansible 'webserver:dbserver:&webserver:!dbserver' -m ping
正则表达式
ansible "webserver:&dbserver" -m ping
ansible "~(web|db).*\.magedu.\com" -m ping

````

## ansible 常见模块

- ping 
- command

````

在远程主机执行命令，默认模块。可忽略-m选项
ansible srvs -m command -a ‘systemctl restart sshd’
ansible srvs -m command -a 'echo magedu | passwd --stdin wang '不成功
此命令不支持$VRNAME< >  | ; & 等，需要用shell模块实现

````

- shell

````

和command相似，用shell执行命令
ansible srv -m shell -a ‘echo magedu | passwd --stdin wang’
调用bash执行命令 类似cat /tmp/stanley.md | awk -F '|' '{print $1,$2}' & >
/tmp/example.txt 这些复杂命令，及时使用shell也可能会失败，解决办法：写到脚本时，copy到远程，执行，再把需要的结果拉回执行命令的机器

````

- script

````

运行脚本
-a “/PATH/TO/SCRIPT_FILE”
ansible webserver -m script -a f1.sh

````

- copy

````

从服务器复制文件到客户端
ansible all -m copy -a 'src=/data/test1 dest=/data/test1 backup=yes mode=000 owner=zhang'  ##如目标存在，默认覆盖，此处是指先备份，并修改全向属主
ansible all -m shell -a 'ls -l /data/'
ansible all -m copy -a "content='test content\n' dest=/tmo/f1.txt" 利用内容，直接生成目标文件

````

- fetch

````

从客户端取文件至服务器端，与copy相反，目录可以先tar
ansible all -m fetch -a ‘src=/root/a.sh dest=/data/f2.sh'

````

- file

````

file:设置文件属性（状态，属组，属主，权限）
ansible all -m file -a “path=/root/a.sh owner=zhang mode=755”
ansible all -m file -a 'src=/data/test1 dest=/tmp/test state=link'
ansible all -m file -a ’name=/data/f3 state=touch‘  #创建文件
ansible all -m file -a ’name=/data/f3 state=absent‘ #删除文件
ansible all -m file -a ’name=/data state=directory‘ #创建目录
ansible all -m file -a ’src=/etc/fstab dest=/data/fstab.link state=link

````

- yum

````

ansible all -m yum -a 'name=httpd state=latest'安装
ansible all -m yum -a 'name=httpd state=ansent' 卸载
ansible all -m yum  -a 'name=dstat update_cache=yes' 更新缓存

````

- service

````

ansible all -m service -a 'name=httpd state=stopped'
ansible all -m service -a 'name=httpd state=started enabled=yes'
ansible all -m service -a 'name=httpd state=reload'
ansible all -m service -a 'name=httpd state=restart'

````

## ansible 系列命令

- ansible-galaxy

````

列出所有已安装的galaxy
ansible-galaxy list
安装galaxy
ansible-galaxy install geerlingguy.redis
删除galaxy
ansible-galaxy remove geerlingguy.redis

````