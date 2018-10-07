---
layout: post
title: "Nginx uWSGI Flask 部署"
---

本人最近在[搬瓦工](https://bandwagonhost.com/ "bandwagonhost")上部署了自己的Django博客，把自己踩的坑先记录下来，防止下次继续踩坑。主要参考uWSGI的[官方文档](http://uwsgi-docs.readthedocs.io/en/latest/tutorials/Django_and_nginx.html "Setting up Django and your web server with uWSGI and nginx")。以下代码中，”（）“内带中文的要修改为自己的，”#“ 后面的内容是注释。

## 1. 基础环境配置

先安装好基础环境

```
cd ~
sudo yum groupinstall "Development tools"
sudo yum install python34 python34-devel python34-pip
pip3 install virtualenv
virtualenv python34
source python34/bin/activate

```

测试python环境

```
python --version
pip --version
```

如果是python3.4的环境，至此python环境配置好了，如果有错，多用一下Google解决。

## 2. Nginx安装

参考Nginx官网的[安装文档](https://nginx.org/en/linux_packages.html "Nginx安装文档")，建议新手使用yum安装，后续升级维护方便。

1. 首先要安装Nginx仓库，`sudo vi /etc/yum.repos.d/nginx.repo`添加以下内容：

```
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/7/$basearch/
gpgcheck=1
enabled=1
```

2. 下载Nginx的key文件，`wget https://nginx.org/keys/nginx_signing.key`
3. 安装key文件，`sudo rpm --import nginx_signing.key`
4. 安装Nginx，`sudo yum install nginx`
5. 测试Nginx，`sudo nginx`

## 3. uWSGI安装

从此往下都需要在Python34的虚拟环境下，检查自己是否在Python34虚拟环境下，不在的话执行`source python34/bin/activate`确认进入虚拟环境，执行`pip install uwsgi	`安装uWSGI，测试uWSGI是否安装好。执行`vi test.py`输入一下内容保存：

```
def application(env, start_response):
    start_response('200 OK', [('Content-Type','text/html')])
    return [b"Hello World"]
```

 开启uWSGI服务，`uwsgi --http :8000 --wsgi-file test.py`，打开浏览器，输入`(你的IP):8000`看到`Hello World`代表web client到uWSGI到Python的连接正常。

## 4. Django配置

先执行`pip install -r requirements.txt`安装好Python包，正式部署前要配置好Django项目的配置(settings.py)，修改一下项目：

```
DEBUG = False
ALLOWED_HOSTS = ['127.0.0.1','localhost','(你的域名)']
STATIC_URL = '/static/'
STATIC_ROOT = os.path.join(BASE_DIR, 'static')
MEDIA_URL = '/media/'
MEDIA_ROOT = os.path.join(BASE_DIR, 'media')
```

然后记得执行一下语句:

```
python manage.py makemigrations
python manage.py migrate
python manage.py createsuperuser # 创建超级用户
python manage.py collectstatic
```

然后执行`python manage.py runserver`测试Django项目运行是否正常。

## 5. uWSGI到Django配置

```
uwsgi --http :8000 --module （你的Djnago项目名）.wsgi
```

打开浏览器测试（8000端口），如果Django项目运行正常，表示Web Client到uWSGI到Django运行正常。

`vi uwsgi.ini`添加以下内容并保存

```
[uwsgi]
socket = :8001
chdir= (你的Django项目地址)
wsgi-file = (你的Django项目地址下wsgi.py所在位置)
touch-reload = ~/reload # 如果reload文件有更改，重启uWSGI
module=(你的Django项目名称).wsgi
master=true
processes = 2 # 进程数
threads = 4 # 线程数
chmod-socket    = 664
chown-socket = （centos用户名）:（centos组）
vacuum=true
pidfile = ~/tmp/uwsgi.pid #pid 文件位置 
```

执行`uWSGI --ini uwsgi.ini  &`运行uWSGI，如果Django项目有更改的话可以执行`touch ~/reload`重启uWSGI。

## 6. 配置Nginx 

执行 `sudo vi /etc/nginx/nginx.conf`修改添加一下内容

```
# 注意，这只是nginx.conf中的一部分，其他内容不熟悉的话不要修改。

http {
    upstream django {
        server 127.0.0.1:8001;
    }

    server {
        listen       80;
        server_name  （你的域名地址）;
        if ($host != 'www.willtunner.me' ) {
            rewrite ^/(.*)$ http://www.willtunner.me/$1 permanent;
        }   # 此处修改自动添加www

        charset utf-8;
        access_log  logs/host.access.log  main;

        location / {
            uwsgi_pass  django;
            include     /etc/nginx/uwsgi_params;
        }

        location /static/ {
            alias （你的Django项目static地址）;
        }

        location /media/ {
            alias （你的Django项目media地址）;
        }

        location ~ .*.txt$ {
            root （robots.txt所在地址）;
        }
    }
}
```

当然Nginx虚拟主机配置最好使用include语法放置在其他文件夹下，多个虚拟主机比较好管理，这里我只有一个虚拟主机，故直接在这里修改了。

改好了之后执行`sudo nginx -s stop`和`sudo nginx`重新加载Nginx的配置。

没有出错的话，至此从Web Client到Nginx到uWSGi到Django的通路便建立好了。网站可以正常访问了。