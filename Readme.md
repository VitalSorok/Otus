# Задание: "Разворачиваем сетевую лабораторию"
#
# 1. Теоретическая часть
# 1.1 Найти свободные подсети
# Свободные поодсети (Исходя из общего адресного диапазона на каждый офис с маской 24)
#  1 офис - нэт
#  2 офис - нэт
#  центральный офис: 
#              192.168.0.16/28
#              192.168.0.48/28
#              192.168.0.128/25
#  Линк между маршрутизаторами - Там тоже будут свободные в зависимости от того,
#  что мы туда назначим и как. В исходных данных про это ничего не сказано, поэтому                           
#  этот момент опустим. Исходных данных нет.
#  1.2 Посчитать сколько узлов в каждой подсети, включая свободные:
#
# 192.168.0.0	28	1	14	192.168.0.15	14	directors
# 192.168.0.16	28	17	30	192.168.0.31	14	Free
# 192.168.0.32	28	33	46	192.168.0.47	14	office hardware
# 192.168.0.48	28	49	62	192.168.0.63	14	Free
# 192.168.0.64	26	65	126	192.168.0.127	62	Wi-fi
# 192.168.0.128	25	129	254	192.168.0.255	126	Free
# Итого хостов по сети Central office					90	
# Free хостов					                        154
#	
# 192.168.1.0	25	1	126	192.168.1.127	126	dev
# 192.168.1.128	26	129	190	192.168.1.191	62	test servers
# 192.168.1.192	26	193	254	192.168.1.254	62	office hardware
# Итого хостов по сети Office2		           			250
#	
# 192.168.2.0	26	1	62	192.168.2.63	62	dev
# 192.168.2.64	26	65	126	192.168.2.127	62	test servers
# 192.168.2.128	26	129	190	192.168.2.191	62	managers
# 192.168.2.192	26	193	254	192.168.2.255	62	office hardware
# Итого хостов по сети Office1					        248	
# Предпоследняя колонка - это количество узлов
#
# 1.3 Третья по счету колонка справа - это broadcast для каждой подсети
#
# 1.4 Ошибок при разбиении нет
#
# 2. Практическая часть
# 2.1 Готовим "Коммутаторы(bridge)" для соединения сетевых узлов на хостовой машине
root@ubuntuhost:/home/vital# apt install bridge-utils
#
# Пишем скрипт для Bridge, который создаст "мосты" в которые мы потом добавим нужные нам интерфейсы,
# а сами интерфейсы создадим и добавим далее при создании VM.
root@ubuntuhost:/home/vital# nano /etc/network/if-up.d/bridgeUP
#!/bin/bash
for f in {\
dir-centr_serv,\                 #Сеть центральный роутер - центральный сервер
hardware32,\                     #Cеть "железок" №1 центральной сети
hardware64,\                     #Cеть "железок" №2 центральной сети
office1Server,\                  #Сеть роутер  офиса 1 - сервер офиса 1
office2Server,\                  #Сеть роутер  офиса 2 - сервер офиса 2
r0-r1,\                          #Сеть соединительная между центральным(r0) и роутером(r1) первого офиса
r0-r2,\                          #Сеть соединительная между центральным(r0) и роутером(r2) второго офиса
rb1net2,\                        #Сеть офиса 1 первая
rb1net3,\                        #Сеть офиса 1 третья
rb2net3,\                        #Сеть офиса 2 третья
rb2net4,\                        #Сеть офиса 2 четвертая
rb1net5,\                        #Сеть офиса 1 пятая
};
do brctl addbr $f;
   brctl stp $f on;
   ip link set $f up;
