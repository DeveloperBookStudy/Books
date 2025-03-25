# Kafka CDC 구현

## 1. CDC(Change Data Capture)

Change Data Capture(CDC, 변경 데이터 캡처)란 데이터베이스 테이블에서 발생하는 삽입(Insert), 수정(Update), 삭제(Delete) 같은 데이터 변경 사항을 실시간 또는 근실시간으로 추적하고 캡처하는 기술 또는 패턴을 말한다.

가장 일반적으로 변경 데이터 캡처는 원본 데이터베이스의 변경 사항을 모니터링하고 해당 변경 사항을 데이터베이스, 데이터 웨어하우스, 데이터 레이크 또는 이벤트 스트리밍 플랫폼에 전파하는 데 사용된다.

 - 실시간 데이터 동기화: 데이터베이스 변경 사항을 빠르게 반영하여 실시간 분석, 로그 처리 등에 활용 가능
 - 데이터 일관성 유지: 수동 배치 방식보다 최신 데이터를 유지할 수 있음
 - 이벤트 기반 아키텍처 지원: Kafka와 같은 메시지 큐와 결합하면 마이크로서비스 아키텍처에서 데이터 변경 사항을 빠르게 공유 가능

## 2. CDC 실무 사례

 - `알림 서비스`
    - 시나리오: 회원 서비스에서 회원 정보가 변경될 경우 변경에 대한 푸시 메시지 발송이나 메일 발송 등 다른 마이크로서비스에서 이 이벤트를 필요로 함
    - 해결: 회원 DB의 변경을 CDC로 감지하여 Kafka로 발행 -> 이 토픽을 구독하여 알림 전송
    - 사레: https://techblog.woowahan.com/10000/
 - `데이터 싱크`
    - 시나리오: A팀에서 데이터 원천을 관리하는데, B팀에서 이 데이터를 계속 싱크 받아야 한다.
    - 해결 1: A팀의 DB에 B팀이 직접 붙어서 서비스에 사용
    - 해결 2: A팀의 DB에 B팀이 주기적으로 붙어서 싱크해온 뒤, B팀 자체적으로 구축한 복사본을 사용
    - 해결 3: A팀이 Kafka를 통해 메시지 형태로 변경사항을 Produce 해주면, B팀에서 Consume해서 데이터를 준실시간으로 싱크
 - `감사 로그 기록`
    - 시나리오: 사용자가 어떤 데이터를 언제 변경했는지 추적해야 한다.
    - 해결: CDC를 통해 변경 이벤트를 로그 서버나 S3로 저장

## 3. Kafka를 이용한 CDC 구현

 - `CdcProducer`
    - CDC 데이터를 프로듀싱한다.
```java
@RequiredArgsConstructor
@Component
public class CdcProducer {

    private final KafkaTemplate<String, String> kafkaTemplate;

    public void sendMessage(CdcMessage message) throws JsonProcessingException {
        try { 
            kafkaTemplate.send(
                Topic.MEMBER_CDC_TOPIC,
                String.valueOf(message.getId()),
                objectMapper.writeValueAsString(message.getData())
            );
        } catch (JsonProcessingException e) {
            throw new CommonException();
        }
    }
}
```

 - `CdcMessage`
    - CdcProducer에 전달할 메시지 객체
```java
@Getter
@NoArgsConstructor
public class CdcMessage {
    private Long id;
    private Object data;
    private OperationType operationType;

    public static CdcMessage of(Long id, Object data, OperationType operationType) {
        this.id = id;
        this.data = data;
        this.operationType = operationType;
    }
}
```

 - `OperationType`
    - DB Write 종류(Create, Update, Delete)
```java
public enum OperationType {
    CREATE,
    UPDATE,
    DELETE;
}
```

### 3-1. @Transactional + 메시지 발행

 - `MemberService`
    - 회원 DB 테이블에 Write(Create, Update, Delete)가 발생했을 때 데이터 동기화(CDC)를 위해 Kafka로 메시지를 발행한다.
