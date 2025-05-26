# ZFS
# Практичекие навыки работы с ZFS
# Исходные данные
root@deb-vm:/home/vital# uname -a
Linux deb-vm 6.1.0-35-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.137-1 (2025-05-07) x86_64 GNU/Linux
root@deb-vm:/home/vital# cat /proc/version
Linux version 6.1.0-35-amd64 (debian-kernel@lists.debian.org) (gcc-12 (Debian 12.2.0-14+deb12u1) 12.2.0, GNU ld (GNU Binutils for Debian) 2.40) #1 SMP PREEMPT_DYNAMIC Debian 6.1.137-1 (2025-05-07)
root@deb-vm:/home/vital# lsblk
NAME                   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sr0                     11:0    1 1024M  0 rom
vda                    254:0    0   10G  0 disk
+-vda1                 254:1    0  487M  0 part /boot
+-vda2                 254:2    0    1K  0 part
L-vda5                 254:5    0  9.5G  0 part
  +-deb--vm--vg-root   253:0    0  4.3G  0 lvm  /
  +-deb--vm--vg-var    253:2    0    1G  0 lvm  /var
  +-deb--vm--vg-swap_1 253:3    0  976M  0 lvm  [SWAP]
  +-deb--vm--vg-tmp    253:5    0  260M  0 lvm  /tmp
  L-deb--vm--vg-home   253:6    0    3G  0 lvm  /home
vdb                    254:16   0    1G  0 disk
+-vg_test-logdisk1     253:1    0  300M  0 lvm
L-vg_test-logdisk2     253:4    0  300M  0 lvm
vdc                    254:32   0    1G  0 disk
vdd                    254:48   0    1G  0 disk

# 1

# Так как zfs На сервере нет, установим zfs, но перед этим подготовим систему
root@deb-vm:/home/vital# apt update && apt upgrade -y

# Поскольку ядро свежее и kmod имеется оганичимся
root@deb-vm:/home/vital# apt install linux-headers-$(uname -r) -y

# Поищем...И к сожалению не найдем...
root@deb-vm:/home/vital# apt search zfs
Sorting... Done
Full Text Search... Done
root@deb-vm:/home/vital#

# Подключим репозитории contrib, дописав в конец каждой строки файла /etc/apt/sources.list слово "contrib"
# Обновим репозитории снова
root@deb-vm:/home/vital# apt update -y

# Попытаем счастья вновь и найдем его...
root@deb-vm:/home/vital# apt search zfs
....
zfsutils-linux/stable,now 2.1.11-1+deb12u1 amd64
  command-line tools to manage OpenZFS filesystems
....

# Далее по накатанной
root@deb-vm:/home/vital# apt install zfsutils-linux -y
# Глянем есть ли среди загруженных модулей zfs
root@deb-vm:/home/vital# kmod list | grep zfs
zfs                  4018176  11
zunicode              335872  1 zfs
zzstd                 589824  1 zfs
zlua                  192512  1 zfs
zavl                   20480  1 zfs
icp                   327680  1 zfs
zcommon               110592  2 zfs,icp
znvpair               118784  2 zfs,zcommon
spl                   122880  6 zfs,icp,zzstd,znvpair,zcommon,zavl
# Получается что есть 

# 2
# Приступим к созданию zfs.Создадим пул и файловые системы
root@deb-vm:/home/vital# zpool create pool1 mirror /dev/vdc /dev/vdd
root@deb-vm:/home/vital# zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
pool1   960M   250K   960M        -         -     0%     0%  1.00x    ONLINE  -
root@deb-vm:/home/vital# zfs create pool1/zfs01; zfs create pool1/zfs02; zfs create pool1/zfs03; zfs create pool1/zfs04

# Посмотрим что вышло
root@deb-vm:/home/vital# df -h
Filesystem                    Size  Used Avail Use% Mounted on
udev                          964M     0  964M   0% /dev
tmpfs                         197M  664K  197M   1% /run
/dev/mapper/deb--vm--vg-root  4.2G  2.2G  1.8G  56% /
tmpfs                         984M     0  984M   0% /dev/shm
tmpfs                         5.0M     0  5.0M   0% /run/lock
/dev/vda1                     455M  132M  298M  31% /boot
/dev/mapper/deb--vm--vg-home  2.9G  396K  2.8G   1% /home
/dev/mapper/deb--vm--vg-tmp   234M   10K  217M   1% /tmp
/dev/mapper/deb--vm--vg-var  1013M  306M  638M  33% /var
pool1                         832M  128K  832M   1% /pool1
pool1/zfs01                   832M  128K  832M   1% /pool1/zfs01
pool1/zfs02                   832M  128K  832M   1% /pool1/zfs02
pool1/zfs03                   832M  128K  832M   1% /pool1/zfs03
pool1/zfs04                   832M  128K  832M   1% /pool1/zfs04
tmpfs                         197M     0  197M   0% /run/user/1000

