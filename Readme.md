# OSPF
# Задание: "Создать домашнюю сетевую лабораторию,yаучится настраивать протокол OSPF в Linux-based системах"

# 1. Поднять три виртуалки
# три виртуалки с прошлого занятия у нас есть
# Вот их исходные данные
# ip v4 адреса можем не прописывать на интерфейсах, они нам без нужды, пропишем их позже на sub(vlan в нашем случае) интерфейсах
# Схема их соединения такова (кольцо одним словом): 
# router1(enp1s0f1)-router2(enp1s0f0)
# router1(enp1s0f2)-router3(enp1s0f0)
# router2(enp1s0f4)-router3(enp1s0f3)
#
# На всех машинах включен роутинг, также как и на router1
root@router1:/home/vital# cat /proc/sys/net/ipv4/ip_forward
1
# 
# 2.Объединить их разными vlan
# Сказано - сделано, приступим:
#
root@router1:/home/vital# cat /etc/network/if-up.d/vlan_script
#!/bin/bash
for v in 100 200
do
ip link add link enp1s0f1 name vlan1$v type vlan id $v
ip link add link enp1s0f2 name vlan2$v type vlan id $v
ip link set vlan1$v up
ip link set vlan2$v up
ip a add 172.18.$v.1/30 dev vlan1$v
ip a add 172.18.$v.5/30 dev vlan2$v
done
#
root@router2:/home/vital# cat /etc/network/if-up.d/vlan_script
#!/bin/bash
for v in 100 200
do
ip link add link enp1s0f0 name vlan0$v type vlan id $v
ip link add link enp1s0f4 name vlan4$v type vlan id $v
ip link set vlan0$v up
ip link set vlan4$v up
ip a add 192.168.$v.2/30 dev vlan4$v
ip a add 172.18.$v.2/30 dev vlan0$v
done
#
root@router3:/home/vital# cat /etc/network/if-up.d/vlan_script
#!/bin/bash
for v in 100 200
do
ip link add link enp1s0f0 name vlan0$v type vlan id $v
ip link add link enp1s0f3 name vlan3$v type vlan id $v
ip link set vlan0$v up
ip link set vlan3$v up
ip a add 192.168.$v.1/30 dev vlan3$v
ip a add 172.18.$v.6/30 dev vlan0$v
done
# В данных if-up скриптах мы объединили все роутеры разными Vlan(100 и 200)
# 
# 3.Поднимем oSPF на базе frr (выполним аналогичные действия на всех трех роутерах)
root@router1:/home/vital# apt install -y frr frr-pythontools
# Нам нужен только ospf демон, поэтому используем только его.
root@router1:/home/vital# cat /etc/frr/daemons
ospfd=yes
все остальное=no(по дефолту)
root@router1:/home/vital# systemctl restart frr
# Произведем базовую настройку наших роутеров и включим вещание ospf на сети c VID=100 Между всеми нашими роутерами
# выполним эти действия на каждом из роутеров(по аналогии)
root@router1:/home/vital# vtysh
router1# conf t                                                    #переходим в режим конфигурирования на нашем роутере
router1(config)# hostname router1                                  #назначаем имя нашему роутеру                                
router1(config)# access-list OUT seq 1 permit 192.168.0.0/28       #создаем access-list помещаем туда нашу подсеть, информацию о которой будем распространять  
router1(config)# access-list OUT seq 2 deny any                    #всю остальную информацию распространять не будем, она нам не нужна(фильтруем)
router1(config) router ospf                                        #переходим в режим конфигурирования ospf на нашем роутере
router1(config-router)# ospf router-id 1.1.1.1                     #назначаем роутер-id нашему роутеру
router1(config-router)# redistribute connected                     #распространим connected-маршруты этого нам будет достаточно
router1(config-router)# network 172.18.100.0/30 area 0.0.0.10      #распространим в сеть соседа (router2) область 10
router1(config-router)# network 172.18.100.4/30 area 0.0.0.10      #распространим в сеть соседа (router3) область 10
router1(config-router)# distribute-list OUT out connected          #распространим лишь то, что в access-list OUT  
#
# конфигурация router2 будет выглядеть чуть по другому
!
hostname router2
router ospf
 ospf router-id 1.1.1.2
 redistribute connected
 network 172.18.100.0/30 area 0.0.0.10
 network 192.168.100.0/30 area 0.0.0.10
 distribute-list OUT out connected
exit
!
access-list OUT seq 1 permit 192.168.2.128/26
access-list OUT seq 2 deny any
#
# конфигурация router3 почти такая же
!
hostname router3
router ospf
 ospf router-id 1.1.1.3
 redistribute connected
 network 172.18.100.4/30 area 0.0.0.10
 network 192.168.100.0/30 area 0.0.0.10
 distribute-list OUT out connected
