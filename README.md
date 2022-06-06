**Домашнее задание**

Systemd

Описание выполнения домашнего задания:

Выполнить следующие задания и подготовить развёртывание результата выполнения с использованием Vagrant и Vagrant shell provisioner (или Ansible, на Ваше усмотрение):

1. Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова (файл лога и ключевое слово должны задаваться в /etc/sysconfig).
2. Из репозитория epel установить spawn-fcgi и переписать init-скрипт на unit-файл (имя service должно называться так же: spawn-fcgi).
3. Дополнить unit-файл httpd (он же apache) возможностью запустить несколько инстансов сервера с разными конфигурационными файлами. 

*4. Скачать демо-версию Atlassian Jira и переписать основной скрипт запуска на unit-файл.

## 1. **Создание сервиса и unit-файла для него**

Создаем конфигурационный файл для сервиса:

[root@systemd ~] cat > /etc/sysconfig/watchlog
```
# Configuration file for my watchlog service
# Place it to /etc/sysconfig
# File and word in that file that we will be monit
WORD="ALERT"
LOG=/var/log/watchlog.log
```
Создаю лог-файл на случай, если ниже созданное задание в crontab отработает позже создаваемого сервиса:

[root@systemd ~] > touch /var/log/watchlog.log

Создаю скрипты для пополнения лога:

[root@systemd ~]# cat > /opt/alert.sh
```
#!/bin/bash
/bin/echo `/bin/date "+%b %d %T"` ALERT >> /var/log/watchlog.log
```
[root@systemd ~]# cat /opt/tail.sh
```
#!/bin/bash
/bin/tail /var/log/messages >> /var/log/watchlog.log
```
[root@systemd ~]# chmod +x /opt/tail.sh /opt/alert.sh

Создаю задания в crontab:

[root@systemd ~] crontab -e
```
*/3  *  *  *  * /opt/tail.sh
*/5  *  *  *  * /opt/alert.sh
```
Создаю непосредственно скрипт, который будет выполняться в ходе работы сервиса:

[root@systemd ~] cat > /opt/watchlog.sh
```
#!/bin/bash
WORD=$1
LOG=$2
DATE=`/bin/date`
if grep $WORD $LOG &> /dev/null; then
    logger "$DATE: I found word, Master!"
	exit 0
else
    exit 0
fi
```
Добавляю скрипту бит исполнения:

[root@systemd ~] chmod +x /opt/watchlog.sh

Формирую unit-файл сервиса:

[root@systemd ~] cat > /etc/systemd/system/watchlog.service
```
[Unit]
Description=My watchlog service

[Service]
Type=oneshot
EnvironmentFile=/etc/sysconfig/watchlog
ExecStart=/opt/watchlog.sh $WORD $LOG
```
Формирую unit-файл таймера:

[root@systemd ~] cat > /etc/systemd/system/watchlog.timer
```
[Unit]
Description=Run watchlog script every 30 second

[Timer]
# Run every 30 second
OnUnitActiveSec=30
Unit=watchlog.service

[Install]
WantedBy=multi-user.target
```
Обновляю информацию о юнитах:
```
[root@systemd ~] systemctl daemon-reload
```
Запускаю юнит:
```
[root@systemd ~] systemctl start watchlog.timer
[root@systemd ~] systemctl enable watchlog.timer
```
Проверяю:

[root@systemd ~]# tail -f /var/log/messages
```
Jun  6 08:47:08 localhost systemd: Started My watchlog service.
Jun  6 08:47:08 localhost systemd: Stopped Run watchlog script every 30 second.
Jun  6 08:47:09 localhost systemd: Started Run watchlog script every 30 second.
```
## **2. Из epel установить spawn-fcgi и переписать init-скрипт на unit-файл. Имя сервиса должно так же называться:**

Устанавливаем
```
yum install epel-release -y && yum install spawn-fcgi php php-cli mod_fcgid httpd -y
```

