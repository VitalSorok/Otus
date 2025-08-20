# Задание: "Настроить бэкапы"

# 1.Имеем две ВМ в одном домене
# Первая ВМ(клиент):
root@deb-vm2:/home/vital# ip a
...
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:32:a5:84 brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.20/24 brd 192.168.122.255 scope global dynamic noprefixroute enp1s0
       valid_lft 3516sec preferred_lft 3516sec
    inet6 fe80::5054:ff:fe32:a584/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
...
root@deb-vm2:/home/vital# iptables -L -n -v
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

# Вторая ВМ(сервер):
root@deb-vm:/home/vital# ip a
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:24:2e:8c brd ff:ff:ff:ff:ff:ff
    inet 192.168.122.220/24 brd 192.168.122.255 scope global enp1s0
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:fe24:2e8c/64 scope link
       valid_lft forever preferred_lft forever
...
root@deb-vm2:/home/vital# iptables -L -n -v
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source 


# 2. Устанавливаем на client и backup сервере borgbackup
root@deb-vm:/home/vital#apt install -y borgbackup
root@deb-vm2:/home/vital#apt install -y borgbackup

# 3. На клиенте генерируем ключ (нам понадобиться файл id_rsa.pub): 
root@deb-vm2:/home/vital# ssh-keygen

# 4. На сервере root@deb-vm создаем пользователя 
root@deb-vm:/home/vital# useradd -m -s /bin/bash borg
# Каталог /var/backup			
root@deb-vm:/home/vital# mkdir /var/backup
# Редактируем права на каталог
root@deb-vm:/home/vital# chown borg:borg /var/backup/

# 5. Создаем и кидаем заранее сгенерированный ключ в соответствующую папку
root@deb-vm:/home/vital# su borg
borg@deb-vm:/home/vital$ cd ~
borg@deb-vm:~$ pwd
/home/borg
borg@deb-vm:~$ mkdir ~/.ssh && touch ~/.ssh/authorized_keys
borg@deb-vm:~$ chmod 700 .ssh
borg@deb-vm:~$ chmod 600 .ssh/authorized_keys
# копируем содержимое файла клиента id_rsa.pub в файл сервера authorized_keys 
borg@deb-vm:~$ nano .ssh/authorized_keys

# 6. Посмотрим имеющиеся в системе диски

root@deb-vm:/home/vital# lsblk
NAME                   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sr0                     11:0    1 1024M  0 rom
vda                    254:0    0   10G  0 disk
+-vda1                 254:1    0  487M  0 part /boot
+-vda2                 254:2    0    1K  0 part
L-vda5                 254:5    0  9.5G  0 part
  +-deb--vm--vg-root   253:0    0  6.3G  0 lvm  /
  +-deb--vm--vg-var    253:1    0    1G  0 lvm  /var
  +-deb--vm--vg-swap_1 253:2    0  976M  0 lvm  [SWAP]
  +-deb--vm--vg-tmp    253:3    0  260M  0 lvm  /tmp
  L-deb--vm--vg-home   253:4    0    1G  0 lvm  /home
vdb                    254:16   0    1G  0 disk 
vdc                    254:32   0    1G  0 disk
vdd                    254:48   0    1G  0 disk

# Возьмем диск vdb и подготовим к монтированию, создадим разделы (постоянно "Enter,в конце w - write, Затем выходим)
root@deb-vm:/home/vital# fdisk -n /dev/vdb
# Отформатируем
root@deb-vm:/home/vital# mkfs.ext4 /dev/vdb
# Cмонтируем
root@deb-vm:/home/vital# mount /dev/vdb /var/backup
# Автомонтирование при загрузке
root@deb-vm:/home/vital# echo "/dev/vdb /var/backup ext4    defaults        0       0" >> /etc/fstab


# 6. Проверяем нашу ssh авторизацию по ключу c клиентской ВМ
root@deb-vm2:/home/vital# ssh borg@192.168.122.220
Linux deb-vm 6.1.0-37-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.140-1 (2025-05-22) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Mon Aug 18 16:07:02 2025 from 192.168.122.20
borg@deb-vm:~$
#Как оказалось удачно

