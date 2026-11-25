# Конфигурация виртуальной машины devops-vm

Разделы 1–9 описывают конфигурацию, созданную вручную в практических работах № 5–7 и КР № 2 (её состояние сохранено в снимке `08-after-pr10`). В практической работе № 11 VM возвращена к снимку `01-clean-install` и настраивается Ansible — см. раздел 10.

## 1. Параметры машины
- Наименование: devops-vm, гостевая система Ubuntu Server 26.04.1 LTS (aarch64), VirtualBox 7.2 на macOS (Apple Silicon).
- Оперативная память 2048 МБ, процессор 2 ядра.
- Диск 25 ГБ, динамически расширяемый (LVM, корневой том 11 ГБ).

## 2. Сетевые интерфейсы
| Адаптер | Тип | Интерфейс | Адрес | Назначение |
|---|---|---|---|---|
| 1 | NAT | enp0s8 | 10.0.2.15/24 (DHCP) | выход в интернет, SSH через проброс порта |
| 2 | Host-only (HostNetwork) | enp0s9 | 192.168.56.2/24 (DHCP) | доступ с хоста по имени devops.local |

На хосте в `/etc/hosts`: `192.168.56.2   devops.local` (и имена ресурсов, например `kv.devops.local`).
Имя узла: `devops-vm`, полное имя `devops-vm.devops.local` (строка `127.0.1.1 devops-vm.devops.local devops-vm` в `/etc/hosts` VM).

## 3. Правило проброса портов
Адаптер 1 (NAT), правило `ssh`: TCP, адрес хоста 127.0.0.1, порт хоста 2222 → порт гостя 2222.

Алиас на хосте (`~/.ssh/config`):
```
Host devops
    HostName 127.0.0.1
    Port 2222
    User devops
    IdentityFile ~/.ssh/devops_vm
    IdentitiesOnly yes
```

## 4. Учётные записи
| Имя | Группы | Аутентификация |
|---|---|---|
| student | sudo, adm и др. (создан установщиком) | только локальная консоль (SSH запрещён `AllowUsers`) |
| devops | sudo, docker | SSH по ключу `~/.ssh/devops_vm` (ed25519, комментарий devops-vm-key) |

## 5. Служба SSH
- Порт 2222; конфигурация: `/etc/ssh/sshd_config.d/99-hardening.conf` (основной `sshd_config` не изменялся, копия `sshd_config.backup`).
- Директивы: `Port 2222`, `PermitRootLogin no`, `PasswordAuthentication no`, `PubkeyAuthentication yes`, `PermitEmptyPasswords no`, `MaxAuthTries 3`, `LoginGraceTime 30`, `AllowUsers devops`, `X11Forwarding no`, `ClientAliveInterval 300`, `ClientAliveCountMax 2`.
- В `50-cloud-init.conf` строка `PasswordAuthentication yes` закомментирована (файлы читаются по алфавиту, действует первое значение).
- `ssh.socket` отключён, `ssh.service` включён (иначе порт задаётся сокетом systemd).
- Ключи узла пересозданы командой `ssh-keygen -A`: установщик оставил их пустыми (0 байт), sshd не запускался.

## 6. Правила межсетевого экрана
- Политики: входящий — deny, исходящий — allow, маршрутизируемый — disabled; журналирование `medium`.
- Правила: `80/tcp ALLOW`, `443/tcp ALLOW`, `2222/tcp LIMIT` (комментарий «SSH rate-limited»), для IPv4 и IPv6.

## 7. Снимки состояния
| Снимок | Момент создания |
|---|---|
| 01-clean-install | после установки ОС, до настройки |
| 02-keys-configured | вход `ssh devops` по ключу |
| 03-ssh-hardened | порт 2222, вход по паролю и от root запрещён |
| 04-nginx-https | ПР № 6: nginx, HTTPS для devops.local |
| 05-docker-compose | ПР № 7: Docker, приложение за обратным прокси |
| 06-before-kr2 | перед КР № 2 |
| 07-kr2 | КР № 2: kr2-app + Redis за nginx (kv.devops.local) |
| 08-after-pr10 | после ПР № 10, перед откатом для ПР № 11 |

Снимки делаются на выключенной машине: сохранение состояния памяти работающей VM в VirtualBox на Apple Silicon выполняется очень медленно.

