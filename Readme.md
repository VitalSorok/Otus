# 1.Сформируем скрипт
nano /home/vital/scripttime

# 2.Добавим туды содержимое
#!/bin/bash
#set -x #Начнем дебаг
{ sttime="$(date -d 'now - 1 hour' +'%s')" #стартовое время в Unix формате час назад
entime="$(date -d now +'%s')" #конечное время в Unix формате сейчас

cat /home/vital/acc.log | while read -r line; do
    timelog=$(echo "$line" | awk '{print $4}' | sed -e 's/\[//g' -e 's/2019:/2019 /g' -e 's/\// /g') #извлекаем время из файла лога и готовим его для конвертирования в Unix формат
    timelogunix=$(date -d "$timelog" +'%s') #конвертируем в unix формат
    if [ "$timelogunix" -ge "$sttime" -a "$timelogunix" -le "$entime" ]; then #если сконвертированные значения из файла находятся в наших пределах то
       echo "$line"
    fi
done } > /home/vital/script/select #файл выборки за последний час #выводим данный кусок лог файла для дальнейшей обработки
vh='cat /home/vital/script/select' #файл выборки за последний час
$vh | awk '{print $1}' | sort | uniq -c | sort -nr > /home/vital/script/ipadd #файл с ip адресами сформированный из выборки за час
$vh | grep -Po '(http|https)://[a-zA-Z0-9./?=_%:-]*' | sort | uniq -c > /home/vital/script/url #файл с url сформированный из выборки за час
$vh | grep -Po '" [4-5][0-5,9][0-9] ' | sort | uniq -c | sort -nr > /home/vital/script/err #файл с ошибками сформированный из выборки за час
$vh | grep -Po '" [1-3][0,2][0-8] ' | sort | uniq -c | sort -nr > /home/vital/script/answ #файл с ответами сформированный из выборки за час
ipadd=$(cat /home/vital/script/ipadd) 
url=$(cat /home/vital/script/url)
err=$(cat /home/vital/script/err)
answ=$(cat /home/vital/script/answ)
start=$(date -d 'now - 1 hour')
finish=$(date -d now)
#set +x #Закончим дебаг
#Отправим письмо с результатами
{ echo To: sorokinvital777@gmail.com; echo From: vitalik40in@yandex.ru; echo Subject: Nginx_Log; echo -e "Начальное время обработки\n$start\nКонечное время обработки\n$finish\nСписок адресов\n$ipadd\nСписок url\n$url\nСписок ошибок\n$err\nСписок ответов\n$answ\n"; } | ssmtp  sorokinvital777@gmail.com

# 3.Сделаем его исполняемым
chmod +x /home/vital/scripttime

# 4.Затолкаем в cron раз в час, предотвратив одновременный запуск нескольких копий, используя утилиту flock
nano /etc/crontab
0 *    * * *   root  /usr/bin/flock  /var/lock/scripttime.lock -c '/home/vital/scripttime'