# Установим разное сжатие для каждой файловой системы
root@deb-vm:/home/vital# zfs set compression=lzjb pool1/zfs01
root@deb-vm:/home/vital# zfs set compression=lz4 pool1/zfs02
root@deb-vm:/home/vital# zfs set compression=gzip9 pool1/zfs03
root@deb-vm:/home/vital# zfs set compression=zle pool1/zfs04

# Проверим что вышло...
root@deb-vm:/home/vital# zfs get compression
NAME         PROPERTY     VALUE           SOURCE
pool1        compression  off             default
pool1/zfs01  compression  lzjb            local
pool1/zfs02  compression  lz4             local
pool1/zfs03  compression  gzip-9          local
pool1/zfs04  compression  zle             local

# Скопируем одни и те же данные в разные fs и посмотрим что же получилось
root@deb-vm:/home/vital# cp -r /var/log /pool1/zfs01
root@deb-vm:/home/vital# cp -r /var/log /pool1/zfs02
root@deb-vm:/home/vital# cp -r /var/log /pool1/zfs03
root@deb-vm:/home/vital# cp -r /var/log /pool1/zfs04
root@deb-vm:/home/vital# df -h | grep pool1
pool1                         768M  128K  768M   1% /pool1
pool1/zfs01                   784M   16M  768M   3% /pool1/zfs01
pool1/zfs02                   781M   14M  768M   2% /pool1/zfs02
pool1/zfs03                   777M  9.0M  768M   2% /pool1/zfs03
pool1/zfs04                   794M   27M  768M   4% /pool1/zfs04

# И немудренно ведь ...
root@deb-vm:/home/vital# zfs get compressratio
NAME         PROPERTY       VALUE  SOURCE
pool1        compressratio  2.18x  -
pool1/zfs01  compressratio  2.20x  -
pool1/zfs02  compressratio  2.66x  -
pool1/zfs03  compressratio  3.94x  -
pool1/zfs04  compressratio  1.33x  -

# Таким образом gzip-9 лидирует по сжатию


# 3
# Определение настроек пула
# Скачиваем архив в домашний каталог: 
root@deb-vm:/home/vital# wget -O archive.tar.gz --no-check-certificate 'https://drive.usercontent.google.com/download?id=1MvrcEp-WgAQe57aDEzxSRalPAwbNN1Bb&export=download'
--2025-05-21 22:46:32--  https://drive.usercontent.google.com/download?id=1MvrcEp-WgAQe57aDEzxSRalPAwbNN1Bb&export=download
Resolving drive.usercontent.google.com (drive.usercontent.google.com)... 173.194.220.132, 2a00:1450:4010:c09::84
Connecting to drive.usercontent.google.com (drive.usercontent.google.com)|173.194.220.132|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 7275140 (6.9M) [application/octet-stream]
Saving to: ‘archive.tar.gz’

archive.tar.gz                                              100%[========================================================================================================================================>]   6.94M  3.30MB/s    in 2.1s

2025-05-21 22:46:44 (3.30 MB/s) - ‘archive.tar.gz’ saved [7275140/7275140]

# Разархивируем
root@deb-vm:/home/vital# tar -xzvf archive.tar.gz
zpoolexport/
zpoolexport/filea
zpoolexport/fileb

# Проверяем возможность импорта
root@deb-vm:/home/vital# zpool import -d zpoolexport/
   pool: otus
     id: 6554193320433390805
  state: ONLINE
status: Some supported features are not enabled on the pool.
        (Note that they may be intentionally disabled if the
        'compatibility' property is set.)
 action: The pool can be imported using its name or numeric identifier, though
        some features will not be available without an explicit 'zpool upgrade'.
 config:

        otus                               ONLINE
          mirror-0                         ONLINE
            /home/vital/zpoolexport/filea  ONLINE
            /home/vital/zpoolexport/fileb  ONLINE

# Пишут что можно импортировать, ну можно так можно... Импортируем...раз можно...
root@deb-vm:/home/vital# zpool import -d zpoolexport/ otus

# Посмотрим на его статус
root@deb-vm:/home/vital# zpool status otus
  pool: otus
 state: ONLINE
status: Some supported and requested features are not enabled on the pool.
        The pool can still be used, but some features are unavailable.