Создаем unit файл
```
[root@systemd ~]# cat > /etc/systemd/system/httpd@first.service
[Unit]
Description=The Apache HTTP Server
After=network.target remote-fs.target nss-lookup.target
Documentation=man:httpd(8)
Documentation=man:apachectl(8)

[Service]
Type=notify
EnvironmentFile=/etc/sysconfig/httpd-%I
ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND
ExecReload=/usr/sbin/httpd $OPTIONS -k graceful
ExecStop=/bin/kill -WINCH ${MAINPID}
# We want systemd to give httpd some time to finish gracefully, but still want
# it to kill httpd after TimeoutStopSec if something went wrong during the
# graceful stop. Normally, Systemd sends SIGTERM signal right after the
# ExecStop, which would kill httpd. We are sending useless SIGCONT here to give
# httpd time to finish.
KillSignal=SIGCONT
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```


Создаю второй unit файл:
```
[root@systemd ~]# cat /etc/systemd/system/httpd@second.service
[Unit]
Description=The Apache HTTP Server
After=network.target remote-fs.target nss-lookup.target
Documentation=man:httpd(8)
Documentation=man:apachectl(8)

[Service]
Type=notify
EnvironmentFile=/etc/sysconfig/httpd-%I
ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND
ExecReload=/usr/sbin/httpd $OPTIONS -k graceful
ExecStop=/bin/kill -WINCH ${MAINPID}
# We want systemd to give httpd some time to finish gracefully, but still want
# it to kill httpd after TimeoutStopSec if something went wrong during the
# graceful stop. Normally, Systemd sends SIGTERM signal right after the
# ExecStop, which would kill httpd. We are sending useless SIGCONT here to give
# httpd time to finish.
KillSignal=SIGCONT
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```
В самом файле окружения (которых будет два) задается опция для запуска веб-сервера с необходимым конфигурационным файлом
```
[root@systemd ~]# grep -P "^OPTIONS" /etc/sysconfig/httpd-first
OPTIONS=-f conf/first.conf

[root@systemd ~]# grep -P "^OPTIONS" /etc/sysconfig/httpd-second
OPTIONS=-f conf/second.conf
```
В директории с конфигами httpd должны лежать два конфига, в нашем случае это будут first.conf и second.conf.

Для удачного запуска, в конфигурационных файлах должны бытя указаны уникальные для каждого экземпляра опции Listen и PidFile. Конфиги можно скопировать и поправить только второй, в нем должны быть след опции: 

PidFile /var/run/httpd-second.pid - т.е. должен быть указан файл пида Listen 8080 - указан порт, который будет отличаться от другого инстанса
```
[root@systemd ~]# grep -P "^PidFile|^Listen" /etc/httpd/conf/first.conf
PidFile "/var/run/httpd-first.pid"
Listen 80
[root@systemd ~]# grep -P "^PidFile|^Listen" /etc/httpd/conf/second.conf
PidFile "/var/run/httpd-second.pid"
Listen 8080
```
Поочередно запускаем юниты:
```
[root@hw07-systemd ~]# systemctl start httpd@first
[root@hw07-systemd ~]# systemctl start httpd@second
```
Проверяем результат:
```
[root@ystemd ~]# ss -tunlp | grep httpd
tcp    LISTEN     0      128    [::]:8080               [::]:*                   users:(("httpd",pid=3444,fd=4),("httpd",pid=3443,fd=4),("httpd",pid=3442,fd=4),("httpd",pid=3441,fd=4),("httpd",pid=3440,fd=4),("httpd",pid=3438,fd=4))
tcp    LISTEN     0      128    [::]:80                 [::]:*                   users:(("httpd",pid=3436,fd=4),("httpd",pid=3435,fd=4),("httpd",pid=3434,fd=4),("httpd",pid=3433,fd=4),("httpd",pid=3432,fd=4),("httpd",pid=3431,fd=4))
```
