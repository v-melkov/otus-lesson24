## Стенд для домашнего задания "Резервное копирование"

Настраиваем бэкапы  
Настроить стенд Vagrant с двумя виртуальными машинами: _backup_server_ и _client_

Настроить удаленный бекап каталога `/etc` c сервера client при помощи borgbackup. Резервные копии должны соответствовать следующим критериям:

-   Директория для резервных копий `/var/backup`. Это должна быть отдельная точка монтирования. В данном случае для демонстрации размер не принципиален, достаточно будет и 2GB.
-   Репозиторий для резервных копий должен быть зашифрован ключом или паролем - на ваше усмотрение
-   Имя бекапа должно содержать информацию о времени снятия бекапа
-   Глубина бекапа должна быть год, хранить можно по последней копии на конец месяца, кроме последних трех. Последние три месяца должны содержать копии на каждый день. Т.е. должна быть правильно настроена политика удаления старых бэкапов

-   Резервная копия снимается каждые 5 минут. Такой частый запуск в целях демонстрации.
-   Написан скрипт для снятия резервных копий. Скрипт запускается из соответствующей Cron джобы, либо systemd timer-а - на ваше усмотрение.
-   Настроено логирование процесса бекапа. Для упрощения можно весь вывод перенаправлять в logger с соответствующим тегом. Если настроите не в syslog, то обязательна ротация логов

Запустите стенд на 30 минут. Убедитесь что резервные копии снимаются. Остановите бэкап, удалите (или переместите) директорию /etc и восстановите ее из бекапа.  
Для сдачи домашнего задания ожидаем настроенные стенд, логи процесса бэкапа и описание процесса восстановления.

* * *

Готовый стенд запускается командой `vagrant up`  
Для упрощения автоматической установки репозиторий в Vagrant создается без шифрования  

Далее описана установка вручную

* * *

Все команды выполняем из учетки root  
Установим borgbackup на обе машины:  
`yum install -y epel-release`  
`yum install -y borgbackup`  

На сервере создаем пользователя borg:  
`useradd -m borg`  
`echo password | passwd borg --stdin`  

Разрешаем вход ssh по паролю:  
`sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config`  
`systemctl restart sshd`

Монтируем директорию /var/backup:  
`mkfs.xfs -q /dev/sdb`  
`mkdir -p /var/backup`  
`mount /dev/sdb /var/backup`  
`chown -R borg:borg /var/backup`  

На клиенте настраиваем вход по ssh-ключам и копируем ключ на сервер:  
`ssh-keygen -b 2048 -t rsa -q -N '' -f ~/.ssh/id_rsa`  
`ssh-copy-id borg@192.168.11.101`

Инициализируем с клиента репозиторий для бэкапов:  
`borg init -e repokey borg@192.168.11.101:/var/backup/repo`  

Создадим первый бэкап вручную для проверки:  
`borg create ssh://borg@192.168.11.101:/var/backup/repo::"FirstBackup-{now:%Y-%m-%d_%H:%M:%S}" /etc`  

Создадим скрипт для автоматического создания резервных копий  
`cat /root/backup.sh`  

    #!/bin/bash
    export BORG_RSH="ssh -i /root/.ssh/id_rsa"
    export BORG_REPO=ssh://borg@192.168.11.101/var/backup/repo
    export BORG_PASSPHRASE='password'
    LOG="/var/log/borg_backup.log"
    [ -f "$LOG" ] || touch "$LOG"
    exec &> >(tee -i ""$LOG")
    exec 2>&1
    echo "Starting backup"
    borg create --verbose --list --stats ::'{now:%Y-%m-%d_%H:%M:%S}' /etc                            

    echo "Pruning repository"
    borg prune --list --keep-daily 90 --keep-monthly 12

Добавим ротацию лога /var/log/borg_backup.log:  
`cat /etc/logrotate.d/borg_backup.conf`  

    /var/log/borg_backup.log {
      rotate 5
      missingok
      notifempty
      compress
      size 1M
      daily
      create 0644 root root
      postrotate
        service rsyslog restart > /dev/null
      endscript
    }

Добавим задачу в cron:  
`crontab -l | { cat; echo "*/5 * * * * /root/backup.sh"; } | crontab -`  

Запустим мониторинг логфайла:  
`tail -f /var/log/borg_backup.log`

    tail: /var/log/borg_backup.log: file truncated
    Creating archive at "ssh://borg@192.168.11.101/var/backup/repo::2020-10-04_14:00:01"
    ------------------------------------------------------------------------------
    Archive name: 2020-10-04_14:00:01
    Archive fingerprint: 1162d2827022e1ceb54b16cc918f6ea1e2ed9cc822f5768cc35c2f7b126256c2
    Time (start): Sun, 2020-10-04 14:00:03
    Time (end):   Sun, 2020-10-04 14:00:04
    Duration: 0.96 seconds
    Number of files: 1699
    Utilization of max. archive size: 0%
    ------------------------------------------------------------------------------
                          Original size      Compressed size    Deduplicated size
    This archive:               28.43 MB             13.43 MB                682 B
    All archives:               56.87 MB             26.86 MB             11.80 MB

                          Unique chunks         Total chunks
    Chunk index:                    1283                 3396
    ------------------------------------------------------------------------------
    Pruning repository
    Keeping archive: 2020-10-04_14:00:01                  Sun, 2020-10-04 14:00:03 [1162d2827022e1ceb54b16cc918f6ea1e2ed9cc822f5768cc35c2f7b126256c2]
    Pruning archive: 2020-10-04_13:55:01                  Sun, 2020-10-04 13:55:03 [75cce128f203807d31b0f353178729cfd9a14f54a8742d4ecefb560623d3a4b2] (1/1)

Бэкапы успешно создаются.

Проверим восстановление из бэкапа:  
`mkdir borg_restore`  
`cd borg_restore`  

Узнаем имя последнего архива:  
`borg list ssh://borg@192.168.11.101/var/backup/repo` (в моем случае это 2020-10-04_14:10:02)  

Восстановим весь архив:  
`borg extract ssh://borg@192.168.11.101/var/backup/repo::2020-10-04_14:10:02`

Задание выполнено.

* * *

### Спасибо за проверку!
