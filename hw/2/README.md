Создадим две коллекции.


Коллекцию заказов/товаров:
```bash
test> db.orders.insertMany(Array.from({length: 20}, (_, i) => ({
...     orderId: i + 1,
...     product: "Product" + (Math.floor(Math.random() * 10) + 1),
...     quantity: Math.floor(Math.random() * 100) + 1,
...     price: Math.floor(Math.random() * 1000) + 10,
...     warehouseId: Math.floor(Math.random() * 5) + 1
... })))
```

Коллекцию складов:
```bash
test> db.warehouses.insertMany(Array.from({length: 5}, (_, i) => ({
...     warehouseId: i + 1,
...     name: "Warehouse" + (i + 1),
...     location: ["North", "South", "East", "West", "Central"][i],
...     capacity: Math.floor(Math.random() * 10000) + 5000
... })))
```



## Aggregation framework

Запрос, который не только показываем количество товаров с разных складов, но и для доп инфы покажем сколько в каждом складе товаров
```bash
db.orders.aggregate([
  {
    $group: {
      _id: {
        product: "$product",
        warehouseId: "$warehouseId"
      },
      quantityPerWarehouse: { $sum: "$quantity" }
    }
  },
  {
    $group: {
      _id: "$_id.product",
      totalQuantity: { $sum: "$quantityPerWarehouse" },
      warehouses: {
        $push: {
          warehouseId: "$_id.warehouseId",
          quantity: "$quantityPerWarehouse"
        }
      }
    }
  },
  {
    $sort: { totalQuantity: -1 }
  }
])
```

Результат:
```bash
[
  {
    _id: 'Product8',
    totalQuantity: 257,
    warehouses: [
      { warehouseId: 3, quantity: 162 },
      { warehouseId: 4, quantity: 13 },
      { warehouseId: 2, quantity: 82 }
    ]
  },
  {
    _id: 'Product1',
    totalQuantity: 203,
    warehouses: [
      { warehouseId: 1, quantity: 90 },
      { warehouseId: 4, quantity: 80 },
      { warehouseId: 3, quantity: 33 }
    ]
  },
  {
    _id: 'Product7',
    totalQuantity: 200,
    warehouses: [ { warehouseId: 5, quantity: 200 } ]
  },
  {
    _id: 'Product3',
    totalQuantity: 106,
    warehouses: [
      { warehouseId: 2, quantity: 42 },
      { warehouseId: 3, quantity: 64 }
    ]
  },
  {
    _id: 'Product4',
    totalQuantity: 89,
    warehouses: [
      { warehouseId: 4, quantity: 55 },
      { warehouseId: 5, quantity: 34 }
    ]
  },
  {
    _id: 'Product5',
    totalQuantity: 64,
    warehouses: [
      { warehouseId: 2, quantity: 26 },
      { warehouseId: 1, quantity: 38 }
    ]
  },
  {
    _id: 'Product9',
    totalQuantity: 48,
    warehouses: [ { warehouseId: 1, quantity: 48 } ]
  },
  {
    _id: 'Product10',
    totalQuantity: 19,
    warehouses: [ { warehouseId: 4, quantity: 19 } ]
  },
  {
    _id: 'Product2',
    totalQuantity: 15,
    warehouses: [ { warehouseId: 5, quantity: 15 } ]
  }
]
```

## Map Reduce

И аналогично для MapReduce:

```bash
test> db.orders.mapReduce(
...   function() {
...     emit(this.product, {
...       warehouseQuantities: [{ warehouseId: this.warehouseId, quantity: this.quantity }],
...       totalQuantity: this.quantity
...     });
...   },
...   function(key, values) {
...     var result = { warehouseQuantities: [], totalQuantity: 0 };
...
...     values.forEach(function(value) {
...       result.totalQuantity += value.totalQuantity;
...       result.warehouseQuantities = result.warehouseQuantities.concat(value.warehouseQuantities);
...     });
...
...     return result;
...   },
...   {
...     out: { inline: 1 },
...     finalize: function(key, reducedValue) {
...       // Group quantities by warehouse
...       var warehouseMap = {};
...       reducedValue.warehouseQuantities.forEach(function(item) {
...         if (!warehouseMap[item.warehouseId]) {
...           warehouseMap[item.warehouseId] = 0;
...         }
...         warehouseMap[item.warehouseId] += item.quantity;
...       });
...
...       // Convert back to array format
...       var warehouses = [];
...       for (var warehouseId in warehouseMap) {
...         warehouses.push({
...           warehouseId: parseInt(warehouseId),
...           quantity: warehouseMap[warehouseId]
...         });
...       }
...
...       return {
...         totalQuantity: reducedValue.totalQuantity,
...         warehouses: warehouses
...       };
...     }
...   }
... )
```

Хоть и запрос получился более громоздким, результат неизменнен:
```bash
{
  results: [
    {
      _id: 'Product3',
      value: {
        totalQuantity: 106,
        warehouses: [
          { warehouseId: 2, quantity: 42 },
          { warehouseId: 3, quantity: 64 }
        ]
      }
    },
    {
      _id: 'Product10',
      value: {
        totalQuantity: 19,
        warehouses: [ { warehouseId: 4, quantity: 19 } ]
      }
    },
    {
      _id: 'Product1',
      value: {
        totalQuantity: 203,
        warehouses: [
          { warehouseId: 1, quantity: 90 },
          { warehouseId: 3, quantity: 33 },
          { warehouseId: 4, quantity: 80 }
        ]
      }
    },
    {
      _id: 'Product4',
      value: {
        totalQuantity: 89,
        warehouses: [
          { warehouseId: 4, quantity: 55 },
          { warehouseId: 5, quantity: 34 }
        ]
      }
    },
    {
      _id: 'Product5',
      value: {
        totalQuantity: 64,
        warehouses: [
          { warehouseId: 1, quantity: 38 },
          { warehouseId: 2, quantity: 26 }
        ]
      }
    },
    {
      _id: 'Product8',
      value: {
        totalQuantity: 257,
        warehouses: [
          { warehouseId: 2, quantity: 82 },
          { warehouseId: 3, quantity: 162 },
          { warehouseId: 4, quantity: 13 }
        ]
      }
    },
    {
      _id: 'Product7',
      value: {
        totalQuantity: 200,
        warehouses: [ { warehouseId: 5, quantity: 200 } ]
      }
    },
    {
      _id: 'Product9',
      value: {
        totalQuantity: 48,
        warehouses: [ { warehouseId: 1, quantity: 48 } ]
      }
    },
    {
      _id: 'Product2',
      value: {
        totalQuantity: 15,
        warehouses: [ { warehouseId: 5, quantity: 15 } ]
      }
    }
  ],
  ok: 1
}
```
