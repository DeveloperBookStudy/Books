# 몽고DB 복제 셋 구성

MongoDB 복제 셋(Replica Set)은 고가용성(High Availability)과 데이터 복원을 보장하기 위해 MongoDB에서 제공하는 데이터 복제 메커니즘입니다. 복제 셋은 동일한 데이터를 보유한 여러 MongoDB 인스턴스(노드)로 구성됩니다. 이를 통해 장애가 발생해도 데이터를 안전하게 보호하고 지속적으로 사용할 수 있습니다.

## 1. 복제 셋 구성 요소

 - `Primary Node`
    - 클라이언트 요청(읽기/쓰기)의 기본 대상이 되는 노드입니다.
    - 쓰기 작업은 항상 Primary 노드에서 수행됩니다.
    - Primary는 복제 셋 내 다른 노드에게 데이터를 복제합니다.
 - `Secondary Node`
    - Primary 노드에서 복제된 데이터를 보유합니다.
    - 기본적으로 읽기 전용입니다. (하지만 readPreference 옵션을 변경하여 읽기 작업을 처리하도록 설정할 수 있음)
    - Primary가 실패할 경우 새 Primary로 승격될 수 있습니다.
 - `Arbiter Node`
    - 투표에만 참여하는 노드로, 데이터를 저장하지 않습니다.
    - 짝수 개의 노드로 인해 투표가 무효화되는 것을 방지하기 위해 사용됩니다.
    - 리소스가 제한적일 때 유용하지만, 데이터 복제에 관여하지 않으므로 데이터 안전성 측면에서는 주의가 필요합니다.

## 2. 복제 셋 작동 원리

 - `데이터 복제`
    - Primary 노드에서 데이터가 변경되면, Secondary 노드가 Primary의 oplog(Operation Log)를 읽고 데이터를 복제합니다.
 - `자동 장애 조치(Failover)`
    - Primary 노드가 실패하면, 남은 Secondary 노드 중 하나가 투표를 통해 새로운 Primary로 승격됩니다.
    - 투표에서 다수결로 승격 노드를 결정합니다.
 - `읽기 및 쓰기`
    - 기본적으로 쓰기 작업은 Primary에서만 이루어지며, 읽기 작업은 Primary 또는 Secondary에서 선택적으로 수행할 수 있습니다.
    - readPreference 설정을 통해 읽기 작업의 대상 노드를 제어할 수 있습니다.

## 3. 복제 셋 구성

복제 셋의 모든 멤버는 같은 셋 내 다른 멤버와 연결할 수 있어야 한다. 몽고DB 3.6+ 부터는 mongod는 기본적으로 로컬호스트(127.0.0.1)에만 바인딩된다. 복제 셋의 각 멤버가 다른 멤버와 통신하려면 다른 멤버가 연결할 수 있는 IP 주소에도 바인딩해야 한다. 인스턴스를 각기 다른 서버의 멤버와 함께 복제 셋의 멤버로 실행하려면, 명령행 매개변수 --bind_ip를 지정하거나 인스턴스 구성 파일에 있는 bind_ip를 사용한다.

```javascript
// MongoDB 노드 설정
mongod --replSet "rs0" --port 27017 --dbpath /data/db1 --bind_ip localhost
mongod --replSet "rs0" --port 27018 --dbpath /data/db2 --bind_ip localhost
mongod --replSet "rs0" --port 27019 --dbpath /data/db3 --bind_ip localhost

// Replica Set 초기화: Primary 노드에서 명령 실행
rs.initiate({
    _id: "rs0",
    members: [
        { _id: 0, host: "localhost:27017" },
        { _id: 1, host: "localhost:27018" },
        { _id: 2, host: "localhost:27019" }
    ]
})

// 복제 상태 확인
rs.status()
```

### 복제 셋 구성 변경

복제 셋 구성에 대한 멤버 추가, 삭제, 변경이 가능하다.

```javascript
// 멤버 추가
rs.add("localhost:27020")

// 멤버 제거
rs.remove("localhost:27017")

// 구성 확인
rs.config()

// 구성 변수로 변경
var config = rs.config()
config.members[0].host = "localhost:27017"
rs.reconfig(config)
```

### 우선순위

특정 멤버가 프라이머리에 대한 우선순위 값을 지정할 수 있다.

 - 0 ~ 100 사이의 값으로 지정한다.
 - 기본값은 1이다.
 - 0으로 지정하면 절대 프라이머리가 될 수 없다. (수동적 멤버)
```javascript
// server-4가 다른 복제 셋 멤버와 같은 최신 데이터로 동기화되면
// 기존 프라이머리는 자격을 내려놓고 server-4 프라이머리로 선출된다.
rs.add({"host" : "server-4:27017", "priority" : 1.5})
```

### 숨겨진 멤버

클라이언트는 숨겨진 멤버에 요청을 라우팅하지 않으며, 숨겨진 멤버는 복제 소스로서 바람직하지 않다.

```javascript
// 숨기는 경우 rs.isMaster()에 노출되지 않는다.
// rs.status(), rs.config() 에서는 노출된다.
var config = rs.config()
config.members[2].hidden = true
config.members[2].priority = 0
rs.reconfig(config)
```

### 아비터 노드

소규모 배포에서는 데이터 복사본을 세 개나 보관하기가 꺼려진다. 복사본은 두 개면 충분하고, 세 번쨰 복사본은 관리, 운영, 비용을 고려하면 별 가치가 없다고 생각할 수 있다.