action: Enable all features using 'zpool upgrade'. Once this is done,
        the pool may no longer be accessible by software that does not support
        the features. See zpool-features(7) for details.
config:

        NAME                               STATE     READ WRITE CKSUM
        otus                               ONLINE       0     0     0
          mirror-0                         ONLINE       0     0     0
            /home/vital/zpoolexport/filea  ONLINE       0     0     0
            /home/vital/zpoolexport/fileb  ONLINE       0     0     0

# Посмотрим все его настройки командой
root@deb-vm:/home/vital# zpool get all otus
NAME  PROPERTY                       VALUE                          SOURCE
otus  size                           480M                           -
otus  capacity                       0%                             -
otus  altroot                        -                              default
otus  health                         ONLINE                         -
otus  guid                           6554193320433390805            -
otus  version                        -                              default
otus  bootfs                         -                              default
otus  delegation                     on                             default
otus  autoreplace                    off                            default
otus  cachefile                      -                              default
otus  failmode                       wait                           default
otus  listsnapshots                  off                            default
otus  autoexpand                     off                            default
otus  dedupratio                     1.00x                          -
otus  free                           478M                           -
otus  allocated                      2.11M                          -
otus  readonly                       off                            -
otus  ashift                         0                              default
otus  comment                        -                              default
otus  expandsize                     -                              -
otus  freeing                        0                              -
otus  fragmentation                  0%                             -
otus  leaked                         0                              -
otus  multihost                      off                            default
otus  checkpoint                     -                              -
otus  load_guid                      4051498291176065774            -
otus  autotrim                       off                            default
otus  compatibility                  off                            default
otus  feature@async_destroy          enabled                        local
otus  feature@empty_bpobj            active                         local
otus  feature@lz4_compress           active                         local
otus  feature@multi_vdev_crash_dump  enabled                        local
otus  feature@spacemap_histogram     active                         local
otus  feature@enabled_txg            active                         local
otus  feature@hole_birth             active                         local
otus  feature@extensible_dataset     active                         local
otus  feature@embedded_data          active                         local
otus  feature@bookmarks              enabled                        local
otus  feature@filesystem_limits      enabled                        local
otus  feature@large_blocks           enabled                        local
otus  feature@large_dnode            enabled                        local
otus  feature@sha512                 enabled                        local
otus  feature@skein                  enabled                        local
otus  feature@edonr                  enabled                        local
otus  feature@userobj_accounting     active                         local
otus  feature@encryption             enabled                        local
otus  feature@project_quota          active                         local
otus  feature@device_removal         enabled                        local
otus  feature@obsolete_counts        enabled                        local
otus  feature@zpool_checkpoint       enabled                        local
otus  feature@spacemap_v2            active                         local
otus  feature@allocation_classes     enabled                        local
otus  feature@resilver_defer         enabled                        local
otus  feature@bookmark_v2            enabled                        local
otus  feature@redaction_bookmarks    disabled                       local
otus  feature@redacted_datasets      disabled                       local
otus  feature@bookmark_written       disabled                       local
otus  feature@log_spacemap           disabled                       local
otus  feature@livelist               disabled                       local
otus  feature@device_rebuild         disabled                       local
otus  feature@zstd_compress          disabled                       local
otus  feature@draid                  disabled                       local

# Запросим также все параметры файловой системы и выберем из них нужное нам
root@deb-vm:/home/vital# zfs get all otus | grep -P 'available|readonly|recordsize|compression|checksum'
otus  available             350M                   -
otus  recordsize            128K                   local
otus  checksum              sha256                 local
otus  compression           zle                    local
otus  readonly              off                    default


# Работа со снапшотом, поиск сообщения от преаподавателя
# скачиваем файл со снапшотом
root@deb-vm:/home/vital# wget -O otus_task2.file --no-check-certificate https://drive.usercontent.google.com/download?id=1wgxjih8YZ-cqLqaZVa0lA3h3Y029c3oI&export=download

# Проверяем наличие
root@deb-vm:/home/vital# ls -la | grep otus_task2.file
-rw-r--r-- 1 root  root  5432736 Dec  6  2023 otus_task2.file
#Все на месте

# Ищем файл с именем "secret_message"
root@deb-vm:/home/vital# find /otus/test -name "secret_message"
/otus/test/task1/file_mess/secret_message
#файл нашелся

# Посмотрим что же там
root@deb-vm:/home/vital# more /otus/test/task1/file_mess/secret_message
https://otus.ru/lessons/linux-hl/
#какая то ссылка. Ссылка на курс "Инфрасруктура высоконагрженных систем"
