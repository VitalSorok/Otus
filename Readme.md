# Занятие 10. Размещаем свой RPM в своем репозитории
# 1.Создать свой RPM пакет (можно взять свое приложение, либо собрать, например,
# Apache с определенными опциями).

# Установливаем необходимые пакеты:
[root@cent-vm vital]# yum install -y wget rpmdevtools rpm-build createrepo yum-utils cmake gcc git nano

# Выполняем подготовку
[root@cent-vm vital]# mkdir rpm && cd rpm

# Загружаем SRPM для nginx
[root@cent-vm rpm]# yumdownloader --source nginx

# Ставим зависимости
[root@cent-vm rpm]# rpm -Uvh nginx*.src.rpm
[root@cent-vm rpm]# yum-builddep nginx

# Получаем исходный код модуля brotli
[root@cent-vm rpm]# cd ..
[root@cent-vm vital]# git clone --recurse-submodules -j8 https://github.com/google/ngx_brotli
[root@cent-vm vital]# cd ngx_brotli/deps/brotli
[root@cent-vm vital]# mkdir out && cd out

# Собираем сам модуль ngx_brotli:
[root@cent-vm out]# cmake -DCMAKE_BUILD_TYPE=Release -DBUILD_SHARED_LIBS=OFF -DCMAKE_C_FLAGS="-Ofast -m64 -march=native -mtune=native -flto -funroll-loops -ffunction-sections -fdata-sections -Wl,--gc-sections" -DCMAKE_CXX_FLAGS="-Ofast -m64 -march=native -mtune=native -flto -funroll-loops -ffunction-sections -fdata-sections -Wl,--gc-sections" -DCMAKE_INSTALL_PREFIX=./installed ..
[root@cent-vm out]# cmake --build . --config Release -j 2 --target brotlienc
[root@cent-vm out]# cd ../../../..

# Редактируем spec файл, чтобы Nginx собирался с необходимыми нам опциями
[root@cent-vm vital]# nano /root/rpmbuild/SPECS/nginx.spec
--add-module=/home/vital/ngx_brotli \

# Приступаем к сборке
[root@cent-vm vital]# cd ~/rpmbuild/SPECS/
[root@cent-vm SPECS]# rpmbuild -ba nginx.spec -D 'debug_package %{nil}'


# Проверим что получилось
[root@cent-vm SPECS]# ls -la ~/rpmbuild/RPMS/x86_64/
drwxr-xr-x. 2 root root    4096 Jun  8 07:01 .
drwxr-xr-x. 4 root root      34 Jun  8 06:52 ..
-rw-r--r--. 1 root root   36259 Jun  8 06:52 nginx-1.20.1-22.el9.x86_64.rpm
-rw-r--r--. 1 root root    7329 Jun  8 07:01 nginx-all-modules-1.20.1-22.el9.noarch.rpm
-rw-r--r--. 1 root root 1033103 Jun  8 06:52 nginx-core-1.20.1-22.el9.x86_64.rpm
-rw-r--r--. 1 root root    8978 Jun  8 07:01 nginx-filesystem-1.20.1-22.el9.noarch.rpm
-rw-r--r--. 1 root root  759896 Jun  8 06:52 nginx-mod-devel-1.20.1-22.el9.x86_64.rpm
-rw-r--r--. 1 root root   19354 Jun  8 06:52 nginx-mod-http-image-filter-1.20.1-22.el9.x86_64.rpm
-rw-r--r--. 1 root root   30876 Jun  8 06:52 nginx-mod-http-perl-1.20.1-22.el9.x86_64.rpm
-rw-r--r--. 1 root root   18160 Jun  8 06:52 nginx-mod-http-xslt-filter-1.20.1-22.el9.x86_64.rpm
-rw-r--r--. 1 root root   53765 Jun  8 06:52 nginx-mod-mail-1.20.1-22.el9.x86_64.rpm
-rw-r--r--. 1 root root   80313 Jun  8 06:52 nginx-mod-stream-1.20.1-22.el9.x86_64.rpm

# Копируем пакеты в общий каталог:
[root@cent-vm SPECS]# cp ~/rpmbuild/RPMS/noarch/* ~/rpmbuild/RPMS/x86_64/
[root@cent-vm SPECS]# cd ~/rpmbuild/RPMS/x86_64

# Устанавливаем наш пакет
[root@cent-vm x86_64]# yum localinstall *.rpm

# Запускаем установленное и проверяем
[root@cent-vm x86_64]# systemctl start nginx
[root@cent-vm x86_64]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabled)
     Active: active (running) since Sun 2025-06-08 07:02:50 EDT; 54min ago
    Process: 41652 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 41653 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 41654 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 41655 (nginx)
      Tasks: 3 (limit: 10377)
     Memory: 7.1M
        CPU: 263ms
     CGroup: /system.slice/nginx.service
             ├─41655 "nginx: master process /usr/sbin/nginx"
             ├─41689 "nginx: worker process"
             └─41690 "nginx: worker process"

