# 🚀 Production Deployment Guide - Unicorns Community
## Деплой на edinorogy.com

### 📋 Предварительные требования
- VPS/сервер с Ubuntu 20.04+ или CentOS 8+
- Доступ по SSH с правами sudo
- Домен edinorogy.com настроен на DNS серверы REG.RU

### 🔑 Сгенерированные ключи для продакшена

```bash
# JWT секрет
JWT_SECRET="UC_prod_jwt_2024_$3cur3_k3y_9Xm8"

# Admin пароль (сгенерированный)
ADMIN_PASSWORD="UC$2024#Pr0d@dm1n!"

# Database URL (PostgreSQL для продакшена)
DATABASE_URL="postgresql://unicorns_user:UC_db_2024_$ecur3@localhost/unicorns_community"

# Webhook секрет для Telegram
WEBHOOK_SECRET="webhook_UC_2024_secure_token_X9mK"
```

---

## 🛠️ Пошаговая инструкция деплоя

### Шаг 1: Подготовка сервера

```bash
# Подключаемся к серверу
ssh root@your-server-ip

# Обновляем систему
apt update && apt upgrade -y

# Устанавливаем необходимые пакеты
apt install -y nginx postgresql postgresql-contrib python3-pip python3-venv git certbot python3-certbot-nginx nodejs npm
```

### Шаг 2: Настройка DNS

В панели REG.RU создайте следующие A-записи:
```
@ (edinorogy.com) → IP_ВАШЕГО_СЕРВЕРА
www → IP_ВАШЕГО_СЕРВЕРА
api → IP_ВАШЕГО_СЕРВЕРА
mini → IP_ВАШЕГО_СЕРВЕРА
```

DNS серверы: 
- ns5.hosting.reg.ru
- ns6.hosting.reg.ru

### Шаг 3: Настройка PostgreSQL

```bash
# Переключаемся на пользователя postgres
sudo -u postgres psql

-- Создаем базу данных и пользователя
CREATE DATABASE unicorns_community;
CREATE USER unicorns_user WITH PASSWORD 'UC_db_2024_$ecur3';
GRANT ALL PRIVILEGES ON DATABASE unicorns_community TO unicorns_user;
\q

# Настраиваем аутентификацию
echo "host    unicorns_community    unicorns_user    127.0.0.1/32    md5" >> /etc/postgresql/14/main/pg_hba.conf
systemctl restart postgresql
```

### Шаг 4: Деплой Backend (FastAPI)

```bash
# Создаем пользователя для приложения
useradd -m -s /bin/bash unicorns
su - unicorns

# Клонируем репозиторий
git clone https://github.com/yourusername/UCReg.git /home/unicorns/app
cd /home/unicorns/app

# Создаем виртуальное окружение
python3 -m venv venv
source venv/bin/activate

# Устанавливаем зависимости
pip install -r requirements.txt

# Создаем production конфигурацию
cat > /home/unicorns/app/.env << EOF
# Production Environment Variables
NODE_ENV=production
DATABASE_URL=postgresql://unicorns_user:UC_db_2024_\$ecur3@localhost/unicorns_community

# Telegram Bot Configuration
BOT_TOKEN=ВАШ_РЕАЛЬНЫЙ_BOT_TOKEN
WEBHOOK_URL=https://api.edinorogy.com/webhook
WEBHOOK_SECRET=webhook_UC_2024_secure_token_X9mK

# Telegram Group Settings
TELEGRAM_GROUP_CHAT_ID=ВАШ_GROUP_CHAT_ID
TELEGRAM_GROUP_NAME=ВАШ_GROUP_NAME
TELEGRAM_GROUP_USERNAME=ВАШ_GROUP_USERNAME
TELEGRAM_GROUP_INVITE_LINK=ВАШ_INVITE_LINK
ADMIN_CHAT_ID=ВАШ_ADMIN_CHAT_ID

# Security
JWT_SECRET=UC_prod_jwt_2024_\$3cur3_k3y_9Xm8
ADMIN_PASSWORD=UC\$2024#Pr0d@dm1n!
CORS_ORIGINS=["https://edinorogy.com", "https://www.edinorogy.com", "https://mini.edinorogy.com"]

# Production settings
DEBUG=false
PRODUCTION=true
EOF

# Устанавливаем права на конфиг
chmod 600 /home/unicorns/app/.env
```

### Шаг 5: Настройка systemd для Backend

