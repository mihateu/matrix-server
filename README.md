# Запуск одиночного matrix-synapse сервера с поддержкой звонков на базе LiveKit + lk-jwt-service + synapse-admin.
**Понадобится 3 домена: livekit.example.com, matrix.example.com, admin.example.com** (могут быть любые, главное, чтобы все вели на IP-адрес сервера).

1. Необходимо открыть следующие порты

| Порт | Протокол |
|------|----------|
| 80 | TCP |
| 443 | TCP |
| 3478 | TCP/UDP |
| 7880-7881 | TCP |
| 8448 | TCP |
| 50000-50100 | UDP |

2. Установим необходимые зависимости

Проверьте синхронизацию времени через ```timedatectl```. Должно быть ```System clock synchronized: yes```, в противном случае выполните ```sudo timedatectl set-ntp true```.

```
sudo apt update && sudo apt upgrade -y
sudo apt install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release \
    certbot \
    python3-certbot-nginx \
    git \
    nano
```

```
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update

sudo apt install -y \
    docker-ce \
    docker-ce-cli \
    containerd.io \
    docker-buildx-plugin \
    docker-compose-plugin
```

```
sudo usermod -aG docker $USER
```

Перелогиньтесь и выполните

```
newgrp docker
```

3. Проверим

```
docker --version
docker compose version
certbot --version
```


4. Получим сертификаты для доменов с помощью ```certbot```

```
sudo certbot certonly --standalone -d livekit.example.com
sudo certbot certonly --standalone -d admin.example.com
sudo certbot certonly --standalone -d matrix.example.com
```

5. Отредактируем .env файл

```
cp .env.example .env
nano .env
```

PROJECT_DIR - полный путь к рабочей директории, \
MATRIX_DOMAIN,
ADMIN_DOMAIN,
LIVEKIT_DOMAIN,
SYNAPSE_SERVER_NAME - домены, с которыми работаем, \
POSTGRES_PASSWORD - пароль пользователя PostgreSQL, получаем через ```openssl rand -base64 32```, \
LIVEKIT_KEY - ключ для LiveKit; можно сгенерировать с помощью ```openssl rand -hex 10```, \
LIVEKIT_SECRET - секрет для LiveKit; можно сгенерировать с помощью ```openssl rand -hex 32```.

6. Сгенерируем Synapse конфигурацию

```docker compose run --rm synapse generate```

Файл **homeserver.yaml** отредактируем в соответствии с **homeserver.yaml.example** (находится в data/synapse/homeserver.yaml). \
Важно обратить внимание на строки:
- password: POSTGRESQL ПАРОЛЬ ИЗ .env
- все строки с matrix.example.com и livekit.example.com замените на свои домены
- turn_shared_secret: "turn секрет, сгенерированный вами", сгенерировать можно с помощью ```openssl rand -base64 32```
- livekit_key: "ключ, сгенерированный вами", вставляем из .env
- livekit_secret: "секрет, сгенерированный вами", вставляем из .env

7. Создайте новую сеть в docker

```
docker network create matrix-network
```

8. Создадим coturn конфигурацию

Находясь в директории проекта, выполните ```cp data/coturn/turnserver.conf.example data/coturn/turnserver.conf```. Отредактируйте конфигурационный файл, изменив realm на ваш matrix домен, вписав туда IP адрес вашего сервера (по идее тот, на который указывают домены), а также указав секрет (turn_shared_secret), сгенерированный для **homeserver.yaml**.

9. Создадим livekit конфигурацию

Находясь в директории проекта, выполните ```cp data/livekit/livekit.yaml.example data/livekit/livekit.yaml```. Отредактируйте конфигурационный файл, вставив вместо LIVEKIT_API_KEY и LIVEKIT_API_SECRET значения, полученные из .env.

10. Создадим nginx конфигурации

Из директории проекта выполните

```
cp nginx/conf.d/admin.conf.example nginx/conf.d/admin.conf
cp nginx/conf.d/livekit.conf.example nginx/conf.d/livekit.conf
cp nginx/conf.d/matrix.conf.example nginx/conf.d/matrix.conf
```

Отредактируйте файлы, заменив домены matrix.example.com, livekit.example.com, admin.example.com на свои (и пути до сертификатов, указаны стандартные). В admin.conf в строке ```allow YOUR_IP``` укажите тот IP-адрес, с которого будет разрешен вход в панель (можно указывать подсети в CIDR нотации).

11. Запуск

Все готово, осталось запустить контейнеры.

```
docker compose up -d
```

Проверить работу контейнеров можно с помощью ```docker ps```. Если около всех контейнеров указано "Up", то все успешно.

Создать пользователя можно через админ-панель по пути **https://admin.example.com**.

В Matrix клиенте (я использовал Element X) указывает провайдером **matrix.example.com** и вводим данные пользователя.
