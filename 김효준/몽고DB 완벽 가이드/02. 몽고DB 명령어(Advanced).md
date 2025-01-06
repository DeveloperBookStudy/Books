# 몽고DB 기초 명령어2

## 1. 인덱스

인덱스는 데이터를 효율적으로 검색할 수 있도록 도와주는 중요한 도구입니다. RDBMS에서의 인덱스와 비슷한 역할을 하며, 데이터를 빠르게 찾기 위해 특정 필드에 인덱스를 생성합니다.

 - __빠른 검색 속도__: 인덱스는 MongoDB가 데이터를 검색할 때, 컬렉션 전체를 스캔하는 대신 특정 필드를 기반으로 효율적으로 검색할 수 있도록 합니다.
 - __쿼리 성능 향상__: 복잡한 쿼리나 대량의 데이터를 처리할 때, 인덱스가 없으면 성능이 크게 저하될 수 있습니다.
```javascript
// 인덱스 생성
db.collection.createIndex({ fieldName: 1 })

// 인덱스 조회
db.collection.getIndexes()

// 인덱스 삭제
db.collection.dropIndex("indexName")
```


 - `Single Field Index (단일 필드 인덱스)`
    - 하나의 필드에 대해 인덱스를 생성합니다.
    - 기본적으로 MongoDB는 _id 필드에 인덱스를 자동으로 생성합니다.
```javascript
db.collection.createIndex({ fieldName: 1 })  // 오름차순
db.collection.createIndex({ fieldName: -1 }) // 내림차순

// `name` 필드에 단일 필드 인덱스 생성
db.users.createIndex({ name: 1 })

// 사용 예: name 필드로 검색
db.users.find({ name: "John" })

// 사용 예: name 필드로 정렬
db.users.find().sort({ name: 1 })
```

 - `Compound Index (복합 인덱스)`
    - 여러 필드를 조합하여 인덱스를 생성합니다.
    - 특정 쿼리가 두 개 이상의 필드를 자주 검색하는 경우 유용합니다.
```javascript
db.collection.createIndex({ field1: 1, field2: -1 })

// 복합 인덱스 생성 (age는 오름차순, name은 내림차순)
db.users.createIndex({ age: 1, name: -1 })

// 사용 예: age와 name을 함께 검색
db.users.find({ age: 25, name: "John" })

// 사용 예: age로 검색하고 name으로 정렬
db.users.find({ age: 25 }).sort({ name: -1 })
```

 - `Multikey Index (다중 키 인덱스)`
    - 배열 형태의 데이터를 인덱싱할 때 사용됩니다.
    - 배열의 각 요소에 대해 인덱스를 생성합니다.
    - 배열 필드가 중첩 배열인 경우 Multikey 인덱스 사용 불가
```javascript
db.collection.createIndex({ arrayField: 1 })

// 배열 필드 tags에 대한 다중 키 인덱스 생성
db.posts.createIndex({ tags: 1 })

// 사용 예: 배열의 특정 요소로 검색
db.posts.find({ tags: "mongodb" })

// 사용 예: 배열의 여러 요소 중 하나라도 일치하는 문서 검색
db.posts.find({ tags: { $in: ["mongodb", "database"] } })
```

 - `Text Index (텍스트 인덱스)`
    - 텍스트 데이터에 대해 효율적으로 검색하기 위한 인덱스입니다.
    - 주로 풀 텍스트 검색에 사용됩니다.
    - $text 연산자를 사용해 키워드 검색 가능
    - 한 컬렉션당 하나의 텍스트 인덱스만 허용
