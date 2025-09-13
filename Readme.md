# №30 Домашнее задание
# Сценарии iptables
# Написать сценарии iptables.
# 1.Реализовать knocking port.СentralRouter может попасть на ssh inetrRouter через knock скрипт

# 1.1 Заходим на inetrRouter, подгатавливаем его
# Пишем правила "доступа", сразу же коментируем их, чтобы понимать, что происходит, так удобнее
# А нам собственно надо чтобы происходило следующее:
# после последовательного стукача в порты которые мы выбрали (8881 затем 7777, а уж после 9991)
# открылся порт 22 - для доступа по ssh.
root@ubuntuhost:/home/vital# nano 5.knock_iptables
#!/bin/bash
# Добавим необходимые нам дополнительные цепочки
iptables -N TRAFFIC
iptables -N SSH-INPUT
iptables -N SSH-INPUTTWO
#
# Весь трафик приходящий на цепочку "INPUT" с интерфейса "vnet3" пусть смело отправляется во вновь созданную цепочку с именем "TRAFFIC"
# И ежели этот трафик уже ESTABLISHED или что страшнее RELATED то пусть идет себе спокойно (будет "ACCEPT")
iptables -A INPUT -i virbr0 -j TRAFFIC
iptables -A TRAFFIC -m state --state ESTABLISHED,RELATED -j ACCEPT
#
# Ну а если он не ESTABLISHED,RELATED, а "NEW" c портом назначения 22 prot tcp и его источник есть в списке SSH2, пусть на 30 секунд будет "ACCEPT"
# Если же трафик не удовлетворяет вышеуказанному правилу, но он есть в списке SSH2,  то удалим его из списка SSH2
iptables -A TRAFFIC -m state --state NEW -m tcp -p tcp --dport 22 -m recent --rcheck --seconds 30 --name SSH2 -j ACCEPT
iptables -A TRAFFIC -m state --state NEW -m tcp -p tcp -m recent --name SSH2 --remove -j DROP
#
# Проверяем трафик далее...Если трафик "NEW" c портом назначения 9991 prot tcp и его источник есть в списке SSH1, пусть идет в цепочку "SSH-INPUTTWO"
# Если же трафик не удовлетворяет вышеуказанному правилу, но он есть в списке SSH1,  то удалим его из списка SSH1
iptables -A TRAFFIC -m state --state NEW -m tcp -p tcp --dport 9991 -m recent --rcheck --name SSH1 -j SSH-INPUTTWO
iptables -A TRAFFIC -m state --state NEW -m tcp -p tcp -m recent --name SSH1 --remove -j DROP
#
# Проверяем трафик далее...Если трафик "NEW" c портом назначения 7777 prot tcp и его источник есть в списке SSH0, пусть идет в цепочку "SSH-INPUT"
# Если же трафик не удовлетворяет вышеуказанному правилу, но он есть в списке SSH0,  то удалим его из списка SSH0
iptables -A TRAFFIC -m state --state NEW -m tcp -p tcp --dport 7777 -m recent --rcheck --name SSH0 -j SSH-INPUT
iptables -A TRAFFIC -m state --state NEW -m tcp -p tcp -m recent --name SSH0 --remove -j DROP
#
# Проверяем трафик далее...Если трафик "NEW" c портом назначения 8881 prot tcp добавим  его источник в список SSH0 и "ВЫБРОСИМ"
iptables -A TRAFFIC -m state --state NEW -m tcp -p tcp --dport 8881 -m recent --name SSH0 --set -j DROP
#
# Трафик залетевший цепочку "SSH-INPUT" добавим его источник в список SSH1 и "ВЫБРОСИМ"
# Трафик залетевший цепочку "SSH-INPUTTWO" добавим его источник в список SSH2 и "ВЫБРОСИМ"
iptables -A SSH-INPUT -m recent --name SSH1 --set -j DROP
iptables -A SSH-INPUTTWO -m recent --name SSH2 --set -j DROP
# Все остальное, что не удовлетворяет ни одному из приведенных правил тихо "ВЫБРОСИМ"
iptables -A TRAFFIC -j DROP
#
# Все готово, опустим момент как сделать текстовый файл скрипта исполняемым, это и так понятно.
# Просто запустим скрипт
root@ubuntuhost:/home/vital# ./5.knock_iptables
# Ну как то так, сохранять тоже не буду данные правила, пусть себе спокойно "сдохнут" после рестарта системы
# это тестовый стенд, после нашего теста они мне не нужны.
# И вот еще что...В таблице filter все цепочки с политикой ACCEPT, и других правил нет
# в других таблицах (за исключением nat - там только маскарад) также нет никаких правил, все ACCEPT
#
# 1.2 Заходим на СentralRouter, подготавливаем его, пишем скрипт
root@router1:/home/vital# nano knock
# Первым аргументом будет хост(HOST) к которому мы подключаемся, остальные(ARG) - порты в которые мы "стучим"
# "Стукачим" утилитой nmap, сканирование завершится через 100 секунд, попытка будет одна, повторов будет 0
# Следующее действие - это подключение по ssh. С помощью sshpass подставляем наш пароль xxxxxxxxx, логин vital
# хост - первый аргумент переданный скрипту
#!/bin/bash
HOST=$1
shift
for ARG in "$@"
do
        nmap -Pn --host-timeout 100 --max-retries 0 -p $ARG $HOST
