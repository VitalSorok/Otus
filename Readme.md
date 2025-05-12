#Занятие 1. Обновление ядра системы
#Цель домашнего задания - научиться обновлять ядро в ОС Linux.
#Описание домашнего задания
#1) Запустить ВМ(Debian 12 в моем случае).
#2) Обновить ядро ОС на новейшую стабильную версию из репозитория debian.
#3) Оформить отчет в README-файле в GitHub-репозитории.

#1)ВМ(Debian 12 запущена успешно)

#2)Приступаем к обновлению ядра

#Проверяем текущую версию ядра. Информацию получили, ядро свежее 25 апреля 25 год. Ну что же так бывает... не будем
#отчаиваться и останавливаться на достигнутом, поищем свежее...
root@deb-vm:/home/vital# uname -av
Linux deb-vm 6.1.0-34-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.135-1 (2025-04-25) x86_64 GNU/Linux


#Ищем свежее....Нашли последнюю версию linux-image-amd64 [installed] кстати...Ну что ж идем дальше... 
root@deb-vm:/home/vital# apt search linux-image
Sorting... Done
Full Text Search... Done
linux-headers-6.1.0-29-amd64/stable 6.1.123-1 amd64
  Header files for Linux 6.1.0-29-amd64

linux-headers-6.1.0-29-cloud-amd64/stable 6.1.123-1 amd64
  Header files for Linux 6.1.0-29-cloud-amd64

linux-headers-6.1.0-29-rt-amd64/stable 6.1.123-1 amd64
  Header files for Linux 6.1.0-29-rt-amd64

linux-headers-6.1.0-32-amd64/stable 6.1.129-1 amd64
  Header files for Linux 6.1.0-32-amd64

linux-headers-6.1.0-32-cloud-amd64/stable 6.1.129-1 amd64
  Header files for Linux 6.1.0-32-cloud-amd64

linux-headers-6.1.0-32-rt-amd64/stable 6.1.129-1 amd64
  Header files for Linux 6.1.0-32-rt-amd64

linux-headers-6.1.0-33-amd64/stable-security 6.1.133-1 amd64
  Header files for Linux 6.1.0-33-amd64

linux-headers-6.1.0-33-cloud-amd64/stable-security 6.1.133-1 amd64
  Header files for Linux 6.1.0-33-cloud-amd64

linux-headers-6.1.0-33-rt-amd64/stable-security 6.1.133-1 amd64
  Header files for Linux 6.1.0-33-rt-amd64

linux-headers-6.1.0-34-amd64/stable-security 6.1.135-1 amd64
  Header files for Linux 6.1.0-34-amd64

linux-headers-6.1.0-34-cloud-amd64/stable-security 6.1.135-1 amd64
  Header files for Linux 6.1.0-34-cloud-amd64

linux-headers-6.1.0-34-rt-amd64/stable-security 6.1.135-1 amd64
  Header files for Linux 6.1.0-34-rt-amd64

linux-image-6.1.0-29-amd64/stable 6.1.123-1 amd64
  Linux 6.1 for 64-bit PCs (signed)

linux-image-6.1.0-29-amd64-dbg/stable 6.1.123-1 amd64
  Debug symbols for linux-image-6.1.0-29-amd64

linux-image-6.1.0-29-amd64-unsigned/stable 6.1.123-1 amd64
  Linux 6.1 for 64-bit PCs

linux-image-6.1.0-29-cloud-amd64/stable 6.1.123-1 amd64
  Linux 6.1 for x86-64 cloud (signed)

linux-image-6.1.0-29-cloud-amd64-dbg/stable 6.1.123-1 amd64
  Debug symbols for linux-image-6.1.0-29-cloud-amd64

linux-image-6.1.0-29-cloud-amd64-unsigned/stable 6.1.123-1 amd64
  Linux 6.1 for x86-64 cloud

linux-image-6.1.0-29-rt-amd64/stable 6.1.123-1 amd64
  Linux 6.1 for 64-bit PCs, PREEMPT_RT (signed)

linux-image-6.1.0-29-rt-amd64-dbg/stable 6.1.123-1 amd64
  Debug symbols for linux-image-6.1.0-29-rt-amd64

linux-image-6.1.0-29-rt-amd64-unsigned/stable 6.1.123-1 amd64
  Linux 6.1 for 64-bit PCs, PREEMPT_RT

linux-image-6.1.0-32-amd64/stable,now 6.1.129-1 amd64 [installed,automatic]
  Linux 6.1 for 64-bit PCs (signed)

linux-image-6.1.0-32-amd64-dbg/stable 6.1.129-1 amd64
  Debug symbols for linux-image-6.1.0-32-amd64

linux-image-6.1.0-32-amd64-unsigned/stable 6.1.129-1 amd64
  Linux 6.1 for 64-bit PCs

linux-image-6.1.0-32-cloud-amd64/stable 6.1.129-1 amd64
  Linux 6.1 for x86-64 cloud (signed)

