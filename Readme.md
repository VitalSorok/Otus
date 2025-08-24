# Задание: "Настраиваем центральный сервер для сбора логов"

# 1. В наличии две ВМ: 

# Клиент вот с такими данными:
root@deb-vm:/home/vital# uname -a
Linux deb-vm 6.1.0-37-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.140-1 (2025-05-22) x86_64 GNU/Linux

root@deb-vm:/home/vital# nginx -v
nginx version: nginx/1.22.1

root@deb-vm:/home/vital# rsyslogd -v
rsyslogd  8.2302.0 (aka 2023.02) compiled with:
        PLATFORM:                               x86_64-pc-linux-gnu
        PLATFORM (lsb_release -d):
        FEATURE_REGEXP:                         Yes
        GSSAPI Kerberos 5 support:              Yes
        FEATURE_DEBUG (debug build, slow code): No
        32bit Atomic operations supported:      Yes
        64bit Atomic operations supported:      Yes
        memory allocator:                       system default
        Runtime Instrumentation (slow code):    No
        uuid support:                           Yes
        systemd support:                        Yes
        Config file:                            /etc/rsyslog.conf
        PID file:                               /run/rsyslogd.pid
        Number of Bits in RainerScript integers: 64

See https://www.rsyslog.com for more information

root@deb-vm:/home/vital# apt search auditd
Sorting... Done
Full Text Search... Done
auditd/oldstable,now 1:3.0.9-1 amd64 [installed]
  User space tools for security auditing

root@deb-vm:/home/vital# ip a | grep inet
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host noprefixroute
    inet 192.168.122.220/24 brd 192.168.122.255 scope global enp1s0
    inet6 fe80::5054:ff:fe24:2e8c/64 scope link

# И почти точно такой же сервер
root@deb-vm2:/home/vital# uname -a
Linux deb-vm2 6.1.0-37-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.140-1 (2025-05-22) x86_64 GNU/Linux
root@deb-vm2:/home/vital# rsyslogd -v
rsyslogd  8.2302.0 (aka 2023.02) compiled with:
        PLATFORM:                               x86_64-pc-linux-gnu
        PLATFORM (lsb_release -d):
        FEATURE_REGEXP:                         Yes
        GSSAPI Kerberos 5 support:              Yes
        FEATURE_DEBUG (debug build, slow code): No
        32bit Atomic operations supported:      Yes
        64bit Atomic operations supported:      Yes
        memory allocator:                       system default
        Runtime Instrumentation (slow code):    No
        uuid support:                           Yes
        systemd support:                        Yes
        Config file:                            /etc/rsyslog.conf
        PID file:                               /run/rsyslogd.pid
        Number of Bits in RainerScript integers: 64

See https://www.rsyslog.com for more information.

root@deb-vm2:/home/vital# ip a | grep inet
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host noprefixroute
    inet 192.168.122.20/24 brd 192.168.122.255 scope global dynamic noprefixroute enp1s0
    inet6 fe80::5054:ff:fe32:a584/64 scope link noprefixroute
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0


# То есть на клиенте работает веб-сервер, утилита для управления логами(запись и отправка логов на сервер), аудит который  
# "отслеживает" нужные нам моменты(в данном случае папка конфигов веб-сервера).
# На сервере - просто приемник логов на базе той же самой утилиты. Сервер и клиент в одном широковещательном домене, порты
# открыты

# 2. Подготовим сервер
# Проверим
root@deb-vm2:/home/vital# cat /etc/rsyslog.conf | grep -P 'imudp|imtcp'
module(load="imudp")
input(type="imudp" port="514")
module(load="imtcp")
input(type="imtcp" port="514")
# Все отлично

# Добавим в конец файли строки
cat <<EOF >> /etc/rsyslog.conf
#Add remote logs
$template RemoteLogs,"/var/log/rsyslog/%HOSTNAME%/%PROGRAMNAME%.log"
*.* ?RemoteLogs
EOF 