```java
@RequiredArgsConstructor
@Service
public class MemberService {

    private final MemberRepository memberRepository;
    private final CdcProvuder cdcProducer;

    @Transactional
    public MemberInfo create(MemberCreateRequest request) {
        // 회원 정보 Create
        MemberEntity savedMember = memberRepository.save(request.toEntity());
        MemberInfo memberInfo = MemberInfo.of(savedMember);

        // CDC 메시지 발행
        cdcProducer.sendMessage(
            CdcMessage.of(
                memberInfo.getId(),
                memberInfo,
                OperationType.CREATE
            )
        );

        return memberInfo;
    }

    @Transactional
    public MemberInfo update(Long memberId, MemberUpdateRequest request) {
        // 회원 정보 Update
        MemberEntity findMember = memberRepository.findById(memberId)
            .orElseThrow(() -> new CommonException(MemberErrorCode.NOT_FOUND));
        findMember.updateMemberProfile(request.toCriteria());
        MemberInfo memberInfo = MemberInfo.of(findMember);

        // CDC 메시지 발행
        cdcProducer.sendMessage(
            CdcMessage.of(
                memberInfo.getId(),
                memberInfo,
                OperationType.UPDATE
            )
        );

        return memberInfo;
    }

    @Transactional
    public void delete(Long memberId) {
        // 회원 정보 Delete
        memberRepository.deleteById(memberId);

        // CDC 메시지 발행
        cdcProducer.sendMessage(
            CdcMessage.of(
                memberId,
                null,
                OperationType.DELETE
            )
        );
    }
}
```

### 3-2. @TransactionalEventListener

@TransactionalEventListener를 사용하면 트랜잭션의 상태(커밋 또는 롤백)에 따라 이벤트를 처리할지 결정할 수 있다. 따라서 데이터가 커밋된 후 안전하게 실행해야 하는 로직(이메일 전송, 캐시 업데이트, 외부 API 호출 등)에 유용하다.

 - `MemberWriteEvent`
    - 회원 DB 테이블에 Write가 발생했을 때 알릴 이벤트
```java
@Getter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class MemberWriteEvent {
    private Long id;
    private Object data;
    private OperationType operationType

    public static MemberWriteEvent of(Long id, Object data, OperationType operationType) {
        return MemberWriteEvent.builder()
            .id(id)
            .data(data)
            .operationType(operationType)
            .build();
    }
}
```

 - `MemberTransactionalEventListener`
    - 회원 DB 테이블에 Write 하고, 커밋이 정상적으로 처리 된 후에 동작하여 Kafka로 메시지를 발행한다.
    - AFTER_COMMIT: 트랜잭션이 성공적으로 커밋된 후 실행(기본값)
    - AFTER_ROLLBACK: 트랜잭션이 롤백된 후 실행
    - AFTER_COMPLETION: 트랜잭션이 커밋되거나 롤백된 후 실행
    - BEFORE_COMMIT: 트랜잭션이 커밋되기 직전 실행
```java
@RequiredArgsConstructor
@Componenet
public class MemberTransactionalEventListener {

    private final CdcProvuder cdcProducer;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Async
    public void transactionalEventListenerAfterCommit(MemberWriteEvent event) {
        cdcProducer.sendMessage(
            CdcMessage.of(
                event.getId(),
                event,
                event.getOperationType()
            )
        );
    }
}
```

 - `MemberService`
    - 회원 DB 테이블에 Write를 수행하고, 메서드 내부에서 Kafka 메시지 발행을 하지 않고, 메시지를 발행하여 TransactionalEventListener에서 DB에 데이터가 반영(커밋)된 이후에 Kafka로 메시지를 발행한다.
    - DB Write > 이벤트 발행 > DB Commit > TransactionalEventListener > Kafka 메시지 발행
```java
@RequiredArgsConstructor
@Service
public class MemberService {

    private final MemberRepository memberRepository;
    private final ApplicationEventPublisher eventPublisher;

    @Transactional
    public MemberInfo create(MemberCreateRequest request) {
        // 회원 정보 Create
        MemberEntity savedMember = memberRepository.save(request.toEntity());
        MemberInfo memberInfo = MemberInfo.of(savedMember);

        // 이벤트 발행
        eventPublisher.publishEvent(
            MemberWriteEvent.of(
                memberInfo.getId(),
                memberInfo,
                OperationType.CREATE
            )
        );

        return memberInfo;
    }

    @Transactional
    public MemberInfo update(Long memberId, MemberUpdateRequest request) {
        // 회원 정보 Update
        MemberEntity findMember = memberRepository.findById(memberId)
            .orElseThrow(() -> new CommonException(MemberErrorCode.NOT_FOUND));
        findMember.updateMemberProfile(request.toCriteria());
        MemberInfo memberInfo = MemberInfo.of(findMember);

        // 이벤트 발행
        eventPublisher.publishEvent(
            MemberWriteEvent.of(
                memberInfo.getId(),
                memberInfo,
                OperationType.UPDATE
            )
        );

        return memberInfo;
    }

    @Transactional
    public void delete(Long memberId) {
        // 회원 정보 Delete
        memberRepository.deleteById(memberId);

        // 이벤트 발행
        eventPublisher.publishEvent(
            MemberWriteEvent.of(
                memberInfo.getId(),
                null,
                OperationType.DELETE
            )
        );
    }
}
```

