# Camunda - Hướng dẫn chi tiết từ góc nhìn Solution Architect

## 1. Camunda là gì?

Camunda là một **workflow/process automation engine** dựa trên chuẩn **BPMN 2.0** (Business Process Model and Notation). Nó cho phép bạn mô hình hóa, thực thi và giám sát các quy trình nghiệp vụ.

---

## 2. Kiến trúc tổng quan

```
┌─────────────────────────────────────────────────┐
│                  BPMN Modeler                    │
│         (Thiết kế process bằng XML/GUI)          │
└──────────────────────┬──────────────────────────┘
                       │ deploy
                       ▼
┌─────────────────────────────────────────────────┐
│              Camunda Engine (Core)               │
│  ┌───────────┐ ┌──────────┐ ┌────────────────┐  │
│  │ Process   │ │ Job      │ │ History        │  │
│  │ Execution │ │ Executor │ │ Service        │  │
│  └───────────┘ └──────────┘ └────────────────┘  │
│  ┌───────────┐ ┌──────────┐ ┌────────────────┐  │
│  │ REST API  │ │ Task     │ │ Decision       │  │
│  │           │ │ Service  │ │ Engine (DMN)   │  │
│  └───────────┘ └──────────┘ └────────────────┘  │
└──────────────────────┬──────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
   ┌─────────┐  ┌───────────┐  ┌──────────┐
   │ Worker 1│  │ Worker 2  │  │ Worker N │
   │ (Java)  │  │ (Node.js) │  │ (Python) │
   └─────────┘  └───────────┘  └──────────┘
```

---

## 3. Hai phiên bản chính

| | Camunda 7 (Platform) | Camunda 8 (Zeebe) |
|---|---|---|
| Engine | Java-based, embedded hoặc standalone | Zeebe - distributed, cloud-native |
| Communication | REST API + External Task | gRPC |
| Database | RDBMS (PostgreSQL, MySQL...) | Không cần RDBMS, dùng RocksDB + Elasticsearch |
| Scaling | Vertical (khó scale horizontal) | Horizontal (partitioning) |
| Deployment | Embedded trong Spring Boot hoặc standalone | Cluster-based |

---

## 4. Các thành phần BPMN quan trọng

### 4.1 Task Types

```
┌─────────────────────────────────────────────────────┐
│                    BPMN Tasks                        │
├──────────────┬──────────────────────────────────────┤
│ Service Task │ Thực thi logic nghiệp vụ             │
│   - Internal │  → Java Delegate (chạy trong engine) │
│   - External │  → Worker polling                     │
├──────────────┼──────────────────────────────────────┤
│ User Task    │ Cần người dùng thao tác (approve...) │
├──────────────┼──────────────────────────────────────┤
│ Send Task    │ Gửi message                          │
├──────────────┼──────────────────────────────────────┤
│ Receive Task │ Chờ nhận message                     │
├──────────────┼──────────────────────────────────────┤
│ Script Task  │ Chạy script (Groovy, JS...)          │
├──────────────┼──────────────────────────────────────┤
│ Business     │ Gọi sub-process hoặc rule            │
│ Rule Task    │ (DMN decision table)                 │
└──────────────┴──────────────────────────────────────┘
```

### 4.2 Internal vs External Task

**Internal Task (Java Delegate):**
```java
// Chạy TRONG engine process → cùng transaction
public class MyDelegate implements JavaDelegate {
    @Override
    public void execute(DelegateExecution execution) {
        // logic ở đây
    }
}
```

**External Task (Worker pattern):**
```java
// Chạy NGOÀI engine → worker chủ động poll
@Component
@ExternalTaskSubscription(topicName = "verifyNewAML")
public class VerifyAmlWorkflow implements ExternalTaskHandler {
    @Override
    public void execute(ExternalTask task, ExternalTaskService service) {
        // logic ở đây
        service.complete(task); // báo hoàn thành
    }
}
```

**Khi nào dùng cái nào?**

| Tiêu chí | Internal | External |
|---|---|---|
| Coupling | Tight (cùng JVM với engine) | Loose (tách biệt hoàn toàn) |
| Ngôn ngữ | Chỉ Java | Bất kỳ (Java, Node, Python...) |
| Scale | Scale cùng engine | Scale độc lập |
| Transaction | Cùng transaction với engine | Transaction riêng |
| Phù hợp | Logic đơn giản, nhanh | Long-running, gọi API bên ngoài |
| Retry | Engine quản lý | Worker tự quản lý |

### 4.3 Gateways

```
◇ Exclusive (XOR)    → Chỉ 1 nhánh đúng điều kiện
◈ Parallel (AND)     → Tất cả nhánh chạy song song
◇ Inclusive (OR)      → 1 hoặc nhiều nhánh
◇ Event-based         → Chờ event nào đến trước
```

### 4.4 Events

```
Start Events:
  ○  None Start        → Khởi tạo thủ công / API
  ✉  Message Start     → Khởi tạo khi nhận message
  ⏰ Timer Start        → Khởi tạo theo lịch (cron)

Intermediate Events:
  ⏰ Timer Catch        → Chờ 1 khoảng thời gian
  ✉  Message Catch     → Chờ nhận message
  ⚡ Signal Catch       → Chờ signal broadcast

End Events:
  ●  None End           → Kết thúc bình thường
  ✉  Message End       → Gửi message khi kết thúc
  ⚠  Error End          → Kết thúc với lỗi (trigger catch)
  ✖  Terminate End      → Kill toàn bộ process instance
```

---

## 5. External Task Deep Dive

### 5.1 Luồng chi tiết

