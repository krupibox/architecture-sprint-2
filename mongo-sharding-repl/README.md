# Шардирование + Репликация 

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

Инициализируем шард №1 с набором реплик

```shell
docker exec -it shard1-node1 mongosh --port 27018
```

```shell
rs.initiate(
    {
      _id : "shard1",
      members: [
        { _id : 0, host : "shard1-node1:27018" },
        { _id : 1, host : "shard1-node2:27018" },
        { _id : 2, host : "shard1-node3:27018" }
      ]
    }
);

exit();
```

Инициализируем шард №2 с набором реплик

```shell
docker exec -it shard2-node1 mongosh --port 27019
```

```shell
rs.initiate(
    {
      _id : "shard2",
      members: [
        { _id : 3, host : "shard2-node1:27019" },
        { _id : 4, host : "shard2-node2:27019" },
        { _id : 5, host : "shard2-node3:27019" }
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
# Добавление реплик как шардов
sh.addShard( "shard1/shard1-node1:27018,shard1-node2:27018,shard1-node3:27018");
sh.addShard( "shard2/shard2-node1:27019,shard2-node2:27019,shard2-node3:27019");

sh.enableSharding("somedb");
sh.shardCollection("somedb.helloDoc", { "name" : "hashed" } );

use somedb;

for(var i = 0; i < 1000; i++) db.helloDoc.insert({age:i, name:"ly"+i});

# Выводим кол-во документов
db.helloDoc.countDocuments();
exit();
```

## Как проверить

Для этого проведем проверку кол-ва документов на шардах и в репликах.

Шард №1 реплика 1

```shell
docker exec -it shard1-node1 mongosh --port 27018
```

```shell
use somedb;
db.helloDoc.countDocuments();
exit();
```

Шард №1 реплика 2

```shell
docker exec -it shard1-node2 mongosh --port 27018
```

```shell
use somedb;
db.helloDoc.countDocuments();
exit();
```

Шард №1 реплика 3

```shell
docker exec -it shard1-node3 mongosh --port 27018
```

```shell
use somedb;
db.helloDoc.countDocuments();
exit();
```

Кол-во документов во всех репликах должно совпадать

Шард №2 реплика 1

```shell
docker exec -it shard2-node1 mongosh --port 27019
```

```shell
use somedb;
db.helloDoc.countDocuments();
exit();
```

Шард №2 реплика 2

```shell
docker exec -it shard2-node2 mongosh --port 27019
```

```shell
use somedb;
db.helloDoc.countDocuments();
exit();
```

Шард №2 реплика 3

```shell
docker exec -it shard2-node3 mongosh --port 27019
```

```shell
use somedb;
db.helloDoc.countDocuments();
exit();
```

Кол-во документов во всех репликах должно совпадать

Сумма документов в обоих шардах (в одной из реплик на каждом) должна быть равна кол-ву документов, которое было получено при проверке на шаге инициализации роутера.