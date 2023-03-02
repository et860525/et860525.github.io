---
title: "Docker: 設定 MongoDB"
date: 2023-03-02
draft: True
author: "Chen Yu Fan"
tags: ["Docker", "MongoDB"]
---

在測試網頁時，我都會使用 [Docker](https://www.docker.com/) 來建置 [MongoDB](https://www.mongodb.com/) 來存取資料。本來都是使用 `docker run` 的方式來建立，不過常常會需要重新建立資料庫，又或是在不同的電腦上使用時，就會需要使用 [Docker Compose file](https://docs.docker.com/compose/compose-file/) 來先把組態寫好，在使用同樣的一串指令就可以獲得一樣的環境。

---

使用 Docker 來建置 MongoDB，可以先到 [docker mongo](https://hub.docker.com/_/mongo) 來選擇自己需要的版本。

> 2023/3/2 - 根據 [Docker Compose file](https://docs.docker.com/compose/compose-file/) 所顯示，2023 六月後就不會支援 Compose V1 ，所以這裡會換成 Compose V2

<!--more-->

# Standalone

Standalone 架構表示只有一個 MongoDB 實體，Mongo Client 只要直接連接它就可以使用。

> 只會在 Standalone 顯示 Compose V1 實作，後面都會以 Compose V2 來實作

## Docker Compose V1

使用 `docker-compose` 來建立 MongoDB：

```yml
# docker-compose.yml
versio: '3.1'
volumes:
  mongo-mount-data:

services:
  mongo:
    image: mongo:latest
    container_name: mongodb
    environment:
      MONGO_INITDB_ROOT_USERNAME: root
      MONGO_INITDB_ROOT_PASSWORD: example
    ports:
      - '27017:27017'
    volumes:
      - mongo-mount-data:/data/db
```

運行 `docker-compose`：

```bash
# Start
docker-compose up -d

# Close
docker-compose down
```

## Docker Compose V2

```yaml
# docker-compose.yaml
services:
  mongo:
    image: mongo:latest
    container_name: mongodb-test
    environment:
      MONGO_INITDB_ROOT_USERNAME: root
      MONGO_INITDB_ROOT_PASSWORD: example
    ports:
      - '27017:27017'
    volumes:
      - mongo-mount-data:/data/db

volumes:
  mongo-mount-data:
```

運行 `docker compose`：

```bash
# Start
docker compose up -d

# Close
docker compose down
```

## 進入 mongoDB 

```bash
docker exec -it <mongo-name> bash
```

輸入後會看到命令列會由 `root` 開頭，接著再輸入：

```bash
# 5.0+
mongosh -u root -p

# 4.0
mongo -u root -p
```

輸入密碼後，就可以進入 MongoDB 了。

## 使用 VM 會遇到的問題

如果在 VM 下使用 MongoDB 5.0+ 以上的版本，有可能會看到：

```bash
WARNING: MongoDB 5.0+ requires a CPU with AVX support, and your current system does not appear to have that!
```

根據 [Mongo DB deployment not working in kubernetes because processor doesn't have AVX support](https://stackoverflow.com/questions/70818543/mongo-db-deployment-not-working-in-kubernetes-because-processor-doesnt-have-avx)

如果是 Windows + VirtualBox 可以在 CMD 輸入：

```powershell
bcdedit /set hypervisorlaunchtype off
DISM /Online /Disable-Feature:Microsoft-Hyper-V
```

重開機後就可以解決。

> 目前我這裡還是有使用 WSL2，如果輸入以上指令可能會造成 WSL2 無法使用，所以我選擇使用舊版本，暫且不使用 `mongo:5.0+`。

