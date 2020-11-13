## **Задание**

Настроить стенд Vagrant с двумя виртуальными машинами server и backup.

Настроить политику бэкапа директории /etc с клиента (server) на бекап сервер (backup):
1) Бекап делаем раз в час
2) Политика хранения бекапов: храним все за последние 30 дней, и по одному за предыдущие два месяца.
3) Настроить логирование процесса бекапа в /var/log/ - название файла на ваше усмотрение
4) Восстановить из бекапа директорию /etc с помощью опции Borg mount

Результатом должен быть скрипт резервного копирования (политику хранения можно реализовать в нем же), а так же вывод команд терминала записанный с помощью script (или другой подобной утилиты).

* Настроить репозиторий для резервных копий с шифрованием ключом.

## **Выполнение задания**

**Установка borgbackup на обе машины**

Скачиваем бинарник с репозиторрия, добавив права на исполнение:
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
Инициируем репозиторий с опцией --encryption и именем `EtcRepo` на хосте backup:
```
[root@server ~]# borg init --encryption=repokey-blake2 borg@192.168.10.20:EtcRepo
Using a pure-python msgpack! This will result in lower performance.
Do you want your passphrase to be displayed for verification? [yN]: y
Your passphrase (between double-quotes): "pass"
Make sure the passphrase displayed above is exactly what you wanted.

```

Проверим содержимое ключа шифрования.переключимся на юзера borg. Должен быть тут `<REPO_DIR>/config`:
```
[borg@backup ~]$ cat /home/borg/EtcRepo/config
[repository]
version = 1
segments_per_dir = 1000
max_segment_size = 524288000
append_only = 0
storage_quota = 0
additional_free_space = 0
id = 56cd16aaf8b9d44ad318ea861d0eaf1f72a44f4675f8c493ed1d985f0b14d5be
key = hqlhbGdvcml0aG2mc2hhMjU2pGRhdGHaAZ6qI2P8vL9OHJ3Wk4+VdrPMJti2xGF86T5WaX
	vQfnf6gIPc+N4ZwnL9gW5GQVZognBQ9o9XVld9xBY/H9wPJnJ1j2zYiGNMgBVLuS+KC9bx
	a82h7h1zu3DiWIZz73PIijafbqRtRAQ1mcD8L35o+PZSin5KClvZ1q8aoz2NwByaWQzIP8
	FJCnHDC66yQ0qztkLBF/mfg7d0UrqfeqPXevhghYzQNwxLIaniIFFr9ll7l4xB9ES73yZE
	so0ZFlqFN+QPBOWn2Q89jBgvCweGRNqZz9dqkr0pCOnmJzKCgJQr0RXxo11JsYiQORijfe
	yP2oH01kssEBXNj1Izpe2kSgisl++vmB0yVUixpdCW3VnXTY5iEYYvZV5FNZp575BIjQx+
	8CRXwKTLRQUNiv+IV8WC60EsuYtT2j8F6Df7MqKIVdZ9+tgEL8retl4appCrsFpSKiRARK
	SsoiCZw2rrfK3Ne3MxZH9nf0ZNNtO7U2CU6gW69jezzmB8bUbNE+cVQXQLnxzV6APMMsZc
	AXrrE+S0X+GuGMHZ1b3ppo4T/rukaGFzaNoAIJUSu4tVwE2bFpAQlzW/ga+lfSi3G4kl4z
	j1EfsmMSkoqml0ZXJhdGlvbnPOAAGGoKRzYWx02gAgaeGRWW9hW3r8PAjoWw2TxgssETiq
	f1zOymUxagH0qv+ndmVyc2lvbgE=

```

Для того, чтобы passphrase не запрашивалась каждый раз при запуске процедуры бэкапа, в скрипте передадим passphrase в переменную окружения BORG_PASSPHRASE:
`export BORG_PASSPHRASE='pass'`.

**Сбор логов**

Borg пишет свои логи в stderr, поэтому для записи логов в файл, нужно перенаправить в него stderr. В скрипте за это отвечают следующие строки:
```
LOG=/var/log/borg_backup.log

borg create \
  --stats --list --debug --progress \
  ${BACKUP_USER}@${BACKUP_HOST}:${BACKUP_REPO}::"etc-server-{now:%Y-%m-%d_%H:%M:%S}" \
  /etc 2>> ${LOG}

borg prune \
  -v --list \
  ${BACKUP_USER}@${BACKUP_HOST}:${BACKUP_REPO} \
  --keep-daily=7 \
  --keep-weekly=4 2>> ${LOG}
```

**Политика хранения бэкапа**

Политика хранения определяется в скрипте командой `borg prune` - она удаляет из репозитория все архивы, которые не подпадают под критерии, которые, в свою очередь, определяются параметрами `--keep-*`.Наша задача хранить бэкапы за последние 30 дней и по одному за предыдущие два месяца:
```
borg prune \
  -v --list \
  ${BACKUP_USER}@${BACKUP_HOST}:${BACKUP_REPO} \
  --keep-daily=30 \
  --keep-monthly=2 2>> ${LOG}
```

