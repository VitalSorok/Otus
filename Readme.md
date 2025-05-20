#Исходные данные
vital@deb-vm:~$ lsblk
NAME                   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sr0                     11:0    1 1024M  0 rom
vda                    254:0    0   10G  0 disk
+-vda1                 254:1    0  487M  0 part /boot
+-vda2                 254:2    0    1K  0 part
L-vda5                 254:5    0  9.5G  0 part
  +-deb--vm--vg-root   253:0    0  2.2G  0 lvm  /
  +-deb--vm--vg-var    253:1    0    1G  0 lvm  /var
  +-deb--vm--vg-swap_1 253:2    0  976M  0 lvm  [SWAP]
  +-deb--vm--vg-tmp    253:3    0  260M  0 lvm  /tmp
  L-deb--vm--vg-home   253:4    0  5.1G  0 lvm  /home
vdb                    254:16   0    1G  0 disk
vdc                    254:32   0    1G  0 disk
vdd                    254:48   0    1G  0 disk


#Будем работать с vdb, vdc
#Создадим PV, затем VG, а уж после LV
#PV
vital@deb-vm:~$ sudo su
root@deb-vm:/home/vital# pvcreate /dev/vdd
WARNING: ext4 signature detected on /dev/vdd at offset 1080. Wipe it? [y/n]: y
Wiping ext4 signature on /dev/vdd.
Physical volume "/dev/vdd" successfully created.
root@deb-vm:/home/vital# pvcreate /dev/vdc
Physical volume "/dev/vdc" successfully created.
#VG
root@deb-vm:/home/vital# vgcreate vg_test /dev/vdc /dev/vdd
Volume group "vg_test" successfully created
#LV
root@deb-vm:/home/vital# lvcreate -L 600 -n logdisk1 vg_test
Logical volume "logdisk1" created.
root@deb-vm:/home/vital# lvcreate -L 600 -n logdisk2 vg_test
Logical volume "logdisk2" created.


#Поглядим что же мы натворили
root@deb-vm:/home/vital# pvs | grep vg_test
  /dev/vdc   vg_test   lvm2 a--  1020.00m 420.00m
  /dev/vdd   vg_test   lvm2 a--  1020.00m 420.00m
root@deb-vm:/home/vital# vgs | grep vg_test
  vg_test     2   2   0 wz--n-  1.99g 840.00m
root@deb-vm:/home/vital# lvs | grep vg_test
  logdisk1 vg_test   -wi-a----- 600.00m
  logdisk2 vg_test   -wi-a----- 600.00m

#Отформатируем и смонтируем файловую систему
root@deb-vm:/home/vital# mkfs.ext4 /dev/mapper/vg_test-logdisk1
mke2fs 1.47.0 (5-Feb-2023)
Discarding device blocks: done
Creating filesystem with 153600 4k blocks and 38400 inodes
Filesystem UUID: 17951d7f-b23e-4db8-8e64-10f0897f1072
Superblock backups stored on blocks:
        32768, 98304

Allocating group tables: done
Writing inode tables: done
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

root@deb-vm:/home/vital# mkfs.ext4 /dev/mapper/vg_test-logdisk2
mke2fs 1.47.0 (5-Feb-2023)
Discarding device blocks: done
Creating filesystem with 153600 4k blocks and 38400 inodes
Filesystem UUID: 581afcc4-c204-41fc-90fc-b2884daa2569
Superblock backups stored on blocks:
        32768, 98304

Allocating group tables: done
Writing inode tables: done
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

#Смонтируем логические тома в папки /mnt/[01,02]
root@deb-vm:/home/vital# mount /dev/mapper/vg_test-logdisk1 /mnt/01
root@deb-vm:/home/vital# mount /dev/mapper/vg_test-logdisk2 /mnt/02

#Посмотрим на все это
root@deb-vm:/home/vital# lsblk | grep vg_test
L-vg_test-logdisk1     253:5    0  600M  0 lvm  /mnt/01
L-vg_test-logdisk2     253:6    0  600M  0 lvm  /mnt/02
root@deb-vm:/home/vital# df -Th | grep vg_test
/dev/mapper/vg_test-logdisk1 ext4      574M   24K  532M   1% /mnt/01
/dev/mapper/vg_test-logdisk2 ext4      574M   24K  532M   1% /mnt/02