exit
!
access-list OUT seq 1 permit 192.168.1.0/25
access-list OUT seq 2 deny any
!
end
# Здесь отличие лишь в том, что на роутерах 2 и 3 мы распространяем другие маршруты и вещаем ospf также в сторону соседей
# Проверим что у нас получилось
root@router1:/home/vital# vtysh -c 'show ip route' | grep O
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       f - OpenFabric,
O   172.18.100.0/30 [110/1000] is directly connected, vlan1100, weight 1, 18:53:41
O   172.18.100.4/30 [110/3000] via 172.18.100.2, vlan1100, weight 1, 18:06:49
O>* 192.168.1.0/25 [110/20] via 172.18.100.2, vlan1100, weight 1, 18:06:48
O>* 192.168.2.128/26 [110/20] via 172.18.100.2, vlan1100, weight 1, 18:53:26
O>* 192.168.100.0/30 [110/2000] via 172.18.100.2, vlan1100, weight 1, 18:53:27
# Все хорошо, поймали три маршрута.Обмен маршрутной информацией идет протокол работает
# На всех остальных участниках OSPF-area ситуация такая же.
# После конфигурирования не забываем сохранить конфигурацию
router1(config-router)# quit
router1(config)# quit
router1# write
#
#
# 4.Изобразим ассиметричный роутинг, для этого искуственно завысим стоимость одного из линков на роутере router1
root@router1:/home/vital# vtysh
router1# conf t
router1(config)# interface 
router1(config)# interface vlan2100
router1(config-if)# ip ospf cost 65535
# запустим icmp запрос с сервера s1 192.168.0.2, распологающегося за роутером router1 на сервер s2 192.168.1.2,
# распологающийся за роутером r3. Наш icmp пакет должен пройти через router1 - router3 затем достигнуть адресата.
# Но так как мы завысили стоимость линка к router3, icmp пакет пойдет по более "дешевому" пути router1 - router2 - router3 и уж после достигнет
# адресата. В обратном же направлении стоимость линков одинакова и icmp пакет должен пройти через router3 - router1 затем достигнет отправителя icmp echo request.
# На лицо несимметричная маршрутизация.
# Проверим это
# Со стороны сервера s1
root@s1:/home/vital# traceroute 192.168.1.2
traceroute to 192.168.1.2 (192.168.1.2), 30 hops max, 60 byte packets
 1  192.168.0.1 (192.168.0.1)  1.682 ms  1.626 ms  1.816 ms
 2  172.18.100.2 (172.18.100.2)  9.654 ms  9.461 ms  9.294 ms
 3  172.18.100.6 (172.18.100.6)  10.017 ms  9.842 ms  10.012 ms
 4  192.168.1.2 (192.168.1.2)  9.600 ms  10.106 ms  10.295 ms
# В обратном направлении(со стороны сервера s3)
traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets
 1  192.168.1.1 (192.168.1.1)  1.302 ms  1.421 ms  1.491 ms
 2  172.18.100.1 (172.18.100.1)  8.170 ms  8.000 ms  8.702 ms
 3  192.168.0.2 (192.168.0.2)  9.153 ms  8.912 ms  8.731 ms
# Примечание: 172.18.100.1 - он же 192.168.0.1
#             172.18.100.6 - он же 192.168.1.1
#
# 5.Сделаем один из линков "дорогим", но что бы при этом роутинг был симметричным.
# Так как дорогой линк мы уже сделали, необходимо теперь избавиться от ассиметриии в маршруте не меняя стоимости линка
# Ассиметрия на линке воникает на участке сети router3. Echo request приходит по линку vlan3100, а вот Echo reply по линку vlan0100
# Как избавиться от этого ? Попробуем использовать PBR и iptables mangle
# Первым делом создадим пользовательскую таблицу маршрутизации
root@router3:/home/vital# echo '2 vlan3100' >> /etc/iproute2/rt_tables 
# Добавим туда требуемый маршрут
root@router3:/home/vital# ip route add default via 192.168.100.2 dev vlan3100 table vlan3100
# Добавим правило использования таблицы маршрутизации для маркированного трафика 
root@router3:/home/vital# ip rule add priority 102 fwmark 0x2 lookup vlan3100
# Ну и наконец промаркируем наш трафик
root@router3:/home/vital# cat ./script_ipt_mark
#!/bin/bash
# Сбрасываем все правила в таблице mangle
iptables -F -t mangle
# Маркируем входящий в интерфейс vlan3100 трафик
iptables -t mangle -A INPUT -j CONNMARK -i vlan3100 --set-mark 0x2
# Присваиваем такую же метку исходящему трафику для его попадания в нашу таблицу
# Таким образом получается откуда пришел туда и ушел
iptables -t mangle -A OUTPUT -j CONNMARK -m connmark --mark 0x2 --restore-mark
# Тем же самым образом маркируем не только входящий, но и транзитный трафик
iptables -t mangle -A PREROUTING -j CONNMARK -m connmark --mark 0x2 --restore-mark
iptables -t mangle -A PREROUTING -j CONNMARK -i vlan3100 --set-mark 0x2
# просматриваем наши правила
iptables -t mangle -L -n -v
# Проверим результат
# Со стороны сервера s1
root@s1:/home/vital# traceroute 192.168.1.2
traceroute to 192.168.1.2 (192.168.1.2), 30 hops max, 60 byte packets
 1  192.168.0.1 (192.168.0.1)  1.682 ms  1.626 ms  1.816 ms
 2  172.18.100.6 (172.18.100.6)  8.017 ms  8.842 ms  8.014 ms
 3  192.168.1.2 (192.168.1.2)  9.600 ms  10.106 ms  10.295 ms
# В обратном направлении(со стороны сервера s3)
traceroute to 192.168.0.2 (192.168.0.2), 30 hops max, 60 byte packets
 1  192.168.1.1 (192.168.1.1)  1.302 ms  1.421 ms  1.491 ms
 2  172.18.100.1 (172.18.100.1)  8.170 ms  8.000 ms  8.702 ms
 3  192.168.0.2 (192.168.0.2)  9.153 ms  8.912 ms  8.731 ms
# Примечание: 172.18.100.1 - он же 192.168.0.1
#             172.18.100.6 - он же 192.168.1.1
# Результат положительный, наш стенд работает.
