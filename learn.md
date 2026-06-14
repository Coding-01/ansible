[toc]

# 安装ansible

```shell
# 控制节点(ubuntu24.04)
rambo@ub24:~$ cat /etc/os-release 
PRETTY_NAME="Ubuntu 24.04.1 LTS"
NAME="Ubuntu"
VERSION_ID="24.04"
VERSION="24.04.1 LTS (Noble Numbat)"
VERSION_CODENAME=noble


rambo@ub24:~$ 
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt search ansible
sudo apt install ansible


rambo@ub24:~$ ansible --version
ansible [core 2.21.0]
  config file = /etc/ansible/ansible.cfg
  configured module search path = ['/home/rambo/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3/dist-packages/ansible
  ansible collection location = /home/rambo/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/bin/ansible
  python version = 3.12.3 (main, Mar 23 2026, 19:04:32) [GCC 13.3.0] (/usr/bin/python3)
  jinja version = 3.1.2
  pyyaml version = 6.0.1 (with libyaml v0.2.5)
rambo@ub24:~$ ansible-playbook --version
ansible-playbook [core 2.21.0]
  config file = /etc/ansible/ansible.cfg
  configured module search path = ['/home/rambo/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3/dist-packages/ansible
  ansible collection location = /home/rambo/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/bin/ansible-playbook
  python version = 3.12.3 (main, Mar 23 2026, 19:04:32) [GCC 13.3.0] (/usr/bin/python3)
  jinja version = 3.1.2
  pyyaml version = 6.0.1 (with libyaml v0.2.5)


# 生成密钥
rambo@ub24:~$ ssh-keygen -t rsa -P ''
rambo@ub24:~$ for i in {5..7};do ssh-copy-id 172.16.186.13$i;done

```

# 安装nginx1.26

```shell
下班、睡觉不是这一天的结束，是为了明天的你在做准备
所以你下班和睡觉是明天的开始



rambo@ub24:~/ansible-project$ cd /usr/local/src/
rambo@ub24:/usr/local/src$ sudo apt install gcc make libpcre3-dev zlib1g-dev libssl-dev -y
rambo@ub24:/usr/local/src$ sudo wget https://nginx.org/download/nginx-1.26.0.tar.gz
rambo@ub24:/usr/local/src$ sudo tar zxvf nginx-1.26.0.tar.gz
rambo@ub24:/usr/local/src$ cd nginx-1.26.0/

rambo@ub24:/usr/local/src/nginx-1.26.0$ sudo ./configure \
--prefix=/opt/nginx/nginx-1.26.0 \
--with-http_ssl_module

rambo@ub24:/usr/local/src/nginx-1.26.0$ sudo make -j4 && sudo make install

# 创建软链接
rambo@ub24:/usr/local/src/nginx-1.26.0$ sudo ln -sfn /opt/nginx/nginx-1.26.0  /opt/nginx/current
rambo@ub24:/usr/local/src/nginx-1.26.0$ ls -l /opt/nginx/current
lrwxrwxrwx 1 root root 23 Jun 13 19:28 /opt/nginx/current -> /opt/nginx/nginx-1.26.0

rambo@ub24:/usr/local/src/nginx-1.26.0$ sudo /opt/nginx/current/sbin/nginx

rambo@ub24:/usr/local/src/nginx-1.26.0$ curl localhost





# 用ansible在web组的机器上部署
rambo@ub24:/usr/local/src/nginx-1.26.0$ cd ~/ansible-project/
rambo@ub24:~/ansible-project$ sudo cp /usr/local/src/nginx-1.26.0.tar.gz  files/
rambo@ub24:~/ansible-project$ sudo chown rambo:rambo  files/nginx-1.26.0.tar.gz


rambo@ub24:~/ansible-project$ vim install_nginx.yml
---
- name: Deploy Nginx from source
  hosts: web
  become: yes
  vars:
    nginx_version: "1.26.0"
    src_dir: "/usr/local/src"
    install_dir: "/opt/nginx/nginx-1.26.0"
    current_link: "/opt/nginx/current"

  tasks:
    - name: Ensure required dependencies are installed
      apt:
        name: [gcc, make, libpcre3-dev, zlib1g-dev, libssl-dev]
        state: present
        update_cache: yes

    - name: Create nginx directories
      file:
        path: "{{ item }}"
        state: directory
      loop: ["{{ src_dir }}", "/opt/nginx"]

    - name: Copy nginx tarball to remote
      copy:
        src: "nginx-{{ nginx_version }}.tar.gz"
        dest: "{{ src_dir }}/"

    - name: Extract nginx source
      unarchive:
        src: "{{ src_dir }}/nginx-{{ nginx_version }}.tar.gz"
        dest: "{{ src_dir }}/"
        remote_src: yes
        creates: "{{ src_dir }}/nginx-{{ nginx_version }}/configure"

    - name: Configure and Compile Nginx
      shell: |
        ./configure --prefix={{ install_dir }} --with-http_ssl_module && make -j4 && make install
      args:
        chdir: "{{ src_dir }}/nginx-{{ nginx_version }}"
        creates: "{{ install_dir }}/sbin/nginx"

    - name: Create current symbolic link
      file:
        src: "{{ install_dir }}"
        dest: "{{ current_link }}"
        state: link
        force: yes

    - name: Start Nginx
      command: "{{ current_link }}/sbin/nginx"
      args:
        creates: /var/run/nginx.pid   # 简单校验是否已启动


rambo@ub24:~/ansible-project$ ansible-playbook -i inventories/prod/hosts.yml install_nginx.yml -u rambo -b -K
BECOME password:         # 输入一次密码
....
	....
TASK [Create current symbolic link] *******************************************************************************************************************
changed: [web01]
changed: [web03]
changed: [web02]

TASK [Start Nginx] ************************************************************************************************************************************
changed: [web01]
changed: [web02]
changed: [web03]

PLAY RECAP ********************************************************************************************************************************************
web01                      : ok=8    changed=5    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
web02                      : ok=8    changed=5    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
web03                      : ok=8    changed=5    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   


rambo@ub24:~/ansible-project$ ansible web -m shell -a "curl -I localhost" -u rambo -b -K
BECOME password: 



```



