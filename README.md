# Linux Projects

Учебные проекты по администрированию Linux, выполненные в Яндекс.Облаке.

## Технологии

Samba (CIFS) · autofs · Nginx (upstream, proxy_pass, балансировка) · Python HTTP Server · Яндекс.Облако (VPC, группы безопасности)

---

## 1. Файловый сервер и автомонтирование

### Цель

Настроить общую папку на ВМ1 через Samba и автоматически монтировать её на ВМ2 через autofs.

### Установка необходимых пакетов

```bash
# На ВМ1 (сервер)
sudo apt update && sudo apt install -y samba

# На ВМ2 (клиент)
sudo apt update && sudo apt install -y autofs cifs-utils
```

### Конфигурация

**ВМ1 — файловый сервер (Samba)**

`/etc/samba/smb.conf`:

```ini
[smbuser]
path = /home/smbuser
read only = yes
valid users = smbuser
```

_Создан пользователь `smbuser`, доступ только для чтения._

**ВМ2 — клиент (autofs)**

`/etc/auto.master`:

```text
/mnt/smb /etc/auto.smbuser --timeout=30
```

`/etc/auto.smbuser`:

```text
smbuser -fstype=cifs,rw,username=smbuser,password=ПАРОЛЬ ://IP_ВМ1/smbuser
```

### Управление и проверка статуса сервисов

```bash
# На ВМ1 (Samba)
sudo systemctl restart smbd
sudo systemctl status smbd

# На ВМ2 (autofs)
sudo systemctl restart autofs
sudo systemctl status autofs
```

### Результат

Папка `/mnt/smb/smbuser` на ВМ2 отображает содержимое `/home/smbuser` с ВМ1. При отсутствии активности в течение 30 секунд папка автоматически размонтируется.

---

## 2. Прокси-сервер и балансировка

### Цель

Настроить Nginx на ВМ1 как прокси-сервер с балансировкой запросов между двумя веб-серверами (ВМ2 и ВМ3).

### Схема архитектуры

| Роль ВМ | Компонент          | Сетевой порт   | Назначение                                        |
| :------ | :----------------- | :------------- | :------------------------------------------------ |
| **ВМ1** | Nginx              | `80` (входной) | Принимает трафик пользователя и балансирует его   |
| **ВМ2** | Python HTTP Server | `8000`         | Веб-сервер, раздает файлы из смонтированной папки |
| **ВМ3** | Python HTTP Server | `8000`         | Веб-сервер, раздает файлы из смонтированной папки |

_Примечание: ВМ2 и ВМ3 монтируют одну и ту же папку с ВМ1 через autofs._

### Установка необходимых пакетов

```bash
# На ВМ1
sudo apt update && sudo apt install -y nginx
```

### Конфигурация Nginx (ВМ1)

`/etc/nginx/nginx.conf`:

```ini
http {
    upstream file_serv {
        server ВМ2:8000;
        server ВМ3:8000;
    }

    server {
        listen 80;
        location / {
            proxy_pass http://file_serv;
        }
    }
}
```

### Запуск и проверка компонентов

**1. На ВМ1 (Nginx):**

```bash
sudo systemctl restart nginx
sudo systemctl status nginx
```

**2. На ВМ2 и ВМ3 (Веб-серверы):**

```bash
python3 -m http.server --directory /mnt/smb/smbuser 8000
```

### Результат

При обращении к ВМ1 на порту 80 запросы распределяются (балансируются) методом Round Robin между ВМ2 и ВМ3, отдавая актуальное содержимое общей папки.

---

**Автор:** s6ade