# Перезапустим rsyslog,проверим статус, слушающие порты
root@deb-vm2:/home/vital# systemctl restart rsyslog && systemctl status rsyslog && ss -tunlp | grep 514
● rsyslog.service - System Logging Service
     Loaded: loaded (/lib/systemd/system/rsyslog.service; enabled; preset: enabled)
     Active: active (running) since Sun 2025-08-24 19:58:41 +05; 23ms ago
TriggeredBy: ● syslog.socket
       Docs: man:rsyslogd(8)
             man:rsyslog.conf(5)
             https://www.rsyslog.com/doc/
   Main PID: 5891 (rsyslogd)
      Tasks: 10 (limit: 2289)
     Memory: 3.1M
        CPU: 9ms
     CGroup: /system.slice/rsyslog.service
             └─5891 /usr/sbin/rsyslogd -n -iNONE

Aug 24 19:58:41 deb-vm2 systemd[1]: Starting rsyslog.service - System Logging Service...
Aug 24 19:58:41 deb-vm2 systemd[1]: Started rsyslog.service - System Logging Service.
Aug 24 19:58:41 deb-vm2 rsyslogd[5891]: imuxsock: Acquired UNIX socket '/run/systemd/journal/syslog' (fd 3) from systemd.  [v8.2302.0]
Aug 24 19:58:41 deb-vm2 rsyslogd[5891]: [origin software="rsyslogd" swVersion="8.2302.0" x-pid="5891" x-info="https://www.rsyslog.com"] start
udp   UNCONN 0      0            0.0.0.0:514        0.0.0.0:*    users:(("rsyslogd",pid=5891,fd=6))
udp   UNCONN 0      0               [::]:514           [::]:*    users:(("rsyslogd",pid=5891,fd=7))
tcp   LISTEN 0      25           0.0.0.0:514        0.0.0.0:*    users:(("rsyslogd",pid=5891,fd=8))
tcp   LISTEN 0      25              [::]:514           [::]:*    users:(("rsyslogd",pid=5891,fd=9))
# Сервер к приему логов готов

# 3. Подготовим клиента, на котором работает nginx
# Допишем в конфиг файл nginx строки
root@deb-vm:/home/vital# nano /etc/nginx/nginx.conf 
error_log /var/log/nginx/error.log;
error_log syslog:server=192.168.122.20:514,tag=nginx_error;
access_log syslog:server=192.168.122.20:514,tag=nginx_access,severity=info combined;

# Проверим все ли хорошо
root@deb-vm:/home/vital# nginx -t && systemctl restart nginx && systemctl status nginx && ss -tunlp | grep 80
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; preset: enabled)
     Active: active (running) since Sun 2025-08-24 20:09:03 +05; 26ms ago
       Docs: man:nginx(8)
    Process: 8246 ExecStartPre=/usr/sbin/nginx -t -q -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
    Process: 8260 ExecStart=/usr/sbin/nginx -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
   Main PID: 8261 (nginx)
      Tasks: 1 (limit: 2312)
     Memory: 1.0M
        CPU: 19ms
     CGroup: /system.slice/nginx.service
             ├─8261 "nginx: master process /usr/sbin/nginx -g daemon on; master_process on;"
             └─8264 "nginx: master process /usr/sbin/nginx -g daemon on; master_process on;"

Aug 24 20:09:03 deb-vm systemd[1]: Starting nginx.service - A high performance web server and a reverse proxy server...
Aug 24 20:09:03 deb-vm systemd[1]: Started nginx.service - A high performance web server and a reverse proxy server.
tcp   LISTEN 0      511          0.0.0.0:80        0.0.0.0:*    users:(("nginx",pid=8264,fd=4),("nginx",pid=8261,fd=4))
tcp   LISTEN 0      511             [::]:80           [::]:*    users:(("nginx",pid=8264,fd=5),("nginx",pid=8261,fd=5))

# 4. Настроим аудит, который проследит за изменением конфигов nginx
root@deb-vm:/home/vital# touch /etc/audit/rules.d/nginx.rules && echo "-w /etc/nginx -p wax -k monitor_nginx" > /etc/audit/rules.d/nginx.rules
root@deb-vm:/home/vital# systemctl restart auditd