```
                    Camunda Engine
                    ┌──────────────────┐
                    │  External Task   │
                    │  Queue           │
                    │                  │
  BPMN Process ───▶│  [verifyNewAML]  │◀─── fetchAndLock (long poll)
  tạo task          │  [checkCIC]      │          │
                    │  [sendSMS]       │          │
                    └──────────────────┘          │
                                                  │
                    ┌──────────────────┐          │
                    │   Worker (App)   │──────────┘
                    │                  │
                    │  1. Fetch & Lock │
                    │  2. Execute      │
                    │  3. Complete     │──── POST /complete
                    │     or Fail      │──── POST /failure
                    └──────────────────┘
```

### 5.2 Fetch & Lock mechanism

```
Worker                          Camunda Engine
  │                                   │
  │── POST /external-task/fetchAndLock ──▶│
  │   {                               │
  │     workerId: "worker-1",         │
  │     maxTasks: 10,                 │
  │     topics: [{                    │
  │       topicName: "verifyNewAML",  │
  │       lockDuration: 30000        │
  │     }]                            │
  │   }                               │
  │                                   │
  │◀──── Response (tasks) ────────────│
  │                                   │
  │   (xử lý task)                    │
  │                                   │
  │── POST /external-task/{id}/complete ─▶│
  │   hoặc                            │
  │── POST /external-task/{id}/failure ──▶│
```

**Lock Duration** rất quan trọng:
- Quá ngắn → task bị unlock, worker khác lấy → **duplicate processing**
- Quá dài → worker chết, task bị lock lâu → **process bị treo**

### 5.3 Retry & Error Handling

```java
taskService.handleFailure(
    externalTask.getId(),
    ERROR_MSG_CAMUNDA,                    // error message
    ExceptionUtils.getRootCauseMessage(e), // error details
    retries,                               // số retry còn lại
    timeout                                // thời gian chờ trước khi retry
);
```

**Retry strategy phổ biến:**
```
Lần 1 fail → chờ 10s  → retry
Lần 2 fail → chờ 30s  → retry
Lần 3 fail → chờ 60s  → retry (hết retry)
         → Task chuyển sang incident
         → Cần xử lý thủ công trên Cockpit
```

---

## 6. Process Variables

```java
// Đọc variable
Long applicationId = (Long) externalTask.getVariable("applicationId");

// Ghi variable khi complete
Map<String, Object> variables = new HashMap<>();
variables.put("amlResult", "PASSED");
variables.put("riskScore", 85);
externalTaskService.complete(externalTask, variables);
```

**Lưu ý:**
- Variables được lưu trong DB của Camunda → **không nên lưu data lớn**
- Chỉ lưu ID, status, flag → query chi tiết từ DB riêng
- Serializable objects có thể gây vấn đề version khi deploy

---

## 7. Kiến trúc triển khai thực tế (Production)

### 7.1 Mô hình cơ bản

```
┌─────────────────────────────────────────────────┐
│            Kubernetes Cluster                    │
│                                                  │
│  ┌──────────────┐     ┌──────────────────────┐  │
│  │ Camunda       │     │ Spring Boot App      │  │
│  │ Engine        │     │ (Workers)            │  │
│  │ (Standalone)  │◀───│                      │  │
│  │               │     │ VerifyAmlWorkflow    │  │
│  │ REST API      │     │ CheckCICWorkflow     │  │
│  │ Cockpit UI    │     │ SendSMSWorkflow      │  │
│  └──────┬───────┘     └──────────────────────┘  │
│         │                                        │
│  ┌──────▼───────┐     ┌──────────────────────┐  │
│  │ PostgreSQL    │     │ App Database         │  │
│  │ (Camunda DB)  │     │ (AML, Application..) │  │
│  └──────────────┘     └──────────────────────┘  │
└─────────────────────────────────────────────────┘
```

### 7.2 Mô hình scale cho high throughput

```
┌──────────────────────────────────────────────────────┐
│                                                       │
│  Camunda Engine (HA)          Worker Pods             │
│  ┌────────┐ ┌────────┐      ┌─────────────────┐     │
│  │Engine 1│ │Engine 2│      │ AML Worker x3   │     │
│  └───┬────┘ └───┬────┘      ├─────────────────┤     │
│      │          │            │ CIC Worker x2   │     │
│      ▼          ▼            ├─────────────────┤     │
│  ┌──────────────────┐       │ SMS Worker x5   │     │
│  │  PostgreSQL (HA)  │       └─────────────────┘     │
│  │  + Read Replicas  │                                │
│  └──────────────────┘                                │
└──────────────────────────────────────────────────────┘
```

→ Mỗi worker type scale **độc lập** theo load.

---

## 8. BPMN Design Patterns thường gặp

### 8.1 Saga Pattern (Compensation)

```
Start → [Tạo đơn] → [Check AML] → [Check CIC] → [Approve] → End
                         │              │
                    (fail)│         (fail)│
                         ▼              ▼
                   [Rollback đơn]  [Rollback AML result]
```

### 8.2 Parallel Execution

```
                    ┌→ [Check AML] ──┐
Start → ◈ (AND) ──┤                 ├── ◈ (AND join) → [Decision] → End
                    └→ [Check CIC] ──┘
```

### 8.3 Event Sub-Process (Timeout handling)

```
┌─────────────────────────────────────┐
│  Main Process                        │
│  Start → [Call API] → [Process] → End│
│                                      │
│  ┌─ Event Sub-Process ────────────┐ │
│  │ ⏰ Timer (30 min) → [Timeout   │ │
│  │                      Handler]  │ │
│  └────────────────────────────────┘ │
└─────────────────────────────────────┘
```

### 8.4 Message Correlation (Async callback)

```
Process A:                          External System:
Start → [Gửi request] →            
        ✉ Message Catch ◀──────── POST /message/correlate
        (chờ callback)              {messageName: "amlResult",
        → [Xử lý result] → End      correlationKeys: {appId: 123}}
```

---

## 9. DMN (Decision Model and Notation)