#Закинем для теста какие нибудь данные в /mnt/01
cp -r /var/log /mnt/01

#Теперь встала задача по расширению файловой системы за счет нового диска 
#создадим PV, добавим его VG
root@deb-vm:/home/vital# pvcreate /dev/vdb
  Physical volume "/dev/vdb" successfully created.
root@deb-vm:/home/vital# vgextend vg_test /dev/vdb
  Volume group "vg_test" successfully extended

#Посмотрим на это все...
root@deb-vm:/home/vital# pvs | grep vg_test
  /dev/vdb   vg_test   lvm2 a--  1020.00m 1020.00m
  /dev/vdc   vg_test   lvm2 a--  1020.00m  420.00m
  /dev/vdd   vg_test   lvm2 a--  1020.00m  420.00m
root@deb-vm:/home/vital# vgs | grep vg_test
  vg_test     3   2   0 wz--n- <2.99g <1.82g
root@deb-vm:/home/vital# lvs | grep vg_test
  logdisk1 vg_test   -wi-a----- 600.00m
  logdisk2 vg_test   -wi-a----- 600.00m

#Расширимся
root@deb-vm:/home/vital# lvextend -L +1500 /dev/mapper/vg_test-logdisk1
  Size of logical volume vg_test/logdisk1 changed from 600.00 MiB (150 extents) to 2.05 GiB (525 extents).
  Logical volume vg_test/logdisk1 successfully resized.

#Проверим
root@deb-vm:/home/vital# lvs | grep vg_test
  logdisk1 vg_test   -wi-ao----   2.05g
  logdisk2 vg_test   -wi-ao---- 600.00m
root@deb-vm:/home/vital# pvs | grep vg_test
  /dev/vdb   vg_test   lvm2 a--  1020.00m      0
  /dev/vdc   vg_test   lvm2 a--  1020.00m      0
  /dev/vdd   vg_test   lvm2 a--  1020.00m 360.00m
root@deb-vm:/home/vital# vgs | grep vg_test
  vg_test     3   2   0 wz--n- <2.99g 360.00m
root@deb-vm:/home/vital# df -Th | grep /mnt/01
/dev/mapper/vg_test-logdisk1 ext4      574M   46M  487M   9% /mnt/01

#Непорядок, необходим resize
root@deb-vm:/home/vital# resize2fs /dev/mapper/vg_test-logdisk1
resize2fs 1.47.0 (5-Feb-2023)
Filesystem at /dev/mapper/vg_test-logdisk1 is mounted on /mnt/01; on-line resizing required
old_desc_blocks = 1, new_desc_blocks = 1
The filesystem on /dev/mapper/vg_test-logdisk1 is now 537600 (4k) blocks long.

#Проверим, теперь все ОК
root@deb-vm:/home/vital# df -Th | grep /mnt/01
/dev/mapper/vg_test-logdisk1 ext4      2.1G   46M  1.9G   3% /mnt/01

#Перезагружаемся...Смотрим после перезагрузки все ли ок...
root@deb-vm:/home/vital# df -Th | grep vg_test
#Ничего нет... Значит не все...Не порядок...

#Редактируем конфигурационный файл /etc/fstab
root@deb-vm:/home/vital# echo "/dev/mapper/vg_test-logdisk1 /mnt/01 ext4    defaults        0       2" >> /etc/fstab
root@deb-vm:/home/vital# echo "/dev/mapper/vg_test-logdisk3 /mnt/02 ext4    defaults        0       2" >> /etc/fstab

#Перезагружаемся и смортрим...Видим что теперь все хорошо.
root@deb-vm:/home/vital# df -Th | grep vg_test
/dev/mapper/vg_test-logdisk2 ext4      574M   24K  532M   1% /mnt/02
/dev/mapper/vg_test-logdisk1 ext4      2.1G   46M  1.9G   3% /mnt/01

