# Настройка аутентификации.

## Настройка аутентификации

Просто открыл доку монги и проследовал по шагам:

```js
db.createUser({user:"admin", pwd:"secret", roles:[{role: "userAdminAnyDatabase", db:"admin"}, {role:"clusterAdmin", db:"admin"}, {role:"readWriteAnyDatabase", db:"admin"}, {role:"dbAdminAnyDatabase", db:"admin"}]})
```

Настроил `/etc/mongod.conf` (закомментированные поля убрал для читаемости)
```
storage:
  dbPath: /var/lib/mongodb
#  engine: wiredTiger
#  engine: inMemory

systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

net:
  port: 27017
  bindIp: 10.0.20.1

security:
  authorization: enabled

auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit.log

```

И перезапустил
```bash
sudo systemctl restart mongod
```


## Проверяем работу аутентификации + проверка с import/export

Создадим пользователя в БД `premier-league`:
```js
db.createUser({ user: "pirogov", pwd: "pirogov!", roles: [ { role: "readWrite", db: "premier-league" } ] })
```

Попробуем зайти без корректных кредов
```js
root@dev-mongo-0:~# mongosh premier-league --host 10.0.20.1 --port 27017 -u pirogov -p qwer --authenticationDatabase=premier-league
Current Mongosh Log ID:	68f61c874b95070c9e80d710
Connecting to:		mongodb://<credentials>@10.0.20.1:27017/premier-league?directConnection=true&authSource=premier-league&appName=mongosh+2.5.6
MongoServerError: Authentication failed.
```

А с корректными все получается
```js
root@dev-mongo-0:~# mongosh premier-league --host 10.0.20.1 --port 27017 -u pirogov -p pirogov! --authenticationDatabase=premier-league
Current Mongosh Log ID:	68f61ca89e4922702d80d710
Connecting to:		mongodb://<credentials>@10.0.20.1:27017/laliga?directConnection=true&authSource=premier-league&appName=mongosh+2.5.6
Using MongoDB:		8.0.12-4
Using Mongosh:		2.5.6
mongosh 2.5.8 is available for download: https://www.mongodb.com/try/download/shell

For mongosh info see: https://www.mongodb.com/docs/mongodb-shell/

premier-league>
```

Проверим работу import и export

### import
Без авторизации сделать import не удается
```bash
root@dev-mongo-0:~# mongoimport --port 27017 -d premier-league -c season11-12 --file /home/pirogov/csvjson.json --jsonArray --host 10.0.20.1
2025-10-19T10:15:36.601-0400	connected to: mongodb://10.0.20.1:27017/
2025-10-19T10:15:36.629-0400	Failed: (Unauthorized) Command insert requires authentication
```

А вот с авторизацией - без проблем 
```bash
root@dev-mongo-0:~# mongoimport --port 27017 -d premier-league -c season11-12 --file /home/pirogov/csvjson.json --jsonArray --host 10.0.20.1 -u pirogov -p pirogov! --authenticationDatabase=premier-league
2025-10-19T10:16:08.732-0400	connected to: mongodb://10.0.20.1:27017/
2025-10-19T10:16:08.973-0400	380 document(s) imported successfully. 0 document(s) failed to import.
```


### export
Без авторизации сделать export не удается
```bash
root@dev-mongo-0:~# mongoexport --collection=season11_12 --db=premier-league --out=events.json
2025-10-19T10:19:09.272-0400	connected to: mongodb://localhost/
2025-10-19T10:19:09.273-0400	Failed: (Unauthorized) Command listCollections requires authentication
```


А вот с авторизацией - без проблем 
```bash
root@dev-mongo-0:~# mongoexport --collection=season11_12 --db=premier-league --out=pl_season_11_12.json -u pirogov -p pirogov! --authenticationDatabase=premier-league --host 10.0.20.1
2025-10-19T10:19:46.711-0400	connected to: mongodb://localhost/
2025-10-19T10:19:46.729-0400	exported 380 records
```

# Настройка валидации