# Проверим что же у нас вышло
root@deb-vm:/home/vital# auditctl -l
-w /etc/nginx -p wxa -k monitor_nginx

# Попробуем вот так
root@deb-vm:/home/vital# touch /etc/nginx/{123,124}
root@deb-vm:/home/vital# rm /etc/nginx/{123,124}
root@deb-vm:/home/vital# ausearch -f /etc/nginx

time->Sun Aug 24 20:16:23 2025
type=PROCTITLE msg=audit(1756048583.452:1854): proctitle=746F756368002F6574632F6E67696E782F313233002F6574632F6E67696E782F313234
type=PATH msg=audit(1756048583.452:1854): item=1 name="/etc/nginx/123" inode=38 dev=fd:01 mode=0100644 ouid=0 ogid=0 rdev=00:00 nametype=NORMAL cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=PATH msg=audit(1756048583.452:1854): item=0 name="/etc/nginx/" inode=68858 dev=fd:01 mode=040755 ouid=0 ogid=0 rdev=00:00 nametype=PARENT cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=CWD msg=audit(1756048583.452:1854): cwd="/home/vital"
type=SYSCALL msg=audit(1756048583.452:1854): arch=c000003e syscall=257 success=yes exit=3 a0=ffffff9c a1=7ffc92aa9ea7 a2=941 a3=1b6 items=2 ppid=7353 pid=8302 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts1 ses=171 comm="touch" exe="/usr/bin/touch" subj=unconfined key="monitor_nginx"
----
time->Sun Aug 24 20:16:23 2025
type=PROCTITLE msg=audit(1756048583.456:1855): proctitle=746F756368002F6574632F6E67696E782F313233002F6574632F6E67696E782F313234
type=PATH msg=audit(1756048583.456:1855): item=1 name="/etc/nginx/124" inode=64 dev=fd:01 mode=0100644 ouid=0 ogid=0 rdev=00:00 nametype=NORMAL cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=PATH msg=audit(1756048583.456:1855): item=0 name="/etc/nginx/" inode=68858 dev=fd:01 mode=040755 ouid=0 ogid=0 rdev=00:00 nametype=PARENT cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=CWD msg=audit(1756048583.456:1855): cwd="/home/vital"
type=SYSCALL msg=audit(1756048583.456:1855): arch=c000003e syscall=257 success=yes exit=0 a0=ffffff9c a1=7ffc92aa9eb6 a2=941 a3=1b6 items=2 ppid=7353 pid=8302 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts1 ses=171 comm="touch" exe="/usr/bin/touch" subj=unconfined key="monitor_nginx"
----
time->Sun Aug 24 20:16:33 2025
type=PROCTITLE msg=audit(1756048593.868:1856): proctitle=726D002F6574632F6E67696E782F313233002F6574632F6E67696E782F313234
type=PATH msg=audit(1756048593.868:1856): item=1 name="/etc/nginx/123" inode=38 dev=fd:01 mode=0100644 ouid=0 ogid=0 rdev=00:00 nametype=DELETE cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=PATH msg=audit(1756048593.868:1856): item=0 name="/etc/nginx/" inode=68858 dev=fd:01 mode=040755 ouid=0 ogid=0 rdev=00:00 nametype=PARENT cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=CWD msg=audit(1756048593.868:1856): cwd="/home/vital"
type=SYSCALL msg=audit(1756048593.868:1856): arch=c000003e syscall=263 success=yes exit=0 a0=ffffff9c a1=55bdfbce34a0 a2=0 a3=7f211cb03f80 items=2 ppid=7353 pid=8303 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts1 ses=171 comm="rm" exe="/usr/bin/rm" subj=unconfined key="monitor_nginx"
----
time->Sun Aug 24 20:16:33 2025
type=PROCTITLE msg=audit(1756048593.872:1857): proctitle=726D002F6574632F6E67696E782F313233002F6574632F6E67696E782F313234
type=PATH msg=audit(1756048593.872:1857): item=1 name="/etc/nginx/124" inode=64 dev=fd:01 mode=0100644 ouid=0 ogid=0 rdev=00:00 nametype=DELETE cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=PATH msg=audit(1756048593.872:1857): item=0 name="/etc/nginx/" inode=68858 dev=fd:01 mode=040755 ouid=0 ogid=0 rdev=00:00 nametype=PARENT cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=CWD msg=audit(1756048593.872:1857): cwd="/home/vital"
type=SYSCALL msg=audit(1756048593.872:1857): arch=c000003e syscall=263 success=yes exit=0 a0=ffffff9c a1=55bdfbce34a0 a2=0 a3=100 items=2 ppid=7353 pid=8303 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts1 ses=171 comm="rm" exe="/usr/bin/rm" subj=unconfined key="monitor_nginx"

