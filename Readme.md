# Занятие.9 Инициализация системы. Systemd
# Systemd — создание unit-файла

# 1.Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова
# (файл лога и ключевое слово должны задаваться в /etc/default).

# Пишем сам скрипт
root@deb-vm:/home/vital# nano SnoopDog

#!/bin/bash

WORD=$1
LOG=$2
DATE=`date`

if grep $WORD $LOG &> /dev/null
then
logger "$DATE: I found word, Master!"
else
exit 0
fi

# Делаем его исполняемым
root@deb-vm:/home/vital# chmod +x SnoopDog

# Рисуем лог файл с произвольным содержимым, добавляем в него слово ALERT
root@deb-vm:/home/vital# nano /var/log/SnoopDog.log

# Подготавливаем конфигурационный файл юнита
root@deb-vm:/home/vital# nano /etc/default/SnoopDog

#Configuration file for my watchlog service
#Place it to /etc/default
#File and word in that file that we will be monit
WORD="ALERT"
LOG=/var/log/SnoopDog.log

# Создаем юнит
[Unit]
Description=My SnoopDog

[Service]
Type=oneshot
EnvironmentFile=/etc/default/SnoopDog
ExecStart=/home/vital/SnoopDog $WORD $LOG

# Создаем таймер
[Unit]
Description=Run Snoopdog script every 30 second

[Timer]
#Run every 30 second
OnUnitActiveSec=30
Unit=SnoopDog.service

[Install]
WantedBy=multi-user.target

# Стартуем таймер
root@deb-vm:/home/vital# systemctl start SnoopDog.timer

# Проверяем
root@deb-vm:/home/vital# journalctl | grep Master
#Ничего нет....(((

# Тогда так
root@deb-vm:/home/vital# systemctl start SnoopDog.service

# Проверяем
root@deb-vm:/home/vital# journalctl | grep Master
Jun 07 21:20:09 deb-vm root[1106]: Sat Jun  7 09:20:09 PM +05 2025: I found word, Master!
Jun 07 21:20:44 deb-vm root[1113]: Sat Jun  7 09:20:44 PM +05 2025: I found word, Master!
Jun 07 21:21:34 deb-vm root[1123]: Sat Jun  7 09:21:34 PM +05 2025: I found word, Master!
#О как ! 30-ю секундами тут и не пахнет, потому как:
#"Еще один важный момент - точность. По умолчанию точность таймеров systemd равна одно минуте, это значит,
#что при наступлении указанного в таймере времени запуск сервиса произойдет в случайный промежуток времени,
#равный одной минуте. Это сделано для того, чтобы разнести срабатывание таймеров, назначенных на одно время
#и исключить повышенную нагрузку на систему. Изменить это поведение можно при помощи директивы AccuracySec,
#с ее помощью можно как увеличить, так и уменьшить значение точности". Добавим в таймер:
AccuracySec=1

# Затем
root@deb-vm:/home/vital# systemctl daemon-reload

# Ждемс...

# Проверяем
root@deb-vm:/home/vital# journalctl | grep Master
Jun 07 22:35:04 deb-vm root[1938]: Sat Jun  7 10:35:04 PM +05 2025: I found word, Master!
Jun 07 22:35:35 deb-vm root[1942]: Sat Jun  7 10:35:35 PM +05 2025: I found word, Master!
Jun 07 22:36:06 deb-vm root[1946]: Sat Jun  7 10:36:06 PM +05 2025: I found word, Master!
Jun 07 22:36:37 deb-vm root[1951]: Sat Jun  7 10:36:37 PM +05 2025: I found word, Master!
#Задание выполнено

# 2.Установить spawn-fcgi и создать unit-файл (spawn-fcgi.sevice) с помощью переделки init-скрипта 
# (https://gist.github.com/cea2k/1318020).
#Устанавливаем все необходимые пакеты
root@deb-vm:/home/vital# apt install spawn-fcgi php php-cgi php-cli apache2 libapache2-mod-fcgid -y

