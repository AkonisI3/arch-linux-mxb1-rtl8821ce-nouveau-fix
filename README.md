# arch-linux-mxb1-rtl8821ce-nouveau-fix
Fix Arch Linux boot freeze (nouveau kernel panic) and Realtek RTL8821CE Wi-Fi drops / PCIe AER errors on Chinese Alder Lake boards (MXB1 / BIOS JH_NXB1_AN_B814).
# Fix Arch Linux Boot Freeze, Nouveau Kernel Panic & Realtek RTL8821CE Wi-Fi Drops on Chinese Alder Lake Boards (`JH_NXB1_AN_B814`)

Гайд по решению сразу двух критических проблем при установке и запуске **Arch Linux** (и других дистрибутивов на свежих ядрах) на китайских ноутбуках / платах на базе **Intel Alder Lake** с прошивкой BIOS `JH_NXB1_AN_B814` и Wi-Fi модулем **Realtek RTL8821CE** (`10ec:c821`).

---

## Симптомы и ошибки

Если при загрузке система намертво зависает («загрузка превращается в рулетку»), а в логах `dmesg` / `journalctl` появляются следующие ошибки — это решение для вас.

### 1. Kernel Panic / зависание на этапе `udev-worker` из-за `nouveau`
При инициализации открытого драйвера NVIDIA на гибридной/китайской разводке происходит падение ядра:
```text
Oops: general protection fault, probably for non-canonical address
CPU: 3 UID: 0 PID: 341 Comm: (udev-worker) Tainted: G D
Hardware name: Default string Default string/MXB1, BIOS JH_NXB1_AN_B814
RIP: 0010:idempotent_init_module
Modules linked in: ... nouveau(+) ...
note: (udev-worker)[341] exited with preempt_count 1
```

### 2. Отвал Wi-Fi и спам ошибками `PCIe Bus Error` (`rtw88_8821ce`)
Из-за кривой реализации энергосбережения PCIe (ASPM) в китайском BIOS встроенный драйвер `rtw88_8821ce` уходит в энергосбережение, отваливается от сети при попытке подключения и забивает консоль ошибками:
```text
pcieport 0000:00:1c.0: AER: Multiple Correctable error message received from 0000:01:00.0
rtw88_8821ce 0000:01:00.0: PCIe Bus Error: severity=Correctable, type=Physical Layer, (Receiver ID)
rtw88_8821ce 0000:01:00.0: device [10ec:c821] error status/mask=00000001/0000c000
rtw88_8821ce 0000:01:00.0: [ 0] RxErr (First)
```

> **Важно:** Не пытайтесь ставить сторонние форки драйверов Realtek с GitHub (например, старые репозитории `lwfinger` 2018–2021 годов) на современные ядра Linux (6.18+). На актуальных ядрах штатный модуль `rtw88_8821ce` работает стабильно, проблема исключительно в управлении питанием шины PCIe.

---

## Быстрое решение (Параметры ядра)

Для устранения обеих проблем нужно добавить в параметры загрузки ядра следующую связку флагов:

```text
modprobe.blacklist=nouveau pci=noaer pcie_aspm=off
```

* `modprobe.blacklist=nouveau` — запрещает загрузку проблемного драйвера `nouveau`, убирая панику ядра при старте.
* `pcie_aspm=off` — отключает агрессивное энергосбережение шины PCIe (Active State Power Management), из-за которого отваливается Wi-Fi модуль RTL8821CE.
* `pci=noaer` — отключает спам корректируемыми ошибками PCIe (Advanced Error Reporting) в системный лог и консоль.

---

## Как применить на постоянной основе

### Первый запуск (из меню загрузчика / Live-флешки)
1. В меню загрузчика (**systemd-boot** или **GRUB**) выберите пункт загрузки Arch Linux и нажмите клавишу **`e`** для редактирования параметров.
2. В конец строки с параметрами ядра (начинается с `options` или `linux`) допишите через пробел:
   ```text
   modprobe.blacklist=nouveau pci=noaer pcie_aspm=off
   ```
3. Нажмите `Enter` (в systemd-boot) или `Ctrl+X` / `F10` (в GRUB), чтобы загрузиться.

---

### Вариант А: Если у вас `systemd-boot`
1. Откройте файл конфигурации вашей записи загрузки в `/boot/loader/entries/` (например, `arch.conf`):
   ```bash
   sudo nano /boot/loader/entries/arch.conf
   ```
2. Добавьте параметры в конец строки `options`:
   ```text
   title   Arch Linux
   linux   /vmlinuz-linux-lts
   initrd  /initramfs-linux-lts.img
   options root=UUID=xxxx-xxxx rw rootfstype=btrfs modprobe.blacklist=nouveau pci=noaer pcie_aspm=off
   ```