done
sshpass -p xxxxxxxxx ssh vital@$HOST
#
# Протестируем
# Сначала так
root@router1:/home/vital# sshpass -p xxxxxxxx ssh vital@192.168.122.1
ssh: connect to host 192.168.122.1 port 22: Connection timed out
# Полнейшая тишина и зафильтрованность
# теперь так
root@router1:/home/vital# ./knock 192.168.122.1 8881 7777 9991
Starting Nmap 7.93 ( https://nmap.org ) at 2025-09-13 11:32 +05
Warning: 192.168.122.1 giving up on port because retransmission cap hit (0).
Nmap scan report for 192.168.122.1
Host is up (0.00045s latency).

PORT     STATE    SERVICE
8881/tcp filtered galaxy4d
MAC Address: 52:54:00:9B:C8:24 (QEMU virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 13.28 seconds
Starting Nmap 7.93 ( https://nmap.org ) at 2025-09-13 11:32 +05
Warning: 192.168.122.1 giving up on port because retransmission cap hit (0).
Nmap scan report for 192.168.122.1
Host is up (0.00034s latency).

PORT     STATE    SERVICE
7777/tcp filtered cbt
MAC Address: 52:54:00:9B:C8:24 (QEMU virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 13.27 seconds
Starting Nmap 7.93 ( https://nmap.org ) at 2025-09-13 11:32 +05
Warning: 192.168.122.1 giving up on port because retransmission cap hit (0).
Nmap scan report for 192.168.122.1
Host is up (0.00031s latency).

PORT     STATE    SERVICE
9991/tcp filtered issa
MAC Address: 52:54:00:9B:C8:24 (QEMU virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 13.27 seconds
Welcome to Ubuntu 24.04.2 LTS (GNU/Linux 6.8.0-79-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Сб 13 сен 2025 06:32:49 UTC

  System load:  0.06                Temperature:               15.0 C
  Usage of /:   51.5% of 142.65GB   Processes:                 220
  Memory usage: 28%                 Users logged in:           1
  Swap usage:   0%                  IPv4 address for enp3s0f0: 172.16.0.4

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

Расширенное поддержание безопасности (ESM) для Applications выключено.

73 обновления может быть применено немедленно.
Чтобы просмотреть дополнительные обновления выполните: apt list --upgradable

Включите ESM Apps для получения дополнительных будущих обновлений безопасности.
Смотрите https://ubuntu.com/esm или выполните: sudo pro status


Last login: Sat Sep 13 06:31:20 2025 from 192.168.122.9
vital@ubuntuhost:~$
# Все хорошо, скрипт работает
# 
#
# 2.Добавить inetRouter2, который виден(маршрутизируется (host-only тип сети для виртуалки)) с хоста или форвардится порт через локалхост-
# - эту роль предоставим router3, который работает с прошлого занятия(сетевая лаборатория)
# запустить nginx на centralServer - уже готов и работает также еще с прошлых занятий
# пробросить 80й порт на inetRouter2 8080.- то есть наша задача: При обращении на порт 8080 router3(192.168.255.6)чтобы нас перенаправило
# на sentralServer(192.168.0.2) порт 80, где у нас трудится наш nginx.
# То есть смоделируем ситуацию - сервер переехал, а пользоатели стучаться по старому адресу. Решим проблему.
# Все действия будем выполнять на router3.Разобьем задачу на две подзадачи:
# 1) Первое что нам нужно поменять адрес назначения вместе с портом с 192.168.255.6:8080 на 192.168.0.2:80
# За замену адреса НАЗНАЧЕНИЯ !!! у нас отвечает DNAT
# Выполним
root@router3:/home/vital# iptables -t nat -A PREROUTING -p tcp --dst 192.168.255.6 --dport 8080 -j DNAT --to-destination 192.168.0.2:80
# Кстати проверим момент
root@router3:/home/vital# cat /proc/sys/net/ipv4/ip_forward
1
# Все ОК
# Ну все можно проверять адрес назначения поменяли - теперь все заработает
root@s2:/home/vital# curl 192.168.255.6:8080
# ФигВам - национальное жилище индейцев, не работает. Заботливо включенный на приемной стороне tcpdump
# сообщил что пакеты доходят до нашего сервера, только вот он их пытается отправить отправителю напрямую,
# минуя наш router3, только вот отправитель про них не в курсе. Он же отправил их на router3 и оттуда же их
# и ждет, а не фиг пойми от кого....Такие дела...
# Тогда нам нужно поменять еще и адрес отправителя
# За замену адреса ИСТОЧНИКА !!! у нас отвечает SNAT
# Выполним
root@router3:/home/vital#iptables -t nat -A POSTROUTING -p tcp --sport 80 --dst 192.168.0.2 -j SNAT --to-source 192.168.255.6:8080
# Что еще важно ? Нам важно понять, что DNAT работает до принятия решения о маршрутизации !!! А SNAT после !!! Все встало на свои места и заработало !!!
root@s2:/home/vital# curl 192.168.0.2:22
curl: (1) Received HTTP/0.9 when not allowed
root@s2:/home/vital# curl 192.168.255.6:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
#
# дефолт в инет оставить через inetRouter.- здесь все понятно
#
# Задача выполнена. Спасибо за внимание !
