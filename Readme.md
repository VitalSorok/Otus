# NFS
# Практические навыки работы с NFS
# 1.Исходные данные, подготовка

# Мы имеем "машину" №1 (хостовую), будущий NFS сервер
vital@ubuntuhost:~$ cat /etc/os-release
PRETTY_NAME="Ubuntu 24.04.2 LTS"

# Интерфейс "virbr0" соединен с гостевой (ВМ) "машиной" 
vital@ubuntuhost:~$ route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
default         _gateway        0.0.0.0         UG    0      0        0 enp3s0f0
172.16.0.0      0.0.0.0         255.255.255.248 U     0      0        0 enp3s0f0
192.168.122.0   0.0.0.0         255.255.255.0   U     0      0        0 virbr0

# Адрес "машины" №1 - 192.168.122.1, будущий сервер NFS
vital@ubuntuhost:~$ ifconfig
virbr0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.122.1  netmask 255.255.255.0  broadcast 192.168.122.255
        ether 52:54:00:9b:c8:24  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

# Мы имеем "машину" №2 (гостевую или ВМ), будущий NFS клиент
vital@deb-vm:~$ cat /etc/os-release
PRETTY_NAME="Debian GNU/Linux 12 (bookworm)"
NAME="Debian GNU/Linux"

# Адрес "машины" №2 - 192.168.122.220, будущий клиент NFS
vital@deb-vm:~$ ifconfig
-bash: ifconfig: command not found
vital@deb-vm:~$ whereis ifconfig
ifconfig: /usr/sbin/ifconfig /usr/share/man/man8/ifconfig.8.gz
vital@deb-vm:~$ /usr/sbin/ifconfig
enp1s0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.122.220  netmask 255.255.255.0  broadcast 192.168.122.255
        inet6 fe80::5054:ff:fe24:2e8c  prefixlen 64  scopeid 0x20<link>
        ether 52:54:00:24:2e:8c  txqueuelen 1000  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

# 2. Действия на сервере. 
# Устанавливаем NFS
root@ubuntuhost:/home/vital# apt install nfs-kernel-server

# Редактируем конфиг сервера. Везде где версия 4 ставим решительное n, дабы сервер работал с Vers3, как от нас этого требует тех.задание
root@ubuntuhost:/home/vital# nano /etc/nfs.conf
[nfsd]
#debug=0
#threads=8
#host=
#port=0
#grace-time=90
#lease-time=90
#udp=n
#tcp=y
#vers3=y
vers4=n
vers4.0=n
vers4.1=n
vers4.2=n
#rdma=n
#rdma-port=20049
....

# Рестартуем nfs сервер. 
root@ubuntuhost:/home/vital# systemctl restart nfs-kernel-server

# Проверяем наличие слушающих портов
root@ubuntuhost:/home/vital# ss -tunlp | grep 2049
tcp   LISTEN 0      64            0.0.0.0:2049       0.0.0.0:*
tcp   LISTEN 0      64               [::]:2049          [::]:*

root@ubuntuhost:/home/vital# ss -tunlp | grep 111
udp   UNCONN 0      0             0.0.0.0:111        0.0.0.0:*    users:(("rpcbind",pid=692,fd=5),("systemd",pid=1,fd=40))
udp   UNCONN 0      0                [::]:111           [::]:*    users:(("rpcbind",pid=692,fd=7),("systemd",pid=1,fd=42))
tcp   LISTEN 0      4096          0.0.0.0:111        0.0.0.0:*    users:(("rpcbind",pid=692,fd=4),("systemd",pid=1,fd=39))
tcp   LISTEN 0      4096             [::]:111           [::]:*    users:(("rpcbind",pid=692,fd=6),("systemd",pid=1,fd=41))

# Подготавливаем директорию для экспорта
root@ubuntuhost:/home/vital# mkdir /home/vital/upload
root@ubuntuhost:/home/vital# chown -R nobody:nogroup /home/vital/upload
root@ubuntuhost:/home/vital# chmod 766 /home/vital/upload