```javascript
db.collection.createIndex({ fieldName: "text" })

// 사용 예제
db.articles.insertMany([
  { title: "Introduction to MongoDB", description: "MongoDB is a NoSQL database." },
  { title: "Advanced MongoDB Queries", description: "Learn about advanced queries in MongoDB." },
  { title: "Text Indexing in MongoDB", description: "Text indexing allows efficient text search." },
  { title: "Other NoSQL Databases", description: "Explore other NoSQL databases like Cassandra and Redis." }
])

// description 필드에 대해 텍스트 인덱스를 생성
db.articles.createIndex({ description: "text" })

// 단순 텍스트 검색: 특정 키워드를 포함하는 문서를 검색
db.articles.find({ $text: { $search: "MongoDB" } })

// AND 검색: 두 개 이상의 키워드가 포함된 문서 검색
db.articles.find({ $text: { $search: "MongoDB advanced" } })

// OR 검색: 키워드 중 하나라도 포함된 문서 검색
db.articles.find({ $text: { $search: "MongoDB OR Redis" } })

// 정확도 점수 포함: 검색 결과를 점수(textScore) 기준으로 정렬
db.articles.find(
  { $text: { $search: "MongoDB" } },
  { score: { $meta: "textScore" } }
).sort({ score: { $meta: "textScore" } })

```

 - `Geospatial Index (지리 공간 인덱스)`
    - 위치 기반 데이터를 검색하기 위해 사용됩니다.
    - 2D 평면 및 2D 구형 데이터에 대해 지원됩니다.
```javascript
db.collection.createIndex({ locationField: "2dsphere" })

// 사용 예제
db.places.insertMany([
  {
    name: "Central Park",
    location: { type: "Point", coordinates: [-73.9654, 40.7829] } // [longitude, latitude]
  },
  {
    name: "Statue of Liberty",
    location: { type: "Point", coordinates: [-74.0445, 40.6892] }
  },
  {
    name: "Times Square",
    location: { type: "Point", coordinates: [-73.9851, 40.7580] }
  },
  {
    name: "Brooklyn Bridge",
    location: { type: "Point", coordinates: [-73.9969, 40.7061] }
  }
])

// Geospatial Index 생성: location 필드에 대해 2dsphere 인덱스를 생성
db.places.createIndex({ location: "2dsphere" })

// 근처의 위치 찾기: 특정 좌표를 기준으로 가까운 위치 검색
db.places.find({
  location: {
    $near: {
      $geometry: { type: "Point", coordinates: [-73.9851, 40.7580] }, // Times Square
      $maxDistance: 1000 // 1km 내
    }
  }
})

// 특정 반경 내 위치 검색: 특정 반경 내의 위치를 검색
db.places.find({
  location: {
    $geoWithin: {
      $centerSphere: [[-73.9851, 40.7580], 1 / 3963.2] // 반경 1마일 (지구 반지름 3963.2마일 기준)
    }
  }
})

// 다각형 영역 내 위치 검색
db.places.find({
  location: {
    $geoWithin: {
      $geometry: {
        type: "Polygon",
        coordinates: [
          [
            [-73.9969, 40.7061],
            [-73.9654, 40.7829],
            [-74.0445, 40.6892],
            [-73.9969, 40.7061] // 첫 좌표와 동일해야 함
          ]
        ]
      }
    }
  }
})

// 멀리 떨어진 위치 찾기: 특정 좌표로부터 가장 먼 위치를 검색
db.places.find({
  location: {
    $near: {
      $geometry: { type: "Point", coordinates: [-73.9851, 40.7580] }, // Times Square
      $minDistance: 5000 // 5km 이상 떨어진 위치
    }
  }
})
```

 - `Wildcard Index (와일드카드 인덱스)`
    - 동적 스키마에 적합한 인덱스입니다.
    - 컬렉션의 여러 필드 또는 하위 문서의 모든 필드에 대해 인덱스를 생성합니다.
```javascript
db.collection.createIndex({ "$**": 1 })

// 컬렉션 전체에 대해 와일드카드 인덱스 생성
db.records.createIndex({ "$**": 1 })

// 사용 예: 어떤 필드의 값이 특정 값과 일치하는 문서 검색
db.records.find({ "dynamicField.someProperty": "value" })
```

## 2. 집계 프레임워크

