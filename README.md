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

[root@systemd ~] systemctl daemon-reload
Запускаю юнит:

[root@systemd ~] systemctl start watchlog.timer
[root@systemd ~] systemctl enable watchlog.timer
Проверяю:
