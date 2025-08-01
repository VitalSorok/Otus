# Vagrant-стенд для обновления ядра и создания образа системы
# Запустить ВМ с помощью Vagrant

# 1. При помощи нашего роутера микротик, какой то там матери и сервиса 
# https://ipspeed.info/ поднимем SSTP до японского сервера 

# 2. Cтартанем, проверим его старт
# [vital@Komar_router] /interface sstp-client> print
Flags: X - disabled, R - running 
 0  R name="sstp-out1" max-mtu=1500 max-mru=1500 mrru=disabled connect-to=vpn393718194.opengw.net:1999 http-proxy=0.0.0.0:1999 certificate=none verify-server-certificate=no verify-server-address-from-certificate=yes user="vpn" 
      password="vpn" profile=default-encryption keepalive-timeout=60 add-default-route=no dial-on-demand=no authentication=pap,chap,mschap1,mschap2 pfs=no tls-version=any

# 3. Промаркируем трафик, завернем весь трафик от нашего хоста в тунель SSTP при помощи Mangle и добавления статического маршрута,
# настроим маскарад на "sstp-out1", разрешим в firewall все это дело, оттраcсируем проверим, все работает
[vital@Komar_router] /ip route> print
Flags: X - disabled, A - active, D - dynamic, C - connect, S - static, r - rip, b - bgp, o - ospf, m - mme, B - blackhole, U - unreachable, P - prohibit 
 #      DST-ADDRESS        PREF-SRC        GATEWAY            DISTANCE

 1 A S  ;;; vagrant
        0.0.0.0/0                          1.0.0.1                   1

 4 ADC  1.0.0.1/32         10.211.1.15     sstp-out1                 0

[vital@Komar_router] > ip fi fi print
Flags: X - disabled, I - invalid, D - dynamic 

24    ;;; Access to vagrant
      chain=forward action=accept in-interface=vlan303Otus log=no log-prefix="" 

[vital@Komar_router] /ip firewall nat> print
 8    ;;; NAT
      chain=srcnat action=masquerade out-interface=sstp-out1 log=no log-prefix=""

# 4. VirtualBox использовать не будем. У нас есть kvm+qemu. Работаем с тем, что есть,
# нашим провайдером будет libvirt

# 5. Тащим вагрант
root@ubuntuhost:/home/vital# curl -O https://releases.hashicorp.com/vagrant/2.4.7/vagrant_2.4.7-1_amd64.deb

# 6. Ставим вагрант
root@ubuntuhost:/home/vital# apt install ./vagrant_2.4.7-1_amd64.deb

# 7. Смотрим вагрант
root@ubuntuhost:/home/vital# vagrant --version

Vagrant 2.4.7
# 8. Ставим необходимые для плагина вагрант зависимости
root@ubuntuhost:/home/vital# apt install -y  build-essential ruby-dev libxml2-dev libxslt1-dev libz-dev

# 9. Ставим сам плагин
root@ubuntuhost:/home/vital# vagrant plugin install vagrant-libvirt

# 10. Смотрим на плагин
root@ubuntuhost:/home/vital# vagrant plugin list
vagrant-libvirt (0.12.2, global)

# 11. Подбираемся к самому интересному....
root@ubuntuhost:/home/vital# nano Vagrantfile
# Чтобы не было проблем с пулом добавляем 
# v.storage_pool_name = "images"

# -*- mode: ruby -*-
# vi: set ft=ruby :

# Описываем Виртуальные машины
MACHINES = {
  # Указываем имя ВМ "kernel update"
  :"kernel-update" => {
              #Какой vm box будем использовать
              :box_name => "generic/centos8s",
              #Указываем box_version
              :box_version => "4.3.4",
              #Указываем количество ядер ВМ
              :cpus => 2,
              #Указываем количество ОЗУ в мегабайтах
              :memory => 1024,
            }
}

Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
    # Отключаем проброс общей папки в ВМ
    config.vm.synced_folder ".", "/vagrant", disabled: true
    # Применяем конфигурацию ВМ
    config.vm.define boxname do |box|
      box.vm.box = boxconfig[:box_name]
      box.vm.box_version = boxconfig[:box_version]
      box.vm.host_name = boxname.to_s
      box.vm.provider "libvirt" do |v|
        v.storage_pool_name = "images"
        v.memory = boxconfig[:memory]
        v.cpus = boxconfig[:cpus]
      end
    end
  end
end

# 12. Запускаем 
root@ubuntuhost:/home/vital# vagrant up

# 13. Смотрим
root@ubuntuhost:/home/vital# virsh list --all
 Id   Name                  State
--------------------------------------
 2    deb-vm                running
 4    vital_kernel-update   running
 -    cent-vm               shut off
 -    deb-vm2               shut off

# 14. Появилась новая машина  "4    vital_kernel-update   running" 

# 15. Пройдем внутрь 
root@ubuntuhost:/home/vital# vagrant ssh
[vagrant@kernel-update ~]$ sudo su
# Первая часть задания выполнена

# Обновить ядро ОС из репозитория ELRepo
# 16. Смотрим что внутри
[root@kernel-update vagrant]# uname -r
4.18.0-516.el8.x86_64

# 17. Не пойдет, подключим репозиторий и обновимся
root@kernel-update vagrant]# yum install -y https://www.elrepo.org/elrepo-release-8.el8.elrepo.noarch.rpm
# Не взлетел, тяги не хваитило, все дело в том что нам необходимо подключить репозиторий vault.centos.org

# 18. Тогда поступим так
[root@kernel-update vagrant]# sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
[root@kernel-update vagrant]# sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*

# 19. Теперь так
root@kernel-update vagrant]# yum install -y https://www.elrepo.org/elrepo-release-8.el8.elrepo.noarch.rpm

# 20. Проверим наш репозиторий 
[root@kernel-update vagrant]# yum repolist | grep elrepo
Failed to set locale, defaulting to C.UTF-8
elrepo              ELRepo.org Community Enterprise Linux Repository - el8

# 21. Установим ядро
[root@kernel-update vagrant]# yum --enablerepo elrepo-kernel install kernel-ml -y

# 22. Далее
[root@kernel-update vagrant]# reboot
Connection to 192.168.121.148 closed by remote host.

# 23. Затем
root@ubuntuhost:/home/vital# vagrant ssh
[fog][WARNING] Unrecognized arguments: libvirt_ip_command
Last login: Fri Aug  1 03:53:51 2025 from 192.168.121.1
[vagrant@kernel-update ~]$ uname -r
6.15.8-1.el8.elrepo.x86_64

# Все готово