MongoDB의 집계 프레임워크는 데이터를 처리하고 변환하여 요약, 분석, 변환 결과를 반환하는 강력한 도구로 파이프라인 기반의 데이터 처리를 제공합니다.

 - __Aggregation Pipeline__: 여러 단계(stage)로 구성되며, 각 단계는 입력 데이터를 처리하고 그 결과를 다음 단계로 전달합니다.
 - __단계(Stage)__: 각 단계는 특정 작업(예: 필터링, 정렬, 그룹화 등)을 수행합니다.
 - __연산자(Operators)__: 특정 연산(수학, 논리, 문자열 등)을 수행하기 위한 도구입니다.
```javascript
db.collection.aggregate([
  { <stage1>: { <options> } },
  { <stage2>: { <options> } },
  ...
])
```

### 주요 파이프라인 단계(Stages)

 - `$match`
    - 필터링 단계로, find와 유사하게 조건에 맞는 문서를 선택합니다.
```javascript
db.orders.aggregate([
  { $match: { status: "completed" } }
])
```

 - `$group`
    - 데이터를 그룹화하고, 요약 통계를 계산합니다.
    - _id 필드를 기준으로 그룹화하며, 각 그룹에 대해 계산된 필드를 반환합니다.
```javascript
db.orders.aggregate([
  { $group: { _id: "$customerId", totalSpent: { $sum: "$amount" } } }
])
```

 - `$project`
    - 필드를 선택하거나 새로운 필드를 생성합니다.
```javascript
db.orders.aggregate([
  { $project: { customerId: 1, totalAmount: { $multiply: ["$quantity", "$price"] } } }
])
```

 - `$sort`
```javascript
db.orders.aggregate([
  { $sort: { amount: -1 } } // amount를 내림차순 정렬
])
```

 - `$limit`
    - 지정된 수만큼의 문서를 반환합니다.
```javascript
db.orders.aggregate([
  { $limit: 5 } // 상위 5개 문서만 반환
])
```

 - `$skip`
    - 지정된 수만큼의 문서를 건너뜁니다.
```javascript
db.orders.aggregate([
  { $skip: 10 } // 첫 10개 문서를 건너뜀
])
```

 - `$unwind`
    - 배열 필드를 개별 요소로 펼칩니다.
```javascript
db.orders.aggregate([
  { $unwind: "$items" }
])
```

 - `$lookup`
    - 조인 연산을 수행합니다 (다른 컬렉션과의 결합).
```javascript
db.orders.aggregate([
  {
    $lookup: {
      from: "customers",
      localField: "customerId",
      foreignField: "_id",
      as: "customerDetails"
    }
  }
])
```

### 집계 연산자 (Operators)

 - `산술 연산자`
    - $sum, $avg, $min, $max, $multiply, $divide 등
```javascript
db.orders.aggregate([
  { $group: { _id: "$customerId", totalSpent: { $sum: "$amount" } } }
])
```

 - `배열 연산자`
    - $size, $arrayElemAt, $concatArrays, $push, $addToSet 등
```javascript
db.orders.aggregate([
  { $group: { _id: "$customerId", items: { $push: "$item" } } }
])
```

 - `문자열 연산자`
    - $concat, $substr, $toLower, $toUpper, $split 등
```javascript
db.orders.aggregate([
  { $project: { customerName: { $toUpper: "$name" } } }
])
```

 - `논리 연산자`
    - $and, $or, $not, $eq, $ne, $gt, $lt 등
```javascript
db.orders.aggregate([
  { $match: { $and: [ { amount: { $gt: 100 } }, { status: "completed" } ] } }
])
```

### 사용 예제

