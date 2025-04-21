# mongo-sharding

## Как запустить

Запускаем mongodb и приложение

```shell
docker compose up -d
```

Подключаемся к серверу конфигурации и проводим инициализацию

```shell
docker exec -it configSrv mongosh --port 27017
```

```shell
rs.initiate(
  {
    _id : "config_server",
       configsvr: true,
    members: [
      { _id : 0, host : "configSrv:27017" }
    ]
  }
);

exit();
```

Инициализируем шард №1

```shell
docker exec -it shard1 mongosh --port 27018
```

```shell
rs.initiate(
    {
      _id : "shard1",
      members: [
        { _id : 0, host : "shard1:27018" }
      ]
    }
);

exit();
```

Инициализируем шард №2

```shell
docker exec -it shard2 mongosh --port 27019
```

```shell
rs.initiate(
    {
      _id : "shard2",
      members: [
        { _id : 1, host : "shard2:27019" }
      ]
    }
);

exit();
```

Инициализируем роутер и наполняем его тестовыми данными

```shell
docker exec -it mongos_router mongosh --port 27020
```

```shell
sh.addShard("shard1/shard1:27018");
sh.addShard("shard2/shard2:27019");
sh.enableSharding("somedb");
sh.shardCollection("somedb.helloDoc", { "name": "hashed" });

use somedb;

for(var i = 0; i < 1000; i++) db.helloDoc.insert({age:i, name:"ly"+i});

# Выводим кол-во документов
db.helloDoc.countDocuments();
exit();
```

## Как проверить

Для этого проведем проверку кол-ва документов на шардах.

Шард №1

```shell
docker exec -it shard1 mongosh --port 27018
```

```shell
use somedb;
db.helloDoc.countDocuments();
exit();
```

Шард №2

```shell
docker exec -it shard2 mongosh --port 27019
```

```shell
use somedb;
db.helloDoc.countDocuments();
exit();
```

Сумма документов в обоих шардах должна быть равна кол-ву документов, которое было получено при проверке на шаге инициализации роутера.