# 企业级安全升级目标

```
升级过程必须满足:
1. 保留旧版本
2. 新版本先验证
3. 不覆盖旧版本
4. 切换成本秒级
5. 自动回滚
6. serial滚动发布
7. 健康检查
8. CMDB登记
```

# 目录结构

```shell
# 推荐目录结构
rambo@ub24:~$ mkdir ansible-project && cd ansible-project
rambo@ub24:~/ansible-project$ mkdir -p inventories/prod  group_vars templates files
rambo@ub24:~/ansible-project$ tree .
.
├── ansible.cfg
├── files
│   ├── nginx-1.26.0.tar.gz
│   ├── nginx-1.28.0.tar.gz
│   └── nginx-1.28.0.tar.gz-bak
├── group_vars
│   └── all.yml
├── install_nginx.yml
├── inventories
│   └── prod
│       └── hosts.yml
├── site.yml
└── templates
    └── nginx.conf.j2


启动命令永远写: /opt/nginx/current/sbin/nginx
而不是: /opt/nginx/nginx-1.26.3/sbin/nginx

```



## inventories/prod/hosts.yml

```shell
rambo@ub24:~/ansible-project$ vim inventories/prod/hosts.yml
all:
  children:
    web:
      hosts:
        web01:
          ansible_host: 172.16.186.135
          ansible_user: rambo
          
        web02:
          ansible_host: 172.16.186.136
          ansible_user: rambo

        web03:
          ansible_host: 172.16.186.137
          ansible_user: rambo
```



## group_vars/all.yml

```shell
rambo@ub24:~/ansible-project$ vim group_vars/all.yml
nginx_port: 80
app_version: "1.2.0"
monitor_server: 10.0.0.100
nginx_base_dir: /opt/nginx
nginx_version: "1.28.0"
ansible_python_interpreter: /usr/bin/python3
healthcheck_url: "http://127.0.0.1:80/"
```



## templates/nginx.conf.j2