```bash
# Возвращаемся к root
exit

# Создаем systemd сервис для FastAPI
cat > /etc/systemd/system/unicorns-api.service << EOF
[Unit]
Description=Unicorns Community FastAPI Application
After=network.target postgresql.service

[Service]
User=unicorns
Group=unicorns
WorkingDirectory=/home/unicorns/app/backend
Environment=PATH=/home/unicorns/app/venv/bin
ExecStart=/home/unicorns/app/venv/bin/python main_production.py
ExecReload=/bin/kill -s HUP \$MAINPID
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

# Запускаем и включаем сервис
systemctl daemon-reload
systemctl enable unicorns-api
systemctl start unicorns-api
systemctl status unicorns-api
```

### Шаг 6: Деплой Frontend (React/Vite)

```bash
# Переходим в папку frontend
cd /home/unicorns/app/frontend

# Создаем production конфигурацию
cat > .env.production << EOF
VITE_API_URL=https://api.edinorogy.com
VITE_BOT_USERNAME=ВАШ_БОТ_USERNAME
VITE_ENVIRONMENT=production
EOF

# Устанавливаем зависимости и собираем
npm install
npm run build

# Копируем статику в nginx директорию
mkdir -p /var/www/edinorogy.com
cp -r dist/* /var/www/edinorogy.com/
chown -R www-data:www-data /var/www/edinorogy.com
```

### Шаг 7: Настройка Nginx

```bash
# Создаем конфигурацию для основного сайта
cat > /etc/nginx/sites-available/edinorogy.com << EOF
# Main Website (Static)
server {
    listen 80;
    server_name edinorogy.com www.edinorogy.com;
    
    root /var/www/edinorogy.com;
    index index.html;
    
    # Security headers
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    
    # Static website
    location / {
        try_files \$uri \$uri/ /index.html;
    }
    
    # Static assets with caching
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}

# API Backend
server {
    listen 80;
    server_name api.edinorogy.com;
    
    # Rate limiting
    limit_req_zone \$binary_remote_addr zone=api:10m rate=10r/s;
    limit_req zone=api burst=20 nodelay;
    
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade \$http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
        proxy_cache_bypass \$http_upgrade;
        
        # Telegram webhook specific
        client_max_body_size 10M;
        proxy_read_timeout 60s;
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
    }
    
    # Admin panel with IP restrictions (опционально)
    location /admin {
        # allow YOUR_ADMIN_IP;
        # deny all;
        
        proxy_pass http://127.0.0.1:8000/admin;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
    }
}

# Mini App (React SPA)
server {
    listen 80;
    server_name mini.edinorogy.com;
    
    root /home/unicorns/app/frontend/dist;
    index index.html;
    
    # Security headers for Mini App
    add_header X-Frame-Options ALLOWALL;
    add_header Content-Security-Policy "frame-ancestors https://web.telegram.org https://*.telegram.org";
    
    location / {
        try_files \$uri \$uri/ /index.html;
    }
    
    # Proxy API calls from Mini App
    location /api/ {
        proxy_pass http://127.0.0.1:8000/api/;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
    }
}
EOF

# Активируем конфигурацию
ln -s /etc/nginx/sites-available/edinorogy.com /etc/nginx/sites-enabled/
nginx -t
systemctl reload nginx
```

### Шаг 8: SSL сертификаты (Let's Encrypt)

```bash
# Получаем SSL сертификаты для всех доменов
certbot --nginx -d edinorogy.com -d www.edinorogy.com -d api.edinorogy.com -d mini.edinorogy.com

# Настраиваем автообновление
echo "0 12 * * * /usr/bin/certbot renew --quiet" | crontab -
```

### Шаг 9: Настройка Telegram Bot

```bash
# Создаем скрипт для настройки webhook
cat > /home/unicorns/app/setup_webhook.py << EOF
#!/usr/bin/env python3
import requests
import os

BOT_TOKEN = "ВАШ_РЕАЛЬНЫЙ_BOT_TOKEN"
WEBHOOK_URL = "https://api.edinorogy.com/webhook"

def setup_webhook():
    url = f"https://api.telegram.org/bot{BOT_TOKEN}/setWebhook"
    payload = {
        "url": WEBHOOK_URL,
        "secret_token": "webhook_UC_2024_secure_token_X9mK",
        "allowed_updates": ["message", "callback_query", "chat_member"]
    }
    
    response = requests.post(url, json=payload)
    print("Webhook setup:", response.json())

def set_menu():
    url = f"https://api.telegram.org/bot{BOT_TOKEN}/setChatMenuButton"
    payload = {
        "menu_button": {
            "type": "web_app",
            "text": "🦄 Открыть приложение",
            "web_app": {
                "url": "https://mini.edinorogy.com"
            }
        }
    }
    
    response = requests.post(url, json=payload)
    print("Menu button setup:", response.json())

if __name__ == "__main__":
    setup_webhook()
    set_menu()
EOF

# Запускаем настройку
cd /home/unicorns/app
python3 setup_webhook.py
```

