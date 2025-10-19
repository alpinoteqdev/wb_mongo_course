Я с помощью LLM создал скрипт, который создает 100 документов, у которых как минимум 20 полей - строк. 
Пример документа:

```javascript
{
  _id: ObjectId('68f4e2400256dc10e580d7d9'),
  productId: 'PROD-1001',
  productName: "L'Oreal Automotive Parts Model 1",
  productCode: 'CODE-2SGJLD',
  skuNumber: 'SKU-10001-3',
  productCategory: 'Automotive Parts',
  productSubcategory: 'Automotive Parts Subcategory 3',
  productType: 'Automotive Parts Type 1',
  productLine: "L'Oreal Automotive Parts Line",
  productFamily: "L'Oreal Family Series",
  brandName: "L'Oreal",
  manufacturer: "L'Oreal Manufacturing Inc.",
  manufacturerCode: "MFR-L'O1",
  supplierName: "L'Oreal Global Supplies",
  distributor: 'Worldwide Distribution Corp',
  shortDescription: "High-quality automotive parts from L'Oreal with advanced features",
  longDescription: "This premium automotive parts product from L'Oreal offers exceptional performance and reliability. Designed for modern consumers who value quality and innovation.",
  keyFeatures: 'Feature 1: Vibration Dampening, Feature 2: Warranty Backed, Feature 3: Direct Fit Replacement, Feature 4: Professional Mechanic Recommended, Feature 5: Performance Guaranteed, Feature 6: Corrosion Resistant Coating, Feature 7: Professional Grade Materials',
  productBenefits: 'Saves Time, Increases Efficiency, Cost-Effective, Environmentally Friendly, Easy to Use',
  color: 'Blue',
  material: 'Silicon',
  sizeDescription: 'Standard Size 1',
  weightClass: 'Class 3',
  dimensions: '22x17x15 cm',
  technologyType: 'Advanced Automotive Parts Technology',
  compatibility: 'Universal Compatibility with Standard Systems',
  connectivity: 'Wireless and Wired Connectivity Options',
  powerRequirements: '12V / 4A',
  usageInstructions: 'Follow manufacturer guidelines for optimal performance and safety',
  applicationArea: 'Home, Office, Commercial, and Industrial Use',
  targetAudience: 'General Consumers, Professionals, Enthusiasts',
  recommendedEnvironment: 'Indoor and Outdoor Use with Proper Care',
  qualityGrade: 'Standard',
  certification: 'ISO 9001, CE Certified, RoHS Compliant',
  safetyStandards: 'International Safety Standards Compliant',
  environmentalRating: 'Eco-Friendly Grade 2',
  packagingType: 'Retail Box Packaging',
  packageContents: 'Main Unit, User Manual, Power Adapter, Accessories',
  shippingClass: 'Standard Shipping Class 1',
  handlingInstructions: 'Handle with Care, Keep Dry, Avoid Extreme Temperatures',
  countryOfOrigin: 'USA',
  assemblyRequired: 'No',
  warrantyType: 'Manufacturer Warranty',
  returnPolicy: '30-Day Money Back Guarantee',
  seasonality: 'Year-Round Product',
  launchDate: '2020-9-6',
  productStatus: 'Discontinued',
  availabilityStatus: 'Out of Stock'
}
```


Без индекса любой поиск, кроме _id, будет использовать `collscan` стратегию:

```javascript
db.products.explain('executionStats').find({qualityGrade: 'Standard'})
```

```javascript
 executionStats: {
    executionSuccess: true,
    nReturned: 26,
    executionTimeMillis: 0,
    totalKeysExamined: 0,
    totalDocsExamined: 100,
    executionStages: {
      isCached: false,
      stage: 'COLLSCAN',
      filter: { qualityGrade: { '$eq': 'Standard' } },
      nReturned: 26,
      executionTimeMillisEstimate: 0,
      works: 101,
      advanced: 26,
      needTime: 74,
      needYield: 0,
      saveState: 0,
      restoreState: 0,
      isEOF: 1,
      direction: 'forward',
      docsExamined: 100
    }
  },
```

Да и если мы хотим найти все товары, которые хоть как-то связаны с органикой (условно говоря такой запрос):
```javascript
db.products.find({
    $or: [
        { keyFeatures: { $regex: 'Organic', $options: 'i' } },
        { shortDescription: { $regex: 'Organic', $options: 'i' } },
        { longDescription: { $regex: 'Organic', $options: 'i' } },
        { productBenefits: { $regex: 'Organic', $options: 'i' } },
        { material: { $regex: 'Organic', $options: 'i' } },
        { productName: { $regex: 'Organic', $options: 'i' } },
        ...
    ]
})
```

мягко говоря неудобно.

Создадим индекс:

```javascript
db.products.createIndex( { "$**": "text" } )
```

Но синтаксис используем немного другой, чтобы задействовать wildcard index:
```javascript
db.products.explain("executionStats").find({$text: {$search: "Organic"}})
```

explain (только часть вывода):
```javascript
winningPlan: {
      isCached: false,
      stage: 'TEXT_MATCH',
      indexPrefix: {},
      indexName: '$**_text',
      ...
 executionStats: {
    executionSuccess: true,
    nReturned: 7,
    executionTimeMillis: 0,
    totalKeysExamined: 7,
    totalDocsExamined: 7,
```
Мы видим, что использовался TEXT_MATCH  с нашим индексом.

А теперь попробуем найти все товары из Taiwan (я сразу выведу explain):
```javascript
 executionStats: {
    executionSuccess: true,
    nReturned: 100,
    executionTimeMillis: 0,
    totalKeysExamined: 100,
    totalDocsExamined: 100,
```
Есть проблема, что для каких-то часто встречающихся слов мы будем пробегаться по всей коллекции. На больших данных это может аукнуться и придется допиливать бизнес-логику на стороне приложения, чтобы 
как-то "фильтровать" обычные слова. В общем нюансов много.


Из интересного, решил проверить размер индекса:
```javascript
test> db.products.totalSize()/(1024*1024)
0.22265625
test> db.products.totalIndexSize()/(1024*1024)
0.16015625
```

Индекс занимает 70% от всей коллекции, что довольно дорого. Конечно, у нас в докуметтах 20 полей и из-за этого требуется много места. 

Я пересоздал коллекцию, где уменьшил количество ключей, которых значение - строка c 20 до 7 и проверил размер:
```javascript
test> db.products.totalSize()/(1024*1024)
0.1796875
test> db.products.totalIndexSize()/(1024*1024)
0.09375
```

и уже 40% от коллекции, что уже приемлемо.

Индекс точно имеет место быть, как поиск по описанию, но:
1) это точно непростая логика на стороне приложения, чтобы индекс не пробегался по всем ключам документам, если имеется часто встречающееся слово
2) очень недешевый в размере индекс