### 3-3. EntityListener

EntityListener는 JPA 엔티티에 대한 생성, 수정, 삭제 이벤트를 감지하여 콜백 메서드를 실행할 수 있다.

 - `MemberService`
    - Service에서 이벤트 발행, 메시지 발행 부분이 필요없다. 단순히 DB Write에 대한 코드만을 작성한다.
    - EntityListener에서 DB Write 발생시 콜백 메서드가 호출되어 리스너에서 Kafka 메시지 발행을 수행한다.
```java
@RequiredArgsConstructor
@Service
public class MemberService {

    private final MemberRepository memberRepository;

    @Transactional
    public MemberInfo create(MemberCreateRequest request) {
        // 회원 정보 Create
        MemberEntity savedMember = memberRepository.save(request.toEntity());

        return MemberInfo.of(savedMember);
    }

    @Transactional
    public MemberInfo update(Long memberId, MemberUpdateRequest request) {
        // 회원 정보 Update
        MemberEntity findMember = memberRepository.findById(memberId)
            .orElseThrow(() -> new CommonException(MemberErrorCode.NOT_FOUND));
        findMember.updateMemberProfile(request.toCriteria());

        return MemberInfo.of(findMember);
    }

    @Transactional
    public void delete(Long memberId) {
        // 회원 정보 Delete
        memberRepository.deleteById(memberId);
    }
}
```

 - `MemberEntityListener`
    - @PrePersist: 엔티티가 저장되기 전 실행
    - @PostPersist: 엔티티가 저장된 후 실행
    - @PreUpdate: 엔티티가 수정되기 전 실행
    - @PostUpdate: 엔티티가 수정된 후 실행
    - @PreRemove: 엔티티가 삭제되기 전 실행
    - @PostRemove: 엔티티가 삭제된 후 실행
    - @PostLoad: 엔티티가 조회된 직후 실행
```java
@Component
public class MyEntityListener {

    @Lazy
    @Autowired
    private CdcProducer cdcProducer;

    // 엔티티가 저장된 후 실행
    @PostPersist
    public void handleCreate(MemberEntity member) {
        MemberInfo memberInfo = MemberInfo.of(member);
        cdcProducer.sendMessage(
            CdcMessage.of(
                memberInfo.getId(),
                memberInfo,
                OperationType.CREATE
            )
        );
    }

    // 엔티티가 수정된 후 실행
    @PostUpdate
    public void handleUpdate(MemberEntity member) {
        MemberInfo memberInfo = MemberInfo.of(member);
        cdcProducer.sendMessage(
            CdcMessage.of(
                memberInfo.getId(),
                memberInfo,
                OperationType.UPDATE
            )
        );
    }

    // 엔티티가 삭제된 후 실행
    @PostRemove
    public void handleDelete(MemberEntity member) {
        cdcProducer.sendMessage(
            CdcMessage.of(
                member.getId(),
                null,
                OperationType.DELETE
            )
        );
    }
}
```

### 3-4. 아웃박스(Outbox) 패턴

아웃박스 패턴은 이런 문제를 해결하기 위해 이벤트 데이터를 데이터베이스 안에 함께 저장하고, 이후 별도의 프로세스(또는 쓰레드/서비스)가 이 이벤트들을 이벤트 브로커에 전송하는 방식입니다.

CDC 이벤트를 별도의 테이블(event_outbox)에 저장한 후, 배치 프로세스 또는 이벤트 큐를 이용하여 비동기로 전송한다.

 - `Dual Write 한계점`
    - DB에 확실히 반영된 것만 Kafka로 메시지 발행하자니, 메시지 발행이 누락될 수 있다.
    - Kafka로 메시지 발행을 먼저 하자니, DB에 반영되지 않는 변경사항이 kAFKA로 유통되어 버릴 수 있다.
    - 근본적인 이유로는 데이터 흐름이 직렬이 아니라 병렬(Dual Write) 이다.
    - Write 목적지인 DB와 Kafka에 대한 작업을 하나의 트랜잭션으로 묶을 수 없다.
 - `CDCEventEntity`
    - 스케줄링에 의하여 Outbox 테이블을 읽어서 메시지를 발행한다. 이때, 메시지 발행이 실패한 데이터는 Outbox 테이블에 발행전 상태로 남아있다.