# Создаем конфигурационный файл, предварительно создав соответствующую директорию
root@deb-vm:/home/vital# mkdir /etc/spawn-fcgi/fcgi.conf && touch /etc/spawn-fcgi/fcgi.conf

# Наполняем файл содержимым
root@deb-vm:/home/vital# nano /etc/spawn-fcgi/fcgi.conf
#You must set some working options before the "spawn-fcgi" service will work.
#If SOCKET points to a file, then this file is cleaned up by the init script.
#
#See spawn-fcgi(1) for all possible options.
#
#Example :
SOCKET=/var/run/php-fcgi.sock
OPTIONS="-u www-data -g www-data -s $SOCKET -S -M 0600 -C 32 -F 1 -- /usr/bin/php-cgi"

# Создаем файл юнита
root@deb-vm:/home/vital# nano /etc/systemd/system/spawn-fcgi.service
[Unit]
Description=Spawn-fcgi startup service by Otus
After=network.target

[Service]
Type=simple
PIDFile=/var/run/spawn-fcgi.pid
EnvironmentFile=/etc/spawn-fcgi/fcgi.conf
ExecStart=/usr/bin/spawn-fcgi -n $OPTIONS
KillMode=process

[Install]
WantedBy=multi-user.target

# Проверяем
root@deb-vm:/home/vital# systemctl start spawn-fcgi
root@deb-vm:/home/vital# systemctl status spawn-fcgi
? spawn-fcgi.service - Spawn-fcgi startup service by Otus
     Loaded: loaded (/etc/systemd/system/spawn-fcgi.service; disabled; preset: enabled)
     Active: active (running) since Sun 2025-06-08 10:13:42 +05; 8s ago
   Main PID: 12972 (php-cgi)
      Tasks: 33 (limit: 2312)
     Memory: 15.5M
        CPU: 70ms
     CGroup: /system.slice/spawn-fcgi.service
             +-12972 /usr/bin/php-cgi
             +-12973 /usr/bin/php-cgi
             +-12974 /usr/bin/php-cgi
             +-12975 /usr/bin/php-cgi
             +-12976 /usr/bin/php-cgi
             +-12977 /usr/bin/php-cgi
             +-12978 /usr/bin/php-cgi
             +-12979 /usr/bin/php-cgi
             +-12980 /usr/bin/php-cgi
             +-12981 /usr/bin/php-cgi
             +-12982 /usr/bin/php-cgi
             +-12983 /usr/bin/php-cgi
             +-12984 /usr/bin/php-cgi
             +-12985 /usr/bin/php-cgi
             +-12986 /usr/bin/php-cgi
             +-12987 /usr/bin/php-cgi
             +-12988 /usr/bin/php-cgi
             +-12989 /usr/bin/php-cgi
             +-12990 /usr/bin/php-cgi
             +-12991 /usr/bin/php-cgi
             +-12992 /usr/bin/php-cgi
             +-12993 /usr/bin/php-cgi
             +-12994 /usr/bin/php-cgi
             +-12995 /usr/bin/php-cgi
             +-12996 /usr/bin/php-cgi
             +-12997 /usr/bin/php-cgi
             +-12998 /usr/bin/php-cgi
             +-12999 /usr/bin/php-cgi
             +-13000 /usr/bin/php-cgi
             +-13001 /usr/bin/php-cgi
             +-13002 /usr/bin/php-cgi
             +-13003 /usr/bin/php-cgi
             L-13004 /usr/bin/php-cgi
root@deb-vm:/home/vital# systemctl stop spawn-fcgi
root@deb-vm:/home/vital# systemctl status spawn-fcgi
0 spawn-fcgi.service - Spawn-fcgi startup service by Otus
     Loaded: loaded (/etc/systemd/system/spawn-fcgi.service; disabled; preset: enabled)
     Active: inactive (dead)

