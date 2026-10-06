# home-backup 1.0

Ежедневный инкрементный бэкап `/home` для Ubuntu 26.04 (ext4):
локальные снапшоты на жёстких ссылках (`rsync --link-dest`) + копия на ficus.

Каждый снапшот выглядит как полная копия `/home`, а на диске занимает место только под изменившиеся файлы.

## Файлы

| Файл | Куда ставится | Назначение |
|---|---|---|
| `home-backup` | `/usr/local/sbin/` | скрипт |
| `home-backup.conf` | `/etc/home-backup/` | настройки |
| `exclude.list` | `/etc/home-backup/` | исключения |
| `home-backup.service` | `/etc/systemd/system/` | сервис (oneshot) |
| `home-backup.timer` | `/etc/systemd/system/` | расписание: каждый день в 02:30 |
| `install.sh` | — | установщик |

## Установка

```bash
sudo sh install.sh
sudo nano /etc/home-backup/home-backup.conf
sudo home-backup run                            # первый запуск — полная копия, может идти долго
sudo systemctl enable --now home-backup.timer
systemctl list-timers home-backup.timer
```

Повторный запуск `install.sh` обновляет скрипт и юниты, но не трогает существующие конфиги.

## Основные настройки

```bash
SOURCES=("/home")              # можно несколько: ("/home" "/root")
DEST="/backup/home"            # лучше отдельный диск, НЕ внутри /home
REQUIRE_MOUNTPOINT="/backup"   # не запускаться, если диск не смонтирован
KEEP_DAILY=7  KEEP_WEEKLY=4  KEEP_MONTHLY=6
MAIL_TO="kvg@ficus"
```

Скрипт откажется работать, если `DEST` лежит внутри источника. Если `DEST` на той же файловой системе, что и `/home`, он выдаст предупреждение.

## Копия на ficus

Копия уходит после каждого снапшота: `rsync -aAXH --delete` всего каталога `DEST`, с сохранением жёстких ссылок.

Владельцы файлов на ficus сохраняются без прав root: благодаря `--fake-super` права и владельцы пишутся в xattr.

**На ficus** (один раз):

```bash
sudo zfs create zroot/BACKUP/$(имя_хоста)_home       # или любой каталог, доступный kvg
sudo zfs set xattr=sa zroot/BACKUP                   # быстрые xattr, нужны для fake-super
```

**На этой машине** (от root):

```bash
sudo ssh-keygen -t ed25519 -N '' -f /root/.ssh/id_home_backup
sudo ssh-copy-id -i /root/.ssh/id_home_backup.pub kvg@ficus
sudo ssh -i /root/.ssh/id_home_backup kvg@ficus true   # обязательно: добавит ficus в known_hosts
```
**Пример на машине gen30:** (от root):

```bash
sudo ssh-keygen -t ed25519 -N '' -C "home-backup@gen30" -f /root/.ssh/id_home_backup
sudo ssh-copy-id -i /root/.ssh/id_home_backup.pub kvg@ficus
sudo ssh -i /root/.ssh/id_home_backup kvg@ficus 'echo OK; ls -ld /zroot/BACKUP'
```

Последний шаг нужен, потому что сервис видит `/root` только для чтения (`ProtectHome=read-only`) и сам записать known_hosts не сможет.

Затем в конфиге:

```bash
REMOTE_ENABLE="yes"
REMOTE_TARGET="kvg@ficus:/zroot/BACKUP/$(hostname -s)_home"
```

Проверка: `sudo home-backup push`.

Учтите, что `/zroot/BACKUP` на ficus ночью разъезжается по gen*, поэтому копия `/home` попадёт и туда.

## Команды

```bash
sudo home-backup run       # снапшот + чистка + отправка
sudo home-backup list      # снапшоты и размер каждого
sudo home-backup prune     # только чистка
sudo home-backup push      # только отправка на ficus
sudo home-backup config    # действующие настройки
journalctl -u home-backup -n 100    # лог последних запусков
systemctl status home-backup        # результат последнего запуска
```

## Структура

```
/backup/home/
├── 2026-10-04_023012/home/kvg/...
├── 2026-10-05_023244/home/kvg/...
├── 2026-10-06_022951/home/kvg/...
└── latest -> 2026-10-06_022951
```

## Восстановление

**Файл или каталог из локального снапшота:**

```bash
ls /backup/home/
sudo cp -a /backup/home/2026-10-05_023244/home/kvg/docs/report.odt /home/kvg/docs/
sudo rsync -aAXH /backup/home/latest/home/kvg/docs/ /home/kvg/docs/   # каталог целиком
```

**Весь /home** (существующие файлы перезаписываются, лишние удаляются):

```bash
sudo rsync -aAXH --numeric-ids --delete /backup/home/latest/home/ /home/
```

**С ficus.** `--fake-super` нужен и при восстановлении, иначе владельцы вернутся не те:

```bash
sudo rsync -aAXH --numeric-ids -e "ssh -i /root/.ssh/id_home_backup" \
  --rsync-path="rsync --fake-super" \
  kvg@ficus:/zroot/BACKUP/HOST_home/latest/home/kvg/ /home/kvg/
```

## Хранение

- **Ежедневные.** Самый свежий снапшот за каждый из последних `KEEP_DAILY` дней.
- **Еженедельные.** Самый свежий за каждую из последних `KEEP_WEEKLY` недель.
- **Ежемесячные.** Самый свежий за каждый из последних `KEEP_MONTHLY` месяцев.
- **Последний снапшот** не удаляется никогда.

Если запустить бэкап за день несколько раз, сохранится только последний снапшот за этот день.

## Поведение при сбоях

- **rsync упал.** Снапшот не сохраняется, а данные остаются в `DEST/.inprogress`. Следующий запуск продолжит с того же места.
- **Код rsync 24** (файлы исчезли во время копирования) считается нормой.
- **Повторный запуск.** Второй экземпляр, запущенный параллельно, не стартует: работает блокировка `flock`.
- **Почта.** Письмо с полным логом приходит после каждого запуска (`MAIL_ON=always`) или только при ошибке (`failure`).
