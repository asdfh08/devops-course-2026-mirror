# Конфигурация виртуальной машины devops-vm

Гипервизор: UTM (QEMU, режим «Виртуализировать»), хост — macOS на Apple Silicon.
Гостевая ОС: Ubuntu Server 26.04.1 LTS (arm64), установка с образа `ubuntu-26.04.1-live-server-arm64.iso`.
Имя узла: `devops-vm`, полное имя: `devops-vm.devops.local`.

## 1. Параметры машины

| Параметр | Значение |
|---|---|
| Оперативная память | 2048 МиБ |
| Процессор | 2 ядра |
| Диск | 25 ГБ (qcow2), разметка по умолчанию: LVM, `/` ≈ 11 ГБ, `/boot` 2 ГБ, `/boot/efi` 1 ГБ |
| Архитектура | ARM64 (aarch64) |
| Вывод на дисплей | включён, OpenGL выключен |

При установке выбраны: тип установки Ubuntu Server (не minimized), диск целиком без шифрования, пункт **Install OpenSSH server**, дополнительные snap-пакеты не устанавливались.

## 2. Сетевые интерфейсы

| Адаптер UTM | Режим | Интерфейс в гостевой ОС | Адрес | Назначение |
|---|---|---|---|---|
| Сеть 1 | Эмулированный VLAN (аналог NAT) | `enp0s1` | `10.0.2.15/24` (DHCP) | выход в интернет, проброс портов с хоста |
| Сеть 2 | Только хост (Host Only) | `enp0s2` | `192.168.128.2/24` (DHCP) | прямой доступ с хоста; адрес хоста в этой сети — `192.168.128.1` |

Проверка: `ip -brief address`.

## 3. Правило проброса портов

Настраивается в UTM: «Изменить» → «Сеть» (первый адаптер) → «Перенаправление портов». Изменение возможно только при выключенной машине.

| Протокол | Адрес хоста | Порт хоста | Гостевой порт | Состояние |
|---|---|---|---|---|
| TCP | 127.0.0.1 | **2223** | **2222** | действующее |
| TCP | 127.0.0.1 | 2222 | 22 | исходное, после смены порта SSH не используется |

Примечание: в используемой версии UTM существующее правило недоступно для редактирования и удаления, поэтому вместо изменения гостевого порта 22 → 2222 добавлено новое правило с портом хоста 2223.

## 4. Учётные записи

| Пользователь | Группы | Способ аутентификации |
|---|---|---|
| `student` (uid 1000) | `student`, `sudo` и др. | создан установщиком; пароль; по SSH не допускается (`AllowUsers devops`), вход только с консоли UTM |
| `devops` (uid 1001) | `devops`, `sudo`, `users` | только по ключу ED25519; пароль используется для `sudo` |

Создание пользователя `devops`:

```bash
sudo adduser devops
sudo usermod -aG sudo devops
```

Ключ создаётся на хосте и копируется на сервер (до ужесточения SSH, пока разрешён вход по паролю):

```bash
ssh-keygen -t ed25519 -C "devops-vm-key" -f ~/.ssh/devops_vm      # пустая passphrase
ssh-copy-id -i ~/.ssh/devops_vm.pub -p 2222 devops@127.0.0.1
```

Права на сервере: `~/.ssh` — 700, `~/.ssh/authorized_keys` — 600 (`stat -c '%a %n' ~/.ssh ~/.ssh/authorized_keys`).

Настройки на хосте (macOS):

`~/.ssh/config`:

```
Host devops
    HostName 127.0.0.1
    Port 2223
    User devops
    IdentityFile ~/.ssh/devops_vm
    IdentitiesOnly yes
```

`/etc/hosts`:

```
192.168.128.2   devops.local
```

После правки — `sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder`.

## 5. Служба SSH

- Порт: **2222**.
- Файл изменений: `/etc/ssh/sshd_config.d/99-hardening.conf` (основной `/etc/ssh/sshd_config` не изменялся; эталонная копия — `/etc/ssh/sshd_config.backup`).
- Содержимое `99-hardening.conf`:

```
Port 2222

PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
MaxAuthTries 3
LoginGraceTime 30

AllowUsers devops

X11Forwarding no
ClientAliveInterval 300
ClientAliveCountMax 2
```

- В файле `/etc/ssh/sshd_config.d/50-cloud-init.conf` (создан cloud-init) строка `PasswordAuthentication yes` закомментирована, так как обрабатывается раньше файла `99-` и переопределяла бы значение `no`:

```bash
sudo sed -i 's/^PasswordAuthentication yes/#PasswordAuthentication yes/' /etc/ssh/sshd_config.d/50-cloud-init.conf
```

- Активация по сокету отключена, иначе порт задаётся юнитом `ssh.socket`, а не директивой `Port`:

```bash
sudo sshd -t && echo "Конфигурация корректна"
sudo systemctl disable --now ssh.socket
sudo systemctl enable --now ssh.service
sudo systemctl restart ssh
sudo ss -tlnp | grep sshd        # 0.0.0.0:2222 и [::]:2222
```

- Проверка действующих значений: `sudo sshd -T | grep -Ei "^(port|permitrootlogin|passwordauthentication|maxauthtries|allowusers)"`.

## 6. Правила межсетевого экрана

UFW, журналирование `medium`.

| Политика по умолчанию | Значение |
|---|---|
| Входящий трафик | deny |
| Исходящий трафик | allow |
| Маршрутизируемый трафик | disabled |

| Порт | Действие | Комментарий |
|---|---|---|
| 2222/tcp (IPv4 и IPv6) | LIMIT | SSH rate-limited |
| 80/tcp (IPv4 и IPv6) | ALLOW | HTTP |
| 443/tcp (IPv4 и IPv6) | ALLOW | HTTPS |

Порядок применения (правило для порта 2222 создаётся до включения):

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw limit 2222/tcp comment 'SSH rate-limited'
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'
sudo ufw enable
sudo ufw logging medium
sudo ufw status verbose
```

Имя узла:

```bash
sudo hostnamectl set-hostname devops-vm
sudo sed -i 's/^127.0.1.1.*/127.0.1.1 devops-vm.devops.local devops-vm/' /etc/hosts
hostname -f        # devops-vm.devops.local
```

## 7. Снимки состояния

В UTM снимки выполнены клонированием выключенной машины («Клонировать», затем переименование копии). Клоны не запускаются.

| Наименование | Момент создания | Состояние |
|---|---|---|
| `01-clean-install` | 22.10.2026, ≈17:00 | чистая установка, обновления применены, SSH на порту 22, проброс 127.0.0.1:2222 → 22 |
| `02-keys-configured` | 22.10.2026, ≈17:40 | создан пользователь `devops`, вход по ключу настроен |
| `03-ssh-hardened` | 22.10.2026, ≈18:15 | SSH на порту 2222 с ужесточёнными параметрами, добавлено правило проброса 127.0.0.1:2223 → 2222 |

Проверка итоговой конфигурации: `scripts/audit.sh` (код возврата 0 при соответствии всем требованиям).