```javascript
// 데이터 삽입
// 주문 데이터
db.orders.insertMany([
  { orderId: 1, customerId: "A123", items: [{ name: "Pen", qty: 10, price: 2.5 }, { name: "Notebook", qty: 5, price: 5 }] },
  { orderId: 2, customerId: "B456", items: [{ name: "Pencil", qty: 20, price: 1 }, { name: "Eraser", qty: 10, price: 0.5 }] },
  { orderId: 3, customerId: "A123", items: [{ name: "Marker", qty: 15, price: 1.5 }] },
  { orderId: 4, customerId: "C789", items: [{ name: "Pen", qty: 5, price: 2.5 }, { name: "Notebook", qty: 3, price: 5 }] }
])

// 고객 데이터
db.customers.insertMany([
  { _id: "A123", name: "Alice", email: "alice@example.com" },
  { _id: "B456", name: "Bob", email: "bob@example.com" },
  { _id: "C789", name: "Charlie", email: "charlie@example.com" }
])

// $unwind: 배열을 펼쳐 각 항목을 별도의 문서로 만듦
db.orders.aggregate([
  { $unwind: "$items" }
])
[
  {
    _id: ObjectId('677bd7b2ee3a7a3fd3e9496a'),
    orderId: 1,
    customerId: 'A123',
    items: { name: 'Pen', qty: 10, price: 2.5 }
  },
  {
    _id: ObjectId('677bd7b2ee3a7a3fd3e9496a'),
    orderId: 1,
    customerId: 'A123',
    items: { name: 'Notebook', qty: 5, price: 5 }
  },
  {
    _id: ObjectId('677bd7b2ee3a7a3fd3e9496b'),
    orderId: 2,
    customerId: 'B456',
    items: { name: 'Pencil', qty: 20, price: 1 }
  },
  // ..
]
```

 - `고객별 총 지출 계산`
    - 각 고객(customerId)이 주문한 상품의 총 금액을 계산합니다.
```javascript
db.orders.aggregate([
  // 1. $unwind: 배열을 펼쳐 각 항목을 별도의 문서로 만듦
  { $unwind: "$items" },
  // 2. $group: 고객별로 그룹화하고 총 지출 계산
  // 각 customerId를 기준으로 그룹화합니다.
  // $multiply를 사용해 각 항목의 qty와 price를 곱하여 금액 계산.
  // $sum으로 모든 항목의 금액을 더해 총 지출(totalSpent)을 구합니다.
  {
    $group: {
      _id: "$customerId", // 그룹 기준
      totalSpent: { $sum: { $multiply: ["$items.qty", "$items.price"] } } // 총 지출 계산
    }
  },
  // 3. $sort: 총 지출 기준으로 내림차순 정렬
  { $sort: { totalSpent: -1 } }
])

// 결과
[
  { _id: 'A123', totalSpent: 72.5 },
  { _id: 'C789', totalSpent: 27.5 },
  { _id: 'B456', totalSpent: 25 }
]
```

 - `상품별 판매량 집계`
    - 각 상품(name)의 총 판매 수량(qty)을 계산합니다.
```javascript
db.orders.aggregate([
  // 1. $unwind: 배열을 펼쳐 각 항목을 개별 문서로 분리
  { $unwind: "$items" },
  // 2. $group: 상품별로 그룹화하고 총 판매 수량 계산
  // items.name 필드를 기준으로 그룹화
  // $sum 연산자를 사용해 각 상품의 총 판매량을 계산
  {
    $group: {
      _id: "$items.name", // 상품명 기준 그룹화
      totalQty: { $sum: "$items.qty" } // 수량 합계
    }
  },
  // 3. $sort: 판매량 기준으로 내림차순 정렬
  { $sort: { totalQty: -1 } }
])

// 결과
[
  { _id: 'Pencil', totalQty: 20 },
  { _id: 'Marker', totalQty: 15 },
  { _id: 'Pen', totalQty: 15 },
  { _id: 'Eraser', totalQty: 10 },
  { _id: 'Notebook', totalQty: 8 }
]
```

 - `고객 정보와 주문 데이터 결합`
    - customers 컬렉션과 결합하여 고객 정보를 포함한 주문 데이터를 조회합니다.