### Шаг 10: Мониторинг и логи

```bash
# Настройка логов
mkdir -p /var/log/unicorns
chown unicorns:unicorns /var/log/unicorns

# Создаем logrotate конфигурацию
cat > /etc/logrotate.d/unicorns << EOF
/var/log/unicorns/*.log {
    daily
    missingok
    rotate 52
    compress
    delaycompress
    notifempty
    create 644 unicorns unicorns
}
EOF

# Команды для мониторинга
echo "# Полезные команды для мониторинга:" >> /root/monitoring_commands.txt
echo "systemctl status unicorns-api" >> /root/monitoring_commands.txt
echo "journalctl -u unicorns-api -f" >> /root/monitoring_commands.txt
echo "tail -f /var/log/nginx/access.log" >> /root/monitoring_commands.txt
echo "tail -f /var/log/nginx/error.log" >> /root/monitoring_commands.txt
```

### Шаг 11: Backup и безопасность

```bash
# Создаем скрипт бэкапа базы данных
cat > /home/unicorns/backup_db.sh << EOF
#!/bin/bash
DATE=\$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/home/unicorns/backups"
mkdir -p \$BACKUP_DIR

# Создаем бэкап PostgreSQL
pg_dump -h localhost -U unicorns_user -d unicorns_community > \$BACKUP_DIR/db_backup_\$DATE.sql

# Удаляем старые бэкапы (старше 30 дней)
find \$BACKUP_DIR -name "db_backup_*.sql" -mtime +30 -delete

echo "Backup completed: db_backup_\$DATE.sql"
EOF

chmod +x /home/unicorns/backup_db.sh

# Настраиваем ежедневный бэкап
echo "0 2 * * * /home/unicorns/backup_db.sh" | crontab -u unicorns -

# Настройка firewall
ufw allow ssh
ufw allow http
ufw allow https
ufw enable
```

---

## 🔧 Конфигурация окончательных файлов

### Backend Production файл (main_production.py)
```python
import os
import uvicorn
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
import logging

# Настройка логирования
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('/var/log/unicorns/api.log'),
        logging.StreamHandler()
    ]
)

# Импорт основного приложения
from main import app

if __name__ == "__main__":
    uvicorn.run(
        "main:app",
        host="127.0.0.1",
        port=8000,
        reload=False,
        workers=2,
        log_config=None
    )
```

---

## 🚀 Запуск в продакшене

```bash
# После завершения всех настроек, проверяем статус сервисов
systemctl status unicorns-api
systemctl status nginx
systemctl status postgresql

# Проверяем доступность
curl https://edinorogy.com
curl https://api.edinorogy.com/api/health
curl https://mini.edinorogy.com

# Просматриваем логи
journalctl -u unicorns-api -f
tail -f /var/log/nginx/access.log
```

---

## 📱 Финальные настройки Telegram Bot

1. **В BotFather:**
   - `/setdomain` - установите `mini.edinorogy.com`
   - `/setmenubutton` - установите Web App URL: `https://mini.edinorogy.com`

2. **Проверка webhook:**
   ```bash
   curl https://api.telegram.org/bot<YOUR_TOKEN>/getWebhookInfo
   ```

---

## 🔒 Дополнительная безопасность

```bash
# Настройка fail2ban для защиты от брутфорса
apt install fail2ban

cat > /etc/fail2ban/jail.local << EOF
[nginx-limit-req]
enabled = true
filter = nginx-limit-req
action = iptables-multiport[name=ReqLimit, port="http,https", protocol=tcp]
logpath = /var/log/nginx/*error.log
findtime = 600
bantime = 7200
maxretry = 10
EOF

systemctl restart fail2ban
```

Ваш Telegram Mini App теперь полностью развернут на production сервере!
