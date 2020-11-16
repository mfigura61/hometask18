## **Задание**

Настроить стенд Vagrant с двумя виртуальными машинами: backup_server и client

 Настроить удаленный бекап каталога /etc c сервера client при помощи borgbackup. Резервные копии должны соответствовать следующим критериям:

 - Директория для резервных копий /var/backup. Это должна быть отдельная точка монтирования. В данном случае для демонстрации размер не принципиален, достаточно будет и 2GB.
 - Репозиторий дле резервных копий должен быть зашифрован ключом или паролем - на ваше усмотрение
 - Имя бекапа должно содержать информацию о времени снятия бекапа
 - Глубина бекапа должна быть год, хранить можно по последней копии на конец месяца, кроме последних трех. Последние три месяца должны содержать копии на каждый день. Т.е. должна быть правильно настроена политика удаления старых бэкапов
 - Резервная копия снимается каждые 5 минут. Такой частый запуск в целях демонстрации.
 - Написан скрипт для снятия резервных копий. Скрипт запускается из соответствующей Cron джобы, либо systemd timer-а - на ваше усмотрение.
 - Настроено логирование процесса бекапа. Для упрощения можно весь вывод перенаправлять в logger с соответствующим тегом. Если настроите не в syslog, то обязательна ротация логов

 Запустите стенд на 30 минут. Убедитесь что резервные копии снимаются. Остановите бекап, удалите (или переместите) директорию /etc и восстановите ее из бекапа. Для сдачи домашнего задания ожидаем настроенные стенд, логи процесса бэкапа и описание процесса восстановления.


## **Выполнение задания**

**Установка borgbackup на обе машины и подготовка**

Предварительно в вагрант файл добавим доп диск sdb, с тем, чтобы потом на нем сделать точку монтирования под бекап.
```
[vagrant@backup ~]$ mkfs.ext4 /dev/sdb
[vagrant@backup ~]$ sudo mkdir /var/backup/
[vagrant@backup ~]$ sudo mount /dev/sdb /var/backup/
[vagrant@backup ~]$ df -h | grep backup
/dev/sdb        9.8G   37M  9.2G   1% /var/backup
[vagrant@backup ~]$

```

Скачиваем бинарник с репозиторрия, добавив права на исполнение.На обеих машинах проделываем:
```
[root@server ~]# curl -L https://github.com/borgbackup/borg/releases/download/1.1.14/borg-linux64 -o /usr/bin/borg && chmod +x /usr/bin/borg
```

Перейдем на хост backup, чтобы  создать пользователя borg:
```

[root@backup ~]# useradd -m borg
```

 Вернемся на хост server, сгенерируем SSH-ключ и добавим его в `~borg/.ssh/authorized_keys` на хосте backup для сквозной авторизации сессий ssh. После чего сменим владельца на borg  :
```
[root@server ~]# ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa):
Created directory '/root/.ssh'.
Enter passphrase (empty for no passphrase):
...
[root@backup ~]# mkdir ~borg/.ssh && echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDKz1BmezN+dZTqgilCerTKjbVhAzcacfYTa/QahK1KFE9zo45hYFOa+OOtnRBST63TKVjcrKT3U98LNZwSq2M/jaiDftN5JfXMKxmsQk9hB6dQO/gjj55v7nxqy1dFqXTBtmq+1DS+OoySfA5hHzurT+WqfsBQ81N8u8LTBidNuVJd+nSPYN4nrCmZM9brRZ0X8zigQqdprtLVL0A+ecSzTaHqBb2EiR2Igfj/5Gg6Q0dm8cF6Bm2srHTppExTk4M4Za3AYrNlyHer+YcQpBYluVzvbU6Uyb0Kk6pOrNQvAjoWI5j5ggbM1e213g3F0e7BSdwmbj5e8rz4k1IHY5od root@server" > ~borg/.ssh/authorized_keys
[root@backup ~]# chown -R borg:borg ~borg/.ssh
```

**Использование опции шифрования**


Borg может шифровать по нескольким алгоритмам, возьмем BLAKE2. При запросе passphrase введем "pass".
Инициируем репозиторий с опцией --encryption и именем `EtcRepo` на хосте backup, задав пароль pass . Но сначала выдадим права на каталог с бакапами юзеру borg. Просто 777, ибо это тестовая среда, так делать нельзя и тд итп..  :

```
[root@backup ~]# chmod -R 777 /var/backup/
```

```
[root@server ~]# borg init --encryption=repokey-blake2 borg@192.168.10.20:/var/backup/EtcRepo
Using a pure-python msgpack! This will result in lower performance.
Remote: Using a pure-python msgpack! This will result in lower performance.
Enter new passphrase:
Enter same passphrase again:
....
Your passphrase (between double-quotes): "pass"
Make sure the passphrase displayed above is exactly what you wanted.

```