```
┌─────────────────────────────────────────────┐
│  Decision Table: "AML Risk Level"            │
├──────────┬───────────┬──────────────────────┤
│ Score    │ Country   │ Result               │
├──────────┼───────────┼──────────────────────┤
│ > 80     │ *         │ HIGH_RISK            │
│ 50-80    │ VN        │ MEDIUM_RISK          │
│ 50-80    │ != VN     │ HIGH_RISK            │
│ < 50     │ *         │ LOW_RISK             │
└──────────┴───────────┴──────────────────────┘
```

→ Thay đổi rule **không cần deploy lại code**, chỉ deploy lại DMN file.

---

## 10. Monitoring & Operations

### Camunda Cockpit
- Xem process instances đang chạy
- Xem **incidents** (task fail hết retry)
- Retry incidents thủ công
- Xem process variables
- Migration process version

### Metrics quan trọng cần monitor

```
- External task queue size (per topic)    → quá lớn = worker chậm
- Job execution throughput                → bao nhiêu task/s
- Incident count                          → cần alert
- Process instance duration               → SLA tracking
- Active process instances                → memory pressure
```

---

## 11. Best Practices

### 1. Idempotency
Worker phải idempotent vì task có thể bị xử lý lại:
```java
var existing = amlService.findByApplicationId(applicationId);
if (existing != null && "COMPLETED".equals(existing.getStatus())) {
    externalTaskService.complete(externalTask);
    return;
}
```

### 2. Không lưu data lớn vào process variables
Chỉ lưu ID/flag, query chi tiết từ DB riêng.

### 3. Versioning
- Process instances cũ chạy trên version cũ
- Instances mới chạy trên version mới
- Dùng **migration API** nếu cần chuyển instances sang version mới

### 4. Phân biệt Business Error vs Technical Error

```java
// Business error → throw BPMN error → đi nhánh khác trong process
externalTaskService.handleBpmnError(externalTask, "AML_REJECTED", "Customer failed AML check");

// Technical error → retry
externalTaskService.handleFailure(externalTask, "API timeout", details, retries, timeout);
```

### 5. Correlation ID
Luôn log processInstanceId + businessKey để trace:
```java
log.info("Processing task [{}] for process [{}] businessKey [{}]",
    externalTask.getId(),
    externalTask.getProcessInstanceId(),
    externalTask.getBusinessKey());
```


---

## 12. Deep Dive: Camunda External Task Polling - Cơ chế bên trong

### 12.1 Tổng quan kiến trúc internal

Khi Spring Boot khởi động, Camunda Client tự động tạo một **polling loop** chạy trên thread riêng. Dưới đây là cách nó hoạt động từ source code thực tế (Camunda 7.22.0):

```
Spring Boot Start
       │
       ▼
┌─────────────────────────────────────────────────────────────┐
│  1. SubscriptionPostProcessor (BeanDefinitionRegistryPostProcessor)  │
│     - Scan tất cả bean có @ExternalTaskSubscription                  │
│     - Tạo SpringTopicSubscriptionImpl cho mỗi topic                 │
│     - Merge config từ annotation + application.yml                   │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  2. ClientFactory (tạo ExternalTaskClient)                           │
│     - Đọc ClientConfiguration (base-url, worker-id, max-tasks...)   │
│     - Tạo EngineClient (HTTP client tới Camunda REST API)           │
│     - Tạo TopicSubscriptionManager                                   │
│     - Start polling thread                                           │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│  3. TopicSubscriptionManager implements Runnable                      │
│     - Chạy vòng lặp while(isRunning) trên thread riêng              │
│     - Mỗi vòng: acquire() → fetchAndLock() → handleExternalTask()   │
│     - Áp dụng BackoffStrategy khi không có task                      │
└─────────────────────────────────────────────────────────────┘
```

### 12.2 TopicSubscriptionManager - Trái tim của polling

Đây là class quan trọng nhất, implement `Runnable` và chạy trên 1 thread riêng:

```java
// Pseudo-code từ decompile TopicSubscriptionManager.class
public class TopicSubscriptionManager implements Runnable {

    protected CopyOnWriteArrayList<TopicSubscription> subscriptions;  // danh sách topic đăng ký
    protected Map<String, ExternalTaskHandler> externalTaskHandlers;   // map topic → handler
    protected EngineClient engineClient;                               // HTTP client
    protected BackoffStrategy backoffStrategy;                         // chiến lược chờ
    protected AtomicBoolean isRunning;
    protected long clientLockDuration;

    // POLLING LOOP - chạy liên tục trên thread riêng
    @Override
    public void run() {
        while (isRunning.get()) {
            try {
                acquire();
            } catch (Throwable t) {
                LOG.exceptionWhileAcquiringTasks(t);
            }
        }
    }

    // MỖI VÒNG LẶP
    protected void acquire() {
        // 1. Clear và chuẩn bị request cho tất cả subscriptions
        taskTopicRequests.clear();
        externalTaskHandlers.clear();
        subscriptions.forEach(this::prepareAcquisition);

        if (!taskTopicRequests.isEmpty()) {
            // 2. Gọi HTTP POST /external-task/fetchAndLock
            FetchAndLockResponseDto response = fetchAndLock(taskTopicRequests);

            // 3. Với mỗi task nhận được → dispatch tới handler tương ứng
            response.getExternalTasks().forEach(task -> {
                String topicName = task.getTopicName();
                ExternalTaskHandler handler = externalTaskHandlers.get(topicName);
                handleExternalTask(task, handler);
            });

            // 4. Áp dụng backoff strategy
            if (!isBackoffStrategyDisabled.get()) {
                runBackoffStrategy(response);
            }
        }
    }

    // CHUẨN BỊ REQUEST CHO MỖI TOPIC
    protected void prepareAcquisition(TopicSubscription subscription) {
        TopicRequestDto request = TopicRequestDto.fromTopicSubscription(subscription, clientLockDuration);
        taskTopicRequests.add(request);
        externalTaskHandlers.put(subscription.getTopicName(), subscription.getExternalTaskHandler());
    }

    // GỌI API CAMUNDA ENGINE
    protected FetchAndLockResponseDto fetchAndLock(List<TopicRequestDto> requests) {
        try {
            List<ExternalTask> tasks = engineClient.fetchAndLock(requests);
            return new FetchAndLockResponseDto(tasks);
        } catch (EngineClientException e) {
            return new FetchAndLockResponseDto(e);  // trả về empty, không crash
        }
    }

    // XỬ LÝ TASK - gọi handler.execute()
    protected void handleExternalTask(ExternalTask task, ExternalTaskHandler handler) {
        handler.execute(task, externalTaskService);
    }
}
```