# 7. Инициализируем репозиторий borg c клиентской на сервеную ВМ:
borg@deb-vm:~$ borg init --encryption=repokey borg@192.168.122.220:/var/backup/

# 8.Запускаем для проверки создания бэкапа
root@deb-vm2:/home/vital# borg create --stats --list borg@192.168.122.220:/var/backup/::"etc-{now:%Y-%m-%d_%H:%M:%S}" /etc
root@deb-vm2:/home/vital# Enter passphrase for key ssh://borg@192.168.122.220/var/backup:

# 9.Проверим что же вышло
root@deb-vm2:/home/vital# borg list borg@192.168.122.220:/var/backup
Enter passphrase for key ssh://borg@192.168.122.220/var/backup:
etc-2025-08-18_16:17:43              Mon, 2025-08-18 16:17:49 [1b63c2e5bbc1aeec6795c379bac435cd234d6199b4f6395494c7fea23efb06aa]
# Все ок

# 10.Автоматизируем создание бэкапов с помощью systemd
# Создаем сервис и таймер в каталоге /etc/systemd/system/

root@deb-vm2:/home/vital# nano /etc/systemd/system/borg-backup.service
[Unit]
Description=Borg Backup

[Service]
Type=oneshot

# Парольная фраза
Environment="BORG_PASSPHRASE=123456"
# Репозиторий
Environment=repo=borg@192.168.122.220:/var/backup/
# Что бэкапим
Environment=target=/etc

# Создание бэкапа
ExecStart=/bin/borg create \
    --stats                \
    ${repo}::etc-{now:%%Y-%%m-%%d_%%H:%%M:%%S} ${target}

# Проверка бэкапа
ExecStart=/bin/borg check ${repo}

# Очистка старых бэкапов
ExecStart=/bin/borg prune \
    --keep-daily  90      \
    --keep-monthly 12     \
    --keep-yearly  1      \
    ${repo}


root@deb-vm2:/home/vital# nano /etc/systemd/system/borg-backup.timer

[Unit]
Description=Borg Backup

[Timer]
OnBootSec=5min
OnUnitActiveSec=5min

[Install]
WantedBy=timers.target

# 11. Включаем и запускаем службу таймера
root@deb-vm2:/home/vital# systemctl enable borg-backup.timer 
root@deb-vm2:/home/vital# systemctl start borg-backup.timer

# 12. Проверяем работу таймера
root@deb-vm2:/home/vital# systemctl list-timers --all
NEXT                        LEFT          LAST                        PASSED       UNIT                         ACTIVATES
Tue 2025-08-19 17:35:36 +05 3min 16s left Tue 2025-08-19 17:30:36 +05 1min 43s ago borg-backup.timer            borg-backup.service

# 13. Проверим настройку процессов логирования
root@deb-vm2:/home/vital# journalctl -u borg-backup.service | tail
Aug 19 17:30:40 deb-vm2 borg[4520]: ------------------------------------------------------------------------------
Aug 19 17:30:40 deb-vm2 borg[4520]:                        Original size      Compressed size    Deduplicated size
Aug 19 17:30:40 deb-vm2 borg[4520]: This archive:                6.80 MB              1.78 MB                652 B
Aug 19 17:30:40 deb-vm2 borg[4520]: All archives:               27.21 MB              7.10 MB              2.10 MB
Aug 19 17:30:40 deb-vm2 borg[4520]:                        Unique chunks         Total chunks
Aug 19 17:30:40 deb-vm2 borg[4520]: Chunk index:                     973                 3907
Aug 19 17:30:40 deb-vm2 borg[4520]: ------------------------------------------------------------------------------
Aug 19 17:30:47 deb-vm2 systemd[1]: borg-backup.service: Deactivated successfully.
Aug 19 17:30:47 deb-vm2 systemd[1]: Finished borg-backup.service - Borg Backup.
Aug 19 17:30:47 deb-vm2 systemd[1]: borg-backup.service: Consumed 3.986s CPU time.
# Как видим логирование ведется

# 14. Настроим ротацию логов по времени
root@deb-vm2:/home/vital# journalctl --vacuum-time=1years
# Задание выполнено
