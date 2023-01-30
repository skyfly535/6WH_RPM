# Управление пакетами. Дистрибьюция софта.
## Развертывание Vagrant стенда с локальным репозиторием и размещенным там своим RPM пакетом.

Преобразуем `Vagrantfile` из ДЗ по NFS. Создадим сервер с локальным репозиторием и размещенным там своим RPM пакетом. И создади клиента, на котором установим пакет из этого репозитория.

Скрипт `repos_script.sh` используемый для развертывания сервера с репозиторием

```
#!/bin/bash
 sudo su

# Устанавливаем необходимый набор пакетов gcc для компиляции сборки NGINX
yum install -y \
 redhat-lsb-core \
 wget \
 rpmdevtools \
 rpm-build \
 createrepo \
 yum-utils \
 gcc

# Загружаем SRPM пакет NGINX (так как разворачиваем CEntOS 7, то и пакет качаем для этой ОС)
cd
wget https://nginx.org/packages/centos/7/SRPMS/nginx-1.14.1-1.el7_4.ngx.src.rpm
rpm -i nginx-1.14.1-1.el7_4.ngx.src.rpm

# Проставляем зависимости
yes | yum-builddep /root/rpmbuild/SPECS/nginx.spec

# Качаем и разархивируем последний исходники для openssl (без параметра --no-check-certificate не загружается)
wget --no-check-certificate https://www.openssl.org/source/openssl-1.1.1k.tar.gz -O /root/openssl-1.1.1k.tar.gz
#mkdir /root/openssl-1.1.1k
tar -xvf openssl-1.1.1k.tar.gz

# Редактируем файлы пакета SRPM
sed -i 's@index.htm;@index.htm;\n        autoindex on;@g' /root/rpmbuild/SOURCES/nginx.vh.default.conf
sed -i 's@--with-ld-opt="%{WITH_LD_OPT}" @--with-ld-opt="%{WITH_LD_OPT}" \\\n    --with-openssl=/root/openssl-1.1.1k @g' /root/rpmbuild/SPECS/nginx.spec

# Производим сборку RPM пакета
rpmbuild -bb /root/rpmbuild/SPECS/nginx.spec

# Производим локальное развертывание NGINX
yum localinstall -y /root/rpmbuild/RPMS/x86_64/nginx-1.14.1-1.el7_4.ngx.x86_64.rpm

# Ставим NGINX в автозагрузку
systemctl enable nginx
systemctl start nginx

# Создадим каталог для будующего репозитория
mkdir -p /usr/share/nginx/html/repo

# Перемещаем в созданный каталог собранный RPM пакет NGINX
mv /root/rpmbuild/RPMS/x86_64/nginx-1.14.1-1.el7_4.ngx.x86_64.rpm /usr/share/nginx/html/repo/

# Качаем в каталог репозитория RPM для установки репозитория Percona-Server
wget https://downloads.percona.com/downloads/percona-release/percona-release-1.0-6/redhat/percona-release-1.0-6.noarch.rpm -O /usr/share/nginx/html/repo/percona-release-0.1-6.noarch.rpm

# Инициализируем репозиторий
createrepo /usr/share/nginx/html/repo/

# Перезапускаем NGINX
nginx -s reload

yes | rm -r /home/vagrant/nginx-1.14.1-1.el7_4.ngx.src.rpm
```
Скрипт `repoс_script.sh` используемый для развертывания клиентской VM.

```
#!/bin/bash
sudo su

# Устанавливаем Lynx
yum install -y lynx

# Добавляем конфиг нашего репозитория на серверне 
cat >> /etc/yum.repos.d/otus.repo << EOF
[otus]
name=otus-linux
baseurl=http://192.168.56.10/repo
gpgcheck=0
enabled=1
EOF
```

## Проверка работоспособности

Идем на сервер.

NGINX установлен из локального репозитория

```
[root@repoServer vagrant]# systemctl status nginx
● nginx.service - nginx - high performance web server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: active (running) since Вс 2023-01-29 14:38:20 UTC; 21min ago
     Docs: http://nginx.org/en/docs/
  Process: 19157 ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx.conf (code=exited, status=0/SUCCESS)
 Main PID: 19158 (nginx)
   CGroup: /system.slice/nginx.service
           ├─19158 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx.conf
           └─19166 nginx: worker process

янв 29 14:38:20 repoServer systemd[1]: Starting nginx - high performance web server...
янв 29 14:38:20 repoServer systemd[1]: Can't open PID file /var/run/nginx.pid (yet?) after start: No such file or directory
янв 29 14:38:20 repoServer systemd[1]: Started nginx - high performance web server.
```
Смотрим при помощи `lynx` адрес локального репозитория http://localhost/repo/
```
[root@repoServer vagrant]# lynx http://localhost/repo/

...

    Index of /repo/
     _____________________________________________________________________________________________

../
repodata/                                          29-Jan-2023 14:38                   -
percona-release-0.1-6.noarch.rpm                   11-Nov-2020 21:49               17560
     _____________________________________________________________________________________________
```
Переходим на клиент.

Прроверяем подключенный репозиторий и его содержимое.

```
[root@repoClient vagrant]# yum repolist enabled | grep otus
Failed to set locale, defaulting to C
otus                            otus-linux                                    2

[root@repoClient vagrant]# yum list | grep otus
Failed to set locale, defaulting to C
percona-release.noarch                      1.0-6                      @otus    
nginx.x86_64                                1:1.14.1-1.el7_4.ngx       otus    
```
Устанавливаем собранный пакет NGINX из репозитория `otus` на сервере

```
[root@repoClient vagrant]# yum install nginx

...

Total download size: 2.1 M
Installed size: 5.9 M
Is this ok [y/d/N]: y
Downloading packages:
No Presto metadata available for otus
nginx-1.14.1-1.el7_4.ngx.x86_64.rpm                                                                                                                                                 | 2.1 MB  00:00:00     
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : 1:nginx-1.14.1-1.el7_4.ngx.x86_64                                                                                                                                                       1/1 
----------------------------------------------------------------------

Thanks for using nginx!

Please find the official documentation for nginx here:
* http://nginx.org/en/docs/

Please subscribe to nginx-announce mailing list to get
the most important news about nginx:
* http://nginx.org/en/support.html

Commercial subscriptions for nginx are available on:
* http://nginx.com/products/

----------------------------------------------------------------------
  Verifying  : 1:nginx-1.14.1-1.el7_4.ngx.x86_64                                                                                                                                                       1/1 

Installed:
  nginx.x86_64 1:1.14.1-1.el7_4.ngx                                                                                                                                                                        

Complete!
```

Все работает

```
[root@repoClient vagrant]# systemctl status nginx
● nginx.service - nginx - high performance web server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sun 2023-01-29 15:56:50 UTC; 1s ago
     Docs: http://nginx.org/en/docs/
  Process: 22369 ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx.conf (code=exited, status=0/SUCCESS)
 Main PID: 22370 (nginx)
   CGroup: /system.slice/nginx.service
           ├─22370 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx.conf
           └─22371 nginx: worker process

Jan 29 15:56:50 repoClient systemd[1]: Starting nginx - high performance web server...
Jan 29 15:56:50 repoClient systemd[1]: Started nginx - high performance web server.
```