3. Сохраните файл (`Ctrl+O`, `Enter`, `Ctrl+X`). Пересборка конфига не требуется.

---

### Вариант Б: Если у вас `GRUB`
1. Откройте `/etc/default/grub`:
   ```bash
   sudo nano /etc/default/grub
   ```
2. Добавьте параметры в переменную `GRUB_CMDLINE_LINUX_DEFAULT`:
   ```text
   GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet modprobe.blacklist=nouveau pci=noaer pcie_aspm=off"
   ```
3. Обновите конфигурацию GRUB:
   ```bash
   sudo grub-mkconfig -o /boot/grub/grub.cfg
   ```

---

## Что делать с видеокартой дальше?

После стабильной загрузки в систему установите проприетарный драйвер NVIDIA вместо заблокированного `nouveau`:
* Для стандартного ядра `linux`: `sudo pacman -S nvidia nvidia-utils`
* Для LTS-ядра `linux-lts`: `sudo pacman -S nvidia-lts nvidia-utils`
* Для современных карт (Turing / RTX 20xx и новее): можно использовать `nvidia-open` или `nvidia-open-lts`.

---


## Настройка Wi-Fi после установки системы

На чипе **Realtek RTL8821CE** стандартный `wpa_supplicant` часто работает нестабильно и приводит к обрывам связи. Оптимальный вариант — использовать связку **NetworkManager + iwd** (где `iwd` выступает в роли Wi-Fi бэкенда).

> **Важно:** Устанавливать и включать эти пакеты нужно **после** установки системы на диск и перезагрузки с новыми параметрами ядра. Если интернета ещё нет — временно раздайте его с телефона по USB-кабелю.

1. Установите оба пакета:
   ```bash
   sudo pacman -S networkmanager iwd
   ```

2. Укажите `NetworkManager` использовать `iwd` вместо `wpa_supplicant` (чтобы службы не конфликтовали за адаптер):
   ```bash
   echo -e "[device]\nwifi.backend=iwd" | sudo tee /etc/NetworkManager/conf.d/wifi_backend.conf
   ```

3. Включите обе службы в автозапуск:
   ```bash
   sudo systemctl enable --now iwd NetworkManager
   ```

4. Подключитесь к сети через `nmcli` (или `nmtui`):
   ```bash
   nmcli device wifi connect "ИМЯ_СЕТИ" password "ПАРОЛЬ"
## Переход на основное (свежее) ядро `linux` (проверено на `7.3.6-arch1-1`)

Фикс полностью универсален и одинаково стабильно работает как на `linux-lts` (`6.18.x`), так и на свежей ветке основного ядра `linux` (`7.3.x+`). Вы можете держать оба ядра параллельно в меню `systemd-boot`, оставив `linux-lts` в качестве резервного.

### 1. Установка ядра
Установите пакет основного ядра и обновите систему:
```bash
sudo pacman -Syu linux
```

### 2. Настройка загрузчика `systemd-boot`

#### Вариант А: Если используется UKI (Unified Kernel Image в `/boot/EFI/Linux/`)
1. Убедитесь, что параметры `modprobe.blacklist=nouveau pci=noaer pcie_aspm=off` прописаны в `/etc/kernel/cmdline`.
2. Откройте пресет `/etc/mkinitcpio.d/linux.preset`, закомментируйте `default_image` и включите сборку UKI в `/boot/EFI/Linux/`:
   ```ini
   PRESETS=('default')

   #default_image="/boot/initramfs-linux.img"
   default_uki="/boot/EFI/Linux/arch-linux.efi"
   default_options="--splash /usr/share/systemd/bootctl/splash-arch.bmp"
   ```
3. Соберите образ:
   ```bash
   sudo mkinitcpio -p linux
   ```

#### Вариант Б: Если используются классические конфиги в `/boot/loader/entries/`
Скопируйте рабочий конфиг от LTS-ядра и уберите суффикс `-lts` из путей к ядру и initramfs (строка `options` с флагами ядра останется без изменений):
```bash
sudo sed 's/-lts//g' /boot/loader/entries/arch-lts.conf | sudo tee /boot/loader/entries/arch.conf
```

### 3. Проверка
Перезагрузитесь (`reboot`), выберите в меню `systemd-boot` пункт **`Arch Linux (7.3.x-arch1-1)`** и проверьте стабильность сети:
```bash
ping -c 3 archlinux.org
```
   ```
## Тестовый стенд
* **OS:** Arch Linux (`6.18.65-1-lts`)
* **Motherboard / BIOS:** `MXB1` / `JH_NXB1_AN_B814` (Intel Alder Lake)
* **Wi-Fi Chip:** Realtek RTL8821CE (`10ec:c821`)
* **File System:** Btrfs
* 
