# OTUS ДЗ 7 Инициализация системы. Systemd и SysV (Centos 7)
-----------------------------------------------------------------------
### Домашнее задание

    Цель: Управление автозагрузкой сервисов происходит через systemd. Вместо cron'а тоже используется systemd. И много других возможностей. В ДЗ нужно написать свой systemd-unit.
    1. Написать сервис, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова. Файл и слово должны задаваться в /etc/sysconfig
    2. Из epel установить spawn-fcgi и переписать init-скрипт на unit-файл. Имя сервиса должно так же называться.
    3. Дополнить юнит-файл apache httpd возможностьб запустить несколько инстансов сервера с разными конфигами
    4*. Скачать демо-версию Atlassian Jira и переписать основной скрипт запуска на unit-файл
    Задание необходимо сделать с использованием Vagrantfile и proviosioner shell (или ansible, на Ваше усмотрение)

1. Написать сервис, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова. Файл и слово должны задаваться в /etc/sysconfig:
- Создаем файл конфигурации для сервиса ```touch /etc/sysconfig/watchlog```
- Зполняем его данными:
```
# Configuration file for my watchdog service
# Place it to /etc/sysconfig
# File and word in that file that we will be monit
WORD="ALERT"
LOG=/var/log/watchlog.log
```
- Создаем ```touch /var/log/watchlog.log``` и пишем туда строку ```ALERT```
- Далее создаем скрипт sh:
```
#!/bin/bash
WORD=$1
LOG=$2
DATE=`date`
if grep $WORD $LOG &> /dev/null
then
logger "$DATE: I found word, Master!" # logger отправляет лог в системный журнал (/var/log/messages)
else
exit 0
fi
```
- Далее создаем unit для сервиса (точнее создаем сам сервис), создаем файл ```nano /etc/systemd/system/watchlog.service```:
```
[Unit]
Description=My watchlog service
[Service]
Type=oneshot
EnvironmentFile=/etc/sysconfig/watchdog
ExecStart=/opt/watchlog.sh $WORD $LOG
```
- Создаем unit для таймера (timer файл), ```nano /etc/systemd/system/watchlog.timer```:
```
[Unit]
Description=Run watchlog script every 30 second
[Timer]
# Run every 30 second
OnActiveSec=1sec # Активировать сервис .service при активации таймера (запуск)
#OnBootSec=1min # Активировать сервис .service файл при загрузке системы
#OnUnitActiveSec=30sec # 1 раз в 30 секунд
OnCalendar=*:*:0/30 # Еще одна форма, для того чтобы указать периодичность, 1 раз в 30 секунд
AccuracySec=1us # Указывается точность
Unit=watchlog.service
[Install]
WantedBy=multi-user.target
```
Для того, чтобы сервис .service работал, необходимо в файле watchlog.timer указать, чтобы сам сервис активировался при активации таймера. Под активацией таймера, подразумевается старт таймера.
- Далее выполняем ```systemctl enable watchlog.timer``` и ```systemctl start watchlog.timer```
- Выполняем команду ```tail -f /var/log/messages```:
```
Sep  9 04:04:00 lvm systemd: Started Cleanup of Temporary Directories.
Sep  9 04:04:30 lvm systemd: Started My watchlog service.
Sep  9 04:04:30 lvm systemd: Starting My watchlog service...
Sep  9 04:04:30 lvm root: Mon Sep  9 04:04:30 UTC 2019: I found word, Master!
Sep  9 04:05:00 lvm systemd: Started My watchlog service.
Sep  9 04:05:00 lvm systemd: Starting My watchlog service...
Sep  9 04:05:00 lvm root: Mon Sep  9 04:05:00 UTC 2019: I found word, Master!
Sep  9 04:05:30 lvm systemd: Started My watchlog service.
Sep  9 04:05:30 lvm systemd: Starting My watchlog service...
Sep  9 04:05:30 lvm root: Mon Sep  9 04:05:30 UTC 2019: I found word, Master!
```
2. Из epel установить spawn-fcgi и переписать init-скрипт на unit-файл. Имя сервиса должно так же называться:
- Устанавливаем ```yum install epel-release -y && yum install spawn-fcgi php php-cli mod_fcgid httpd -y```
- ```etc/rc.d/init.d/spawn-fcg``` - скрипт, который будет переписан
- Заходим в ```/etc/sysconfig/spawn-fcgi``` и приводим его к следующему виду:
```
# You must set some working options before the "spawn-fcgi" service will work.
# If SOCKET points to a file, then this file is cleaned up by the init script.
#
# See spawn-fcgi(1) for all possible options.
#
# Example :
SOCKET=/var/run/php-fcgi.sock
OPTIONS="-u apache -g apache -s $SOCKET -S -M 0600 -C 32 -F 1 -- /usr/bin/php-cgi"
```
- Создаем unit файл ```nano /etc/systemd/system/spawn-fcgi.service```:
```
[Unit]
Description=Spawn-fcgi startup service by Otus
After=network.target
[Service]
Type=simple
PIDFile=/var/run/spawn-fcgi.pid
EnvironmentFile=/etc/sysconfig/spawn-fcgi
ExecStart=/usr/bin/spawn-fcgi -n $OPTIONS
KillMode=process
[Install]
WantedBy=multi-user.target
```
- ```systemctl start spawn-fcgi``` и ```systemctl status spawn-fcgi```:
```
[root@lvm system]# systemctl status spawn-fcgi
● spawn-fcgi.service - Spawn-fcgi startup service by Otus
   Loaded: loaded (/etc/systemd/system/spawn-fcgi.service; enabled; vendor preset: disabled)
   Active: active (running) since Mon 2019-09-09 03:49:12 UTC; 23min ago
 Main PID: 924 (php-cgi)
    Tasks: 33
   Memory: 5.3M
   CGroup: /system.slice/spawn-fcgi.service
           ├─ 924 /usr/bin/php-cgi
           ├─ 981 /usr/bin/php-cgi
           ├─ 982 /usr/bin/php-cgi
           ├─ 983 /usr/bin/php-cgi
           ├─ 984 /usr/bin/php-cgi
           ├─ 985 /usr/bin/php-cgi
           ├─ 986 /usr/bin/php-cgi
           ├─ 987 /usr/bin/php-cgi
           ├─ 988 /usr/bin/php-cgi
           ├─ 989 /usr/bin/php-cgi
           ├─ 990 /usr/bin/php-cgi
           ├─ 991 /usr/bin/php-cgi
           ├─ 992 /usr/bin/php-cgi
           ├─ 993 /usr/bin/php-cgi
           ├─ 994 /usr/bin/php-cgi
           ├─ 995 /usr/bin/php-cgi
           ├─ 996 /usr/bin/php-cgi
           ├─ 997 /usr/bin/php-cgi
           ├─ 998 /usr/bin/php-cgi
           ├─ 999 /usr/bin/php-cgi
           ├─1000 /usr/bin/php-cgi
           ├─1001 /usr/bin/php-cgi
           ├─1002 /usr/bin/php-cgi
           ├─1003 /usr/bin/php-cgi
           ├─1004 /usr/bin/php-cgi
           ├─1005 /usr/bin/php-cgi
           ├─1006 /usr/bin/php-cgi
           ├─1007 /usr/bin/php-cgi
           ├─1008 /usr/bin/php-cgi
           ├─1009 /usr/bin/php-cgi
           ├─1010 /usr/bin/php-cgi
           ├─1011 /usr/bin/php-cgi
           └─1012 /usr/bin/php-cgi

Sep 09 03:49:12 lvm systemd[1]: Started Spawn-fcgi startup service by Otus.
Sep 09 03:49:12 lvm systemd[1]: Starting Spawn-fcgi startup service by Otus...
```
3. Дополнить юнит-файл apache httpd возможностьб запустить несколько инстансов сервера с разными конфигами:
- Для запуска нескольких экземпляров сервиса будем использовать шаблон в конфигурации файла окружения, т.е. шаблонизацию