# Добавляем директорию для экспорта
root@ubuntuhost:/home/vital# echo '/home/vital/upload  192.168.122.220(rw,sync,root_squash)' >> /etc/exports
root@ubuntuhost:/home/vital# exportfs -r

# проверяем 
root@ubuntuhost:/home/vital# exportfs -s
/home/vital/upload  192.168.122.220(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)

# 2. Действия на клиенте. 
# Устанавливаем NFS клиент
root@deb-vm:/home/vital# apt install nfs-common

# Монтируем саму NFS
root@deb-vm:/home/vital# mount 192.168.122.1:/home/vital/upload /mnt/01

# Проверяем содержимое, все сходится, оно самое...
root@deb-vm:/home/vital# ls -la /mnt/01
total 16
drwxrw-rw- 2 nobody nogroup 4096 May 28 08:41 .
drwxr-xr-x 9 root   root    4096 May 19 21:01 ..
-rw-r--r-- 1 nobody nogroup   25 May 27 22:40 file1
-rw-r--r-- 1 nobody nogroup    0 May 26 14:45 file2
-rw-r--r-- 1 nobody nogroup    7 May 28 08:41 file3

# Теперь необходимо настроить автомонтирование
#Согласно презентации к занятиям есть три пути

#Вариант1 /etc/fstab
#1.Добавим туда строку:192.168.122.1:/home/vital/upload /mnt/01 nfs3 defaults        0       0

#2.перезагрузим, проверим, но увы и ах где то ошибка, не взлетел, тяги не хватило, система не грузится.

#3.На запрос доступа по ssh - посылает в далекое пешее... На мое "ssh vital@192.168.122.220" отвечает немедля, а не пойти бы
#вам: "connection refused". На попытку поговорить по человечески "virsh console deb-vm" полный игнор - виснет наглухо.

#4.Значит надо как то отредактировать проклятый /etc/fstab, располагающийся на диске "deb-vm.gcow2" с которого стартует злополучная
#"deb-vm". Как это сделать?

#5.Останавливаем машину "virsh destroy deb-vm", потому как на "virsh shutdown deb-vm" мы не реагируем никак. Команда отрабатывет 
#без ошибок, а машина по прежнему в состоянии running.

#6.Установим 
root@deb-vm:/home/vital#apt install guestfs-tools

#Далее расковыряем злополучный deb-vm.qcow2
root@deb-vm:/home/vital# guestfish --rw -a /var/lib/libvirt/images/deb.qcow2
><fs>
><fs> launch
><fs> list-filesystems
... 
#Найдем корневой раздел(LVM), смонтируем его в /, октроем vi /etc/fstab,
#Закоментируем строку: 192.168.122.1:/home/vital/upload /mnt/01 nfs3 defaults        0       0
><fs> quit

#Ну с богом...
root@deb-vm:/home/vital# virsh start deb-vm
Domain 'deb-vm' started
root@deb-vm:/home/vital# ssh vital@192.168.122.220
#Все открывается и работает...

#Вариант2 (sudo systemctl restart remote-fs.target) пропустим 

#Вариант3 Отработаем.
root@deb-vm:/home/vital# nano /etc/systemd/system/mnt-01.mount

[Unit]
Description=Mount NFS 192.168.122.1

[Mount]
What=192.168.122.1:/home/vital/upload
Where=/mnt/01
Type=nfs
Options=defaults

[Install]
WantedBy=multi-user.target

#Далее
root@deb-vm:/home/vital# nano /etc/systemd/system/mnt-01.automount

[Unit]
Description=Automount NFS 192.168.122.1

[Automount]
Where=/mnt/01
TimeoutIdleSec=30

[Install]
WantedBy=multi-user.target

#Перезагружаем файлы служб, обновляем конфигурацию
root@deb-vm:/home/vital# systemctl daemon-reload