Проверим содержимое ключа шифрования.переключимся на юзера borg. Должен быть тут `<REPO_DIR>/config`:
```
[borg@backup ~]$ cat /var/backup/EtcRepo/config
[repository]
version = 1
segments_per_dir = 1000
max_segment_size = 524288000
append_only = 0
storage_quota = 0
additional_free_space = 0
id = 8bc509743530b43789f1af75bfe886907fd61af23b1c19fc346e522a61126888
key = hqlhbGdvcml0aG2mc2hhMjU2pGRhdGHaAZ5jQ1vSjwF/JXOKK2Gfu0PPT+rIN8zCj3THc9
	AtI7KUSw2oEcyBwPjJB5vU3sil8LVldnAbHZ+qiy5ErDgoTvRZLF0YUNwgB9JLGCAHWNm5
	FkUs52OSQQP3k7R0jDb1gd/x8jw59gbPcSS7jZWZ/PTI+wFf9N7uwVIwvNA1sofpvbSm24
	0lsqyjuyZ/OLRZ9ys0/xExTb2QcpymVAP6Uy2QvhrTsyDiJGgyAs7Gpm6vZD/Wmb0CLVRo
	JyOUeQ42S3meA08x4a5oFVu0cgcGW9IERBqxGoOJJYiMa0oJCt75zSF7h1DngGeYPSLo51
	xbLpLzCYjLTpBO3bamuwO7bg/3cgniH79cDDdgQWAvGrfgymMqPSAMaGfU77B59euwXlx0
	SQo8CyK3eCyL2dO41WDkMzZ7HaYAXctXnyknhdNIe5JzwWDwkbQ8A+1ThVS+D1a1IMIE6N
	MSgYSudfEtrJJN396ta1BoE0m0boPbud9utrVjxDfFmXp21ZD2Jvb2jwZ8XcJPIvxZNiDu
	R1RVbZmJjwMPLC8cOg4EW6Y4o9qkaGFzaNoAIOWdYbPdFVizvyYMpDWX1evk9tnptR7cVL
	rtibK26HtOqml0ZXJhdGlvbnPOAAGGoKRzYWx02gAg8FpH2pB/pYsDK2GHoucoPpqvJPlj
	yfto5NDtqVHmTjqndmVyc2lvbgE=

```

Для того, чтобы passphrase не запрашивалась каждый раз при запуске процедуры бэкапа, в скрипте полезно передавать passphrase в переменную окружения BORG_PASSPHRASE:
`export BORG_PASSPHRASE='pass'`.

**Сбор логов**

Borg пишет свои логи в stderr, поэтому для записи логов в файл, нужно перенаправить в него stderr. В скрипте за это отвечают следующие строки:
```
LOG=/var/log/borg_backup.log

borg create \
  --stats --list --debug --progress \
  ${BACKUP_USER}@${BACKUP_HOST}:${BACKUP_REPO}::"etc-server-{now:%Y-%m-%d_%H:%M:%S}" \
  /etc 2>> ${LOG}


```

**Политика хранения бэкапа**

Политика хранения определяется в скрипте командой `borg prune` - она удаляет из репозитория все архивы, которые не подпадают под критерии, которые, в свою очередь, определяются параметрами `--keep-*`.Наша задача хранить бэкапы за последние 12 месяцев, причем последние 3 месяца(93 дня) ежедневные:
```
borg prune \
  -v --list \
  ${BACKUP_USER}@${BACKUP_HOST}:${BACKUP_REPO} \
  --keep-daily=93 \
  --keep-monthly=12 2>> ${LOG}
```

**Автоматическое выполнение бэкапа**

Необходимо бэкап делать каждые 5 минут. Можно задействовать cron, но современнее systemd service и systemd timer.Скопируем подтянувшиеся в виртуалки файлики-заготовки:
```
[root@server ~]# cp /vagrant/borg-backup.service /etc/systemd/system/
[root@server ~]# cp /vagrant/borg-backup.timer /etc/systemd/system/ && cp /vagrant/borg-backup.sh /root/

[root@server ~]# cat /etc/systemd/system/borg-backup.service
[Unit]
Description=Borg /etc backup
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
ExecStart=/root/borg-backup.sh
```
```
[root@server ~]# cat /etc/systemd/system/borg-backup.timer
[Unit]
Description=Borg backup timer

[Timer]
#run hourly
OnBootSec=3min
OnUnitActiveSec=5min
Unit=borg-backup.service

[Install]
WantedBy=multi-user.target
```

Обновим конфигурацию systemd и запустим таймер:
```
[root@server ~]# systemctl daemon-reload
[root@server ~]# systemctl enable --now borg-backup.timer
Created symlink from /etc/systemd/system/multi-user.target.wants/borg-backup.timer to /etc/systemd/system/borg-backup.timer.

```

**Работа с архивом**

Создадим `/etc/testdir` с файлами внутри:
```
[root@server ~]# mkdir /etc/testdir && touch /etc/testdir/testfile{01..05}
[root@server ~]# ll /etc/testdir/
total 0
-rw-r--r--. 1 root root 0 Nov 16 14:35 testfile01
-rw-r--r--. 1 root root 0 Nov 16 14:35 testfile02
-rw-r--r--. 1 root root 0 Nov 16 14:35 testfile03
-rw-r--r--. 1 root root 0 Nov 16 14:35 testfile04
-rw-r--r--. 1 root root 0 Nov 16 14:35 testfile05

```