**Điểm quan trọng:**
- Tất cả topics được gom vào **1 request fetchAndLock duy nhất** → giảm số HTTP calls
- Task được dispatch tới đúng handler dựa trên `topicName`
- Nếu fetchAndLock fail → không crash, chỉ log và tiếp tục vòng lặp
- `handleExternalTask` chạy **đồng bộ trên cùng thread** → task được xử lý tuần tự

### 12.3 BackoffStrategy - Chiến lược chờ khi không có task

Camunda dùng `ExponentialBackoffStrategy` mặc định:

```java
// Default values từ decompile:
public class ExponentialBackoffStrategy implements BackoffStrategy {
    protected long initTime = 500;     // 500ms ban đầu
    protected float factor = 2.0f;     // nhân đôi mỗi lần
    protected long maxTime = 60000;    // tối đa 60s
    protected int level = 0;

    // Khi không có task → tăng level
    // Khi có task → reset level = 0
    public void reconfigure(List<ExternalTask> tasks) {
        if (tasks.isEmpty()) {
            level++;        // không có task → chờ lâu hơn
        } else {
            level = 0;      // có task → reset, poll ngay
        }
    }

    // Tính thời gian chờ: initTime * factor^(level-1), tối đa maxTime
    public long calculateBackoffTime() {
        if (level == 0) return 0;   // có task → không chờ
        long backoff = (long)(initTime * Math.pow(factor, level - 1));
        return Math.min(backoff, maxTime);
    }
}
```

**Ví dụ thực tế:**
```
Lần poll 1: có task    → xử lý ngay, poll tiếp ngay (0ms)
Lần poll 2: có task    → xử lý ngay, poll tiếp ngay (0ms)
Lần poll 3: không task → chờ 500ms
Lần poll 4: không task → chờ 1000ms  (500 * 2^1)
Lần poll 5: không task → chờ 2000ms  (500 * 2^2)
Lần poll 6: không task → chờ 4000ms  (500 * 2^3)
...
Lần poll N: không task → chờ 60000ms (max cap)
Lần poll N+1: có task  → reset, xử lý ngay
```

### 12.4 Long Polling (async-response-timeout)

Ngoài backoff, Camunda còn hỗ trợ **long polling** ở phía Engine:

```
Worker                              Camunda Engine
  │                                       │
  │── POST /fetchAndLock ────────────────▶│
  │   { asyncResponseTimeout: 1000 }      │
  │                                       │
  │   Engine GIỮA connection mở           │
  │   chờ tối đa 1000ms                   │
  │   nếu có task mới → trả về ngay       │
  │   nếu hết timeout → trả về empty      │
  │                                       │
  │◀──── Response ────────────────────────│
```

**Kết hợp cả 2:**
```
async-response-timeout (server-side) + BackoffStrategy (client-side)
= Giảm thiểu số request không cần thiết + phản hồi nhanh khi có task
```

---

## 13. Cấu hình chi tiết trong project

### 13.1 Cấu hình Global (application.yml)

```yaml
camunda.bpm.client:
  # URL tới Camunda Engine REST API
  base-url: ${CAMUNDA_URL:http://localhost:8080/engine-rest}

  # Long polling timeout (ms) - Engine giữ connection bao lâu nếu chưa có task
  async-response-timeout: ${CAMUNDA_ASYNC_RES_TIMEOUT:1000}

  # Định danh worker - quan trọng khi có nhiều worker instances
  worker-id: ${CAMUNDA_WORKER_ID:bbs3-worker-client}

  # Số task tối đa fetch trong 1 lần poll
  max-tasks: ${CAMUNDA_MAX_TASK:2}

  # Thời gian lock mặc định (ms) - task bị lock bao lâu
  lock-duration: ${CAMUNDA_LOCK_DURATION:20000}

  # Process definition key mặc định - chỉ fetch task từ process này
  process-definition-key: ${CAMUNDA_PROCESS_DEF_KEY:onboarding-flow}
```

### 13.2 Cấu hình Per-Topic (subscriptions)

```yaml
camunda.bpm.client:
  subscriptions:
    # Topic verifyNewAML - lắng nghe từ 2 process definitions
    verifyNewAML:
      process-definition-key-in: ${CAMUNDA_RM_PROCESS_DEF_KEY:onboarding-flow,onboarding-assistant-flow}

    # Topic syncLeadId - chỉ lắng nghe từ 1 process khác
    syncLeadId:
      process-definition-key: ${CAMUNDA_GCM_PROCESS_KEY:gcm-sync-lead}

    # Topic Topic_create_merchant - override lock-duration riêng
    Topic_create_merchant:
      process-definition-key: ${CAMUNDA_PROCESS_DEF_KEY:onboarding-flow}
      lock-duration: ${CREATE_MERCHANT_LOCK_DURATION:30000}  # 30s thay vì 20s mặc định
```

### 13.3 Cấu hình Retry (AbstractExternalTask)