done
#
# Выполним скрипт, который будет запускаться каждый раз при старте хостовой машины
root@ubuntuhost:/home/vital# chmod +x /etc/network/if-up.d/bridgeUP && /etc/network/if-up.d/bridgeUP
#
# 2.2 Готовим centralRouter, роль которого также будет выполнять хостовая машина
# Так как хостовая машина(192.168.122.1) у нас будет выполнять роль inetRouter, и ей необходимо знать про всех участников нашей
# сети, чтобы доставить им кажному по маленькому "кусочку" этого вашего "энтырнета", объсним ей что все остальные будущие участники,
# которых мы даже еще не создали, спрятались за centralRouter(192.168.122.9). Запустим этот скрипт позже.
root@ubuntuhost:/home/vital# nano /etc/network/if-up.d/bridgeUP
#!/bin/bash
for i in \
192.168.255.8/30 \
192.168.255.4/30 \
192.168.0.0/28 \
192.168.0.32/28 \
192.168.0.64/26 \
192.168.2.0/26 \
192.168.2.64/28 \
192.168.2.128/26 \
192.168.2.192/26 \
192.168.1.0/25 \
192.168.1.128/26 \
192.168.1.192/26;
do ip route add $i via 192.168.122.9;
done
#
# Далее рассмотрим iptables(nf-tables).
root@ubuntuhost:/home/vital# cat /etc/network/if-up.d/vitalIptables
#!/bin/bash
/sbin/iptables-restore < /etc/vital/vitalIptables
root@ubuntuhost:/home/vital# nano /etc/vital/vitalIptables
#По iptables во всех таблицах, во всех цепочках ACCEPT!, кроме того все таблицы пусты кроме, 
# mangle
-A POSTROUTING -j LIBVIRT_PRT
-A LIBVIRT_PRT -o virbr0 -p udp -m udp --dport 68 -j CHECKSUM --checksum-fill
# nat
-A POSTROUTING -j LIBVIRT_PRT
-A LIBVIRT_PRT -s 192.168.122.0/24 -d 224.0.0.0/24 -j RETURN
-A LIBVIRT_PRT -s 192.168.122.0/24 -d 255.255.255.255/32 -j RETURN
-A LIBVIRT_PRT -s 192.168.122.0/24 ! -d 192.168.122.0/24 -p tcp -j MASQUERADE --to-ports 1024-65535
-A LIBVIRT_PRT -s 192.168.122.0/24 ! -d 192.168.122.0/24 -p udp -j MASQUERADE --to-ports 1024-65535
-A LIBVIRT_PRT -s 192.168.122.0/24 ! -d 192.168.122.0/24 -j MASQUERADE
# То есть если говорить совсем грубо работает только маскарад и то лишь для сети 192.168.122.0/24,
# все остальное просто форвард и все !
# Кстати посмотрим на наш форвард
root@ubuntuhost:/home/vital# cat /etc/sysctl.conf | grep net.ipv4.ip_
net.ipv4.ip_forward=1
# Все ок
# 
# Добавим в самый низ /etc/vital/vitalIptables строку, учитывая что внешний наш адрес 172.16.0.4 он статический,
# учтем также что enp3s0f0 это наш выходной интерфейс чтобы нат работал не только для уже существующих сетей
# с адресами 192.168.122.0/24
-A LIBVIRT_PRT -o enp3s0f0 -j SNAT --to-source 172.16.0.4
# inetRouter готов
#
# 2.3 Готовим centralRouter.
# У нас уже есть готовые виртуальные машины c предыдущих занятий, запущенные посредством libvirt qemu/kvm 
# Посмотрим
root@ubuntuhost:/home/vital# virsh list --all
 Id   Name      State
-------------------------
 1    deb-vm2   running
 2    cent-vm   running
 3    deb-vm    running
#
# Возьмем за основу deb-vm, остановим ее, скопируем xml файл, доработаем под свои нужды
root@ubuntuhost:/home/vital# virsh shutdown deb-vm $$ virsh dumpxml deb-vm > v1.xml
# 
# Удалим uuid и mac address, так как libvirt сгенерирует их позже сам
root@ubuntuhost:/home/vital# sed -i '/uuid/d' v1.xml; sed -i '/mac address/d' v1.xml
#
# Подготовим диск для нашего роутера
root@ubuntuhost:/home/vital# cp /var/lib/libvirt/images/{deb-vm.qcow2,router1.qcow2}
#
# Заменим в файле v1.xml 'deb-vm' на 'router1' в названии VM и пути диска.
#
# Добавим в v1.xml пять сетевых интерфейсов, при этом тип=бридж, source bridge - к какому бриджу подключаем,
# target dev - имя нашего интерфейсаб которое отобразится на host машине.
root@ubuntuhost:/home/vital# nano v1.xml
...
    <interface type='network'>
      <mac address='52:54:00:78:cd:31'/>
      <source network='default'/>
      <model type='virtio'/>
      <address type='pci' domain='0x0000' bus='0x01' slot='0x00' function='0x0'/>
    </interface>
    <interface type='bridge'>
      <source bridge='r0-r1'/>
      <target dev='r0net1'/>
      <address type='pci' domain='0x0000' bus='0x01' slot='0x00' function='0x1'/>
    </interface>
    </interface>
    <interface type='bridge'>
      <source bridge='r0-r2'/>
      <target dev='r0net2'/>
      <address type='pci' domain='0x0000' bus='0x01' slot='0x00' function='0x2'/>
    </interface>
    </interface>
    <interface type='bridge'>
      <source bridge='dir-centr_serv'/>
      <target dev='r0net3'/>
      <address type='pci' domain='0x0000' bus='0x01' slot='0x00' function='0x3'/>
    </interface>
    </interface>
    <interface type='bridge'>
      <source bridge='hardware32'/>
      <target dev='r0net4'/>
      <address type='pci' domain='0x0000' bus='0x01' slot='0x00' function='0x4'/>
    </interface>
    </interface>
    <interface type='bridge'>
      <source bridge='hardware64'/>
      <target dev='r0net5'/>
      <address type='pci' domain='0x0000' bus='0x01' slot='0x00' function='0x5'/>
    </interface>