Выполним задание бэкапа вручную:
```
[root@server ~]# ./borg-backup.sh
```

Посмотрим, что получилось в репозитории EtcRepo:
```
[root@server ~]# borg list borg@192.168.10.20:/var/backup/EtcRepo
Using a pure-python msgpack! This will result in lower performance.
Remote: Using a pure-python msgpack! This will result in lower performance.
Enter passphrase for key ssh://borg@192.168.10.20/var/backup/EtcRepo:
etc-server-2020-11-16_15:00:49       Mon, 2020-11-16 15:00:50 [fd90e6fad474c272f276d1bc8ac904e51e8122422839fd84fcb7e92ab3d013b7]

```

Видим пока один архив `etc-server-2020-11-16_15:00:49`, посмотрим его содержимое,пока временно закомментив блок prune в скрипте, чтобы немного накопилось бекапов через 5 минут:
```
[root@server ~]# borg list borg@192.168.10.20:/var/backup/EtcRepo::etc-server-2020-11-16_15:00:49
Using a pure-python msgpack! This will result in lower performance.
Remote: Using a pure-python msgpack! This will result in lower performance.
Enter passphrase for key ssh://borg@192.168.10.20/var/backup/EtcRepo:
drwxr-xr-x root   root          0 Mon, 2020-11-16 14:51:16 etc
-rw-r--r-- root   root        450 Mon, 2020-11-16 13:51:04 etc/fstab
-rw------- root   root          0 Thu, 2020-04-30 22:04:55 etc/crypttab
lrwxrwxrwx root   root         17 Thu, 2020-04-30 22:04:55 etc/mtab -> /proc/self/mounts
-rw-r--r-- root   root          7 Sun, 2020-11-15 14:59:47 etc/hostname
-rw-r--r-- root   root       2388 Thu, 2020-04-30 22:08:36 etc/libuser.conf

...
```

Удалим тестовую папку `/etc/testdir`:
```
[root@server ~]# rm -rf /etc/testdir/
[root@server ~]# ll /etc/testdir
ls: cannot access /etc/testdir: No such file or directory
```

 Теперь попробуем достать `/etc/testdir` из бэкапа. Для этого создадим директорию `/mnt/borgbackup` и примонтируем в неё репозиторий с бэкапом:
```
[root@server ~]# mkdir /mnt/borgbackup/
[root@server ~]# borg mount  borg@192.168.10.20:/var/backup/EtcRepo::etc-server-2020-11-16_15:00:49 /mnt/borgbackup/
Using a pure-python msgpack! This will result in lower performance.
Remote: Using a pure-python msgpack! This will result in lower performance.
Enter passphrase for key ssh://borg@192.168.10.20/var/backup/EtcRepo:

```

 Посмотрим есть ли `testdir` в `/mnt/borgbackup/etc`:
```
[root@server ~]# ll /mnt/borgbackup/etc/testdir/
total 0
-rw-r--r--. 1 root root 0 Nov 16 14:35 testfile01
-rw-r--r--. 1 root root 0 Nov 16 14:35 testfile02
-rw-r--r--. 1 root root 0 Nov 16 14:35 testfile03
-rw-r--r--. 1 root root 0 Nov 16 14:35 testfile04
-rw-r--r--. 1 root root 0 Nov 16 14:35 testfile05

```

Вернем `testdir` в `/etc`:
```
[root@server ~]# cp -Rp /borgbackup/etc/testdir/ /etc
ll /etc/testdir/
total 0
-rw-r--r--. 1 root root 0 Nov 16 14:35 testfile01
-rw-r--r--. 1 root root 0 Nov 16 14:35 testfile02
-rw-r--r--. 1 root root 0 Nov 16 14:35 testfile03
-rw-r--r--. 1 root root 0 Nov 16 14:35 testfile04
-rw-r--r--. 1 root root 0 Nov 16 14:35 testfile05
[root@server ~]#

```

Отмонтируем репу с бэкапом, раз все успешно восстановилось:
```
[root@server ~]# borg umount /mnt/borgbackup/
Using a pure-python msgpack! This will result in lower performance.
[root@server ~]# ll /borgbackup/
total 0
```
Расскомментим обратно блок prune, убедившись, что бекапы создаются
```
borg list borg@192.168.10.20:/var/backup/EtcRepo
Using a pure-python msgpack! This will result in lower performance.
Remote: Using a pure-python msgpack! This will result in lower performance.
Enter passphrase for key ssh://borg@192.168.10.20/var/backup/EtcRepo:
etc-server-2020-11-16_15:00:49       Mon, 2020-11-16 15:00:50 [fd90e6fad474c272f276d1bc8ac904e51e8122422839fd84fcb7e92ab3d013b7]
etc-server-2020-11-16_15:06:49       Mon, 2020-11-16 15:06:50 [1d84f9bc9f79e48d15de0161eaaf9e7915295e6c5cba90264759b5f689796b7b]
```