Jun 08 10:14:05 deb-vm systemd[1]: spawn-fcgi.service: Unit process 12997 (php-cgi) remains running after unit stopped.
Jun 08 10:14:05 deb-vm systemd[1]: spawn-fcgi.service: Unit process 12999 (php-cgi) remains running after unit stopped.
Jun 08 10:14:05 deb-vm systemd[1]: spawn-fcgi.service: Unit process 13000 (php-cgi) remains running after unit stopped.
Jun 08 10:14:05 deb-vm systemd[1]: spawn-fcgi.service: Unit process 13001 (php-cgi) remains running after unit stopped.
Jun 08 10:14:05 deb-vm systemd[1]: spawn-fcgi.service: Unit process 13002 (php-cgi) remains running after unit stopped.
Jun 08 10:14:05 deb-vm systemd[1]: spawn-fcgi.service: Unit process 13003 (php-cgi) remains running after unit stopped.
Jun 08 10:14:05 deb-vm systemd[1]: spawn-fcgi.service: Unit process 13004 (php-cgi) remains running after unit stopped.
Jun 08 10:14:05 deb-vm systemd[1]: Stopped spawn-fcgi.service - Spawn-fcgi startup service by Otus.
Jun 08 10:14:08 deb-vm systemd[1]: /etc/systemd/system/spawn-fcgi.service:7: PIDFile= references a path below legacy directory /var/run/, updating /var/run/spawn-fcgi.pid > /run/spawn-fcgi.pid; >
Jun 08 10:14:51 deb-vm systemd[1]: /etc/systemd/system/spawn-fcgi.service:7: PIDFile= references a path below legacy directory /var/run/, updating /var/run/spawn-fcgi.pid > /run/spawn-fcgi.pid;

# Корректируем 
root@deb-vm:/home/vital# nano /etc/systemd/system/spawn-fcgi.service
#Заменяем 
PIDFile=/var/run/spawn-fcgi.pid
#На
PIDFile=/run/spawn-fcgi.pid

# Проверяем еще раз
root@deb-vm:/home/vital# systemctl daemon-reload
root@deb-vm:/home/vital# systemctl start spawn-fcgi
root@deb-vm:/home/vital# systemctl status spawn-fcgi
root@deb-vm:/home/vital# systemctl stop spawn-fcgi
root@deb-vm:/home/vital# systemctl status spawn-fcgi
#Все работает,задание выполнено

# 3.Доработать unit-файл Nginx (nginx.service) для запуска нескольких инстансов сервера с разными конфигурационными 
# файлами одновременно.

# Ставим nginx
root@deb-vm:/home/vital# apt install nginx -y

# Копируем дефолтный файл юнита, затем модифицируем его в шаблон для запуска нескольких экземпляров
root@deb-vm:/home/vital# cp /usr/lib/systemd/system/nginx.service /etc/systemd/system/nginx@.service
root@deb-vm:/home/vital# nano /etc/systemd/system/nginx@.service
#Stop dance for nginx
#=======================
#
#ExecStop sends SIGSTOP (graceful stop) to the nginx process.
#If, after 5s (--retry QUIT/5) nginx is still running, systemd takes control
#and sends SIGTERM (fast shutdown) to the main process.
#After another 5s (TimeoutStopSec=5), and if nginx is alive, systemd sends
#SIGKILL to all the remaining processes in the process group (KillMode=mixed).
#
#nginx signals reference doc:
#http://nginx.org/en/docs/control.html
#
[Unit]
Description=A high performance web server and a reverse proxy server number %I
Documentation=man:nginx(8)
After=network.target nss-lookup.target

[Service]
Type=forking
PIDFile=/run/nginx%I.pid
ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx/nginx%I.conf -q -g 'daemon on; master_process on;'
ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx%I.conf -g 'daemon on; master_process on;'
ExecReload=/usr/sbin/nginx -c /etc/nginx/nginx%I.conf -g 'daemon on; master_process on;' -s reload
ExecStop=-/sbin/start-stop-daemon --quiet --stop --retry QUIT/5 --pidfile /run/nginx%I.pid
TimeoutStopSec=5
KillMode=mixed