# Все хорошо, все отслеживается

# Настроим отправку на сборщик логов
root@deb-vm:/home/vital# nano /etc/rsyslog.d/audit.conf

$ModLoad imfile
$InputFileName /var/log/audit/audit.log
$InputFileTag nginx_edit_conf:
$InputFileStateFile audit_nginx_log
$InputFileSeverity info
$InputFileFacility local0
$InputRunFileMonitor
$InputFilePollInterval 10
*.* @@192.168.122.20:514

# Сделаем рестарт службы
root@deb-vm:/home/vital# systemctl restart rsyslog && systemctl status rsyslog
● rsyslog.service - System Logging Service
     Loaded: loaded (/lib/systemd/system/rsyslog.service; enabled; preset: enabled)
     Active: active (running) since Sun 2025-08-24 20:24:29 +05; 22ms ago
TriggeredBy: ● syslog.socket
       Docs: man:rsyslogd(8)
             man:rsyslog.conf(5)
             https://www.rsyslog.com/doc/
   Main PID: 8342 (rsyslogd)
      Tasks: 5 (limit: 2312)
     Memory: 996.0K
        CPU: 9ms
     CGroup: /system.slice/rsyslog.service
             └─8342 /usr/sbin/rsyslogd -n -iNONE

Aug 24 20:24:29 deb-vm systemd[1]: Starting rsyslog.service - System Logging Service...
Aug 24 20:24:29 deb-vm systemd[1]: Started rsyslog.service - System Logging Service.
Aug 24 20:24:29 deb-vm rsyslogd[8342]: imuxsock: Acquired UNIX socket '/run/systemd/journal/syslog' (fd 3) from systemd.  [v8.2302.0]
Aug 24 20:24:29 deb-vm rsyslogd[8342]: [origin software="rsyslogd" swVersion="8.2302.0" x-pid="8342" x-info="https://www.rsyslog.com"] start

# 5. Идем проверять сервер сбора логов

# Сначала "постучимся" на nginx
root@deb-vm2:/home/vital# curl 192.168.122.220
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.22.1</center>
</body>
</html>

# Потом проверим наши логи
root@deb-vm2:/home/vital# ls -la /var/log/rsyslog/deb-vm/ | grep nginx
-rw-r----- 1 root adm   3153 Aug 24 20:25 nginx_access.log
-rw-r----- 1 root adm  69904 Aug 24 20:24 nginx_edit_conf.log

# Все интересующее нас к счастью имеется
root@deb-vm2:/home/vital# cat /var/log/rsyslog/deb-vm/nginx_access.log | grep 20:
2025-08-22T20:19:28+05:00 deb-vm nginx_access: 192.168.20.2 - - [22/Aug/2025:20:19:28 +0500] "GET / HTTP/1.1" 404 187 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36"
2025-08-22T20:19:36+05:00 deb-vm nginx_access: 192.168.20.2 - - [22/Aug/2025:20:19:36 +0500] "GET / HTTP/1.1" 404 187 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36"
2025-08-22T20:19:37+05:00 deb-vm nginx_access: 192.168.20.2 - - [22/Aug/2025:20:19:37 +0500] "GET / HTTP/1.1" 404 187 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/109.0.0.0 Safari/537.36"
2025-08-24T20:25:54+05:00 deb-vm nginx_access: 192.168.122.20 - - [24/Aug/2025:20:25:54 +0500] "GET / HTTP/1.1" 404 153 "-" "curl/7.88.1"

