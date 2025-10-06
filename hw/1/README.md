Установил mongo на виртуалку proxmox'a. 

Не забываем запустить:
```bash
root@stage-pirogovlabs-0:~#  sudo systemctl enable mongod
root@stage-pirogovlabs-0:~#  sudo systemctl start mongod
```

Заходим в нее
```bash
root@stage-pirogovlabs-0:~#  mongosh
```

И создаем случайное количество документов
```bash
test> db.collection.insertMany(
...   Array.from({length: Math.floor(Math.random() * 20) + 1}, () => ({
...     foo: Math.floor(Math.random() * 100)
...   }))
... )
{
  acknowledged: true,
  insertedIds: {
    '0': ObjectId('68e38b391a87682a9ace5f56'),
    '1': ObjectId('68e38b391a87682a9ace5f57'),
    '2': ObjectId('68e38b391a87682a9ace5f58'),
    '3': ObjectId('68e38b391a87682a9ace5f59'),
    '4': ObjectId('68e38b391a87682a9ace5f5a'),
    '5': ObjectId('68e38b391a87682a9ace5f5b'),
    '6': ObjectId('68e38b391a87682a9ace5f5c'),
    '7': ObjectId('68e38b391a87682a9ace5f5d'),
    '8': ObjectId('68e38b391a87682a9ace5f5e'),
    '9': ObjectId('68e38b391a87682a9ace5f5f')
  }
}

```

И посчитает количество документов
```bash
test> db.collection.countDocuments()
10
```

Или так
```bash
test> db.collection.estimatedDocumentCount()
10
```