```yaml
tcb:
  external-task:
    max-retry: ${EXTERNAL_TASK_MAX_RETRY:5}    # Số lần retry tối đa
    time-out: ${EXTERNAL_TASK_TIME_OUT:60}      # Base timeout giữa các retry (seconds)
  bpmn-error: ${BPMN_ERROR:false}               # Flag bật/tắt BPMN error handling
```

**Retry timeout tăng dần (từ AbstractExternalTask):**
```
Retry 5 (lần đầu fail): chờ 60s   (1 * 60s)
Retry 4:                 chờ 60s   (1 * 60s)  
Retry 3:                 chờ 120s  (2 * 60s)
Retry 2:                 chờ 180s  (3 * 60s)
Retry 1:                 chờ 240s  (4 * 60s)
Retry 0:                 → INCIDENT (cần xử lý thủ công)
```

---

## 14. Tất cả config có thể tuỳ chỉnh

### 14.1 Global Config (ClientConfiguration)

| Property | Mô tả | Default |
|---|---|---|
| `base-url` | URL tới Camunda REST API | bắt buộc |
| `worker-id` | ID định danh worker | auto-generated |
| `max-tasks` | Số task tối đa mỗi lần fetch | 10 |
| `lock-duration` | Thời gian lock mặc định (ms) | 20000 |
| `async-response-timeout` | Long polling timeout (ms) | không set = short polling |
| `disable-auto-fetching` | Tắt auto polling khi start | false |
| `disable-backoff-strategy` | Tắt exponential backoff | false |
| `use-priority` | Ưu tiên task theo priority | true |
| `use-create-time` | Sắp xếp theo thời gian tạo | false |
| `order-by-create-time` | asc hoặc desc | asc |
| `default-serialization-format` | Format serialize variables | application/json |
| `date-format` | Format ngày tháng | yyyy-MM-dd'T'HH:mm:ss.SSSZ |

### 14.2 Per-Topic Config (SubscriptionConfiguration)

| Property | Mô tả | Ví dụ |
|---|---|---|
| `lock-duration` | Override lock duration cho topic này | 30000 |
| `variable-names` | Chỉ fetch các variable cần thiết | applicationId,caseId |
| `local-variables` | Chỉ lấy local variables | false |
| `business-key` | Filter theo business key | |
| `process-definition-id` | Filter theo process definition ID | |
| `process-definition-key` | Filter theo 1 process key | onboarding-flow |
| `process-definition-key-in` | Filter theo nhiều process keys | onboarding-flow,assistant-flow |
| `process-definition-version-tag` | Filter theo version tag | v2.0 |
| `process-variables` | Filter theo giá trị variable | |
| `without-tenant-id` | Chỉ lấy task không có tenant | false |
| `tenant-id-in` | Filter theo tenant IDs | |
| `include-extension-properties` | Bao gồm extension properties | false |
| `auto-open` | Tự động bắt đầu subscribe | true |

### 14.3 Annotation vs YAML - Thứ tự ưu tiên

```
@ExternalTaskSubscription(topicName="verifyNewAML", lockDuration=30000)
                    ↓
         YAML config override annotation
                    ↓
camunda.bpm.client.subscriptions.verifyNewAML.lock-duration=50000  ← THẮNG
```

**YAML luôn override annotation** → cho phép thay đổi config mà không cần build lại code.

---

## 15. Ví dụ tuỳ chỉnh theo nghiệp vụ

### 15.1 Topic gọi API chậm (AML check) - tăng lock duration

```yaml
camunda.bpm.client:
  subscriptions:
    verifyNewAML:
      lock-duration: 120000  # 2 phút vì API AML response chậm
      process-definition-key-in: onboarding-flow,onboarding-assistant-flow
```

### 15.2 Topic cần xử lý nhanh (Send SMS) - giảm lock, tăng max-tasks

```yaml
camunda.bpm.client:
  subscriptions:
    sendSMS:
      lock-duration: 5000    # 5s là đủ
      variable-names: phoneNumber,message  # chỉ fetch variable cần thiết → giảm payload
```

### 15.3 Topic chỉ chạy cho 1 version cụ thể

```yaml
camunda.bpm.client:
  subscriptions:
    newFeatureTask:
      process-definition-version-tag: v2.0  # chỉ xử lý process version 2.0
```

### 15.4 Multi-tenant - mỗi worker xử lý tenant riêng

```yaml
# Worker cho tenant A
camunda.bpm.client:
  worker-id: worker-tenant-a
  subscriptions:
    verifyNewAML:
      tenant-id-in: tenant-a

# Worker cho tenant B (deploy riêng)
camunda.bpm.client:
  worker-id: worker-tenant-b
  subscriptions:
    verifyNewAML:
      tenant-id-in: tenant-b
```

### 15.5 Tắt auto-fetching để control thủ công

```yaml
camunda.bpm.client:
  disable-auto-fetching: true  # không tự poll khi start
```

```java
// Start/stop polling thủ công
@Autowired
private SpringTopicSubscription verifyNewAMLSubscription;

public void startPolling() {
    verifyNewAMLSubscription.open();   // bắt đầu poll
}

public void stopPolling() {
    verifyNewAMLSubscription.close();  // dừng poll
}
```


---

## 16. Phân tích cơ chế Polling trên service coc-origination

### 16.1 Tổng quan: 10 workers, 1 polling thread, 1 HTTP request

Service này có **10 External Task Workers** nhưng tất cả đều được gom vào **1 polling thread duy nhất** bởi `TopicSubscriptionManager`:

```
                         TopicSubscriptionManager (1 thread)
                                    │
                         while(isRunning) {
                              acquire();
                         }
                                    │
                    ┌───────────────┼───────────────┐
                    ▼                               ▼
          prepareAcquisition()              fetchAndLock()
          (gom 10 topics)                   (1 HTTP POST duy nhất)
                    │                               │
                    ▼                               ▼
    ┌─────────────────────────────┐    POST /external-task/fetchAndLock
    │ topics: [                   │    {
    │   "verifyNewAML",           │      workerId: "bbs3-worker-client",
    │   "syncLeadId",             │      maxTasks: 2,
    │   "remind_customer",        │      topics: [ ...10 topics... ]
    │   "first_remind_customer",  │    }
    │   "releaseLan",             │
    │   "inActiveCorpCus",        │         │
    │   "ekycExpiredOrFailed",    │         ▼
    │   "origination_check_...",  │    Response: List<ExternalTask>
    │   "Topic_create_merchant",  │         │
    │   "after_1day_remind_..."   │         ▼
    │ ]                           │    forEach(task -> {
    └─────────────────────────────┘      handler = handlers.get(task.topicName);
                                         handler.execute(task, service);
                                       })
```

### 16.2 Bảng tổng hợp 10 Workers

| # | Topic | Worker Class | Process Definition | Lock Duration | Đặc điểm |
|---|---|---|---|---|---|
| 1 | `verifyNewAML` | VerifyAmlWorkflow | onboarding-flow, onboarding-assistant-flow | 20s (global) | Gọi API AML bên ngoài, custom retry lưu AML khi hết retry |
| 2 | `syncLeadId` | GcmSyncLead | gcm-sync-lead | 20s (global) | Process riêng biệt, auto-complete khi hết retry |
| 3 | `remind_customer` | RemindCustomerTask | onboarding-flow | 20s (global) | Gửi email nhắc nhở, logic đơn giản |
| 4 | `first_remind_customer` | FirstRemindCustomerTask | onboarding-flow-first-remind | 20s (global) | Tương tự remind nhưng process khác |
| 5 | `after_1day_remind_customer` | AfterOpsFeedbackRemindCustomerTask | *(không config trong YAML)* | 20s (global) | ⚠️ Thiếu config subscription trong YAML |
| 6 | `releaseLan` | ReleaseLanTask | onboarding-flow, onboarding-assistant-flow | 20s (global) | Gọi API T24 release Lucky Account Number |
| 7 | `inActiveCorpCus` | InActiveCustomerTask | onboarding-flow, onboarding-assistant-flow | 20s (global) | Gọi API T24 inactive CIF |
| 8 | `ekycExpiredOrFailed` | CustomerEkycWorkFollow | onboarding-assistant-flow | 20s (global) | Không có try-catch retry, task luôn complete |
| 9 | `origination_check_enrollment_status` | CheckEnrollmentStatusTask | onboarding-flow | 20s (global) | Auto-complete với flag=false khi hết retry |
| 10 | `Topic_create_merchant` | CreateMerchantTask | onboarding-flow | **30s (override)** | Duy nhất override lock-duration, luôn complete kể cả lỗi |

### 16.3 Phân tích luồng Polling chi tiết

#### Bước 1: Khởi động (Spring Boot Start)

```
Spring Boot Start
       │
       ▼
SubscriptionPostProcessor scan 10 bean có @ExternalTaskSubscription
       │
       ▼
Tạo 10 SpringTopicSubscriptionImpl, merge config:
  - verifyNewAML:  annotation(topicName) + YAML(process-definition-key-in)
  - syncLeadId:    annotation(topicName) + YAML(process-definition-key)
  - Topic_create_merchant: annotation(topicName) + YAML(lock-duration=30000)
  - ...
       │
       ▼
ClientFactory tạo ExternalTaskClient với config:
  - base-url: http://camunda-engine/engine-rest
  - worker-id: bbs3-worker-client
  - max-tasks: 2
  - lock-duration: 20000 (default)
  - async-response-timeout: 1000
       │
       ▼
TopicSubscriptionManager.start()
  → Tạo Thread mới, chạy run() → vòng lặp polling bắt đầu
```

#### Bước 2: Mỗi vòng lặp Polling

```
┌─────────────────────────────────────────────────────────────────┐
│  acquire() - MỖI VÒNG LẶP                                       │
│                                                                   │
│  1. Clear taskTopicRequests & externalTaskHandlers                │
│                                                                   │
│  2. Duyệt 10 subscriptions → prepareAcquisition():               │
│     taskTopicRequests = [                                         │
│       { topicName: "verifyNewAML",                                │
│         lockDuration: 20000,                                      │
│         processDefinitionKeyIn: ["onboarding-flow",               │
│                                   "onboarding-assistant-flow"] }, │
│       { topicName: "syncLeadId",                                  │
│         lockDuration: 20000,                                      │
│         processDefinitionKey: "gcm-sync-lead" },                  │
│       { topicName: "Topic_create_merchant",                       │
│         lockDuration: 30000,           ← override riêng           │
│         processDefinitionKey: "onboarding-flow" },                │
│       ... 7 topics khác                                           │
│     ]                                                             │
│     externalTaskHandlers = {                                      │
│       "verifyNewAML" → VerifyAmlWorkflow instance,                │
│       "syncLeadId" → GcmSyncLead instance,                       │
│       ...                                                         │
│     }                                                             │
│                                                                   │
│  3. fetchAndLock(taskTopicRequests)                                │
│     → POST http://camunda-engine/engine-rest/external-task/       │
│            fetchAndLock                                            │
│     → Body:                                                       │
│       {                                                           │
│         "workerId": "bbs3-worker-client",                         │
│         "maxTasks": 2,              ← CHỈ LẤY TỐI ĐA 2 TASK     │
│         "asyncResponseTimeout": 1000,                             │
│         "topics": [ ...10 topic requests... ]                     │
│       }                                                           │
│     → Engine trả về tối đa 2 tasks (có thể từ bất kỳ topic nào) │
│                                                                   │
│  4. Với mỗi task nhận được:                                       │
│     handler = externalTaskHandlers.get(task.topicName)            │
│     handler.execute(task, externalTaskService)                    │
│     → XỬ LÝ TUẦN TỰ trên cùng thread                            │
│                                                                   │
│  5. runBackoffStrategy(response)                                  │
│     → Nếu có task: reset level=0, poll ngay                      │
│     → Nếu không task: level++, chờ 500ms → 1s → 2s → ... → 60s  │
└─────────────────────────────────────────────────────────────────┘
```