이러한 배포에 대해 몽고DB는 프라이머리 선출에 참여하는 용도로만 쓰이는 아비터라는 특수한 멤버를 지원한다. 아비터는 데이터를 가지지 않으며 클라이언트에 의해 사용되지 않는다. 오로지 멤버 복제 셋에서 과반수를 구성하는 데 사용된다. 일반적으로는 아비터가 없는 배포가 바람직하다.

 - 아비터는 mongod 서버 작동과 아무런 연관이 없다.
 - 아비터는 일반적으로 몽고DB에 사용하는 서버보다 사양이 낮은 서버에서 경량화 프로세스로 실행할 수 있다.
 - 아비터를 한 번 구성하고 나면 프라이머리나 세컨더리로 변경할 수 없다.
```javascript
rs.addArb("server-5:27017")
rs.add({"_id": 4, "host": "server-5:27017", "arbiterOnly": true})
```

## 4. 복제 셋 구성시 고려사항

### 데이터 안정성과 성능의 균형

#### WriteConcern

w 옵션은 쓰기 작업이 성공으로 간주되기 위해 필요한 복제 셋의 응답 노드 수를 설정합니다.

 - 데이터의 안전성을 높이기 위해 writeConcern을 majority로 설정하면 쓰기 작업이 복제된 데이터의 확인을 기다립니다.
 - 하지만 데이터 복제가 완료될 때까지 기다리므로 쓰기 성능에 영향을 줄 수 있습니다.
 - 0: 쓰기 작업을 네트워크 버퍼에 기록한 후 성공으로 간주합니다. 서버 응답을 기다리지 않으므로 성능은 높지만 데이터 손실 가능성이 있습니다.
 - 1 (기본값): Primary 노드가 쓰기 작업을 저장한 후 성공으로 간주합니다. 복제 노드에는 복제가 완료되지 않을 수 있습니다.
 - majority: 과반수 이상의 복제 노드가 데이터를 복제한 후 성공으로 간주합니다. 데이터 안정성이 높지만 성능은 상대적으로 낮습니다.
 - 정수 값(예: 2, 3): 지정된 숫자만큼의 노드가 데이터를 복제한 후 성공으로 간주합니다. w 값이 복제 셋의 노드 수보다 클 경우 오류가 발생합니다.

<br/>

j 옵션은 쓰기 작업이 journal에 기록되었는지 여부를 확인합니다.

 - true: 쓰기 작업이 journal에 기록된 후 성공으로 간주합니다. 시스템이 비정상 종료되더라도 데이터가 복구될 가능성이 높아집니다.
 - false (기본값): journal 기록을 기다리지 않습니다. 데이터 안정성이 다소 낮지만 성능이 더 높을 수 있습니다.

```javascript
db.collection.insertOne(
  { name: "example" },
  { writeConcern: { w: "majority", j: true, wtimeout: 5000 } }
);
```

 - __빠른 성능이 중요한 경우__
    - w: 0, j: false를 사용하여 쓰기 작업의 속도를 높입니다.
    - 데이터 안정성은 낮아지므로 테스트 환경 또는 로그 데이터처럼 덜 중요한 데이터에 적합합니다.
 - __일반적인 안정성이 필요한 경우__
    - w: 1, j: false를 사용하여 Primary에 쓰기가 성공적으로 기록되면 작업을 완료합니다.
 - __높은 데이터 안전성이 중요한 경우__
    - w: "majority", j: true를 사용하여 복제 셋의 과반수 노드에 데이터가 저장되고 journal에도 기록된 후 성공으로 간주합니다.
    - 금융 데이터나 중요한 비즈니스 애플리케이션에 적합합니다.

#### ReadPreference

ReadPreference 옵션은 클라이언트가 읽기 작업(쿼리)을 수행할 때 사용할 복제 셋 노드를 결정하는 방법을 정의합니다.

 - 기본적으로 읽기 작업은 Primary에서 이루어지지만, 읽기 부하를 줄이기 위해 Secondary를 사용할 수 있습니다.
 - 최신 데이터가 필요하면 primary로 설정하고, 부하 분산이 중요하면 secondaryPreferred로 설정합니다.
 - primary (기본값): 읽기 작업을 반드시 Primary 노드에서 수행합니다. 가장 최신 데이터를 보장합니다.
 - primaryPreferred: 우선 프라이머리에서 읽기 요청
 - secondary: 항상 세컨더리에 읽기 요청 (세컨더리가 없으면 에러)
 - secondaryPreferred: 우선 세컨더리에 읽기 요청 (세컨더리가 없으면 프라이머리로 요청)
 - nearest: 요청 일관성보다 낮은 지연율이 중요한 경우(핑 시간 기반 지연율이 가장 낮은 멤버에서 읽기)
 - __최신 데이터가 필요한 경우__
    - primary 또는 primaryPreferred를 사용
    - 실시간 데이터 분석 또는 최신 상태가 중요한 애플리케이션에 적합
 - __읽기 부하 분산__
    - secondary 또는 secondaryPreferred를 사용
    - 읽기 요청이 많은 애플리케이션에서 Primary의 부하를 줄이기 위해 사용
 - __최소 네트워크 지연__
    - nearest를 사용
    - 글로벌 분산 시스템에서 가장 가까운 노드에서 데이터를 읽도록 설정
 - __장애 복구와 읽기 분산 병행__
    - secondaryPreferred를 사용
    - Primary 장애 시에도 읽기 작업이 가능하도록 설정