#Добавляем в автозагрузку
root@deb-vm:/home/vital# systemctl enable mnt-01.automount 

#Перезагружаемся и радуемся жизни... Все на месте... правда не сразу, а спустя 30 сек
root@deb-vm:/home/vital# ls -la /mnt/01
total 16
drwxrw-rw- 2 nobody nogroup 4096 May 28 20:31 .
drwxr-xr-x 9 root   root    4096 May 19 21:01 ..
-rw-r--r-- 1 nobody nogroup   25 May 27 22:40 file1
-rw-r--r-- 1 nobody nogroup    0 May 26 14:45 file2
-rw-r--r-- 1 nobody nogroup    7 May 28 08:41 file3

#Проверка работоспособности
#Заходим на сервер. 
#Заходим в каталог /home/vital/upload.
#Создаём тестовый файл touch check_file.
root@ubuntuhost:/home/vital/upload# touch check_file

#Заходим на клиент.
#Заходим в каталог /mnt/01.
root@deb-vm:/home/vital# cd /mnt/01
 
#Проверяем наличие ранее созданного файла.
root@deb-vm:/mnt/01# ls -la
total 16
drwxrw-rw- 2 nobody nogroup 4096 May 28 20:34 .
drwxr-xr-x 9 root   root    4096 May 19 21:01 ..
-rw-r--r-- 1 root   root       0 May 28 20:34 check_file
-rw-r--r-- 1 nobody nogroup   25 May 27 22:40 file1
-rw-r--r-- 1 nobody nogroup    0 May 26 14:45 file2
-rw-r--r-- 1 nobody nogroup    7 May 28 08:41 file3

#Создаём тестовый файл touch client_file.
root@deb-vm:/mnt/01# touch client_file
 
#Проверяем, что файл успешно создан.
root@deb-vm:/mnt/01# ls -la
total 16
drwxrw-rw- 2 nobody nogroup 4096 May 28 20:37 .
drwxr-xr-x 9 root   root    4096 May 19 21:01 ..
-rw-r--r-- 1 root   root       0 May 28 20:34 check_file
-rw-r--r-- 1 nobody nogroup    0 May 28 20:37 client_file
-rw-r--r-- 1 nobody nogroup   25 May 27 22:40 file1
-rw-r--r-- 1 nobody nogroup    0 May 26 14:45 file2
-rw-r--r-- 1 nobody nogroup    7 May 28 08:41 file3

#Перезагружаем клиент
root@deb-vm:/mnt/01# reboot

Broadcast message from root@deb-vm on pts/1 (Wed 2025-05-28 20:39:46 +05):

The system will reboot now!

root@deb-vm:/mnt/01# Connection to 192.168.122.220 closed by remote host.
Connection to 192.168.122.220 closed.

#Предварительно проверяем клиент
#Заходим на клиент
vital@ubuntuhost:~$ ssh vital@192.168.122.220
Linux deb-vm 6.1.0-35-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.137-1 (2025-05-07) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
You have new mail.
Last login: Wed May 28 19:44:42 2025 from 192.168.122.1
vital@deb-vm:~$

#Заходим в каталог /mnt/01
root@deb-vm:/home/vital# cd /mnt/01

#Проверяем наличие ранее созданных файлов
root@deb-vm:/mnt/01# ls -la
total 16
drwxrw-rw- 2 nobody nogroup 4096 May 28 20:37 .
drwxr-xr-x 9 root   root    4096 May 19 21:01 ..
-rw-r--r-- 1 root   root       0 May 28 20:34 check_file
-rw-r--r-- 1 nobody nogroup    0 May 28 20:37 client_file
-rw-r--r-- 1 nobody nogroup   25 May 27 22:40 file1
-rw-r--r-- 1 nobody nogroup    0 May 26 14:45 file2
-rw-r--r-- 1 nobody nogroup    7 May 28 08:41 file3
#Все на месте