#### Bước 3: Xử lý task (ví dụ verifyNewAML)

```
TopicSubscriptionManager thread
       │
       ▼
VerifyAmlWorkflow.execute(externalTask, externalTaskService)
       │
       ├─ Đọc applicationId từ process variable
       ├─ Tạo AML entity
       ├─ Build AML request
       ├─ Gọi API AML (HTTP call ra ngoài)  ← BLOCKING trên polling thread!
       │
       ├─ Thành công:
       │    amlService.save(aml)
       │    externalTaskService.complete(externalTask)
       │    → POST /external-task/{id}/complete
       │
       └─ Thất bại:
            retries = getRetries(task)  // null → 5, rồi 4, 3, 2, 1, 0
            timeout = getNextTimeout(retries)
            │
            ├─ retries > 0:
            │    taskService.handleFailure(id, msg, details, retries, timeout)
            │    → POST /external-task/{id}/failure
            │    → Engine sẽ unlock task sau timeout, worker poll lại
            │
            └─ retries <= 0:
                 amlService.save(aml)  // lưu AML với trạng thái lỗi
                 taskService.handleFailure(id, msg, details, 0, timeout)
                 → Task chuyển thành INCIDENT trên Cockpit
```

### 16.4 Timeline thực tế của 1 chu kỳ polling

```
Timeline (giả sử có 2 task: 1 verifyNewAML + 1 remind_customer)

t=0ms     │ POST /fetchAndLock (10 topics, maxTasks=2)
          │ Engine giữ connection (long poll)
t=50ms    │ Engine trả về 2 tasks
          │
t=50ms    │ ── Bắt đầu xử lý task 1: verifyNewAML ──
          │ Đọc variable, build request
t=100ms   │ Gọi API AML... (chờ response)
t=2100ms  │ API AML trả về (2s)
          │ Save DB, complete task
          │ POST /external-task/{id1}/complete
          │ ── Kết thúc task 1 ──
          │
t=2150ms  │ ── Bắt đầu xử lý task 2: remind_customer ──
          │ Đọc variable, gửi email
t=2300ms  │ Email sent, complete task
          │ POST /external-task/{id2}/complete
          │ ── Kết thúc task 2 ──
          │
t=2300ms  │ BackoffStrategy: có task → level=0 → không chờ
          │
t=2300ms  │ POST /fetchAndLock (vòng tiếp theo)
          │ ...
```

### 16.5 Phân tích các vấn đề và rủi ro

#### ⚠️ Vấn đề 1: max-tasks=2 + xử lý tuần tự = BOTTLENECK

```
Hiện tại:
- 10 topics nhưng mỗi lần chỉ fetch 2 tasks
- Xử lý TUẦN TỰ trên 1 thread
- Nếu 1 task mất 5s (gọi API chậm) → task còn lại phải chờ

Ví dụ worst case:
  Task 1 (CreateMerchant): 10s (gọi nhiều API)
  Task 2 (VerifyAML): 5s (API AML chậm)
  → Tổng 15s cho 2 tasks
  → Throughput: ~8 tasks/phút
  → Nếu có 100 tasks pending → xử lý hết ~12 phút
```

#### ⚠️ Vấn đề 2: async-response-timeout=1000 (1s) quá ngắn

```
Hiện tại: Engine chỉ giữ connection 1s
- Nếu không có task → trả về empty sau 1s
- Client áp dụng backoff: 500ms → 1s → 2s → ... → 60s
- Khi idle, mỗi phút vẫn có ~1 request (sau khi đạt max backoff)

Khuyến nghị: Tăng lên 10000-30000ms
- Giảm số request không cần thiết
- Engine sẽ trả về ngay khi có task mới (không cần chờ hết timeout)
```

#### ⚠️ Vấn đề 3: lock-duration=20s có thể không đủ cho một số task

```
verifyNewAML:
  - Gọi API AML bên ngoài, có thể mất 5-10s
  - Lock 20s → còn 10-15s buffer → OK nhưng sát

CreateMerchant:
  - Gọi nhiều API: getCustomer, getSoftposFee, submitMerchant
  - Lock 30s (đã override) → hợp lý hơn

releaseLan:
  - Gọi API T24 (core banking) có thể chậm
  - Lock 20s → có thể không đủ nếu T24 chậm
  - ⚠️ Nên override lock-duration riêng
```

#### ⚠️ Vấn đề 4: Topic `after_1day_remind_customer` thiếu config YAML

```yaml
# Trong YAML chỉ có:
subscriptions:
  verifyNewAML: ...
  syncLeadId: ...
  remind_customer: ...
  first_remind_customer: ...
  releaseLan: ...
  inActiveCorpCus: ...
  ekycExpiredOrFailed: ...
  origination_check_enrollment_status: ...
  Topic_create_merchant: ...
  # ❌ THIẾU: after_1day_remind_customer

# Trong code:
@ExternalTaskSubscription(topicName = "after_1day_remind_customer")
public class AfterOpsFeedbackRemindCustomerTask ...
```

**Hậu quả:** Topic này sẽ dùng global `process-definition-key: onboarding-flow` thay vì có config riêng. Nếu đúng ý đồ thì OK, nhưng nên explicit config để rõ ràng.

#### ⚠️ Vấn đề 5: Error handling không nhất quán giữa các workers