```javascript
db.orders.aggregate([
  // 1. $lookup: customers 컬렉션과 조인
  // orders.customerId와 customers._id를 매칭하여 두 컬렉션을 결합
  // 결과는 customerDetails 필드에 배열로 저장됩니다.
  {
    $lookup: {
      from: "customers", // 결합할 컬렉션
      localField: "customerId", // orders의 필드
      foreignField: "_id", // customers의 필드
      as: "customerDetails" // 결합된 데이터를 저장할 필드
    }
  },
  // 2. $unwind: 결합된 배열을 펼침
  // customerDetails 배열을 펼쳐 각 문서에 단일 고객 데이터를 포함
  { $unwind: "$customerDetails" },

  // 3. $project: 필요한 필드만 선택
  {
    $project: {
      orderId: 1,
      customerName: "$customerDetails.name",
      email: "$customerDetails.email",
      items: 1
    }
  }
])

// 결과
[
  {
    _id: ObjectId('677bd7b2ee3a7a3fd3e9496a'),
    orderId: 1,
    items: [
      { name: 'Pen', qty: 10, price: 2.5 },
      { name: 'Notebook', qty: 5, price: 5 }
    ],
    customerName: 'Alice',
    email: 'alice@example.com'
  },
  {
    _id: ObjectId('677bd7b2ee3a7a3fd3e9496b'),
    orderId: 2,
    items: [
      { name: 'Pencil', qty: 20, price: 1 },
      { name: 'Eraser', qty: 10, price: 0.5 }
    ],
    customerName: 'Bob',
    email: 'bob@example.com'
  },
  {
    _id: ObjectId('677bd7b2ee3a7a3fd3e9496c'),
    orderId: 3,
    items: [ { name: 'Marker', qty: 15, price: 1.5 } ],
    customerName: 'Alice',
    email: 'alice@example.com'
  },
  {
    _id: ObjectId('677bd7b2ee3a7a3fd3e9496d'),
    orderId: 4,
    items: [
      { name: 'Pen', qty: 5, price: 2.5 },
      { name: 'Notebook', qty: 3, price: 5 }
    ],
    customerName: 'Charlie',
    email: 'charlie@example.com'
  }
]
```

## 3. 트랜잭션

MongoDB는 __다중 문서 트랜잭션(Multi-Document Transactions)__ 을 지원하여 관계형 데이터베이스의 ACID 특성을 제공합니다. 트랜잭션은 주로 복잡한 데이터 변경 작업에서 데이터의 일관성을 유지하기 위해 사용됩니다.

 - `트랜잭션 사용 예제(Javascript)`
    - startSession(): 트랜잭션을 시작하기 위해 세션을 생성한다.
    - startTransaction(): 트랜잭션을 시작한다. 격리 수준(readConcern, writeConcern)을 설정할 수 있다.
    - commitTransaction(): 트랜잭션을 커밋하여 작업을 확정한다.
    - abortTransaction(): 트랜잭션을 롤백하여 작업을 취소한다.
    - endSession(): 트랜잭션 세션을 종료한다.
```javascript
const { MongoClient } = require("mongodb");

const uri = "mongodb://localhost:27017";
const client = new MongoClient(uri);

async function runTransaction() {
  try {
    await client.connect();

    const db = client.db("bank");
    const accounts = db.collection("accounts");
    const transactions = db.collection("transactions");

    // 트랜잭션 세션 생성
    const session = client.startSession();

    try {
      // 트랜잭션 시작
      session.startTransaction();

      // 1. 계좌 A에서 100 감소
      await accounts.updateOne(
        { accountId: "A123" },
        { $inc: { balance: -100 } },
        { session }
      );

      // 2. 계좌 B에서 100 증가
      await accounts.updateOne(
        { accountId: "B456" },
        { $inc: { balance: 100 } },
        { session }
      );

      // 3. 거래 내역 추가
      await transactions.insertOne(
        {
          from: "A123",
          to: "B456",
          amount: 100,
          date: new Date(),
        },
        { session }
      );

      // 트랜잭션 커밋
      await session.commitTransaction();
      console.log("Transaction committed successfully!");
    } catch (error) {
      console.error("Transaction aborted due to error:", error);

      // 트랜잭션 롤백
      await session.abortTransaction();
    } finally {
      // 세션 종료
      session.endSession();
    }
  } finally {
    await client.close();
  }
}

runTransaction().catch(console.error);
```