#Проверяем сервер
#Заходим на сервер в отдельном окне терминала
#Перезагружаем сервер
#Заходим на сервер
#Проверяем наличие файлов в каталоге /home/vital/upload/
root@ubuntuhost:/home/vital# ls -la /home/vital/upload
total 16
drwxrw-rw- 2 nobody nogroup 4096 мая 28 15:37 .
drwxr-x--- 6 vital  vital   4096 мая 27 16:43 ..
-rw-r--r-- 1 root   root       0 мая 28 15:34 check_file
-rw-r--r-- 1 nobody nogroup    0 мая 28 15:37 client_file
-rw-r--r-- 1 nobody nogroup   25 мая 27 17:40 file1
-rw-r--r-- 1 nobody nogroup    0 мая 26 09:45 file2
-rw-r--r-- 1 nobody nogroup    7 мая 28 03:41 file3

#Проверяем экспорты exportfs -s
root@ubuntuhost:/home/vital# exportfs -s
/home/vital/upload  192.168.122.220(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)

#Проверяем работу RPC showmount -a 192.168.122.1.
root@ubuntuhost:/home/vital# showmount -a 192.168.122.1
All mount points on 192.168.122.1:
192.168.122.220:/home/vital/upload

#Проверяем клиент: 
#Возвращаемся на клиент
#Перезагружаем клиент
#Заходим на клиент
#Проверяем работу RPC showmount -a 192.168.122.1
root@deb-vm:/home/vital# showmount -a 192.168.122.1
All mount points on 192.168.122.1:
192.168.122.220:/home/vital/upload

#Заходим в каталог /mnt/01
root@deb-vm:/home/vital# cd /mnt/01
root@deb-vm:/mnt/01#

#Проверяем статус монтирования mount | grep mnt
root@deb-vm:/mnt/01# mount | grep mnt
systemd-1 on /mnt/01 type autofs (rw,relatime,fd=55,pgrp=1,timeout=30,minproto=5,maxproto=5,direct,pipe_ino=14248)
192.168.122.1:/home/vital/upload on /mnt/01 type nfs (rw,relatime,vers=3,rsize=1048576,wsize=1048576,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,mountaddr=192.168.122.1,mountvers=3,mountport=43365,mountproto=udp,local_lock=none,addr=192.168.122.1)

#Проверяем наличие ранее созданных файлов
root@deb-vm:/mnt/01# ls -la
total 16
drwxrw-rw- 2 nobody nogroup 4096 May 28 20:37 .
drwxr-xr-x 9 root   root    4096 May 19 21:01 ..
-rw-r--r-- 1 root   root       0 May 28 20:34 check_file
-rw-r--r-- 1 nobody nogroup    0 May 28 20:37 client_file
-rw-r--r-- 1 nobody nogroup   25 May 27 22:40 file1
-rw-r--r-- 1 nobody nogroup    0 May 26 14:45 file2
-rw-r--r-- 1 nobody nogroup    7 May 28 08:41 file3

#Создаём тестовый файл touch final_check
root@deb-vm:/mnt/01# touch final_check

#Проверяем, что файл успешно создан.
root@deb-vm:/mnt/01# ls -la
total 16
drwxrw-rw- 2 nobody nogroup 4096 May 28 20:58 .
drwxr-xr-x 9 root   root    4096 May 19 21:01 ..
-rw-r--r-- 1 root   root       0 May 28 20:34 check_file
-rw-r--r-- 1 nobody nogroup    0 May 28 20:37 client_file
-rw-r--r-- 1 nobody nogroup   25 May 27 22:40 file1
-rw-r--r-- 1 nobody nogroup    0 May 26 14:45 file2
-rw-r--r-- 1 nobody nogroup    7 May 28 08:41 file3
-rw-r--r-- 1 nobody nogroup    0 May 28 20:58 final_check
