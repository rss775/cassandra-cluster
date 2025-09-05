# Автоматизация SSH в контейнерах Cassandra



Создайте файл `.env` для хранения пароля (подходит для тестовой среды) 
```bash
# .env
ROOT_PASSWORD=root
```

## Варианты автоматизации

### Вариант 1: Build через Dockerfile

**Создайте `Dockerfile`**
```dockerfile
FROM cassandra:5.0.5

# Устанавливаем SSH при сборке образа с зеркал
RUN sed -i 's|http://archive.ubuntu.com/ubuntu|https://mirror.yandex.ru/ubuntu|g' /etc/apt/sources.list && \
    sed -i 's|http://security.ubuntu.com/ubuntu|https://mirror.yandex.ru/ubuntu|g' /etc/apt/sources.list && \
    apt-get update && \
    apt-get install -y openssh-server && \
    rm -rf /var/lib/apt/lists/*

# Настраиваем SSH
RUN sed -i 's/.*PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config && \
    mkdir -p /run/sshd

# Устанавливаем пароль из аргумента сборки
ARG ROOT_PASSWORD
RUN echo "root:${ROOT_PASSWORD:-root}" | chpasswd

# Открываем SSH порт
EXPOSE 22

# Запускаем SSH и Cassandra
ENTRYPOINT ["/bin/bash", "-c", "/usr/sbin/sshd -D & exec /usr/local/bin/docker-entrypoint.sh cassandra -f"]
```

**Изменения в docker-compose.yml:**
```yaml
services:
  cassandra1:
    build:
      context: .
      args:
        - ROOT_PASSWORD=${ROOT_PASSWORD}  # Передает пароль из .env в сборку
    env_file: .env
    environment:
    # остальные настройки...
```

**Запуск кластера:**
```bash
# Собираем образ и запускаем кластер
docker compose up -d --build
```

- SSH предустановлен в образе при сборке
- Ключи сохраняются до ребилда
- Быстрый запуск контейнера (без apt-get)

**Очистить build кэш если больше не нужен**
```bash
docker compose down
docker builder prune -f
```

### Вариант 2: docker exec + setup-ssh.sh

Создайте файл `setup-ssh.sh`

```bash
#!/bin/bash

# Устанавливаем SSH с зеркал, если его нет
if ! which sshd; then
    echo "- Installing openssh-server"
    sed -i 's|http://archive.ubuntu.com/ubuntu|https://mirror.yandex.ru/ubuntu|g' /etc/apt/sources.list
    sed -i 's|http://security.ubuntu.com/ubuntu|https://mirror.yandex.ru/ubuntu|g' /etc/apt/sources.list
    apt-get update && apt-get install -y openssh-server
fi

mkdir -p /run/sshd

# Разрешаем root вход по паролю или ключу
sed -i 's/.*PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config

# Устанавливаем пароль из переменной окружения
ROOT_PASSWORD=${ROOT_PASSWORD:-root}
echo "root:${ROOT_PASSWORD}" | chpasswd

# Запускаем SSH сервер в фоне
/usr/sbin/sshd -D &
```

Сделайте файл исполняемым
```bash
chmod +x setup-ssh.sh
```

**Запуск кластера:**
```bash
# 1. Запускаем кластер
docker compose up -d

# 2. Настраиваем SSH. Замените имя контейнера
docker exec --env-file .env cassandra-cluster-cassandra1-1 bash -c "$(cat setup-ssh.sh)"
```

### Вариант 3: Автоматический entrypoint в docker-compose.yml

Создайте файл `setup-ssh.sh` как для второго варианта

**Изменения в docker-compose.yml:**
```yaml
services:
  cassandra1:
    image: cassandra:5.0.5
    env_file: .env  # Передает переменные из .env в контейнер
    volumes:
      - ./setup-ssh.sh:/setup-ssh.sh:ro
    entrypoint: ["/bin/bash", "-c", "source /setup-ssh.sh && exec /usr/local/bin/docker-entrypoint.sh cassandra -f"]
    environment:
    # остальные настройки...
    healthcheck:
    # ...
        retries: 15 # старт идет дольше
```

**Запуск кластера:**
```bash
# 1. Запускаем кластер (SSH настроится автоматически)
docker compose up -d
docker compose logs -f cassandra1 # в отдельном окне
```

- SSH настраивается автоматически при запуске
- Конейнер стартует долго если интернет медленный

## Подключение

```bash
# Сбросить сохраненные SSH ключи хоста при необходимости
ssh-keygen -R 192.168.1.200
# Подключение по SSH (пароль из .env)
ssh root@192.168.1.200
```

## Скрипт для автоподнятия сети
`setup-ipvlan-shim.sh`
```bash
#!/bin/bash

# Создание ipvlan интерфейса для доступа к контейнеру
sudo ip link add ipvlan-shim link eth0 type ipvlan  # делаем виртуальный интерфейс
sudo ip addr add 192.168.1.199/24 dev ipvlan-shim   # даем ему IP
sudo ip link set ipvlan-shim up                     # включаем
sudo ip route add 192.168.1.200/32 dev ipvlan-shim  # добавляем маршрут к контейнеру
```

**Использование:**
```bash
chmod +x setup-ipvlan-shim.sh
./setup-ipvlan-shim.sh
```

