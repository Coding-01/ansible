[toc]

# 真正的升级顺序

```shell
真正企业:
旧版本运行
↓
启动新版本
↓
健康检查
↓
LB摘流量
↓
切换流量
↓
停止旧版本

例如:
current -> nginx-1.26

先启动: nginx-1.28
监听: 8080
检查: curl 127.0.0.1:8080
正常后: current -> nginx-1.28
最后: nginx-1.26 -s quit
这才是真正生产升级
```



# readlink

```shell
readlink 是一个用于读取符号链接(Symbolic Link)内容的系统调用/库函数
它的核心作用是读取符号链接文件中存储的目标路径字符串，而不是解析或跟随该链接
需要特别注意：
- 不会打开目标文件
- 不会解析路径
- 只返回符号链接内部保存的字符串
```



# changed_when

```shell
例如:
- command: nginx -t

实际上只是检查配置,不会修改任何东西, 但Ansible默认认为以下模块执行了命令:
command
shell
raw

就算:changed
结果如下, 很难看:
TASK [nginx test]
changed

所以:
- command: nginx -t
  changed_when: false
  
告诉Ansible这个任务永远不算修改, 结果如下，这是最常见用途:
TASK [nginx test]
ok


# 企业实际最常见的changed_when
配置检查:
- command: nginx -t
  changed_when: false

查看版本:
- command: nginx -v
  changed_when: false

获取软链接:
- command: readlink current
  changed_when: false

获取进程:
- shell: pgrep nginx
  changed_when: false

这些本质都是查询,不是修改

更高级的用法,甚至可以自己定义, 例如:
- shell: useradd deploy
  register: result
  changed_when:
    - result.rc == 0           # 只有返回0时才算changed

例如:
- shell: |
    if grep deploy /etc/passwd
    then
      echo EXIST
    else
      useradd deploy
      echo CREATED
    fi
  register: result
  changed_when:
    - "'CREATED' in result.stdout"

如果创建了用户: changed
否则: ok
这才是真正符合幂等思想的写法



register   保存结果
when       条件执行
changed_when   定义什么情况下算修改
这三个一起出现的频率，在企业Playbook里可能超过70%

```



# assert是什么

```shell
相当于: assert 条件
或者:
if 条件不成立
then
    exit 1
fi

例如:
- assert:
    that:
      - nginx_port > 0
      - nginx_port < 65535

如果 nginx_port: 80 则通过
如果 nginx_port: 99999 则直接报错FAILED, 后续任务不执行



- name: verify version
  assert:
    that:
      - nginx_version is defined
      - nginx_version in allowed_versions
意思是:
nginx_version is defined           # 检查变量是否存在
例如有定义则通过:
nginx_version: 1.28.0
如果没写:
nginx_version:
或者压根没有这个变量则直接失败

nginx_version in allowed_versions
例如:
allowed_versions:
  - 1.26.0
  - 1.28.0

这样允许 nginx_version: 1.26.0 或 nginx_version: 1.28.0
如果有人执行 ansible-playbook site.yml -e nginx_version=6.6.6 也直接失败
企业大量使用 assert 来防止误操作


```



# serial数量如何定义

```shell
真正定义原则是一次允许多少台机器离线而不影响业务
小公司 3台Web 通常 serial: 1
10台机器通常 serial: 2
50台机器通常 serial: 5
100台机器通常:(很多公司)
serial:
  - 1
  - 5
  - 10

这是高级写法.
则第一批1台   观察，没问题
第二批5台  还没问题
第三批开始
10台
10台
10台
...
直到结束.
这就是金丝雀发布(Canary Release)


企业里真正会配合的参数不是单独用serial,而是:
serial: 10%
max_fail_percentage: 20

例如100台
每批10台
如果失败2台则还能继续.
如果失败超过20%则停止后面90台不再升级


```



