```java
@Entity
@Table(name = "cdc_event")
public class CDCEventEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private Long messageId;

    private String message;

    @Enumerated(EnumType.STRING)
    private OperationType operationType;

    private LocalDateTime createdAt = LocalDateTime.now();

    public CDCEventEntity(Long messageId, String message, OperationType operationType) {
        this.messageId = messageId;
        this.message = message;
        this.operationType = operationType;
    }
}
```

 - `MemberService`
    - DB Write시에 원본 데이터 Write와 CDC 데이터(Outbox 테이블) Write를 하나의 트랜잭션 안에서 수행한다.
    - 트랜잭션 안에서 원본 데이터에 대한 DB Write와 Outbox 테이블에 대한 Write가 원자적으로 보장된다.
```java
@RequiredArgsConstructor
@Service
public class MemberService {

    private final MemberRepository memberRepository;
    private final CdcEventRepository eventRepository;
    private final CustomObjectMapper objectMapper;

    @Transactional
    public MemberInfo create(MemberCreateRequest request) {
        // 회원 정보 Create
        MemberEntity savedMember = memberRepository.save(request.toEntity());
        MemberInfo memberInfo = MemberInfo.of(savedMember);

        // CDC 이벤트(Outbox 테이블) 저장
        eventRepository.save(
            new CDCEventEntity(
                memberInfo.getId(),
                objectMapper.writeValueAsString(memberInfo),
                OperationType.CREATE
            )
        );

        return memberInfo
    }

    @Transactional
    public MemberInfo update(Long memberId, MemberUpdateRequest request) {
        // 회원 정보 Update
        MemberEntity findMember = memberRepository.findById(memberId)
            .orElseThrow(() -> new CommonException(MemberErrorCode.NOT_FOUND);
        findMember.updateMemberProfile(request.toCriteria());
        MemberInfo memberInfo = MemberInfo.of(findMember);

        // CDC 이벤트(Outbox 테이블) 저장
        eventRepository.save(
            new CDCEventEntity(
                memberInfo.getId(),
                objectMapper.writeValueAsString(memberInfo),
                OperationType.UPDATE
            )
        );

        return memberInfo;
    }

    @Transactional
    public void delete(Long memberId) {
        // 회원 정보 Delete
        memberRepository.deleteById(memberId);

        // CDC 이벤트(Outbox 테이블) 저장
        eventRepository.save(
            new CDCEventEntity(
                memberId,
                null,
                OperationType.DELETE
            )
        );
    }
}
```

 - `CDCEventPublisher`
    - 별도의 배치 프로세스 혹은 스케줄러를 만들어서 Outbox 테이블의 데이터를 읽어서 메시지 발행을 처리한다.
    - 예제는 간단한 의사코드(Pseudocode)로 작성하였다.
        - 데이터 조회시 날짜 혹은 발행 상태 등을 검색 조건으로 넣을 수 있다.
        - 메시지 발행이 성공하면 데이터를 삭제하거나 발행 성공 상태로 변경한다.
```java
@RequiredArgsConstructor
@Componenet
public class CDCEventPublisher {

    private final CDCEventRepository cdcEventRepository;
    private final CdcProvuder cdcProducer;

    @Scheduled(fixedRate = 5000) // 5초마다 실행
    public void publishEvents() {
        List<CDCEventEntity> events = cdcEventRepository.findAll();

        for (CDCEventEntity event : events) {
            try {
                // Kafka로 CDC 메시지 발행
                cdcProducer.sendMessage(
                    CdcMessage.of(
                        event.getMessageId(),
                        event.getMessage(),
                        event.getOperationType()
                    )
                );

                // 성공적으로 전송되면 해당 이벤트 삭제(혹은 상태 변경)
                cdcEventRepository.delete(event);
            } catch (Exception e) {
                System.out.println("⚠️ CDC 메시지 전송 실패: " + e.getMessage());
                // 실패 시 재시도 로직 필요
            }
        }
    }
}
```