...
# Запустим нашу VM из созданного нами файла, зайдем в консоль
root@ubuntuhost:/home/vital# virsh define v1.xml && virsh start router1 && virsh console router1
#
# Пропишем все необходимое для работы наших сетей
root@deb-vm:/home/vital# nano /etc/network/interfaces
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

auto enp1s0f0
allow-hotplug enp1s0f0
auto enp1s0f1
allow-hotplug enp1s0f1
auto enp1s0f2
allow-hotplug enp1s0f2
auto enp1s0f3
allow-hotplug enp1s0f3
auto enp1s0f4
allow-hotplug enp1s0f4
auto enp1s0f5

iface enp1s0f0 inet dhcp

iface enp1s0f1 inet static
address 192.168.255.9/30

iface enp1s0f2 inet static
address 192.168.255.5/30

iface enp1s0f3 inet static
address 192.168.0.1/28

iface enp1s0f4 inet static
address 192.168.0.33/28

iface enp1s0f5 inet static
address 192.168.0.65/26

# То есть нулевой интерфейс получает все по dhcp от хостовой машины (inetRouter),
# все остальное прописываем статически
#
# Далее 
root@deb-vm:/home/vital# echo router1 > /etc/hostname
root@deb-vm:/home/vital# sed -i 's/deb-vm/router1/g' /etc/hosts
# 
# Маршруты в Inet есть, пропишем маршруты для остальных сетей
root@router1:/home/vital# nano /etc/network/if-up.d/vital_route
#!/bin/bash
for i in \
192.168.2.0/26 \
192.168.2.64/28 \
192.168.2.128/26 \
192.168.2.192/26;
do ip route add $i via 192.168.255.10;
done;
for i in \
192.168.1.0/25 \
192.168.1.128/26 \
192.168.1.192/26;
do ip route add $i via 192.168.255.6;
done
root@router1:/home/vital# chmod +x /etc/network/if-up.d/vital_route
#
# Пока больше ничего не делаем, перезапустимся позже и у нас "встанут на свои места" и адреса интерфейсов и имя системы и маршруты
#
# 2.4 Готовим по такому же принципу officeRouter1(router2), officeRouter2(router3)
# Только из конфига .xml убираем это:
    <interface type='network'>
      <mac address='52:54:00:78:cd:31'/>
      <source network='default'/>
      <model type='virtio'/>
      <address type='pci' domain='0x0000' bus='0x01' slot='0x00' function='0x0'/>
# Оставляем только "мостовые" интерфейсы в router2 - 5 мостовых интерфейсов,
# а уж в router3 - итого меньше 4, все остальное удаляем
# Также прописываем имена машин, адресацию сетей, маршруты здесь уже не нужны.
# потому что в настройку сети для соответствующего интерфейса добавляем
root@router2:/home/vital# nano /etc/network/interfaces
gateway 192.168.255.9 - для router2
gateway 192.168.255.5 - для router2
#
# 2.5 Готовим три сервера по такому же принципу, только там еще проще. Остается один интерфейс со статическим адресом 
# а также добавляется gateway
# 
# всю инфраструктуру будующей сетевой лаборатории перезагружаем 
# На xocт машине делаем так:
root@ubuntuhost:/home/vital# for i in router1 router2 router3 s1 s2 s3; do virsh autostart $i; done
#
#
# 2.6 Проверим результат нашей работы
root@ubuntuhost:/home/vital# brctl show
bridge name             bridge id              STP enabled     interfaces
dir-centr_serv          8000.9699959cc946       yes             r0net3     #интерф 3 router1(central)
                                                                s1         #интерф 1 server1(central)