```shell
rambo@ub24:~/ansible-project$ vim templates/nginx.conf.j2
worker_processes auto;

events {
    worker_connections 1024;
}

http {
    upstream backend {
# 会把inventory中的web作为一个组来循环，结果就是依次循环web01、web02、web03
{% for host in groups['web'] %}
    server {{ hostvars[host]['ansible_host'] }};
{% endfor %}
}

server {
    listen {{ nginx_port }};
    location / {
        return 200 "ok\n";
    }
  }
}



# 释义
hostvars是企业里经常考的, Ansible自动维护一个大字典
结构类似:
hostvars:
  web01:
    ansible_host: 172.16.186.135

  web02:
    ansible_host: 172.16.186.136

  web03:
    ansible_host: 172.16.186.137

访问 hostvars['web01'] 得到 ansible_host: 172.16.186.135
访问 hostvars['web01']['ansible_host'] 得到 172.16.186.135
所以 server {{ hostvars[host]['ansible_host'] }}; 实际生成:
server 172.16.186.135;
server 172.16.186.136;
server 172.16.186.137;

```



## vault

```shell
mysql_password: supersecret

加密:
ansible-vault encrypt vault.yml
```



## 企业级中级Playbook

## 版本1

```shell
# 先编译好高版本的nginx
rambo@ub24:~/ansible-project$ wget https://nginx.org/download/nginx-1.28.0.tar.gz -O files/nginx-1.28.0.tar.gz
rambo@ub24:~/ansible-project$ cd files/
rambo@ub24:~/ansible-project/files$ tar -zxvf nginx-1.28.0.tar.gz 
rambo@ub24:~/ansible-project/files$ cd nginx-1.28.0/
rambo@ub24:~/ansible-project/files/nginx-1.28.0$ sudo ./configure --prefix=/opt/nginx/nginx-1.28.0 --with-http_ssl_module

rambo@ub24:~/ansible-project/files/nginx-1.28.0$ sudo make -j4 && sudo make install

# 再重新打包新版本，是把安装好的目录重新打包，并不是源码包
rambo@ub24:~/ansible-project/files/nginx-1.28.0$ cd ..
rambo@ub24:~/ansible-project/files$ mv nginx-1.28.0.tar.gz{,-bak}
rambo@ub24:~/ansible-project/files$ cd /opt/nginx/
rambo@ub24:~/ansible-project/files$ tar zcvf ~/ansible-project/files/nginx-1.28.0.tar.gz  nginx-1.28.0

# 验证压缩包结构
rambo@ub24:~/ansible-project/files$ tar tf ~/ansible-project/files/nginx-1.28.0.tar.gz | head -10
正确应该类似:
nginx-1.28.0/
nginx-1.28.0/sbin/
nginx-1.28.0/sbin/nginx
nginx-1.28.0/html/
nginx-1.28.0/html/index.html
nginx-1.28.0/html/50x.html
nginx-1.28.0/conf/
nginx-1.28.0/conf/koi-win
nginx-1.28.0/conf/fastcgi.conf
nginx-1.28.0/conf/scgi_params


rambo@ub24:/opt/nginx$ cd -
rambo@ub24:~/ansible-project/files$ ls -alh nginx-1.28.0.tar.gz*
-rw-rw-r-- 1 rambo rambo 2.0M Jun 13 20:55 nginx-1.28.0.tar.gz         # 新的
-rw-rw-r-- 1 rambo rambo 1.3M Apr 23  2025 nginx-1.28.0.tar.gz-bak     # 原来的


rambo@ub24:~/ansible-project/files$ cd ..



rambo@ub24:~/ansible-project$ vim site.yml
---
- name: Enterprise Nginx Rolling Upgrade
  hosts: web         # 对web组所有机器执行
  become: true       # 等于sudo

  serial: 1          # 滚动发布,先web01，成功后再web02...不会一起升级 

  vars_files:        # 加载变量文件,导入到当前Play
    - vault.yml

  pre_tasks:
    - name: collect facts
      setup:          # 收集Facts,比如hostname/ip addr/uname -r/cpu...

    - name: show current host
      debug:
        msg: "{{ inventory_hostname }}"

    - name: verify nginx base dir
      stat:
        path: "{{ nginx_base_dir }}"
      register: nginx_base

    - name: fail if nginx dir missing
      fail:
        msg: "{{ nginx_base_dir }} not found"
      when: not nginx_base.stat.exists

# 主任务
  tasks:

# 当前版本
    - name: check current symlink
      stat:
        path: "{{ nginx_base_dir }}/current"
      register: current_symlink

    - name: get current version
      command: readlink {{ nginx_base_dir }}/current
      register: current_link
      changed_when: false
      when: current_symlink.stat.exists

    - name: save rollback version
      set_fact:
        rollback_version: "{{ current_link.stdout }}"
      when: current_symlink.stat.exists

    - name: show current version
      debug:
        msg: "{{ rollback_version }}"

# 上传新版本
    - name: upload nginx package
      unarchive:
        src: files/nginx-{{ nginx_new_version }}.tar.gz
        dest: "{{ nginx_base_dir }}"
        remote_src: false
        creates: "{{ nginx_base_dir }}/nginx-{{ nginx_version }}"

# 下发配置
    - name: deploy nginx config
      template:
        src: templates/nginx.conf.j2
        dest: "{{ nginx_base_dir }}/nginx-{{ nginx_version }}/conf/nginx.conf"


# 升级 + 自动回滚
    - block:
# 新版本配置检查
        - name: validate nginx config
          command:
            cmd: >
              {{ nginx_base_dir }}/nginx-{{ nginx_version }}/sbin/nginx
              -t
              -c
              {{ nginx_base_dir }}/nginx-{{ nginx_version }}/conf/nginx.conf
          register: nginx_test
          changed_when: false

        - name: stop old nginx
          shell: |
            if pgrep nginx >/dev/null 2>&1; then
              {{ rollback_version }}/sbin/nginx -s quit
            fi

        - name: wait old nginx stopped
          wait_for:
            port: "{{ nginx_port }}"
            state: stopped
            timeout: 20
# 切换软链接
        - name: switch current symlink
          file:
            src: "{{ nginx_base_dir }}/nginx-{{ nginx_version }}"
            dest: "{{ nginx_base_dir }}/current"
            state: link
            force: true

# restart nginx
        - name: start new nginx
          command:
            cmd: "{{ nginx_base_dir }}/current/sbin/nginx"

        - name: wait nginx started
          wait_for:
            port: "{{ nginx_port }}"
            timeout: 20

# 健康检查
        - name: health check
          uri:
            url: "{{ healthcheck_url }}"
            status_code: 200

# 回滚逻辑
      rescue:
        - name: stop failed nginx
          shell: |
            pkill nginx || true

        - name: restore symlink
          file:
            src: "{{ rollback_version }}"
            dest: "{{ nginx_base_dir }}/current"
            state: link
            force: true

        - name: start rollback nginx
          command:
            cmd: "{{ rollback_version }}/sbin/nginx"

        - name: wait rollback nginx
          wait_for:
            port: "{{ nginx_port }}"
            timeout: 20

        - name: rollback health check
          uri:
            url: "{{ healthcheck_url }}"
            status_code: 200

        - name: fail deployment
          fail:
            msg: "upgrade failed, rollback completed"


# 无论成功失败都执行
      always:
        - name: write deploy log
          lineinfile:
            path: /var/log/ansible-nginx-deploy.log
            create: true
            line: >
              {{ ansible_date_time.iso8601 }}
              host={{ inventory_hostname }}
              target=nginx-{{ nginx_version }}

# CMDB注册(这里我没加上)
    - name: register host to cmdb
      uri:
        url: "http://{{ monitor_server }}/register"
        method: POST
        body_format: json
        body:
          hostname: "{{ inventory_hostname }}"
          ip: "{{ ansible_host }}"
          version: "{{ nginx_version }}"
      delegate_to: localhost



rambo@ub24:~/ansible-project$ ansible web -m shell -a "hostname"
注：
Ansible默认去找 /etc/ansible/hosts
而当前的inventory在 ~/ansible-project/inventories/prod/hosts.yml
所以提示provided hosts list is empty
意思是没找到任何主机


先验证 inventory 是否正常:
rambo@ub24:~/ansible-project$ ansible-inventory -i inventories/prod/hosts.yml --graph
@all:
  |--@ungrouped:
  |--@web:
  |  |--web01
  |  |--web02
  |  |--web03


然后测试:
rambo@ub24:~/ansible-project$ ansible web -i inventories/prod/hosts.yml -m ping


更方便的方法是在项目根目录创建:
rambo@ub24:~/ansible-project$ vim ansible.cfg
[defaults]
inventory = inventories/prod/hosts.yml
host_key_checking = False
forks = 10
timeout = 30

目录则变成:
ansible-project/
├── ansible.cfg
├── files
├── group_vars
├── inventories
├── site.yml
└── templates

在项目目录直接执行ansible命令
cd ~/ansible-project
rambo@ub24:~/ansible-project$ ansible web -m ping
就不用每次写 -i inventories/prod/hosts.yml

# 冒烟测试
rambo@ub24:~/ansible-project$ ansible-playbook site.yml --check -k -K

# 如果没问题后再实际跑playbook
rambo@ub24:~/ansible-project$ ansible-playbook site.yml -k -K


```