linux-image-6.1.0-32-cloud-amd64-dbg/stable 6.1.129-1 amd64
  Debug symbols for linux-image-6.1.0-32-cloud-amd64

linux-image-6.1.0-32-cloud-amd64-unsigned/stable 6.1.129-1 amd64
  Linux 6.1 for x86-64 cloud

linux-image-6.1.0-32-rt-amd64/stable 6.1.129-1 amd64
  Linux 6.1 for 64-bit PCs, PREEMPT_RT (signed)

linux-image-6.1.0-32-rt-amd64-dbg/stable 6.1.129-1 amd64
  Debug symbols for linux-image-6.1.0-32-rt-amd64

linux-image-6.1.0-32-rt-amd64-unsigned/stable 6.1.129-1 amd64
  Linux 6.1 for 64-bit PCs, PREEMPT_RT

linux-image-6.1.0-33-amd64/stable-security 6.1.133-1 amd64
  Linux 6.1 for 64-bit PCs (signed)

linux-image-6.1.0-33-amd64-dbg/stable-security 6.1.133-1 amd64
  Debug symbols for linux-image-6.1.0-33-amd64

linux-image-6.1.0-33-amd64-unsigned/stable-security 6.1.133-1 amd64
  Linux 6.1 for 64-bit PCs

linux-image-6.1.0-33-cloud-amd64/stable-security 6.1.133-1 amd64
  Linux 6.1 for x86-64 cloud (signed)

linux-image-6.1.0-33-cloud-amd64-dbg/stable-security 6.1.133-1 amd64
  Debug symbols for linux-image-6.1.0-33-cloud-amd64

linux-image-6.1.0-33-cloud-amd64-unsigned/stable-security 6.1.133-1 amd64
  Linux 6.1 for x86-64 cloud

linux-image-6.1.0-33-rt-amd64/stable-security 6.1.133-1 amd64
  Linux 6.1 for 64-bit PCs, PREEMPT_RT (signed)

linux-image-6.1.0-33-rt-amd64-dbg/stable-security 6.1.133-1 amd64
  Debug symbols for linux-image-6.1.0-33-rt-amd64

linux-image-6.1.0-33-rt-amd64-unsigned/stable-security 6.1.133-1 amd64
  Linux 6.1 for 64-bit PCs, PREEMPT_RT

linux-image-6.1.0-34-amd64/stable-security,now 6.1.135-1 amd64 [installed,automatic]
  Linux 6.1 for 64-bit PCs (signed)

linux-image-6.1.0-34-amd64-dbg/stable-security 6.1.135-1 amd64
  Debug symbols for linux-image-6.1.0-34-amd64

linux-image-6.1.0-34-amd64-unsigned/stable-security 6.1.135-1 amd64
  Linux 6.1 for 64-bit PCs

linux-image-6.1.0-34-cloud-amd64/stable-security 6.1.135-1 amd64
  Linux 6.1 for x86-64 cloud (signed)

linux-image-6.1.0-34-cloud-amd64-dbg/stable-security 6.1.135-1 amd64
  Debug symbols for linux-image-6.1.0-34-cloud-amd64

linux-image-6.1.0-34-cloud-amd64-unsigned/stable-security 6.1.135-1 amd64
  Linux 6.1 for x86-64 cloud

linux-image-6.1.0-34-rt-amd64/stable-security 6.1.135-1 amd64
  Linux 6.1 for 64-bit PCs, PREEMPT_RT (signed)

linux-image-6.1.0-34-rt-amd64-dbg/stable-security 6.1.135-1 amd64
  Debug symbols for linux-image-6.1.0-34-rt-amd64

linux-image-6.1.0-34-rt-amd64-unsigned/stable-security 6.1.135-1 amd64
  Linux 6.1 for 64-bit PCs, PREEMPT_RT

linux-image-amd64/stable-security,now 6.1.135-1 amd64 [installed]
  Linux for 64-bit PCs (meta-package)

linux-image-amd64-dbg/stable-security 6.1.135-1 amd64
  Debugging symbols for Linux amd64 configuration (meta-package)

linux-image-amd64-signed-template/stable-security 6.1.135-1 amd64
  Template for signed linux-image packages for amd64

linux-image-cloud-amd64/stable-security 6.1.135-1 amd64
  Linux for x86-64 cloud (meta-package)

linux-image-cloud-amd64-dbg/stable-security 6.1.135-1 amd64
  Debugging symbols for Linux cloud-amd64 configuration (meta-package)

linux-image-rt-amd64/stable-security 6.1.135-1 amd64
  Linux for 64-bit PCs (meta-package)

linux-image-rt-amd64-dbg/stable-security 6.1.135-1 amd64
  Debugging symbols for Linux rt-amd64 configuration (meta-package)


#Пробуем установить самое свежее и самое последнее и выясняем что оно у нас есть...Уже новейшая версия...
#устанавливать и обновлять нечего, все новое...
root@deb-vm:/home/vital# apt install linux-image-amd64
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
linux-image-amd64 is already the newest version (6.1.135-1).
0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.