[Install]
WantedBy=multi-user.target

# Создаем конфигурационные файлы для экземпляров юнита, указанные в юните
root@deb-vm:/home/vital# сp /etc/nginx/{nginx.conf,nginx1.conf}; сp /etc/nginx/{nginx.conf,nginx2.conf}

# Правим файл экземпляра под свою задачу
root@deb-vm:/home/vital# nano /etc/nginx/nginx1.conf
pid /run/nginx1.pid
server {
		listen 9001;
	}
# Правим файл экземпляра под свою задачу
root@deb-vm:/home/vital# nano /etc/nginx/nginx2.conf
pid /run/nginx2.pid
server {
		listen 9002;
	}

# Запускаем экземпляры и проверяем работу
root@deb-vm:/home/vital# systemctl start nginx@1
root@deb-vm:/home/vital# systemctl start nginx@2
root@deb-vm:/home/vital# systemctl status nginx@*
● nginx@2.service - A high performance web server and a reverse proxy server number 2
     Loaded: loaded (/etc/systemd/system/nginx@.service; disabled; preset: enabled)
     Active: active (running) since Sun 2025-06-08 14:05:11 +05; 13min ago
       Docs: man:nginx(8)
    Process: 1013 ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx/nginx2.conf -q -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
    Process: 1014 ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx2.conf -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
   Main PID: 1015 (nginx)
      Tasks: 2 (limit: 2312)
     Memory: 1.7M
        CPU: 21ms
     CGroup: /system.slice/system-nginx.slice/nginx@2.service
             ├─1015 "nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx2.conf -g daemon on; master_process on;"
             └─1016 "nginx: worker process"

Jun 08 14:05:11 deb-vm systemd[1]: Starting nginx@2.service - A high performance web server and a reverse proxy server number 2...
Jun 08 14:05:11 deb-vm systemd[1]: Started nginx@2.service - A high performance web server and a reverse proxy server number 2.

● nginx@1.service - A high performance web server and a reverse proxy server number 1
     Loaded: loaded (/etc/systemd/system/nginx@.service; disabled; preset: enabled)
     Active: active (running) since Sun 2025-06-08 14:05:08 +05; 13min ago
       Docs: man:nginx(8)
    Process: 1007 ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx/nginx1.conf -q -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
    Process: 1008 ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx1.conf -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
   Main PID: 1009 (nginx)
      Tasks: 2 (limit: 2312)
     Memory: 1.7M
        CPU: 21ms
     CGroup: /system.slice/system-nginx.slice/nginx@1.service
             ├─1009 "nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx1.conf -g daemon on; master_process on;"
             └─1010 "nginx: worker process"

Jun 08 14:05:08 deb-vm systemd[1]: Starting nginx@1.service - A high performance web server and a reverse proxy server number 1...
Jun 08 14:05:08 deb-vm systemd[1]: Started nginx@1.service - A high performance web server and a reverse proxy server number 1.

root@deb-vm:/home/vital# ss -tunlp | grep nginx
tcp   LISTEN 0      511          0.0.0.0:9002      0.0.0.0:*    users:(("nginx",pid=1016,fd=5),("nginx",pid=1015,fd=5))
tcp   LISTEN 0      511          0.0.0.0:9001      0.0.0.0:*    users:(("nginx",pid=1010,fd=5),("nginx",pid=1009,fd=5))
tcp   LISTEN 0      511          0.0.0.0:80        0.0.0.0:*    users:(("nginx",pid=676,fd=5),("nginx",pid=670,fd=5))
tcp   LISTEN 0      511             [::]:80           [::]:*    users:(("nginx",pid=676,fd=6),("nginx",pid=670,fd=6))
#Все работает, задание выполнено