Jun 08 07:02:50 cent-vm systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jun 08 07:02:50 cent-vm nginx[41653]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jun 08 07:02:50 cent-vm nginx[41653]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jun 08 07:02:50 cent-vm systemd[1]: Started The nginx HTTP and reverse proxy server.

# 2.Создать свой репозиторий и разместить там ранее собранный RPM
# Готовим директорию
[root@cent-vm vital]# mkdir /usr/share/nginx/html/repo

# Копируем туда наши собранные RPM-пакеты
[root@cent-vm vital]# cp ~/rpmbuild/RPMS/x86_64/*.rpm /usr/share/nginx/html/repo/

# Инициализируем репозиторий командой:
[root@cent-vm vital]# createrepo /usr/share/nginx/html/repo/

# Настроим в NGINX доступ к листингу каталога. В файле /etc/nginx/nginx.conf в блоке server добавим следующие директивы:

	index index.html index.htm;
	autoindex on;

# Проверяем синтаксис и перезапускаем NGINX:
[root@cent-vm vital]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
[root@cent-vm vital]# nginx -s reload

# Проверяем
[root@cent-vm vital]# curl -a http://localhost/repo/
<html>
<head><title>Index of /repo/</title></head>
<body>
<h1>Index of /repo/</h1><hr><pre><a href="../">../</a>
<a href="repodata/">repodata/</a>                                          08-Jun-2025 11:19                   -
<a href="nginx-1.20.1-22.el9.x86_64.rpm">nginx-1.20.1-22.el9.x86_64.rpm</a>                     08-Jun-2025 11:06               36259
<a href="nginx-all-modules-1.20.1-22.el9.noarch.rpm">nginx-all-modules-1.20.1-22.el9.noarch.rpm</a>         08-Jun-2025 11:06                7329
<a href="nginx-core-1.20.1-22.el9.x86_64.rpm">nginx-core-1.20.1-22.el9.x86_64.rpm</a>                08-Jun-2025 11:06             1033103
<a href="nginx-filesystem-1.20.1-22.el9.noarch.rpm">nginx-filesystem-1.20.1-22.el9.noarch.rpm</a>          08-Jun-2025 11:06                8978
<a href="nginx-mod-devel-1.20.1-22.el9.x86_64.rpm">nginx-mod-devel-1.20.1-22.el9.x86_64.rpm</a>           08-Jun-2025 11:06              759896
<a href="nginx-mod-http-image-filter-1.20.1-22.el9.x86_64.rpm">nginx-mod-http-image-filter-1.20.1-22.el9.x86_6..&gt;</a> 08-Jun-2025 11:06               19354
<a href="nginx-mod-http-perl-1.20.1-22.el9.x86_64.rpm">nginx-mod-http-perl-1.20.1-22.el9.x86_64.rpm</a>       08-Jun-2025 11:06               30876
<a href="nginx-mod-http-xslt-filter-1.20.1-22.el9.x86_64.rpm">nginx-mod-http-xslt-filter-1.20.1-22.el9.x86_64..&gt;</a> 08-Jun-2025 11:06               18160
<a href="nginx-mod-mail-1.20.1-22.el9.x86_64.rpm">nginx-mod-mail-1.20.1-22.el9.x86_64.rpm</a>            08-Jun-2025 11:06               53765
<a href="nginx-mod-stream-1.20.1-22.el9.x86_64.rpm">nginx-mod-stream-1.20.1-22.el9.x86_64.rpm</a>          08-Jun-2025 11:06               80313
<a href="percona-release-latest.noarch.rpm">percona-release-latest.noarch.rpm</a>                  12-Feb-2025 14:02               28300
</pre><hr></body>
</html>

# Добавим репозиторий в /etc/yum.repos.d:
[root@cent-vm vital]# nano /etc/yum.repos.d/otus.repo
[otus]
name=otus-linux
baseurl=http://localhost/repo
gpgcheck=0
enabled=1


# Убедимся, что репозиторий подключился и посмотрим, что в нем есть
[root@cent-vm vital]# yum repolist enabled | grep otus
otus otus-linux

# Добавим пакет в наш репозиторий:
[root@cent-vm vital]# cd /usr/share/nginx/html/repo/
[root@cent-vm repo]# wget https://repo.percona.com/yum/percona-release-latest.noarch.rpm

# Обновим список пакетов в репозитории:
[root@cent-vm repo]# createrepo /usr/share/nginx/html/repo/
[root@cent-vm repo]# yum makecache
[root@cent-vm repo]# yum list | grep otus
percona-release.noarch 	1.0-30 		otus

# Так как Nginx у нас уже стоит, установим репозиторий percona-release:

[root@cent-vm repo]# yum install -y percona-release.noarch

# Все прошло успешно. 
#Ссылка на репозиторий
http://a36a0aa61be7.sn.mynetname.net:1200/repo/

