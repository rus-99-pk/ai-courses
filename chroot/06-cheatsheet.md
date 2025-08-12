# Модуль 6: Шпаргалка по chroot

Здесь собраны все основные команды из курса для быстрого доступа.

### 1. Создание окружения

```bash
# Корневая директория
CHROOT_DIR="/mnt/mychroot"

# Базовая структура
sudo mkdir -p ${CHROOT_DIR}/{bin,lib,lib64,etc,proc,sys,dev,tmp}

# Установка прав на /tmp
sudo chmod 1777 ${CHROOT_DIR}/tmp
```

### 2. Копирование утилит и зависимостей

```bash
# Копирование бинарника (например, bash)
sudo cp /bin/bash ${CHROOT_DIR}/bin/

# Просмотр зависимостей
ldd /bin/bash

# Скрипт-методичка для автоматического копирования зависимостей
# Использование: ./copy_deps.sh /mnt/mychroot /bin/bash /bin/ls
TARGET_DIR=$1
shift
for BIN in "$@"; do
    # Копируем сам бинарник
    sudo cp "$BIN" "${TARGET_DIR}${BIN}"
    # Копируем его зависимости
    DEPS=$(ldd "$BIN" | grep "=>" | awk '{print $3}')
    for DEP in $DEPS; do
        sudo cp "$DEP" "${TARGET_DIR}$(dirname $DEP)/"
    done
done
# Не забудьте скопировать загрузчик ld-linux!
# sudo cp /lib64/ld-linux-x86-64.so.2 ${CHROOT_DIR}/lib64/
```

### 3. Копирование базовых конфигов

```bash
sudo cp /etc/resolv.conf ${CHROOT_DIR}/etc/
sudo cp /etc/passwd ${CHROOT_DIR}/etc/
sudo cp /etc/group ${CHROOT_DIR}/etc/
```

### 4. Монтирование и размонтирование

```bash
# Монтирование
sudo mount --bind /proc ${CHROOT_DIR}/proc
sudo mount --bind /sys ${CHROOT_DIR}/sys
sudo mount --bind /dev ${CHROOT_DIR}/dev
sudo mount --bind /dev/pts ${CHROOT_DIR}/dev/pts

# Размонтирование (в обратном порядке или рекурсивно)
sudo umount ${CHROOT_DIR}/dev/pts
sudo umount ${CHROOT_DIR}/dev
sudo umount ${CHROOT_DIR}/sys
sudo umount ${CHROOT_DIR}/proc
# или
sudo umount -R ${CHROOT_DIR}
```

### 5. Вход и выход

```bash
# Вход с оболочкой bash
sudo chroot ${CHROOT_DIR} /bin/bash

# Вход и выполнение одной команды
sudo chroot ${CHROOT_DIR} ls -l /

# Выход
exit
```

### 6. Практические примеры

#### Восстановление GRUB
```bash
# После загрузки с LiveUSB
sudo mount /dev/sdXN /mnt
sudo mount /dev/sdXY /mnt/boot/efi  # Если UEFI
sudo mount --bind /dev /mnt/dev
sudo mount --bind /proc /mnt/proc
sudo mount --bind /sys /mnt/sys
sudo chroot /mnt
# --> Внутри chroot <--
grub-install /dev/sdX
update-grub
exit
# --> Снаружи <--
sudo umount -R /mnt
```

#### SFTP-тюрьма (в `/etc/ssh/sshd_config`)
```apacheconf
Match User username
    ChrootDirectory %h
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
```
И помните про права на `ChrootDirectory` (владелец `root`, права `755`).

---
**Поздравляем! Вы завершили курс "Мастерская Chroot".**