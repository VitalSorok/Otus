# Загрузка системы
# Занятие 7

# 1.Включить отображение меню Grub
root@deb-vm2:/home/vital# nano /etc/default/grub

# По было так и меню было видно, не интересно...
GRUB_TIMEOUT=5

# Сделаем так
GRUB_TIMEOUT_STYLE=menu
GRUB_TIMEOUT=10

# Проверяем дальше
root@deb-vm2:/home/vital# update-grub
root@deb-vm2:/home/vital# reboot

#В результате ничего не поменялось, только стало 10 секунд вместо 5-ти,
#задание выполнено

# 2.Попасть в систему без пароля несколькими способами
# Способ №1 Recovery mode
#Выбираем Advanced options в menu Grub
#Выбираем Debian GNU/Linux, with Linux 6.1.0-32-amd64 (recovery mode)
#Получаем Give root password for maintenance (or press Control-D to continue):
#Жмем Control-D для продолжения, и как результат получаем предложение к аутентификации
#Расстраиваемся и начинаем действовать способом №2

# Способ №2 init=/bin/bash
#Выбираем Advanced options в menu Grub
#Выбираем Debian GNU/Linux, with Linux 6.1.0-32-amd64 (recovery mode)
#Hажимаем 'е'
#Находим linux /vmlinuz-6.1.0-32-amd64 root=/dev/mapper/deb--vm2--vg-root ro single
#Меняем ro single на rw init=/bin/bash
#Жмем Control-X для продолжения, и как результат получаем root-console
#В этот раз не расстраиваемся, а радуемся и меняем пароль командой passwd
#Перезагружаемся заходим под рутом, доступ есть= все отлично.Задание выполнено.

# Способ №3 Использование guestfish и openssl 
#Испробуем способ который на лекции озвучил студент Иван Алифанов
# Останавливаем машину
root@ubuntuhost:/home/vital# virsh destroy deb-vm2

# Готовим пароль по вкусу, можно даже с солью, можно без
root@ubuntuhost:/home/vital# openssl passwd -1 "Password"

# Берем в свои теплые руки наш образ диска 
root@ubuntuhost:/home/vital# guestfish --rw -a /var/lib/libvirt/images/deb-vm.qcow2
><fs>
><fs> launch

# Получаем информацию
><fs> list-filesystems
/dev/sda1: ext2
/dev/deb-vm2/root: ext4
/dev/deb-vm2/swap_1: swap

# Примонтируем то что нас интересует в / 
><fs> mount /dev/sda1 /

# В данном фаиле меняем текущий хэш пароля на свежеприготовленный нами хэш
><fs> vi /etc/shadow

# Уходим
><fs> quit

# Стартуем
root@ubuntuhost:/home/vital# virsh destroy deb-vm2

# Заходим с паролем "Password",радуемся жизни

# 3.Установить систему с LVM, после чего переименовать VG
# Система установлена и готова к бою
root@deb-vm2:/home/vital# df -h
Filesystem                 Size  Used Avail Use% Mounted on
udev                       955M     0  955M   0% /dev
tmpfs                      197M 1012K  196M   1% /run
/dev/mapper/deb--vm2-root  8.4G  4.9G  3.1G  62% /
tmpfs                      984M     0  984M   0% /dev/shm
tmpfs                      5.0M     0  5.0M   0% /run/lock
/dev/vda1                  455M  128M  303M  30% /boot
tmpfs                      197M   36K  197M   1% /run/user/1000

# Смотрим что мы имеем
root@deb-vm2:/home/vital# vgs
  VG      #PV #LV #SN Attr   VSize  VFree
 deb-vm2-vg   1   2   0 wz--n- <9.52g 24.00m

# Переименуем
root@deb-vm2:/home/vital# vgrename deb-vm2-vg deb-vm2

# Лезем в grub.cfg, хотя в нем написано что своими руками там не трогать, а в методичке написано что можно трогать. Поверим методичке
root@deb-vm2:/home/vital# nano /boot/grub/grub.cfg
#Везде где видим что то наподобие /vmlinuz-6.1.0-34-amd64 root=/dev/mapper/deb--vm2--vg-root ro  quiet
#Очень аккуратно меняем на /vmlinuz-6.1.0-34-amd64 root=/dev/mapper/deb--vm2-root ro  quiet
#Сохраняем, выходим, десять раз думаем, проверяем еще 10 раз, перезагружаемся и теряем доступ к системе. Система не перезагружается
#Чешем голову сзади вспоминаем про /etc/fstab, материмся

# Прибегаем к методу предложеному студентом Иваном Алифановым и описанному в способе №3, задания 2 нашей дом. работы
#Только в данном случае редактируем файл /etc/fstab

# Вот это закоментируем
#/dev/mapper/deb--vm2--vg-root /               ext4    errors=remount-ro 0     

# Вот это наберем заново
/dev/mapper/deb--vm2-root /                    ext4    errors=remount-ro 0
     
# Вот это оставим также
# /boot was on /dev/vda1 during installation
UUID=ce998c62-9a29-45ac-9a54-d4eb0ac3d46e /boot           ext2    defaults

# Вот это редактируем, затираем символы --vg 
/dev/mapper/deb--vm2-swap_1 none               swap    sw              0       0

# Вот это оставим также
/dev/sr0        /media/cdrom0   udf,iso9660 user,noauto     0       0

# Сохраняем, перезагружаем и успешно входим в систему. Задание выполнено