**Автоматическое выполнение бэкапа**

Необходимо бэкап делать каждый час. Можно здайствовать cron, но современнее systemd service и systemd timer.Все вспомогательные файл прокинулись в диреторию vagrant, оттуда их и переместим в необходимые места, например в /etc/systemd/system в данном случае :
```
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
OnBootSec=5min
OnUnitActiveSec=1h
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
-rw-r--r--. 1 root root 0 Nov 13 08:58 testfile01
-rw-r--r--. 1 root root 0 Nov 13 08:58 testfile02
-rw-r--r--. 1 root root 0 Nov 13 08:58 testfile03
-rw-r--r--. 1 root root 0 Nov 13 08:58 testfile04
-rw-r--r--. 1 root root 0 Nov 13 08:58 testfile05

```

Выполним задание бэкапа взяв подготовленный скрипт из /vagrant :
```
[root@server ~]# ./borg-backup.sh
```

Посмотрим, что получилось в репозитории EtcRepo:
```
[root@server ~]# borg list borg@192.168.10.20:EtcRepo
Using a pure-python msgpack! This will result in lower performance.
Remote: Using a pure-python msgpack! This will result in lower performance.
Enter passphrase for key ssh://borg@192.168.10.20/./EtcRepo:
etc-server-2020-11-13_09:05:14       Fri, 2020-11-13 09:05:15 [d278952c0755292c7672358bc38d1e31d5d67f3e6026f54c66605c0f1829595d]
```

Видим пока один архив `etc-server-2020-11-13_09:05:14`, посмотрим его содержимое:
```
[root@server ~]# borg list borg@192.168.10.20:EtcRepo::etc-server-2020-11-13_09:05:14
Using a pure-python msgpack! This will result in lower performance.
Remote: Using a pure-python msgpack! This will result in lower performance.
Enter passphrase for key ssh://borg@192.168.10.20/./EtcRepo:
Enter passphrase for key ssh://borg@192.168.10.20/./EtcRepo:
drwxr-xr-x root   root          0 Fri, 2020-11-13 08:58:55 etc
drwxr-xr-x root   root          0 Fri, 2020-11-13 08:58:55 etc/testdir
-rw-r--r-- root   root          0 Fri, 2020-11-13 08:58:55 etc/testdir/testfile01
-rw-r--r-- root   root          0 Fri, 2020-11-13 08:58:55 etc/testdir/testfile02
-rw-r--r-- root   root          0 Fri, 2020-11-13 08:58:55 etc/testdir/testfile03
-rw-r--r-- root   root          0 Fri, 2020-11-13 08:58:55 etc/testdir/testfile04
-rw-r--r-- root   root          0 Fri, 2020-11-13 08:58:55 etc/testdir/testfile05
-rw------- root   root          0 Thu, 2020-04-30 22:04:55 etc/crypttab

...
```

Удалим тестовую папку `/etc/testdir`:
```
[root@server ~]# rm -rf /etc/testdir/
[root@server ~]# ll /etc/testdir
ls: cannot access /etc/testdir: No such file or directory
```

 Теперь попробуем достать `/etc/testdir` из бэкапа. Для этого создадим директорию `/borgbackup` и примонтируем в неё репозиторий с бэкапом:
```
[root@server ~]# mkdir /borgbackup
[root@server ~]# borg mount borg@192.168.10.20:EtcRepo::etc-server-2020-11-13_09:05:14 /borgbackup/
Using a pure-python msgpack! This will result in lower performance.
Remote: Using a pure-python msgpack! This will result in lower performance.
Enter passphrase for key ssh://borg@192.168.10.20/./EtcRepo:

```

 Посмотрим есть ли `testdir` в `/borgbackup/etc`:
```
[root@server ~]# ll /borgbackup/etc/testdir/
total 0
-rw-r--r--. 1 root root 0 Nov 13 08:58 testfile01
-rw-r--r--. 1 root root 0 Nov 13 08:58 testfile02
-rw-r--r--. 1 root root 0 Nov 13 08:58 testfile03
-rw-r--r--. 1 root root 0 Nov 13 08:58 testfile04
-rw-r--r--. 1 root root 0 Nov 13 08:58 testfile05

```

Вернем `testdir` в `/etc`:
```
[root@server ~]# cp -Rp /borgbackup/etc/testdir/ /etc
[root@server ~]# ll /etc/testdir/
total 0
-rw-r--r--. 1 root root 0 Nov 13 08:58 testfile01
-rw-r--r--. 1 root root 0 Nov 13 08:58 testfile02
-rw-r--r--. 1 root root 0 Nov 13 08:58 testfile03
-rw-r--r--. 1 root root 0 Nov 13 08:58 testfile04
-rw-r--r--. 1 root root 0 Nov 13 08:58 testfile05

```

Отмонтируем репу с бэкапом, раз все успешно восстановилось:
```
[root@server ~]# borg umount /borgbackup/
Using a pure-python msgpack! This will result in lower performance.
[root@server ~]# ll /borgbackup/
total 0
```
