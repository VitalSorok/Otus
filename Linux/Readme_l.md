# Строим бонды и вланы
# Задание: "Развести вланами сервера и клиентские машины,соединить линками роутеры, объединить их в бонд"

# 1. Развести вланами сервера и клиентские машины
# У нас в наличии 6-ть VM: c1,c2(клиентские машины),s1,s2(серверные машины),r1,r0(роутеры)
# Начнем c клиентских машин, их адреса должны быть 10.10.10.254/24
# Вот их исходные данные. Их настройки с1 от с2 не отличаются, стоит лишь отметить что
# на них нет vlan, порт подключаемый к ним со стороны роутера работает в access режиме
root@c1:/home/vital# ip a | grep inet
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host noprefixroute
    inet 10.10.10.254/24 brd 10.10.10.255 scope global enp1s0
    inet6 fe80::5054:ff:fea9:3e33/64 scope link
# Далее сервера(s1,s2)
root@s1:/home/vital# ip a | grep inet
    inet 127.0.0.1/8 scope host lo
    inet6 ::1/128 scope host noprefixroute
    inet 192.168.1.2/25 brd 192.168.1.127 scope global enp1s0
    inet6 fe80::5054:ff:fe41:25c3/64 scope link
    inet 10.10.10.1/24 scope global vlan100
    inet6 fe80::5054:ff:fe41:25c3/64 scope link
# На s2 та же самая ситуация, за исключением вместо vlan100 vlan101
# здесь стоит отметить что порты серверов работают уже с тэгируемым трафиком и порты, подключаемые
# к роутеру работают  в trunk режиме.
# Продемонстрируем настройку vlan на серверах на примере s1
# Первой командой включили саб-интерфейс на существующем физическом интерфейсе, второй - назначили ip адрес,
# третьей - включили интерфейс vlan100. Стоит отметить, что для сервера s2 все тоже самое, что для s1,
# за исключением Vlan101 Вместо Vlan100 и id 101 вместо id 100
root@s1:/home/vital# ip link add link enp1s0 name vlan100 type vlan id 100
root@s1:/home/vital# ip address add 10.10.10.1/24 dev vlan100
root@s1:/home/vital# ip link set vlan100 up
#
# Займемся роутером r1. Роутер располагает шестью ethernet интерфейсами
# В нулевой и в первый интерфейс включены s1 и s2 соответственно, а во второй и третий с1 и с2 соответственно
# не забываем, что таким образом интерфейсы 0 и 1 необходимо настроить в транк режиме, а 2 и 3 - в access,
# кроме того необходимо указать трафик с какими тэгами можно пропускать, а с какими нет. Приступим к настройке
# 
root@r1:/home/vital# ip link add name br1 type bridge vlan_filtering 1
root@r1:/home/vital# for i in {0..3}; do ip link set enp1s0f$i master br1
root@r1:/home/vital# bridge vlan add dev enp1s0f2 vid 100 pvid untagged
root@r1:/home/vital# bridge vlan add dev enp1s0f3 vid 101 pvid untagged
root@r1:/home/vital# bridge vlan add dev enp1s0f0 vid 100
root@r1:/home/vital# bridge vlan add dev enp1s0f1 vid 101
root@r1:/home/vital# for l in {0..3}; do ip link set dev enp1s0f$l up; done
#
# Проверим что же получилось
root@r1:/home/vital# ip a | grep enp
2: enp1s0f0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master br1 state UP group default qlen 1000
3: enp1s0f1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master br1 state UP group default qlen 1000
4: enp1s0f2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master br1 state UP group default qlen 1000
5: enp1s0f3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel master br1 state UP group default qlen 1000
6: enp1s0f4: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
7: enp1s0f5: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
# все прекрасно. проверим ping запросами связность между s1 и c1, а также s2 и c2
# и увидим, что связность у нас есть s1 связан c c1 также как и s2 связан c c2 - все работает. Задание выполнено
#
# 2. Cоединить линками роутеры, объединить их в бонд
# Соединим двумя линками роутер r1 с роутером r0
# Для начала на VM r1 и r0 для поддержки бондинга ifenslave установим
root@r0:/home/vital# apt install ifenslave
# Настроим конфигурацию для использования bonding
root@r1:/home/vital# cat /etc/network/interfaces
auto bond0
iface bond0 inet static
address 192.168.33.2/30
bond-slaves enp1s0f4 enp1s0f5
bond-mode active-backup
bond-miimon 100
root@r0:/home/vital# cat /etc/network/interfaces
auto bond0
iface bond0 inet static
address 192.168.33.1/30
bond-slaves enp1s0f0 enp1s0f1
bond-mode active-backup
bond-miimon 100
# Делаем рестарт сетевых сервисов
root@r1:/home/vital# systemctl restart networking
root@r0:/home/vital# systemctl restart networking
# Запускаем пинг - запросы, при этом поочередно выключаем интерфейсы входящие в bond0
# Делаем это с одной и с другой стороны, убеждаемся, что ответы приходят.
# наш bond0 работает, задача выполнена