## 版本2

```shell
rambo@ub24:~/ansible-project$ vim site.yml 
---
- name: Enterprise Nginx Rolling Upgrade
  hosts: web
  become: true

  serial: 1
  max_fail_percentage: 20

  pre_tasks:

    - name: gather facts
      setup:

    - name: check nginx base dir
      stat:
        path: "{{ nginx_base_dir }}"
      register: nginx_base

    - name: fail if nginx base dir missing
      fail:
        msg: "{{ nginx_base_dir }} not found"
      when: not nginx_base.stat.exists           # 当nginx_base状态不存在时

    - name: verify version
      assert:
        that:
          - nginx_version is defined
          - nginx_version in allowed_versions
        fail_msg: "invalid nginx version"

  tasks:

    - name: get current version path
      command:
        cmd: readlink -f {{ nginx_base_dir }}/current
      register: current_link
      changed_when: false

    - name: save rollback version
      set_fact:
        rollback_version: "{{ current_link.stdout }}"

    - name: show current version
      debug:
        msg: "current={{ rollback_version }}"

    - name: upload nginx package
      copy:
        src: "files/nginx-{{ nginx_version }}.tar.gz"
        dest: "/tmp/nginx-{{ nginx_version }}.tar.gz"

    - name: extract nginx package
      unarchive:
        src: "/tmp/nginx-{{ nginx_version }}.tar.gz"
        dest: "{{ nginx_base_dir }}"
        remote_src: true
        creates: "{{ nginx_base_dir }}/nginx-{{ nginx_version }}/sbin/nginx"

    - name: verify nginx binary
      stat:
        path: "{{ nginx_base_dir }}/nginx-{{ nginx_version }}/sbin/nginx"
      register: nginx_binary

    - name: fail if binary missing
      fail:
        msg: "nginx binary not found"
      when: not nginx_binary.stat.exists

    - name: deploy nginx config
      template:
        src: templates/nginx.conf.j2
        dest: "{{ nginx_base_dir }}/nginx-{{ nginx_version }}/conf/nginx.conf"

    - block:

        - name: validate nginx config
          command:
            cmd: >
              {{ nginx_base_dir }}/nginx-{{ nginx_version }}/sbin/nginx
              -t
              -c
              {{ nginx_base_dir }}/nginx-{{ nginx_version }}/conf/nginx.conf
          register: nginx_test
          changed_when: false

        - name: stop old nginx
          command:
            cmd: "{{ rollback_version }}/sbin/nginx -s quit"
          ignore_errors: true

        - name: wait old nginx stopped
          wait_for:
            port: "{{ nginx_port }}"
            state: stopped
            timeout: 20

        - name: switch current symlink
          file:
            src: "{{ nginx_base_dir }}/nginx-{{ nginx_version }}"
            dest: "{{ nginx_base_dir }}/current"
            state: link
            force: true

        - name: start new nginx
          command:
            cmd: "{{ nginx_base_dir }}/current/sbin/nginx"

        - name: wait nginx started
          wait_for:
            port: "{{ nginx_port }}"
            timeout: 20

        - name: health check
          uri:
            url: "{{ healthcheck_url }}"
            status_code: 200

      rescue:

        - name: stop failed nginx
          shell: |
            pkill nginx || true

        - name: restore symlink
          file:
            src: "{{ rollback_version }}"
            dest: "{{ nginx_base_dir }}/current"
            state: link
            force: true

        - name: start rollback nginx
          command:
            cmd: "{{ rollback_version }}/sbin/nginx"

        - name: wait rollback nginx
          wait_for:
            port: "{{ nginx_port }}"
            timeout: 20

        - name: rollback health check
          uri:
            url: "{{ healthcheck_url }}"
            status_code: 200

        - name: fail deployment
          fail:
            msg: "upgrade failed, rollback completed"

      always:

        - name: write deploy log
          lineinfile:
            path: /var/log/ansible-nginx-deploy.log
            create: true
            line: >
              {{ ansible_facts.date_time.iso8601 }}
              host={{ inventory_hostname }}
              target=nginx-{{ nginx_version }}

#    - name: register host
#      uri:
#        url: "http://{{ monitor_server }}/register"
#        method: POST
#        body_format: json
#        body:
#          hostname: "{{ inventory_hostname }}"
#          ip: "{{ ansible_host }}"
#      delegate_to: localhost


rambo@ub24:~/ansible-project$ cat group_vars/all.yml
nginx_port: 80
app_version: "1.2.0"
monitor_server: 10.0.0.100
nginx_base_dir: /opt/nginx
nginx_version: "1.28.0"
ansible_python_interpreter: /usr/bin/python3
healthcheck_url: "http://127.0.0.1:80/"
allowed_versions:               # 添加这个变量
  - 1.26.0
  - 1.28.0



rambo@ub24:~/ansible-project$ ansible-playbook site.yml -k -K
SSH password: 
BECOME password[defaults to SSH password]: 
....
	....
TASK [health check] **********************************************************************************************************************
ok: [web03]

TASK [write deploy log] ******************************************************************************************************************
changed: [web03]

PLAY RECAP *******************************************************************************************************************************
web01                      : ok=18   changed=3    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   
web02                      : ok=18   changed=4    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   
web03                      : ok=18   changed=4    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0   







```