На коллекции `cars` создадим базовую валидацию - пусть число будет положительным и модель обязательно должны быть указаны "make", "model", "year", "price".
```js
autopark> db.createCollection("cars", {
...   validator: {
...     $jsonSchema: {
...       bsonType: "object",
...       required: ["make", "model", "year", "price"],
...       properties: {
...         model: {
...           bsonType: "string",
...           description: "must be a string and is required"
...         },
...         price: {
...           bsonType: "double",
...           minimum: 0,
...           description: "must be a positive number and is required"
...         }
...       }
...     }
...   },
...   validationLevel: "strict",
...   validationAction: "error"
... });
{ ok: 1 }
```

Попробуем добавить машину с отрицательным числом:
```js
autopark> db.cars.insertOne({ make: "Ford", model: "Focus", price: -10000.50, "year": 2025 });
Uncaught:
MongoServerError: Document failed validation
Additional information: {
  failingDocumentId: ObjectId('68f6208950e763875480d717'),
  details: {
    operatorName: '$jsonSchema',
    schemaRulesNotSatisfied: [
      {
        operatorName: 'properties',
        propertiesNotSatisfied: [
          {
            propertyName: 'price',
            description: 'must be a positive number and is required',
            details: [
              {
                operatorName: 'minimum',
                specifiedAs: { minimum: 0 },
                reason: 'comparison failed',
                consideredValue: -10000.5
              }
            ]
          }
        ]
      }
    ]
  }
}
```

Без указаания года:
```js
autopark> db.cars.insertOne({ make: "Ford", model: "Focus", price: 100.500, });
Uncaught:
MongoServerError: Document failed validation
Additional information: {
  failingDocumentId: ObjectId('68f620a250e763875480d718'),
  details: {
    operatorName: '$jsonSchema',
    schemaRulesNotSatisfied: [
      {
        operatorName: 'required',
        specifiedAs: { required: [ 'make', 'model', 'year', 'price' ] },
        missingProperties: [ 'year' ]
      }
    ]
  }
}
```

А если все указать:
```js
autopark> db.cars.insertOne({ make: "Ford", model: "Focus", price: 100.50, "year": 2025 });
{
  acknowledged: true,
  insertedId: ObjectId('68f620b850e763875480d719')
}
```

Из интересного - если мы укажем цену даже в виде вещественного числа `100.00` - валидатор не пропустит:
```js
autopark> db.cars.insertOne({ make: "Ford", model: "Focus", price: 100.00, "year": 2025 });
Uncaught:
MongoServerError: Document failed validation
Additional information: {
  failingDocumentId: ObjectId('68f620cd50e763875480d71a'),
  details: {
    operatorName: '$jsonSchema',
    schemaRulesNotSatisfied: [
      {
        operatorName: 'properties',
        propertiesNotSatisfied: [
          {
            propertyName: 'price',
            description: 'must be a positive number and is required',
            details: [
              {
                operatorName: 'bsonType',
                specifiedAs: { bsonType: 'double' },
                reason: 'type did not match',
                consideredValue: 100,
                consideredType: 'int'
              }
            ]
          }
        ]
      }
    ]
  }
}
```

# mongo-perf

mongo-perf работает только с клиентом mongo 5.0, что очень неудобно. Я малень переписал их Dockerfile, чтобы заходил в контейнер через bash и оттуда запускал бенч. Без авторизации не работает. 

```Dockerfile
FROM mongo:5.0.8

RUN groupadd -r mongo-shell && useradd -r -g mongo-shell mongo-shell

RUN apt-get update -y \
    &&  apt-get install -y python3 python3-pip \
    && rm -rf /var/lib/apt/lists/*


WORKDIR /workdir

COPY requirements.txt .

RUN pip3 install -r requirements.txt

COPY . .

RUN chown -R mongo-shell:mongo-shell .

USER mongo-shell

#ENTRYPOINT ["python3", "benchrun.py"] # убрал
ENTRYPOINT ["bash"]
CMD []
```


Я запустил на двух конфигурациях
1 cpu 4gb ram - узнать как монго отработает на минимальном конфиге

Команда запуска
```bash
python3 benchrun.py -f testcases/simple_commands.js testcases/simple_in_queries.js testcases/simple_multi_update.js testcases/simple_sort_queries.js testcases/simple_insert.js testcases/simple_query.js -t 1 --includeFilter insert update -u admin -p secret --host 10.0.20.1 --trialTime 5
```

