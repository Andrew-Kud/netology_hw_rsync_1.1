# Домашнее задание к занятию "`Резервное копирование`" - `Кудряшов Андрей`


### Инструкция по выполнению домашнего задания

   1. Сделайте `fork` данного репозитория к себе в Github и переименуйте его по названию или номеру занятия, например, https://github.com/имя-вашего-репозитория/git-hw или  https://github.com/имя-вашего-репозитория/7-1-ansible-hw).
   2. Выполните клонирование данного репозитория к себе на ПК с помощью команды `git clone`.
   3. Выполните домашнее задание и заполните у себя локально этот файл README.md:
      - впишите вверху название занятия и вашу фамилию и имя
      - в каждом задании добавьте решение в требуемом виде (текст/код/скриншоты/ссылка)
      - для корректного добавления скриншотов воспользуйтесь [инструкцией "Как вставить скриншот в шаблон с решением](https://github.com/netology-code/sys-pattern-homework/blob/main/screen-instruction.md)
      - при оформлении используйте возможности языка разметки md (коротко об этом можно посмотреть в [инструкции  по MarkDown](https://github.com/netology-code/sys-pattern-homework/blob/main/md-instruction.md))
   4. После завершения работы над домашним заданием сделайте коммит (`git commit -m "comment"`) и отправьте его на Github (`git push origin`);
   5. Для проверки домашнего задания преподавателем в личном кабинете прикрепите и отправьте ссылку на решение в виде md-файла в вашем Github.
   6. Любые вопросы по выполнению заданий спрашивайте в чате учебной группы и/или в разделе “Вопросы по заданию” в личном кабинете.
   
Желаем успехов в выполнении домашнего задания!
   
### Дополнительные материалы, которые могут быть полезны для выполнения задания

1. [Руководство по оформлению Markdown файлов](https://gist.github.com/Jekins/2bf2d0638163f1294637#Code)

---

### Задание 1

<img width="1271" height="1271" alt="1" src="https://github.com/user-attachments/assets/0eb35719-1743-432d-91b0-b47ec434cc0c" />



---

### Задание 2

<img width="1279" height="1400" alt="2" src="https://github.com/user-attachments/assets/e7c95cff-7cb7-4565-9ded-8f4dedfa3559" />

<img width="496" height="457" alt="3" src="https://github.com/user-attachments/assets/5c2e1f1f-4693-4826-9877-a45ca87a0cd6" />


---

### Задание 3

<img width="2553" height="603" alt="4" src="https://github.com/user-attachments/assets/b8cbe2b6-f9d9-4ac1-9b49-b21ee246077d" />

<img width="2560" height="552" alt="5" src="https://github.com/user-attachments/assets/bcd6262d-2a70-4d4b-a241-d93987b09ba9" />


---

### Задание 4

Скрипт и скриншот работы скрипта на бэкап. Ограничение в 10 мбит, логи, 5 штук бэкапов, синхронизация, проверка кэша и.т.д:

```
#!/bin/bash

USER="kudryashov"
REMOTE_USER="kudryashov"

LOCAL_BASE="/home/$USER"
REMOTE_BASE="/home/$REMOTE_USER/backup"

LOG_FILE="/var/log/backup_rsync.log"
TIMESTAMP=$(date '+%Y-%m-%d_%H-%M-%S')

BACKUP_DIR="$REMOTE_BASE/$TIMESTAMP"
REMOTE_HOST="192.168.10.209"

#килобайт / 0 - безлимит.
SPEED_LIMIT=0

BACKUPS=5



log() {
    echo "$TIMESTAMP [INFO]  $1" >> "$LOG_FILE"
}
error() {
    echo "$TIMESTAMP [ERROR] $1" >> "$LOG_FILE"
}



if [ ! -d "$LOCAL_BASE" ]; then
    error "local directory is NOT found: $LOCAL_BASE"
    exit 1
fi



if ! ssh "$REMOTE_USER@$REMOTE_HOST" "[ -d '$REMOTE_BASE' ] || mkdir -p '$REMOTE_BASE'"; then
    error "failed to connect: $REMOTE_HOST or create directory: $REMOTE_BASE"
    exit 1
fi



LATEST=$(ssh "$REMOTE_USER@$REMOTE_HOST" "cd '$REMOTE_BASE' 2>/dev/null && \
    find . -maxdepth 1 -type d ! -name '.' ! -name '..' -printf '%f\n' 2>/dev/null | \
    sort -r | head -1")



if [ -n "$LATEST" ]; then
    LATEST="$REMOTE_BASE/$LATEST"
    log "The previous backup will be used for: $LATEST"
else
    log "Previous backup not found. This backup will be full."
fi



RSYNC_CMD="rsync"
if [ -n "$LATEST" ]; then
    RSYNC_CMD="rsync --link-dest='$LATEST'"
fi



LIMIT=""
if [ "$SPEED_LIMIT" -ne 0 ] 2>/dev/null; then
    LIMIT="--bwlimit=$SPEED_LIMIT"
    log "Bandwidth limits included: $SPEED_LIMIT KB/s (~$((SPEED_LIMIT * 8 / 1000)) Mbps)"
else
    log "No bandwidth restrictions."
fi

if rsync -avc --delete --exclude='.*/' $LIMIT --rsync-path="$RSYNC_CMD" "$LOCAL_BASE/" \
    "$REMOTE_USER@$REMOTE_HOST:$BACKUP_DIR/" >> "$LOG_FILE" 2>&1; then
    log "Backup created successfully: $BACKUP_DIR"

    CLEANUP_LIST=$(ssh "$REMOTE_USER@$REMOTE_HOST" "cd '$REMOTE_BASE' 2>/dev/null && \
    find * -maxdepth 0 -type d 2>/dev/null | sort -r | tail -n +$((BACKUPS + 1))")

    if [ -n "$CLEANUP_LIST" ]; then
        while IFS= read -r dir; do
            [ -z "$dir" ] && continue
            log "Removing old backup: $dir"
            ssh "$REMOTE_USER@$REMOTE_HOST" "rm -rf '$REMOTE_BASE/$dir'"
        done <<< "$CLEANUP_LIST"
    else
        log "No old backups to remove."
    fi
else
    error "rsync failed for backup: $BACKUP_DIR"
    exit 1
fi

```

<img width="2559" height="1439" alt="4" src="https://github.com/user-attachments/assets/1fa25f1b-e29b-4ef3-9930-55323c4c1637" />


Скрипт восстановления:
```
#!/bin/bash

#set -euo pipefail

USER="kudryashov"
REMOTE_USER="kudryashov"

LOCAL_BASE="/home/$USER"
REMOTE_BASE="/home/$REMOTE_USER/backup"

REMOTE_HOST="192.168.10.209"

LOG_FILE="/var/log/backup_rsync.log"
TIMESTAMP=$(date '+%Y-%m-%d_%H-%M-%S')



log() {
    echo "$TIMESTAMP [INFO]  $1" >> "$LOG_FILE"
}
error() {
    echo "$TIMESTAMP [ERROR] $1" >> "$LOG_FILE"
}



if [ ! -d "$LOCAL_BASE" ]; then
    echo "[ERROR] Directory $LOCAL_BASE not exist."
    exit 1
fi



BACKUP_LIST=()
readarray -t BACKUP_LIST < <(ssh "$REMOTE_USER@$REMOTE_HOST" "cd '$REMOTE_BASE' 2>/dev/null || exit 1
    for dir in *; do
        if [ -d \"\$dir\" ]; then
            size=\$(du -sh \"\$dir\" 2>/dev/null | awk '{print \$1}')
            printf '%s|%s\n' \"\$dir\" \"\$size\"
        fi
    done | sort -r")



if [ ${#BACKUP_LIST[@]} -eq 0 ]; then
    echo "[ERROR] On the $REMOTE_HOST in $REMOTE_BASE no backup copies."
    exit 1
fi



BACKUPS=()
SIZES=()



echo "Available backups"
for i in "${!BACKUP_LIST[@]}"; do
    BACKUP_NAME="${BACKUP_LIST[$i]%%|*}"
    BACKUP_SIZE="${BACKUP_LIST[$i]#*|}"
    BACKUPS+=("$BACKUP_NAME")
    SIZES+=("$BACKUP_SIZE")
    printf "  %2d) %-30s — %8s\n" "$i" "$BACKUP_NAME" "$BACKUP_SIZE"
done



echo
read -p "Select file to recover: " SELECTED_NUM

if ! [[ "$SELECTED_NUM" =~ ^[0-9]+$ ]] || [ "$SELECTED_NUM" -ge "${#BACKUPS[@]}" ]; then
    echo "[ERROR] invalid number."
    exit 1
fi

RESTORE_SOURCE="${BACKUPS[$SELECTED_NUM]}"
RESTORE_PATH="$REMOTE_BASE/$RESTORE_SOURCE"
RESTORE_SIZE="${SIZES[$SELECTED_NUM]}"



echo
echo "Selected: $RESTORE_SOURCE — $RESTORE_SIZE"
echo



echo "Select recovery mode:"
echo "  1) Full recovery (will replace all duplicate files)"
echo "  2) Partial recovery (missing files only)"
read -p "Enter 1 or 2: " MODE

case "$MODE" in
    1)
        echo
        read -p "All data in $LOCAL_BASE will be COMPLETELY replaced. Continue? (y/N): " CONFIRM
        if [[ ! "$CONFIRM" =~ ^[Yy]$ ]]; then
            echo "Cancelled."
            exit 0
        fi

        log "Full recovery: $LOCAL_BASE"
        find "$LOCAL_BASE" -mindepth 1 -maxdepth 1 ! -name ".*" -exec rm -rf {} + || {
            error "Failed to recovery $LOCAL_BASE"
            exit 1
        }

        log "Restoring: $RESTORE_PATH (Size: $RESTORE_SIZE)"
        if rsync -av --delete --exclude='.*/' "$REMOTE_USER@$REMOTE_HOST:$RESTORE_PATH/" "$LOCAL_BASE/" >> "$LOG_FILE" 2>&1; then
            log "Full restore successful: $RESTORE_SOURCE"
            echo "Restore complete: $RESTORE_SOURCE (Size: $RESTORE_SIZE)"
        else
            error "rsync failed full restore from $RESTORE_SOURCE"
            echo "[ERROR] Full restore"
            exit 1
        fi
        ;;

    2)
        echo
        read -p "ONLY deleted/missing files will be restored. Continue? (y/N): " CONFIRM
        if [[ ! "$CONFIRM" =~ ^[Yy]$ ]]; then
            echo "Cancelled."
            exit 0
        fi

        log "Partial restore: $RESTORE_PATH (Size: $RESTORE_SIZE)"
        if rsync -av --exclude='.*/' --ignore-existing "$REMOTE_USER@$REMOTE_HOST:$RESTORE_PATH/" "$LOCAL_BASE/" >> "$LOG_FILE" 2>&1; then
            log "Partial restore successful: $RESTORE_SOURCE"
            echo "ONLY missing files restored: $RESTORE_SOURCE (Size: $RESTORE_SIZE)"
        else
            error "rsync failed partial restore from $RESTORE_SOURCE"
            echo "[ERROR] Patrial restore."
            exit 1
        fi
        ;;

    *)
        echo "Invalid selection"
        exit 1
        ;;
esac


```

<img width="2541" height="986" alt="5" src="https://github.com/user-attachments/assets/bf0c4b2a-825f-4b33-a571-02b037c9d096" />