### 트랜잭션 격리 수준

트랜잭션 격리 수준은 여러 트랜잭션이 동시에 수행될 때 데이터의 일관성을 유지하기 위해 MongoDB가 제공하는 메커니즘입니다.

 - `읽기 격리 수준(Read Isolation)`
    - local
        - 데이터베이스의 로컬 데이터만 읽습니다.
        - 복제본(replica set) 또는 샤드(clustered)에서 일관성이 보장되지 않을 수 있습니다.
        - 복제본 세트의 복제 지연을 무시하고, 로컬에 저장된 데이터를 반환하므로 성능이 가장 빠릅니다.
        - 가장 최근에 쓰여진 데이터를 보장하지 않음.
        - 복제본 세트에서 데이터가 동기화되지 않았을 수 있음.
    - majority
        - 복제본 집합(replica set)에서 다수의 복제본 노드에 확인된 데이터를 읽습니다.
        - 더 높은 읽기 일관성을 보장하지만, 지연(latency)이 발생할 수 있습니다.
    - snapshot
        - 트랜잭션 내에서 읽기 작업은 동일한 스냅샷에 기반합니다.
        - 트랜잭션 시작 이후에 발생한 변경 사항은 보이지 않습니다.
        - MongoDB 트랜잭션과 함께 사용되며, 트랜잭션 시작 시점의 데이터 상태를 기준으로 읽습니다.
        - 트랜잭션 내의 모든 읽기 작업이 동일한 데이터 상태를 유지.
        - 쓰기 작업 중에도 일관성 있는 데이터를 읽음.
 - `쓰기 격리 수준 (Write Isolation)`
    - `w (Acknowledged/Unacknowledged 쓰기 확인 수준)`
        - __w: 0 (Unacknowledged)__
            - 클라이언트가 쓰기 작업을 서버로 보낸 후, 서버의 확인을 기다리지 않음
            - 쓰기 작업의 성공 여부를 알 수 없으므로 성능은 좋지만, 데이터 손실 가능성이 있음
            - 사용 사례: 쓰기 성능이 최우선이고, 데이터 안정성이 덜 중요한 경우 (로그 데이터 처리 등)
        - __w: 1 (Acknowledged)__
            - 쓰기 작업이 Primary 노드에 기록된 것을 확인 후 성공 응답 반환
            - MongoDB의 기본 Write Concern 값
            - 사용 사례: 기본적인 쓰기 작업에서 적당한 일관성을 보장
        - __w: n (Custom)__
            - 쓰기 작업이 지정된 n개의 복제본 노드에 기록된 것을 확인 후 성공 응답 반환
            - n은 Primary를 포함한 복제본 노드의 개수
            - 사용 사례: 데이터 안정성을 강화하고자 하는 경우
        - __w: majority__
            - 쓰기 작업이 복제본 세트의 과반수 노드에 기록된 것을 확인 후 성공 응답 반환
            - 고가용성과 데이터 안정성을 보장
            - 사용 사례: 중요한 금융 거래, 결제 시스템 등에서 데이터 유실 방지
    - `j (Journaling)`
        - __j: true__
            - 쓰기 작업이 디스크의 저널 파일에 기록된 것을 확인 후 성공 응답 반환
            - 전원이 꺼지는 등의 장애 상황에서도 데이터의 안정성을 보장
            - 사용 사례: 데이터 손실 가능성을 최소화해야 하는 애플리케이션
        - __j: false__
            - 쓰기 작업이 저널에 기록되지 않아도 성공 응답을 반환
            - 디스크 기록이 완료되지 않았을 수 있으므로 데이터 손실 가능성이 있음
    - `타임아웃 설정 (wtimeout)`
        - 쓰기 작업이 지정된 시간 내에 완료되지 않으면 실패로 간주
        - 단위: 밀리초(ms)

