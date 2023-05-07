# Как опубликовать Web-сервер на C# (Wep Api)

Проект написан на C# с использованием .NET 6 и шаблоном Web Api.

В инструкции есть пример подключения базы данных, в нашем случае PostgreSQL (вместе с PGADMIN) и Portainer.

## Подготовка веб-сервера

1. Аренда

Каждому свое, потому выбирайте сервер у того провайдера, у которого пожелаете. От меня лишь рекомандация брать там, кто бесплатно дает IP-адрес, чтобы по SSH было удобнее подключаться.

2. Установка Docker

[Ссылка на мануал](https://docs.docker.com/engine/install/ubuntu/) по установке Docker из официальной документации.

3. Установка docker-compose

Выполните установку [по манулу](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-compose-on-ubuntu-22-04). Достаточно пройти **Step 1**.

4. Авторизация в Docker

Следует авторизоваться в Docker Hub, чтобы иметь возможность скачивать изображения со своего репозитория.

```
$ docker login
```

##  Публикация Wep Api
1. собрать релизную версия проекта.

```
dotnet publish -c Release
```

2. Перейти в файлы сборки.

```
cd bin/Release/net6.0/publish
```

3. Написать докерфайл для последующей сборки проекта в image. Если не знаешь, как писать, то можно следующим образом.
```
FROM mcr.microsoft.com/dotnet/aspnet:6.0

WORKDIR /src

COPY . ./
ENV ASPNETCORE_URLS="http://+:5050"
EXPOSE 5050

ENTRYPOINT  ["dotnet", "/src/Thesis.Images.dll", "http://*:5050"]
```

Вместо **Thesis.Images** следует использовать ИМЯ проекта.

4. Собрать изображение из проекта.

```
docker build -t name .
```

Если процессор на ARM-архитектуре, то следующей командой
```
docker buildx build --platform linux/amd64 -t name:amd64 .
```

Если планируется разворачивать проект на веб-сервере, то лучше в наименовании изображения использовать никнейм на docker hub, например, **username/project-name**.

> 
5. Написать docker-compose
```
version: '3.4'

services:
  thesis-images:
    image: seljmov/thesis-images:amd64
    container_name: thesis-images
    hostname: thesis-images
    ports:
     - 10010:5050
```

6. Запустить docker-compose на сервере

```
$ docker-compose up -d
```

## Дополнительно

1. Переменные окружения

Если в проекте необходимо использовать переменные окружения, то делается это при помощи файла .env.

Заполняется следующим образом
```
ASPNETCORE_ENVIRONMENT=Production
JWT_ISSUER=test
JWT_AUDIENCE=test
JWT_KEY="test key"
JWT_ACCESS_TOKEN_LIFETIME=15
JWT_REFRESH_TOKEN_LIFETIME=10080
```

Дополняем docker-compose
```
version: '3.4'

services:
  thesis-images:
    image: seljmov/thesis-images:amd64
    container_name: thesis-images
    hostname: thesis-images
    ports:
     - 10010:5050
    environment:
     - ASPNETCORE_ENVIRONMENT
     - JwtOptions__Issuer=${JWT_ISSUER}
     - JwtOptions__Audience=${JWT_AUDIENCE}
     - JwtOptions__Key=${JWT_KEY}
     - JwtOptions__AccessTokenLifetime=${JWT_ACCESS_TOKEN_LIFETIME}
     - JwtOptions__RefreshTokenLifetime=${JWT_REFRESH_TOKEN_LIFETIME}
```

2. База данных

При необходимости использовать базу данных, можно дописать docker-compose.
```
version: '3.4'

networks:
  local-net:
    external: true

services:
  postgres: 
    image: postgres:12
    container_name: postgres
    hostname: postgres
    environment: 
      - POSTGRES_USER=${PG_USER}
      - POSTGRES_PASSWORD=${PG_PASSWORD}
    ports:
      - 5432:5432
    volumes:
      - pg-data:/var/lib/postgresql/data
      - pg-conf:/etc/postgresql
      - pg-log:/var/log/postgresql
      - pg-backup:/backup
    networks: 
      - local-net
    restart: always

  thesis-images:
    image: seljmov/thesis-images:amd64
    container_name: thesis-images
    hostname: thesis-images
    ports:
     - 10010:5050
    networks: 
      - local-net
    environment:
     - ASPNETCORE_ENVIRONMENT
     - ConnectionStrings__DefaultConnection=${AUTH_DEFAULT_CONNECTION}
     - JwtOptions__Issuer=${JWT_ISSUER}
     - JwtOptions__Audience=${JWT_AUDIENCE}
     - JwtOptions__Key=${JWT_KEY}
     - JwtOptions__AccessTokenLifetime=${JWT_ACCESS_TOKEN_LIFETIME}
     - JwtOptions__RefreshTokenLifetime=${JWT_REFRESH_TOKEN_LIFETIME}

volumes:
  pg-data: {}
  pg-conf: {}
  pg-log: {}
  pg-backup: {}
  rabbit: {}
  pgadmin: {}
```

В какой-то момент может захотеться посмотреть, что лежит в базе. Сделать это можно при помощи PGADMIN, который тоже можно добавить в docker-compose.
```
  pgadmin:
    image: dpage/pgadmin4
    container_name: pgadmin4
    hostname: pgadmin4
    networks:
      - local-net
    ports:
      - 54320:80
    volumes:
      - pgadmin:/var/lib/pgadmin
    environment:
      - PGADMIN_DEFAULT_EMAIL=${PGADMIN_DEFAULT_EMAIL}
      - PGADMIN_DEFAULT_PASSWORD=${PGADMIN_DEFAULT_PASSWORD}
    depends_on:
      - postgres
    restart: always
```

Добавляем данные в файл окружения.
```
ASPNETCORE_ENVIRONMENT=Production
JWT_ISSUER=test
JWT_AUDIENCE=test
JWT_KEY="test key"
JWT_ACCESS_TOKEN_LIFETIME=15
JWT_REFRESH_TOKEN_LIFETIME=10080
PG_NAME=postgres
PG_PASSWORD=postgres
PGADMIN_DEFAULT_EMAIL=email@example.com
PGADMIN_DEFAULT_PASSWORD=password123
AUTH_DEFAULT_CONNECTION="Host=postgres;Database=DatabaseName;Username=postgres;Password=postgres;Pooling=true;Persist Security Info=true"
```

3. Мониторинг контейнеров

Следить за состояниями контейнеров, логами и тп, позволит Portainer. Поставить его можно следующим образом.
```
$ sudo su
$ docker volume create portainer_data
$ docker run -d -p 8000:8000 -p 9443:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:latest
```