root@deb-vm2:/home/vital# cat /var/log/rsyslog/deb-vm/nginx_edit_conf.log | grep /etc/nginx | tail
2025-08-24T20:05:43+05:00 deb-vm nginx_edit_conf: type=PATH msg=audit(1756047937.672:1840): item=0 name="/etc/nginx/" inode=68858 dev=fd:01 mode=040755 ouid=0 ogid=0 rdev=00:00 nametype=PARENT cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0#035OUID="root" OGID="root"
2025-08-24T20:05:43+05:00 deb-vm nginx_edit_conf: type=PATH msg=audit(1756047937.672:1840): item=1 name="/etc/nginx/.nginx.conf.swp" inode=67 dev=fd:01 mode=0100644 ouid=0 ogid=0 rdev=00:00 nametype=DELETE cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0#035OUID="root" OGID="root"
2025-08-24T20:16:23+05:00 deb-vm nginx_edit_conf: type=PATH msg=audit(1756048583.452:1854): item=0 name="/etc/nginx/" inode=68858 dev=fd:01 mode=040755 ouid=0 ogid=0 rdev=00:00 nametype=PARENT cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0#035OUID="root" OGID="root"
2025-08-24T20:16:23+05:00 deb-vm nginx_edit_conf: type=PATH msg=audit(1756048583.452:1854): item=1 name="/etc/nginx/123" inode=38 dev=fd:01 mode=0100644 ouid=0 ogid=0 rdev=00:00 nametype=NORMAL cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0#035OUID="root" OGID="root"
2025-08-24T20:16:23+05:00 deb-vm nginx_edit_conf: type=PATH msg=audit(1756048583.456:1855): item=0 name="/etc/nginx/" inode=68858 dev=fd:01 mode=040755 ouid=0 ogid=0 rdev=00:00 nametype=PARENT cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0#035OUID="root" OGID="root"
2025-08-24T20:16:23+05:00 deb-vm nginx_edit_conf: type=PATH msg=audit(1756048583.456:1855): item=1 name="/etc/nginx/124" inode=64 dev=fd:01 mode=0100644 ouid=0 ogid=0 rdev=00:00 nametype=NORMAL cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0#035OUID="root" OGID="root"
2025-08-24T20:16:33+05:00 deb-vm nginx_edit_conf: type=PATH msg=audit(1756048593.868:1856): item=0 name="/etc/nginx/" inode=68858 dev=fd:01 mode=040755 ouid=0 ogid=0 rdev=00:00 nametype=PARENT cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0#035OUID="root" OGID="root"
2025-08-24T20:16:33+05:00 deb-vm nginx_edit_conf: type=PATH msg=audit(1756048593.868:1856): item=1 name="/etc/nginx/123" inode=38 dev=fd:01 mode=0100644 ouid=0 ogid=0 rdev=00:00 nametype=DELETE cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0#035OUID="root" OGID="root"
2025-08-24T20:16:33+05:00 deb-vm nginx_edit_conf: type=PATH msg=audit(1756048593.872:1857): item=0 name="/etc/nginx/" inode=68858 dev=fd:01 mode=040755 ouid=0 ogid=0 rdev=00:00 nametype=PARENT cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0#035OUID="root" OGID="root"
2025-08-24T20:16:33+05:00 deb-vm nginx_edit_conf: type=PATH msg=audit(1756048593.872:1857): item=1 name="/etc/nginx/124" inode=64 dev=fd:01 mode=0100644 ouid=0 ogid=0 rdev=00:00 nametype=DELETE cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0#035OUID="root" OGID="root"

# Все получилось, задача выполнена