Копируем файл из ```/usr/lib/systemd/system/```, ```cp /usr/lib/systemd/system/httpd.service /etc/systemd/system```, далее переименовываем ```mv /etc/systemd/system/httpd.service /etc/systemd/system/httpd@.service``` и приводим к виду:
```
[Unit]
Description=The Apache HTTP Server
After=network.target remote-fs.target nss-lookup.target
Documentation=man:httpd(8)
Documentation=man:apachectl(8)
[Service]

Type=notify
EnvironmentFile=/etc/sysconfig/httpd-%I #(%I - это и есть подстановка)
ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND
ExecReload=/usr/sbin/httpd $OPTIONS -k graceful
ExecStop=/bin/kill -WINCH ${MAINPID}
KillSignal=SIGCONT
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```
- В самом файле окружения (которых будет два) задается опция для запуска веб-сервера с необходимым конфигурационным файлом
```
# /etc/sysconfig/httpd-first
OPTIONS=-f conf/first.conf
```
```
# /etc/sysconfig/httpd-second
OPTIONS=-f conf/second.conf
```
В директории с конфигами httpd должны лежать два конфига, в нашем случае это будут first.conf и second.conf.
- Для удачного запуска, в конфигурационных файлах должны бытя указаны уникальные для каждого экземпляра опции Listen и PidFile. Конфиги можно скопировать и поправить только второй, в нем должны быть след опции:
```PidFile /var/run/httpd-second.pid``` - т.е. должен быть указан файл пида
```Listen 8080``` - указан порт, который будет отличаться от другого инстанса
```
[root@lvm system]# systemctl status httpd@second.service
● httpd@second.service - The Apache HTTP Server
   Loaded: loaded (/etc/systemd/system/httpd@.service; disabled; vendor preset: disabled)
   Active: inactive (dead)
     Docs: man:httpd(8)
           man:apachectl(8)
[root@lvm system]# systemctl start httpd@second.service
[root@lvm system]# systemctl start httpd@first.service
[root@lvm system]# systemctl status httpd@first.service
● httpd@first.service - The Apache HTTP Server
   Loaded: loaded (/etc/systemd/system/httpd@.service; disabled; vendor preset: disabled)
   Active: active (running) since Mon 2019-09-09 04:30:05 UTC; 5s ago
     Docs: man:httpd(8)
           man:apachectl(8)
 Main PID: 2050 (httpd)
   Status: "Processing requests..."
   CGroup: /system.slice/system-httpd.slice/httpd@first.service
           ├─2050 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           ├─2051 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           ├─2052 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           ├─2053 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           ├─2054 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           ├─2055 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           └─2056 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND

Sep 09 04:30:05 lvm systemd[1]: Starting The Apache HTTP Server...
Sep 09 04:30:05 lvm httpd[2050]: AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 127.0.0.1. Set the 'ServerName' directive globally to suppress this message
Sep 09 04:30:05 lvm systemd[1]: Started The Apache HTTP Server.
```
```
[root@lvm system]# systemctl status httpd@second.service
● httpd@second.service - The Apache HTTP Server
   Loaded: loaded (/etc/systemd/system/httpd@.service; disabled; vendor preset: disabled)
   Active: active (running) since Mon 2019-09-09 04:30:00 UTC; 28s ago
     Docs: man:httpd(8)
           man:apachectl(8)
 Main PID: 2033 (httpd)
   Status: "Total requests: 0; Current requests/sec: 0; Current traffic:   0 B/sec"
   CGroup: /system.slice/system-httpd.slice/httpd@second.service
           ├─2033 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           ├─2038 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           ├─2039 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           ├─2040 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           ├─2041 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           ├─2042 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           └─2043 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND

Sep 09 04:29:59 lvm systemd[1]: Starting The Apache HTTP Server...
Sep 09 04:30:00 lvm httpd[2033]: AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 127.0.0.1. Set the 'ServerName' directive globally to suppress this message
Sep 09 04:30:00 lvm systemd[1]: Started The Apache HTTP Server.
```
```
[root@lvm system]# netstat -tunap
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:111             0.0.0.0:*               LISTEN      611/rpcbind
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      923/sshd
tcp        0      0 127.0.0.1:25            0.0.0.0:*               LISTEN      1054/master
tcp        0      0 10.0.2.15:22            10.0.2.2:5773           ESTABLISHED 1412/sshd: vagrant
tcp6       0      0 :::111                  :::*                    LISTEN      611/rpcbind
tcp6       0      0 :::80                   :::*                    LISTEN      2050/httpd
tcp6       0      0 :::8080                 :::*                    LISTEN      2033/httpd
```
