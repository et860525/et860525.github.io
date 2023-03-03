---
title: "Docker: 設定 MongoDB"
date: 2023-03-03
draft: false
author: "Chen Yu Fan"
tags: ["Docker", "MongoDB", "Backend"]
---

使用 Docker 來建置 MongoDB，可以先到 [docker mongo](https://hub.docker.com/_/mongo) 來選擇版本。

MongoDB 會有幾種架構：

1. Standalone
	- 建立難度 **低**
	- 單一 MongoDB 資料庫
2. [Replica Set](https://www.mongodb.com/docs/manual/replication/)
	- 建立難度 **中**
	- 資料會有多個副本提供容錯空間
3. [Sharded Cluster](https://www.mongodb.com/docs/manual/sharding/)
	- 建立難度 **高**
	- 資料放置在不同的 shard，每個 shard 或 config servers 都是 `Replica Set`
  ![Docker-sharded-cluster-production-architecture.png](/images/Docker-mongodb/Docker-sharded-cluster-production-architecture.png)

<!--more-->

# Standalone

Standalone 表示只有一個 MongoDB 實體，Mongo Client 只要直接連接它就可以使用。

> 根據 [Docker compose file](https://docs.docker.com/compose/compose-file/) 所顯示，2023年六月後就不會支援 Compose V1 ，所以除了在 Standalone 會試做一個 V1，其他都會換成 Compose V2 的格式

## Docker compose V1

使用 `docker-compose` 來建立 MongoDB：

```yml
# docker-compose.yml
versio: '3.1'

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

## Docker compose V2

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

---

# Replica Set

[MongoDB Replication](https://www.mongodb.com/docs/manual/replication/) 可以建立一份資料庫副本，在主要的資料庫發生問題時，副本可以直接接手主要資料庫的工作。

使用 Replica Set 的重點：
- Replica Set 會提供 [High availability](https://www.mongodb.com/docs/manual/reference/glossary/#std-term-high-availability)
- 依照硬體規格來調整 Replica Set，例如：更大更快的硬碟
- 不會影響 Primary，使用 [Read Preference](https://www.mongodb.com/docs/manual/core/read-preference/) 來讀取更快更近的副本

## MongoDB Replication 的架構

MongoDB Replication 主要的架構的構成：

![Docker-replica-set-read-write-operations-primary.png](/images/Docker-mongodb/Docker-replica-set-read-write-operations-primary.png)

[Replica Set Member](https://www.mongodb.com/docs/manual/core/replica-set-members/#std-label-replica-set-primary-member) 有兩種：

- `Primary`：接受所有讀寫的操作
- `Secondaries`：複製 `Primary` 的資料

最基本的 Replica Set 需要有三個**數據承載成員 ( data-bearing members )**：一個 `Primary` 與二個 `Secondary`：

![Docker-replica-set-primary-with-two-secondaries.png](/images/Docker-mongodb/Docker-replica-set-primary-with-two-secondaries.png)

在某些情況下 ( 例如：硬體限制 ) 可以將某個 member 改為 [arbiter](https://www.mongodb.com/docs/manual/core/replica-set-arbiter/)，它只會參與 [投票 ( elections )](https://www.mongodb.com/docs/manual/core/replica-set-elections/#std-label-replica-set-elections) 並不會擁有任何資料，更不會成為 `Primary`：

![Docker-replica-set-primary-secondary-arbiter.png](/images/Docker-mongodb/Docker-replica-set-primary-secondary-arbiter.png)

以下使用 Docker compose 來建構 Replica Set。

## Docker compose

使用 Docker compose 來建立下方的架構：

![Docker-replica-set-compose.png](/images/Docker-mongodb/Docker-replica-set-compose.png)

- 這一個 Replica Set 命名為 `RS`，需要使用 `--replSet RS`
- 建立三個 container 來滿足最基本的 Replica Set：
	- container (`rs1`/`rs2`/`rs3`) ；port ( `27041`/`27042`/`27043`)

`docker-compose-replica-set.yaml`：

```yaml
services:
  rs1:
    image: mongo:4.4.19-focal
    container_name: rs1
    network_mode: host
    command: mongod --replSet RS --port 27041 --dbpath /data/db --config /resource/mongod.yaml
    volumes:
      - ./replica/config/mongod.yaml:/resource/mongod.yaml
      - ./replica/data/rs1:/data/db
    extra_hosts:
      - "rs1.local:127.0.0.1"
      - "rs2.local:127.0.0.1"
      - "rs3.local:127.0.0.1"
  rs2:
    image: mongo:4.4.19-focal
    container_name: rs2
    network_mode: host
    command: mongod --replSet RS --port 27042 --dbpath /data/db --config /resource/mongod.yaml
    volumes:
      - ./replica/config/mongod.yaml:/resource/mongod.yaml
      - ./replica/data/rs2:/data/db
    extra_hosts:
      - "rs1.local:127.0.0.1"
      - "rs2.local:127.0.0.1"
      - "rs3.local:127.0.0.1"
  rs3:
    image: mongo:4.4.19-focal
    container_name: rs3
    network_mode: host
    command: mongod --replSet RS --port 27043 --dbpath /data/db --config /resource/mongod.yaml
    volumes:
      - ./replica/config/mongod.yaml:/resource/mongod.yaml
      - ./replica/data/rs3:/data/db
    extra_hosts:
      - "rs1.local:127.0.0.1"
      - "rs2.local:127.0.0.1"
      - "rs3.local:127.0.0.1"
```

- `network_mode` 設定為 `host`，資料庫會如同架設在本機一樣，讓 `rs1.local`、`rs2.local` 與 `rs3.local` 都會對應到 `127.0.0.1`
- `command` 裡面的設定：
	- `--dbpath`：指定 MongoDB 存放資料的資料夾
	- `--port`：指定 MongoDB 要開啟的 Port
	- `--config`：指定 MongoDB Config file
- `volumes` 裡面的設定：
	- `./replica/config/mongod.yaml:/resource/mongod.yaml`：其格式為 `A:B`，A 為主機端的路徑；B 為容器內的路徑。當容器掛載完成，容器端就會有 `/resource/mongod.yaml` 檔案，達到路徑能共享資料的功能
	- `./replica/data/rs3:/data/db`：與上方設定很像，資料共享與綁定。當目前的容器被刪除後，其資料都會被存放在本機端，等待下一個容器指向相同的位置，就可以繼續使用以前的資料
- `extra_hosts` 設定解析的域名 `rs1.local`、`rs2.local` 與 `rs3.local` 都會解析 `127.0.0.1`，這樣就不用再打 `127.0.0.1`

接下來設定 MongoDB config file `replica/config/mongod.yaml`：

```yaml
net:
  bindIpAll: true
storage:
  engine: wiredTiger
  wiredTiger:
    engineConfig:
      cacheSizeGB: 0.1
```

- [net Option](https://www.mongodb.com/docs/manual/reference/configuration-options/#net-options) 裡的 `bindIpAll`：設定那些 Client IP 可以連進 MongoDB，預設為 `localhost` 也就是只有本機。假設 Client IP 來自不同的主機，設定 `true` 就表示任何 Client IP 都可以連進來
- [storage Option](https://www.mongodb.com/docs/manual/reference/configuration-options/#storage-options) 裡的 `engine`：有 `wiredTiger` 與 `inMemory` 兩種，不論選擇哪一個都要小心設定 `cacheSizeGB` 或 `inMemorySizeGB` 的大小。這是因為 Mongo 會依照硬體配置來計算[內部緩存的預設值](https://www.mongodb.com/docs/manual/reference/configuration-options/#mongodb-setting-storage.wiredTiger.engineConfig.cacheSizeGB)，而 Mongo 並不會知道自己運行在 Docker 容器中，所以在同一個 Docker 上架設兩台以上的 Mongo，就有可能會出現問題，如：斷線或無法連線進去。所以使用 Docker 架設 Mongo **一定要設定**

以上只用使用幾個重要的設定，完整的 Mongo config 可以到 [Configuration File Options](https://www.mongodb.com/docs/manual/reference/configuration-options/)。

都完成後，使用 `docker compose up -d` 來運行。

## Replica Set 設定

### 確認資料庫連線狀況

進入 **rs1** 容器，來測試能否用域名連線到其他資料庫：

```bash
docker exec -it rs1 bash
```

確認資料庫連線：

```bash
mongo rs1.local:27041 --eval "print('rs1 ok')"
mongo rs2.local:27042 --eval "print('rs2 ok')"
mongo rs3.local:27043 --eval "print('rs3 ok')"
```

如果都能顯示這些字串，就可以開始設定 Replica Set。

### 設定 Replica Set

隨便進入一個 member：

```bash
mongo rs1.local:27041
```

設定 Replica Set config：

```bash
cfg = {
  "_id": "RS",
  "members": [{
      "_id": 0,
      "host": "rs1.local:27041"
    },
    {
      "_id": 1,
      "host": "rs2.local:27042"
    },
    {
      "_id": 2,
      "host": "rs3.local:27043"
    }
  ]
};
rs.initiate(cfg);
```

顯示 `ok: 1` 就表示完成。

### 確認 Replica Set 狀態

當要增加或減少 member 時，或是要檢查 member 是否有連線，就要確認 member 的狀態。最常用的有三種：

- `rs.status()`：members 的狀態
	```bash
	rs.status().members.forEach(m => print(`${m.name} =>  ${m.stateStr}`))
	```
	會列出：
	```bash
	rs1.local:27041 =>  PRIMARY
	rs2.local:27042 =>  SECONDARY
	rs3.local:27043 =>  SECONDARY
	```
	當然也可以直接使用 `rs.status()` 來看更詳細的狀態。
- `rs.printSecondaryReplicationInfo()`：members 的同步狀態
	```bash
	source: rs2.local:27042
        syncedTo: Fri Mar 03 2023 11:50:04 GMT+0000 (UTC)
        0 secs (0 hrs) behind the primary
    source: rs3.local:27043
        syncedTo: Fri Mar 03 2023 11:50:04 GMT+0000 (UTC)
        0 secs (0 hrs) behind the primary
	```
- `rs.printReplicationInfo()`：members 的 [oplog](https://www.mongodb.com/docs/manual/reference/glossary/#std-term-oplog) 狀態
	```bash
	configured oplog size:   6040.000781059265MB
	log length start to end: 18735secs (5.2hrs)
	oplog first event time:  Fri Mar 03 2023 06:33:09 GMT+0000 (UTC)
	oplog last event time:   Fri Mar 03 2023 11:45:24 GMT+0000 (UTC)
	now:                     Fri Mar 03 2023 11:45:31 GMT+0000 (UTC)
	```

# 結語

起初會接觸到 Replica Set 的原因，是出自於我想使用 Prisma 連接 local MongoDB。不過當連接時就會跳出錯誤：

```bash
prisma needs to perform transactions, which requires your mongodb server to be run as a replica set.
```

這也讓我想了解一下這個 Replica Set 到底是什麼東西。

而看了該文章才知道，在 Docker 裡建立 MongoDB 時需要設定 `cacheSizeGB` 來設定緩存大小。可能是通常我也只會使用到一個的 MongoDB 資料庫，所以才沒有遇到那些問題。

---

# Reference

- [[資料庫]使用 Docker 構築不同 MongoDB 架構 (三) - Replica Set](https://ithelp.ithome.com.tw/articles/10227851)