hardware32              8000.3e625e42950e       yes             r0net4     #интерф 4 router1(central)
hardware64              8000.9a15792053a2       yes             r0net5     #интерф 5 router1(central)
office1Server           8000.1ee5bcc9a402       yes             r1net4     #интерф 4 router2(office1)
                                                                s2         #интерф 1 server2(office1)
office2Server           8000.f244d5301eb4       yes             r2net2     #интерф 2 router3(office2)
                                                                s3         #интерф 1 server3(office2)
r0-r1                   8000.a2d0bb9eaae5       yes             r0net1     #интерф 1 router1(central)
                                                                r1net1     #интерф 1 router2(office1)
r0-r2                   8000.966f8d2fdde9       yes             r0net2     #интерф 2 router1(central)
                                                                r2net1     #интерф 1 router3(office2)
rb1net2                 8000.5e378ebc1ee8       yes             r1net2     #интерф 2 router2(office1)
rb1net3                 8000.664ecbf0ac02       yes             r1net3     #интерф 3 router2(office1)
rb1net5                 8000.5e7de6c19e45       yes             r1net5     #интерф 5 router2(office1)
rb2net3                 8000.e2f7e53058bc       yes             r2net3     #интерф 3 router3(office2)
rb2net4                 8000.a2c141efd613       yes             r2net4     #интерф 4 router3(office2)
#
# Теперь зайдем на самый дальний сервер(сервер s3 в office2) и выполним трассировку,
# это будет показательная проверка 
# до интернета
root@s3:/home/vital# traceroute 8.8.8.8
traceroute to 8.8.8.8 (8.8.8.8), 30 hops max, 60 byte packets
 1  192.168.1.1 (192.168.1.1)  1.520 ms  1.548 ms  1.557 ms
 2  192.168.255.5 (192.168.255.5)  4.320 ms  4.139 ms  7.853 ms
 3  ubuntuhost (192.168.122.1)  7.247 ms  7.485 ms  7.310 ms
 4  _gateway (172.16.0.1)  7.445 ms  7.525 ms  7.597 ms
 5  h31-8-0-1.dyn.bashtel.ru (31.8.0.1)  14.362 ms  15.628 ms  14.017 ms
 6  185.140.151.144 (185.140.151.144)  11.079 ms 185.140.151.138 (185.140.151.138)  7.986 ms  7.450 ms
 7  * * *
 8  * * *
 9  * * *
10  72.14.235.226 (72.14.235.226)  27.994 ms 108.170.225.36 (108.170.225.36)  27.840 ms *
11  * * 192.178.243.132 (192.178.243.132)  39.429 ms
12  * 216.239.51.32 (216.239.51.32)  42.095 ms *
13  209.85.254.20 (209.85.254.20)  44.869 ms * 172.253.66.110 (172.253.66.110)  43.295 ms
14  142.250.238.181 (142.250.238.181)  41.113 ms 216.239.49.3 (216.239.49.3)  43.106 ms 216.239.63.129 (216.239.63.129)  40.614 ms
15  * * *
16  * * *
17  * * *
18  * * *
19  * * *
20  * * *
21  * * *
22  * * *
23  * * *
24  dns.google (8.8.8.8)  41.599 ms  43.713 ms *
#
# до сервера в центральном офисе (s1)
root@s3:/home/vital# traceroute 192.168.0.2
traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets
 1  192.168.1.1 (192.168.1.1)  1.641 ms  1.634 ms  1.441 ms
 2  192.168.255.5 (192.168.255.5)  6.221 ms  6.062 ms  6.639 ms
 3  192.168.0.2 (192.168.0.2)  8.806 ms  8.946 ms  8.693 ms
# 
# Ну и наконец до сервера s2 в offise1
root@s3:/home/vital# traceroute 192.168.2.130
traceroute to 192.168.2.130 (192.168.2.130), 30 hops max, 60 byte packets
 1  192.168.1.1 (192.168.1.1)  1.685 ms  1.396 ms  1.524 ms
 2  192.168.255.5 (192.168.255.5)  3.723 ms  3.568 ms  7.871 ms
 3  192.168.255.10 (192.168.255.10)  8.063 ms  7.343 ms  10.127 ms
 4  192.168.2.130 (192.168.2.130)  12.238 ms  12.341 ms  12.599 ms
#
# Все соответствует схеме сети. Проверять необходимости больше нет.
