# Мониторинг кластера

Создал ВМ в proxmox через tofu, через ansible galaxy накатил mongo и экспортеры percona, сделал шаблон виртуалки и затем клонировал шаблон и расскатывал через tofu уже непосредственно ноды с mongo. 
Также локально расскатил prometheus + grafana + PMM. Документация везде про это есть, делается довольно просто.

Главное не забыть установить нужные роли для экспортера:
```
admin> db.getUsers()
{
  users: [
    ...,
    {
      _id: 'admin.node_exporter',
      userId: UUID('07728ac7-e3e1-4220-ab10-19d1a78cdfd8'),
      user: 'node_exporter',
      db: 'admin',
      roles: [
        { role: 'clusterMonitor', db: 'admin' },
        { role: 'read', db: 'local' }
      ],
      mechanisms: [ 'SCRAM-SHA-1', 'SCRAM-SHA-256' ]
    },
    {
      _id: 'admin.root',
      userId: UUID('578cd5b6-4a4a-45d9-bf53-2f98d6952462'),
      user: 'root',
      db: 'admin',
      roles: [ { role: 'root', db: 'admin' } ],
      mechanisms: [ 'SCRAM-SHA-1', 'SCRAM-SHA-256' ]
    }
  ],
  ok: 1
}
```

И правильно запустить контейнер экспортера (пароли естественно другие):
```bash
docker run -d -p 9216:9216 --network host --name mongo_exporter percona/mongodb_exporter:0.40 --mongodb.uri=mongodb://node_exporter:securepassword@127.0.0.1:27017 --collect-all --compatible-mode --collector.profile  --discovering-mode
```
Обратить внимание на тип `--network=host` - это обязательно. 
И также обязательны флаги `--compatible-mode --collector.profile  --discovering-mode` - без них не все метрики могут собираться


Хар-ки ВМ:
8GB ram
hdd
CPU: 4 cores, type host - intel core i7


Решил прогнать `mongo-perf` на все тесткейсы, заодно и посмотреть на мониторинг. 
Для начала посмотрим, как хорошо монго справляется с CRUD:
![](https://github.com/alpinoteqdev/wb_mongo_course/blob/main/hw/7/docs_ops.png)

150к на вставку и 100к на чтение выглядит весьма не дурно

Также хорошо справилась не только с CRUD, но и Command:
![](https://github.com/alpinoteqdev/wb_mongo_course/blob/main/hw/7/command_ops.png)

Через мониторинг удобно глянуть эффективность использования индексов:
![](https://github.com/alpinoteqdev/wb_mongo_course/blob/main/hw/7/index_op%20counteres.png)


Но!

Мониторить систему, при этом использую mongo-perf не лучшая идея. Несмотря на пройденный mongo-perf, некоторые дашборды ни о чем не говорят:

Например, статистика о курсорах:
![](https://github.com/alpinoteqdev/wb_mongo_course/blob/main/hw/7/useless_cursor.png)

А вот казалось бы важнейший latency:
![](https://github.com/alpinoteqdev/wb_mongo_course/blob/main/hw/7/latency.png)


При этом сама нода нагружена хорошо (обратите внимание, как низок iowait по сравнению с другими параметрами, а у меня стоит hdd - по сути самое узкое место):
![](https://github.com/alpinoteqdev/wb_mongo_course/blob/main/hw/7/node_summary.png)



До курса я тестировал монго полгода назад по-другому - создал клиентов и имитировал реальную нагрузку. Там и latency приближенный к реальности был, и iowait ощутил. Так что если хотим мониторить систему, то лучше это делать без утилит тестирования) Они предназначены для другого как мне кажется (дать первоначальную картину instance без его эксплуатации).

В целом я пробовал различные варианты мониторинга - от готовых дашбордов графаны (которые не рекомендовал бы использовать, не нашел ничего стоящего) до PMM. И PMM - один из лучших варианто, которое дает очень многое из коробки.

---

Скрипт не делал, расскатывал все через IaC :)
