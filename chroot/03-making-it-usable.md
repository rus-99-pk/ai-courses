# Модуль 3: Делаем окружение рабочим

Минимальное окружение из прошлого модуля не очень полезно. Многие программы ожидают найти специальные файловые системы, такие как `/proc`, `/sys` и `/dev`, для получения информации о системе и устройствах.

### Зачем нужны `/proc`, `/sys` и `/dev`?

*   `/proc`: Виртуальная файловая система, предоставляющая информацию о процессах и ядре. Команда `ps` берет данные отсюда.
*   `/sys`: Виртуальная файловая система для взаимодействия с драйверами и устройствами.
*   `/dev`: Содержит файлы устройств. Самые важные:
    *   `/dev/null`: "черная дыра".
    *   `/dev/zero`: источник нулевых байтов.
    *   `/dev/random`, `/dev/urandom`: источники случайных данных.
    *   `/dev/pts`, `/dev/tty`: для терминалов.

Мы не будем их *копировать*. Вместо этого мы их *примонтируем* из основной системы в наше chroot-окружение.

### Шаг 1: Создание точек монтирования

Сначала создадим пустые каталоги внутри chroot, куда мы будем монтировать.

```bash
# Выполнять в основной системе, не в chroot!
sudo mkdir -p /mnt/mychroot/{proc,sys,dev/pts}
```

### Шаг 2: Монтирование с опцией `--bind`

Опция `--bind` создает "зеркало" каталога в другом месте. Изменения в одном отражаются в другом.

```bash
# Монтируем виртуальные ФС
sudo mount --bind /proc /mnt/mychroot/proc
sudo mount --bind /sys /mnt/mychroot/sys
sudo mount --bind /dev /mnt/mychroot/dev
sudo mount --bind /dev/pts /mnt/mychroot/dev/pts # Для корректной работы псевдо-терминалов
```

### Шаг 3: Вход и проверка

Теперь снова войдем в chroot:

```bash
sudo chroot /mnt/mychroot /bin/bash
```

Внутри chroot попробуйте выполнить команды, которые раньше не работали (возможно, придется сначала скопировать бинарники `ps`, `ping` и их зависимости!):

```bash
# Внутри chroot
# Если вы скопировали /bin/ps и его зависимости:
ps aux
# Вы должны увидеть процессы системы
```

### Шаг 4: Размонтирование (Очень важно!)

После завершения работы в chroot **обязательно** нужно размонтировать эти файловые системы. Если этого не сделать, вы не сможете, например, удалить каталог `/mnt/mychroot`.

```bash
# Выполнять в основной системе после выхода из chroot
sudo umount /mnt/mychroot/dev/pts
sudo umount /mnt/mychroot/dev
sudo umount /mnt/mychroot/sys
sudo umount /mnt/mychroot/proc
```
> **Совет**: `sudo umount -R /mnt/mychroot` может рекурсивно размонтировать все точки внутри, но будьте осторожны с этой командой.

### Методичка: Скрипт для автоматизации

Чтобы не вводить все это руками каждый раз, можно написать небольшой скрипт.

**Файл `enter-chroot.sh`**:
```bash
#!/bin/bash

CHROOT_DIR="/mnt/mychroot"

# Монтируем все необходимое
echo "==> Mounting filesystems..."
sudo mount --bind /proc ${CHROOT_DIR}/proc
sudo mount --bind /sys ${CHROOT_DIR}/sys
sudo mount --bind /dev ${CHROOT_DIR}/dev
sudo mount --bind /dev/pts ${CHROOT_DIR}/dev/pts

# Запускаем chroot
echo "==> Entering chroot. Type 'exit' to leave."
sudo chroot ${CHROOT_DIR} /bin/bash

# Размонтируем после выхода
echo "==> Unmounting filesystems..."
sudo umount ${CHROOT_DIR}/dev/pts
sudo umount ${CHROOT_DIR}/dev
sudo umount ${CHROOT_DIR}/sys
sudo umount ${CHROOT_DIR}/proc

echo "==> Done."
```

Сделайте его исполняемым (`chmod +x enter-chroot.sh`) и запускайте (`./enter-chroot.sh`).

---
**Что дальше?** Теперь у нас есть полнофункциональное окружение. Пора применить его для решения реальных задач.

➡️ **Следующий модуль**: [Практические задачи](./04-practical-use-cases.md)