4 cpu 8gb ram - узнать как монго отработает на более-менее конфиге

Команда запуска
```bash
python3 benchrun.py -f testcases/simple_commands.js testcases/simple_in_queries.js testcases/simple_multi_update.js testcases/simple_sort_queries.js testcases/simple_insert.js testcases/simple_query.js -t 1 --includeFilter insert update -u admin -p secret --host 10.0.20.1 --trialTime 5
```

Обе команды запускают со временем теста в 5 секунд на определенных тестах из `testcase` и исполняют часть команд.

Диск- hdd.



Результат:
| Test Name | 1 CPU / 4 GB RAM (ops/sec) | 4 CPU / 8 GB RAM (ops/sec) | Performance Change |
|-----------|----------------------------|----------------------------|-------------------|
| **MultiUpdate.Uncontended.SingleDoc.NoIndex** | 7,572.52 | 29,176.53 | +285.26% |
| **MultiUpdate.Uncontended.SingleDoc.Indexed** | 7,198.41 | 26,969.84 | +274.68% |
| **MultiUpdate.Uncontended.TwoDocs.NoIndex** | 5,366.44 | 20,528.53 | +282.53% |
| **MultiUpdate.Uncontended.TwoDocs.Indexed** | 4,850.49 | 17,835.92 | +267.67% |
| **MultiUpdate.Contended.Low.NoIndex** | 7,483.69 | 27,099.43 | +262.14% |
| **MultiUpdate.Contended.Low.Indexed** | 6,915.79 | 26,458.64 | +282.58% |
| **MultiUpdate.Contended.Medium.NoIndex** | 45.42 | 151.76 | +234.16% |
| **MultiUpdate.Contended.Medium.Indexed** | 25.69 | 95.31 | +270.98% |
| **MultiUpdate.Contended.Hot.NoIndex** | 2,812.12 | 9,803.53 | +248.62% |
| **MultiUpdate.Contended.Hot.Indexed** | 1,958.41 | 7,124.02 | +263.76% |
| **MultiUpdate.Contended.Doc.Seq.NoIndex** | 7,822.92 | 23,626.89 | +202.06% |
| **MultiUpdate.Contended.Doc.Seq.Indexed** | 7,143.98 | 20,870.13 | +192.14% |
| **MultiUpdate.Contended.Doc.Rnd.NoIndex** | 8,817.96 | 31,669.39 | +259.24% |
| **MultiUpdate.Contended.Doc.Rnd.Indexed** | 9,128.83 | 29,341.86 | +321.43% |
| **Insert.Empty** | 9,595.90 | 35,514.03 | +270.10% |
| **Insert.EmptyCapped** | 8,541.45 | 16,536.15 | +93.60% |
| **Insert.EmptyCapped.SeqIntID** | 8,281.86 | 16,004.16 | +93.24% |
| **Insert.JustID** | 9,568.05 | 35,994.46 | +276.20% |
| **Insert.IntVector** | 2,993.44 | 8,799.92 | +193.97% |
| **Insert.SeqIntID.Indexed** | 9,056.82 | 33,643.52 | +271.47% |
| **Insert.IntIDUpsert** | 7,401.38 | 28,116.65 | +279.88% |
| **Insert.JustNum** | 9,235.43 | 33,755.97 | +265.49% |
| **Insert.JustNumIndexed** | 9,066.86 | 34,092.50 | +275.99% |
| **InsertIndexedStringsSimpleCollation** | 8,900.79 | 33,280.06 | +273.90% |
| **InsertIndexedStringsNonSimpleCollation** | 8,650.35 | 27,583.32 | +218.87% |
| **Insert.UniqueIndex** | 8,560.31 | 28,188.14 | +229.29% |
| **Insert.UniqueIndexCompound** | 8,423.43 | 30,817.12 | +265.85% |
| **Insert.UniqueIndexCompoundReverse** | 8,398.77 | 25,909.34 | +208.49% |
| **Insert.LargeDocVector** | 1,953.81 | 5,154.11 | +163.80% |

4 ядра и 8 оперативки дают прирост почте везде в 2-2.5 раза. Очень недурно для моего старого сервера :) 
