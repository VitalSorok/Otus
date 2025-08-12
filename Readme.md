# Задание: "Запретить всем пользователям, кроме группы admin логин в выходные (суббота и воскресенье), без учета праздников"

# 1. Создадим двух пользователей на нашем сервере.Эти пользователи будут входить в группу "vit". Установим им также пароль.
root@deb-vm2:/home/vital# useradd -m -d /home/vital1 -g vit vital1; useradd -m -d /home/vital2 -g vit vital2
root@deb-vm2:/home/vital# passwd vital1
root@deb-vm2:/home/vital# passwd vital2

# 2. Создадим группу "admin". Затолкаем туда уже существующего пользователя vital.
root@deb-vm2:/home/vital# groupadd admin
root@deb-vm2:/home/vital# usermod -a -G admin vital

# 3. Посмотрим что же получилось
root@deb-vm2:/home/vital# cat /etc/passwd | grep -E '10[0-9][0-9]'
vital:x:1000:1000:vital,,,:/home/vital:/bin/bash
vital1:x:1001:1001::/home/vital1:/bin/bash
vital2:x:1002:1001::/home/vital2:/bin/bash

root@deb-vm2:/home/vital# cat /etc/group | grep -E '10[0-9][0-9]'
vital:x:1000:
vit:x:1001:
admin:x:1002:vital

root@deb-vm2:/home/vital# groups vital
vital : vital cdrom floppy audio dip video plugdev users netdev bluetooth lpadmin scanner admin
root@deb-vm2:/home/vital# groups vital1
vital1 : vit
root@deb-vm2:/home/vital# groups vital2
vital2 : vit
# По итогу у нас всего три несистемных пользователя, за исключением root: vital - это у нас admin, vital1, vital2 - эти двое нет

# 4. Далее приходит на ум метод pam_time.Выполним его.
root@deb-vm2:/home/vital# nano /etc/security/time.conf
sshd|login;*;vital1|vital2;!SaSu0000-2400

# Услуги(службы) sshd (это ssh) и login (это обычная консоль)
# Списки устройств * - все утройства
# Пользователь(и) vital1|vital2 - vital1 и vital2, так как они не входят в группу "admin"
# Время !SaSu0000-2400 - в любое время, но только не в Sa c 00:00 до 24:00 и не в Su c 00:00 до 24:00, то есть время любое, кроме субботы и воскресенья круглосуточно

root@deb-vm2:/home/vital# echo "account requisite pam_time.so" >> /etc/pam.d/common-auth

# 5. Пробуем - задание выполнено, но выполнено "коряво". Неудобно таким образом управлять пользователями

# 6. Сделаем по-другому.Создадим скрипт и дадим ему права на выполнение
root@deb-vm2:/home/vital# cat << EOF > /usr/local/bin/login.sh
#!/bin/bash
#Первое условие: если день недели суббота или воскресенье
if [ $(date +%a) = "Sat" ] || [ $(date +%a) = "Sun" ]; then
 #Второе условие: входит ли пользователь в группу admin
 if getent group admin | grep -qw "$PAM_USER"; then
        #Если пользователь входит в группу admin, то он может подключиться
        exit 0
      else
        #Иначе ошибка (не сможет подключиться)
        exit 1
    fi
  #Если день не выходной, то подключиться может любой пользователь
  else
    exit 0
fi
EOF
root@deb-vm2:/home/vital# chmod +x /usr/local/bin/login.sh

# 7. Допишем правило в конфогурацию
root@deb-vm2:/home/vital# echo "auth required pam_exec.so debug /usr/local/bin/login.sh" >> /etc/pam.d/sshd

# 8. Проверим и убедимся что все работает