## 涉及的中级知识点

| 知识点      | 是否覆盖 |
| ----------- | -------- |
| Inventory   | √        |
| group_vars  | √        |
| host_vars   | √        |
| Vault       | √        |
| Facts       | √        |
| Register    | √        |
| Debug       | √        |
| Loop        | √        |
| Condition   | 部分     |
| Notify      | √        |
| Handler     | √        |
| Template    | √        |
| Jinja2      | √        |
| Tags        | √        |
| Serial      | √        |
| Delegate_to | √        |
| Run_once    | √        |
| Block       | √        |
| Rescue      | √        |
| Always      | √        |
| Package     | √        |
| User        | √        |
| File        | √        |
| URI API调用 | √        |



# FAQ

```shell
inventory_hostname是最重要的变量之一.
假设:
web:
  hosts:
    web01:
      ansible_host: 172.16.186.135

    web02:
      ansible_host: 172.16.186.136

Playbook执行时:
hosts: web

运行到web01:
inventory_hostname=web01

运行到web02:
inventory_hostname=web02

所以:
- debug:
    msg: "{{ inventory_hostname }}"
输出:
web01
web02

注意: 它不是IP, 而是Inventory里的名字



# ansible_host是什么
web01:
  ansible_host: 172.16.186.136

这里:
inventory_hostname=web01
ansible_host=172.16.186.135

区别:
变量						值
inventory_hostname		web01
ansible_host			172.16.186.136


# group_names
例如:
web:
  hosts:
    web01:

执行:
- debug:
    var: group_names
结果:
[
 "web"
]

如果:
prod:
  children:
    web:

web:
  hosts:
    web01:

结果:
[
 "prod",
 "web"
]











```

































