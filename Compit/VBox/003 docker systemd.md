как проинсталировать docker  c systemd на ubuntu

----------------------

Вот пошаговая инструкция по установке Docker с systemd на Ubuntu:

## 1. Подготовка системы

### Обновление пакетов:
```bash
sudo apt update
sudo apt upgrade -y
```

### Установка зависимостей:
```bash
sudo apt install -y apt-transport-https ca-certificates curl gnupg lsb-release
```

## 2. Добавление официального репозитория Docker

### Добавление GPG-ключа:
```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

### Добавление репозитория:
```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Обновление списка пакетов:
```bash
sudo apt update
```

## 3. Установка Docker

### Установка последней версии:
```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Или установка конкретной версии:
```bash
# Посмотреть доступные версии
apt-cache madison docker-ce

# Установить конкретную версию
sudo apt install -y docker-ce=5:24.0.7-1~ubuntu.22.04~jammy docker-ce-cli=5:24.0.7-1~ubuntu.22.04~jammy containerd.io docker-buildx-plugin docker-compose-plugin
```

## 4. Настройка systemd для Docker

### Включение автозапуска:
```bash
sudo systemctl enable docker.service
sudo systemctl enable containerd.service
```

### Запуск Docker:
```bash
sudo systemctl start docker
```

### Проверка статуса:
```bash
sudo systemctl status docker
```

## 5. Настройка Docker Daemon для systemd

### Создание конфигурационного файла:
```bash
sudo mkdir -p /etc/docker
sudo nano /etc/docker/daemon.json
```

### Пример конфигурации:
```json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
```

### Альтернативная конфигурация (для продвинутых настроек):
```json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  },
  "storage-driver": "overlay2",
  "live-restore": true,
  "metrics-addr": "127.0.0.1:9323",
  "experimental": false
}
```

## 6. Перезагрузка Docker

### Применение изменений:
```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### Проверка конфигурации:
```bash
sudo docker info | grep -i cgroup
```
Должно показывать: `Cgroup Driver: systemd`

## 7. Настройка прав пользователя

### Добавление пользователя в группу docker:
```bash
sudo usermod -aG docker $USER
```

### Применение изменений группы:
```bash
newgrp docker
# или перелогиньтесь в систему
```

## 8. Проверка установки

### Тестовый запуск контейнера:
```bash
docker run hello-world
```

### Проверка версий:
```bash
docker --version
docker-compose --version
docker system info
```

## 9. Дополнительные настройки systemd

### Создание drop-in файла для кастомизации:
```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo nano /etc/systemd/system/docker.service.d/override.conf
```

```ini
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TimeoutStartSec=0
```

### Применение изменений:
```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

## 10. Управление сервисом Docker

### Основные команды systemd:
```bash
# Статус
sudo systemctl status docker

# Остановка
sudo systemctl stop docker

# Запуск
sudo systemctl start docker

# Перезагрузка
sudo systemctl restart docker

# Просмотр логов
sudo journalctl -u docker.service -f
```

## 11. Устранение проблем

### Если возникают проблемы с cgroups:
```bash
# Проверка поддержки cgroups
cat /proc/filesystems | grep cgroup

# Проверка монтирования cgroups
mount | grep cgroup
```

### Проверка логов Docker:
```bash
sudo journalctl -u docker.service
```

### Проверка конфигурации:
```bash
sudo docker info
```

## 12. Установка Docker Compose (отдельно)

Если нужно установить отдельно Docker Compose:

```bash
# Скачать последнюю версию
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# Дать права на выполнение
sudo chmod +x /usr/local/bin/docker-compose

# Проверить установку
docker-compose --version
```

Теперь Docker установлен и настроен для работы с systemd на вашей Ubuntu системе!
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTU0MjQ1OTgxNF19
-->