```
Pattern 1 - Retry rồi thành Incident (mặc định):
  → RemindCustomerTask, FirstRemindCustomerTask, AfterOpsFeedbackRemindCustomerTask
  → InActiveCustomerTask, ReleaseLanTask

Pattern 2 - Retry rồi auto-complete với error info:
  → GcmSyncLead: hết retry → complete(task, {exceptionName: ...})
  → CheckEnrollmentStatusTask: hết retry → complete(task, {isBioMetric: "FALSE"})

Pattern 3 - Retry rồi lưu DB khi hết retry:
  → VerifyAmlWorkflow: hết retry → amlService.save(aml) rồi handleFailure

Pattern 4 - Không retry, luôn complete:
  → CreateMerchantTask: catch Exception → externalTaskService.complete(externalTask)
  → CustomerEkycWorkFollow: không có try-catch cho exception
```

#### ⚠️ Vấn đề 6: CreateMerchantTask nuốt exception

```java
// CreateMerchantTask.java
} catch (Exception ex) {
    log.error("...", applicationId, ExceptionUtils.getStackTrace(ex));
    externalTaskService.complete(externalTask);  // ← LUÔN COMPLETE dù lỗi!
}
```

**Hậu quả:** Nếu tạo merchant thất bại, process vẫn tiếp tục như bình thường. Không có cơ chế retry hay thông báo lỗi qua Camunda. Lỗi chỉ nằm trong log.

#### ⚠️ Vấn đề 7: CustomerEkycWorkFollow không có error handling

```java
// CustomerEkycWorkFollow.java - KHÔNG có try-catch
@Override
public void execute(ExternalTask externalTask, ExternalTaskService externalTaskService) {
    var applicationId = Long.parseLong(externalTask.getVariable("applicationId"));
    // ... logic ...
    externalTaskService.complete(externalTask);
    // Nếu exception xảy ra → task bị lock 20s → unlock → retry vô hạn
}
```

**Hậu quả:** Nếu exception xảy ra (ví dụ DB down), task sẽ bị retry vô hạn (không có handleFailure giảm retries).

### 16.6 Retry Strategy chi tiết (AbstractExternalTask)

```
Config: max-retry=5, time-out=60s

Lần fail đầu tiên:
  retries = null → set = 5
  timeout = 1 * 60s = 60s (vì maxRetries == retries)

Lần fail thứ 2:
  retries = 5 → 5-1 = 4
  timeout = (5-4) * 60s = 60s

Lần fail thứ 3:
  retries = 4 → 4-1 = 3
  timeout = (5-3) * 60s = 120s

Lần fail thứ 4:
  retries = 3 → 3-1 = 2
  timeout = (5-2) * 60s = 180s

Lần fail thứ 5:
  retries = 2 → 2-1 = 1
  timeout = (5-1) * 60s = 240s

Lần fail thứ 6:
  retries = 1 → 1-1 = 0
  timeout = (5-0) * 60s = 300s
  → Task chuyển thành INCIDENT (retries=0)

Tổng thời gian retry: 60+60+120+180+240+300 = 960s ≈ 16 phút
```

### 16.7 Sơ đồ tổng hợp toàn bộ cơ chế

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        coc-origination Service                          │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  TopicSubscriptionManager (1 Thread)                             │   │
│  │                                                                   │   │
│  │  Config: worker-id=bbs3-worker-client, max-tasks=2               │   │
│  │          async-response-timeout=1000ms                            │   │
│  │          backoff: 500ms → 1s → 2s → ... → 60s (exponential)     │   │
│  │                                                                   │   │
│  │  while(isRunning) {                                               │   │
│  │    1. Gom 10 topics vào 1 request                                 │   │
│  │    2. POST /fetchAndLock (max 2 tasks)                            │   │
│  │    3. Xử lý tuần tự từng task                                     │   │
│  │    4. Backoff nếu không có task                                   │   │
│  │  }                                                                │   │
│  └──────────────────────────┬──────────────────────────────────────┘   │
│                              │ dispatch by topicName                    │
│         ┌────────────────────┼────────────────────┐                    │
│         ▼                    ▼                    ▼                    │
│  ┌─────────────┐  ┌──────────────────┐  ┌──────────────────┐         │
│  │ onboarding   │  │ onboarding       │  │ gcm-sync-lead    │         │
│  │ -flow        │  │ -assistant-flow  │  │                  │         │
│  │              │  │                  │  │ • syncLeadId     │         │
│  │ • verifyAML  │  │ • verifyAML      │  └──────────────────┘         │
│  │ • remind     │  │ • releaseLan     │                                │
│  │ • firstRemind│  │ • inActiveCus    │  ┌──────────────────┐         │
│  │ • releaseLan │  │ • ekycExpired    │  │ onboarding-flow  │         │
│  │ • inActiveCus│  │                  │  │ -first-remind    │         │
│  │ • checkEnroll│  └──────────────────┘  │                  │         │
│  │ • merchant   │                         │ • firstRemind   │         │
│  │ • afterRemind│                         └──────────────────┘         │
│  └─────────────┘                                                       │
│                                                                         │
│  AbstractExternalTask: max-retry=5, base-timeout=60s                   │
│  Retry: 60s → 60s → 120s → 180s → 240s → 300s → INCIDENT             │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                    POST /fetchAndLock
                    POST /complete
                    POST /failure
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     Camunda Engine (Standalone)                          │
│                                                                         │
│  BPMN Processes:                                                        │
│  • onboarding-flow              (7 external tasks)                      │
│  • onboarding-assistant-flow    (4 external tasks)                      │
│  • onboarding-flow-first-remind (1 external task)                       │
│  • gcm-sync-lead                (1 external task)                       │
│                                                                         │
│  REST API: /engine-rest/external-task/fetchAndLock                      │
│  Cockpit: Monitoring, Incidents, Retry                                  │
└─────────────────────────────────────────────────────────────────────────┘
```