## 8. Веб-сервер
- Пакет `nginx` (1.28.3); конфигурация ресурса `/etc/nginx/sites-available/devops-site`, ссылка в `sites-enabled`, ссылка `default` удалена.
- Каталог ресурса `/var/www/devops-site`, владелец devops:devops, каталоги 755, файлы 644; доставка `rsync` из каталога `site/`.
- Сертификат `/etc/ssl/certs/devops.crt` (644), ключ `/etc/ssl/private/devops.key` (600, root), срок 365 дней:
```
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/devops.key \
  -out /etc/ssl/certs/devops.crt \
  -subj "/CN=devops.local" \
  -addext "subjectAltName=DNS:devops.local"
```
- HTTP перенаправляется на HTTPS (301); на хосте сертификат хранится как `~/devops.crt`.

## 9. Контейнеризация
- Docker Engine 29.9.3 и плагин Compose установлены из официального репозитория `download.docker.com` (suite resolute); служба `docker` включена.
- Группа docker: devops (членство в группе равносильно правам root).
- Проект ПР № 7: `~/app` (Flask + PostgreSQL 16), файл `~/app/.env` с правами 600 (только на сервере).
- Публикация: `127.0.0.1:5000:5000`; порт СУБД 5432 не публикуется; именованный том `app_db_data`.
- Обратный прокси-сервер: `location /app/` в HTTPS-блоке devops-site → `http://127.0.0.1:5000/`.

## Особенности стенда
На хосте работает VPN; через него TCP проходит, а UDP и IPv6 — нет. Поэтому:
- в VirtualBox для NAT включён резолвер хоста (`--nat-dns-host-resolver1=on`);
- systemd-resolved использует DNS over TLS: `/etc/systemd/resolved.conf.d/10-dns-over-tls.conf` (1.1.1.1, 9.9.9.9);
- контейнеры обращаются к DNS по TCP: `/etc/docker/daemon.json` — `"dns": ["1.1.1.1", "9.9.9.9"]`, `"dns-opts": ["use-vc"]`;
- IPv6 отключён на NAT-интерфейсе: `/etc/netplan/60-nat-no-ipv6.yaml` (`dhcp6: false`, `accept-ra: false`, `link-local: []` для enp0s8).

## 10. Ansible
- Расположение проекта: каталог `ansible/` репозитория devops-course-2026; запуск из каталога `ansible/` (там `ansible.cfg`).
- Управляющий узел: macOS (Apple Silicon), Ansible 15.0.0 (ansible-core 2.22.0), коллекции из `requirements.yml`: ansible.posix 1.6.2, community.docker 4.5.0, community.general 9.5.0.
- Управляемый узел: devops-vm, Ubuntu Server 26.04.1 LTS (aarch64), адрес Host-only `192.168.56.2`, SSH на порту 22.
- Inventory: `inventory/bootstrap.ini` — первичный запуск от `student` (ключ установлен `ssh-copy-id`, пароль sudo вводится `--ask-become-pass`); `inventory/prod.ini` — последующие запуски от `devops` по ключу `~/.ssh/devops_vm` без интерактивного ввода (NOPASSWD).
- Роли:
  - `base` — проверка платформы, DNS over TLS (стенд с VPN), базовые пакеты и `python3-requests`, `qemu-user-binfmt` на arm64, репозиторий и служба Docker;
  - `ssh` — пользователь `devops`, открытый ключ, sudo без пароля, hardening `/etc/ssh/sshd_config.d/00-ansible-hardening.conf`, UFW (22, 80, deny incoming);
  - `nginx` — reverse proxy `:80` → `127.0.0.1:5000`, отключение сайта по умолчанию, handlers `nginx -t` и reload;
  - `app` — загрузка образа по полному SHA (платформа `linux/amd64`), контейнер `devops-ci-app` с `restart_policy: unless-stopped`, проверка `/health`.
- Образ: `ghcr.io/mist1qu3x/devops-ci-app`, тег — полный SHA `6cb25027b38e6da3e15acac762a1ca6c2a5f9d49` (`group_vars/all.yml`, переменная `app_tag`); публикация `127.0.0.1:5000:5000`; health endpoint `/health`.
- Команды:
  - подготовка: `ansible-galaxy collection install -r requirements.yml`;
  - первичный запуск: `ssh-copy-id -i ~/.ssh/devops_vm.pub student@192.168.56.2`, затем `ansible-playbook -i inventory/bootstrap.ini site.yml --ask-become-pass`;
  - повторный запуск: `ansible-playbook site.yml` (ожидается `changed=0`);
  - read-only аудит: `ansible-playbook verify.yml`.
- Подготовка чистого снимка `01-clean-install` перед bootstrap (вручную в консоли VM): пересоздать пустые ключи узла (`ssh-keygen -A`) и разрешить `22/tcp` в UFW, который в снимке уже включён без правила для порта 22.

