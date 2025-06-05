# Bank Statement Processing System - Complete Design

## High-Level Design (HLD)

### 1. System Components

#### Core Components
- **Statement Generator Service**: Orchestrates the entire statement generation process
- **Account Data Service**: Manages account information and transaction data retrieval
- **Template Engine Service**: Handles statement formatting and PDF generation
- **Notification Service**: Manages multi-channel customer notifications
- **File Storage Service**: Handles statement storage and retrieval
- **Queue Management Service**: Manages asynchronous processing queues
- **Audit Service**: Tracks all operations for compliance and monitoring

#### Supporting Components
- **Load Balancer**: Distributes incoming requests across service instances
- **API Gateway**: Single entry point for external communications
- **Configuration Service**: Manages system configurations and feature flags
- **Monitoring Service**: Real-time system health and performance monitoring
- **Security Service**: Authentication, authorization, and encryption

#### External Integrations
- **Core Banking System**: Source of account and transaction data
- **Email Service Provider**: Third-party email delivery (SendGrid, AWS SES)
- **SMS Gateway**: SMS notification delivery
- **Mobile Push Service**: Push notifications for mobile app users
- **Postal Service API**: Physical mail delivery integration

### 2. Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                           Load Balancer                              │
└─────────────────────┬───────────────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────────────┐
│                        API Gateway                                   │
└─────────────────────┬───────────────────────────────────────────────┘
                      │
      ┌───────────────┼───────────────┐
      │               │               │
      ▼               ▼               ▼
┌─────────┐    ┌─────────────┐   ┌──────────────┐
│Statement│    │   Account   │   │ Notification │
│Generator│    │Data Service │   │   Service    │
│Service  │    │             │   │              │
└────┬────┘    └──────┬──────┘   └──────┬───────┘
     │                │                 │
     ▼                ▼                 ▼
┌─────────┐    ┌─────────────┐   ┌──────────────┐
│Template │    │    Cache    │   │Queue Mgmt    │
│Engine   │    │   Service   │   │Service       │
│Service  │    │   (Redis)   │   │(Kafka/RMQ)   │
└────┬────┘    └─────────────┘   └──────┬───────┘
     │                                  │
     ▼                                  ▼
┌─────────┐                      ┌──────────────┐
│  File   │                      │External      │
│Storage  │                      │Integrations  │
│Service  │                      │(Email/SMS)   │
└─────────┘                      └──────────────┘
     │
     ▼
┌─────────────────────────────────────────────────┐
│              Database Layer                     │
│  ┌─────────────┐  ┌─────────────┐ ┌──────────┐ │
│  │ PostgreSQL  │  │   MongoDB   │ │  Redis   │ │
│  │(Metadata)   │  │(Documents)  │ │(Cache)   │ │
│  └─────────────┘  └─────────────┘ └──────────┘ │
└─────────────────────────────────────────────────┘
```

## Low-Level Design (LLD)

### 1. Class Diagram

```java
┌─────────────────────────────────────────────────────────────────────┐
│                        StatementController                          │
├─────────────────────────────────────────────────────────────────────┤
│ - statementService: StatementService                                │
│ - notificationService: NotificationService                          │
├─────────────────────────────────────────────────────────────────────┤
│ + generateMonthlyStatements(): ResponseEntity                       │
│ + getStatementStatus(jobId: String): ResponseEntity                 │
│ + retryFailedStatements(jobId: String): ResponseEntity              │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        StatementService                             │
├─────────────────────────────────────────────────────────────────────┤
│ - accountService: AccountService                                    │
│ - templateEngine: TemplateEngine                                    │
│ - fileStorageService: FileStorageService                           │
│ - queueService: QueueService                                        │
│ - auditService: AuditService                                        │
├─────────────────────────────────────────────────────────────────────┤
│ + processStatements(request: StatementRequest): StatementJob        │
│ + generateStatement(account: Account): Statement                    │
│ + validateStatement(statement: Statement): ValidationResult         │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                   ┌────────────────┼────────────────┐
                   ▼                ▼                ▼
┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐
│   AccountService    │  │   TemplateEngine    │  │ NotificationService │
├─────────────────────┤  ├─────────────────────┤  ├─────────────────────┤
│- accountRepo:       │  │- templateRepo:      │  │- emailService:      │
│  AccountRepository  │  │  TemplateRepository │  │  EmailService       │
│- transactionRepo:   │  │- pdfGenerator:      │  │- smsService:        │
│  TransactionRepo    │  │  PDFGenerator       │  │  SMSService         │
├─────────────────────┤  ├─────────────────────┤  ├─────────────────────┤
│+ getAccountData():  │  │+ generatePDF():     │  │+ sendNotification():│
│  AccountData        │  │  byte[]             │  │  NotificationResult │
│+ getTransactions(): │  │+ validateTemplate():│  │+ retryFailed():     │
│  List<Transaction>  │  │  boolean            │  │  void               │
└─────────────────────┘  └─────────────────────┘  └─────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Domain Models                                  │
├─────────────────────────────────────────────────────────────────────┤
│  Account                    │  Statement                            │
│  ├─ accountId: Long         │  ├─ statementId: String               │
│  ├─ customerId: Long        │  ├─ accountId: Long                   │
│  ├─ accountType: enum       │  ├─ period: StatementPeriod           │
│  ├─ balance: BigDecimal     │  ├─ transactions: List<Transaction>   │
│  ├─ status: AccountStatus   │  ├─ summary: StatementSummary         │
│  └─ preferences: Prefs      │  └─ generatedDate: LocalDateTime      │
│                             │                                       │
│  Transaction               │  NotificationRequest                   │
│  ├─ id: String             │  ├─ accountId: Long                    │
│  ├─ amount: BigDecimal     │  ├─ customerId: Long                  │
│  ├─ type: TransactionType  │  ├─ channels: List<Channel>           │
│  ├─ description: String    │  ├─ content: NotificationContent      │
│  └─ timestamp: LocalDT     │  └─ priority: Priority                │
└─────────────────────────────────────────────────────────────────────┘
```

### 2. Java Implementation

#### Core Service Implementation
```java
@Service
@Transactional
public class StatementService {
    
    private final AccountService accountService;
    private final TemplateEngine templateEngine;
    private final FileStorageService fileStorageService;
    private final QueueService queueService;
    private final AuditService auditService;
    
    public StatementService(AccountService accountService, 
                           TemplateEngine templateEngine,
                           FileStorageService fileStorageService,
                           QueueService queueService,
                           AuditService auditService) {
        this.accountService = accountService;
        this.templateEngine = templateEngine;
        this.fileStorageService = fileStorageService;
        this.queueService = queueService;
        this.auditService = auditService;
    }
    
    @Async("statementExecutor")
    public CompletableFuture<StatementJob> processStatements(StatementRequest request) {
        String jobId = generateJobId();
        StatementJob job = new StatementJob(jobId, request.getPeriod(), JobStatus.RUNNING);
        
        try {
            auditService.logJobStart(job);
            
            List<Account> accounts = accountService.getActiveAccounts(request.getFilters());
            List<List<Account>> batches = partitionAccounts(accounts, request.getBatchSize());
            
            List<CompletableFuture<BatchResult>> batchFutures = batches.stream()
                .map(batch -> processBatch(batch, request.getPeriod()))
                .collect(Collectors.toList());
            
            CompletableFuture.allOf(batchFutures.toArray(new CompletableFuture[0]))
                .thenApply(v -> batchFutures.stream()
                    .map(CompletableFuture::join)
                    .collect(Collectors.toList()))
                .thenAccept(results -> {
                    job.setResults(aggregateResults(results));
                    job.setStatus(JobStatus.COMPLETED);
                    auditService.logJobCompletion(job);
                    
                    // Trigger notifications
                    queueService.publishNotificationBatch(job);
                });
                
            return CompletableFuture.completedFuture(job);
            
        } catch (Exception e) {
            job.setStatus(JobStatus.FAILED);
            job.setErrorMessage(e.getMessage());
            auditService.logJobError(job, e);
            throw new StatementProcessingException("Failed to process statements", e);
        }
    }
    
    @Async("batchExecutor")
    public CompletableFuture<BatchResult> processBatch(List<Account> accounts, 
                                                      StatementPeriod period) {
        BatchResult result = new BatchResult();
        
        for (Account account : accounts) {
            try {
                Statement statement = generateStatement(account, period);
                ValidationResult validation = validateStatement(statement);
                
                if (validation.isValid()) {
                    String fileLocation = storeStatement(statement);
                    result.addSuccess(account.getAccountId(), fileLocation);
                } else {
                    result.addFailure(account.getAccountId(), validation.getErrors());
                }
                
            } catch (Exception e) {
                result.addFailure(account.getAccountId(), List.of(e.getMessage()));
            }
        }
        
        return CompletableFuture.completedFuture(result);
    }
    
    private Statement generateStatement(Account account, StatementPeriod period) {
        // Get account data and transactions
        AccountData accountData = accountService.getAccountData(account.getAccountId());
        List<Transaction> transactions = accountService.getTransactions(
            account.getAccountId(), period.getStartDate(), period.getEndDate());
        
        // Calculate statement summary
        StatementSummary summary = calculateSummary(accountData, transactions);
        
        // Create statement object
        Statement statement = Statement.builder()
            .statementId(generateStatementId(account.getAccountId(), period))
            .accountId(account.getAccountId())
            .period(period)
            .transactions(transactions)
            .summary(summary)
            .generatedDate(LocalDateTime.now())
            .build();
        
        return statement;
    }
    
    private String storeStatement(Statement statement) {
        try {
            // Generate PDF
            byte[] pdfContent = templateEngine.generatePDF(statement);
            
            // Store in file system
            String fileName = generateFileName(statement);
            String fileLocation = fileStorageService.store(fileName, pdfContent);
            
            // Update database with metadata
            StatementMetadata metadata = new StatementMetadata(
                statement.getStatementId(),
                statement.getAccountId(),
                fileLocation,
                pdfContent.length,
                LocalDateTime.now()
            );
            
            statementRepository.saveMetadata(metadata);
            
            return fileLocation;
            
        } catch (Exception e) {
            throw new StatementStorageException("Failed to store statement", e);
        }
    }
}

@Entity
@Table(name = "accounts")
public class Account {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long accountId;
    
    @Column(name = "customer_id")
    private Long customerId;
    
    @Enumerated(EnumType.STRING)
    @Column(name = "account_type")
    private AccountType accountType;
    
    @Column(name = "current_balance", precision = 15, scale = 2)
    private BigDecimal currentBalance;
    
    @Enumerated(EnumType.STRING)
    private AccountStatus status;
    
    @Column(name = "statement_cycle_day")
    private Integer statementCycleDay;
    
    @Type(type = "jsonb")
    @Column(name = "delivery_preferences", columnDefinition = "jsonb")
    private DeliveryPreferences deliveryPreferences;
    
    @CreationTimestamp
    @Column(name = "created_date")
    private LocalDateTime createdDate;
    
    @UpdateTimestamp
    @Column(name = "last_updated")
    private LocalDateTime lastUpdated;
    
    // Constructors, getters, setters
}

@Entity
@Table(name = "transactions")
public class Transaction {
    @Id
    private String transactionId;
    
    @Column(name = "account_id")
    private Long accountId;
    
    @Column(precision = 15, scale = 2)
    private BigDecimal amount;
    
    @Enumerated(EnumType.STRING)
    private TransactionType type;
    
    private String description;
    
    @Column(name = "transaction_date")
    private LocalDateTime transactionDate;
    
    @Column(name = "balance_after", precision = 15, scale = 2)
    private BigDecimal balanceAfter;
    
    @Column(name = "reference_number")
    private String referenceNumber;
    
    // Constructors, getters, setters
}

@Component
public class NotificationService {
    
    private final EmailService emailService;
    private final SMSService smsService;
    private final PushNotificationService pushService;
    private final QueueService queueService;
    
    @Async("notificationExecutor")
    public CompletableFuture<NotificationResult> sendNotification(NotificationRequest request) {
        NotificationResult result = new NotificationResult();
        
        for (NotificationChannel channel : request.getChannels()) {
            try {
                switch (channel.getType()) {
                    case EMAIL:
                        EmailResult emailResult = emailService.sendEmail(
                            channel.getAddress(), 
                            request.getContent().getEmailContent()
                        );
                        result.addChannelResult(channel, emailResult);
                        break;
                        
                    case SMS:
                        SMSResult smsResult = smsService.sendSMS(
                            channel.getAddress(), 
                            request.getContent().getSmsContent()
                        );
                        result.addChannelResult(channel, smsResult);
                        break;
                        
                    case PUSH:
                        PushResult pushResult = pushService.sendPush(
                            channel.getDeviceToken(), 
                            request.getContent().getPushContent()
                        );
                        result.addChannelResult(channel, pushResult);
                        break;
                }
            } catch (Exception e) {
                result.addChannelError(channel, e.getMessage());
                // Queue for retry
                queueService.queueForRetry(request, channel);
            }
        }
        
        return CompletableFuture.completedFuture(result);
    }
}
```

### 3. Database Selection Analysis

#### Recommended Database Architecture: **Polyglot Persistence**

**Primary Database: PostgreSQL**
- **Use Case**: Account metadata, transaction records, statement metadata
- **Reasoning**: 
  - ACID compliance for financial data integrity
  - Strong consistency for balance calculations
  - Advanced indexing for fast transaction queries
  - JSON support for flexible customer preferences
  - Excellent performance with proper partitioning

**Document Database: MongoDB**
- **Use Case**: Statement templates, configuration data, audit logs
- **Reasoning**:
  - Flexible schema for different statement formats
  - Horizontal scaling for large document storage
  - Built-in sharding capabilities
  - Rich querying for template management

**Cache Layer: Redis**
- **Use Case**: Session data, frequently accessed account info, rate limiting
- **Reasoning**:
  - Sub-millisecond response times
  - Built-in data structures (sets, sorted sets, hashes)
  - Pub/Sub for real-time notifications
  - Automatic expiration for temporary data

**File Storage: Amazon S3 / Azure Blob**
- **Use Case**: PDF statement storage, document archives
- **Reasoning**:
  - Unlimited scalability
  - Built-in redundancy and durability
  - Cost-effective for large file storage
  - Integration with CDN for fast delivery

#### Database Schema Design

```sql
-- PostgreSQL Schema
CREATE TABLE accounts (
    account_id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    account_type VARCHAR(50) NOT NULL,
    current_balance DECIMAL(15,2) NOT NULL,
    status VARCHAR(20) NOT NULL,
    statement_cycle_day INTEGER,
    delivery_preferences JSONB,
    created_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) PARTITION BY RANGE (account_id);

CREATE TABLE transactions (
    transaction_id VARCHAR(50) PRIMARY KEY,
    account_id BIGINT NOT NULL,
    amount DECIMAL(15,2) NOT NULL,
    transaction_type VARCHAR(50) NOT NULL,
    description TEXT,
    transaction_date TIMESTAMP NOT NULL,
    balance_after DECIMAL(15,2) NOT NULL,
    reference_number VARCHAR(100)
) PARTITION BY RANGE (transaction_date);

CREATE TABLE statement_metadata (
    statement_id VARCHAR(100) PRIMARY KEY,
    account_id BIGINT NOT NULL,
    statement_period_start DATE NOT NULL,
    statement_period_end DATE NOT NULL,
    file_location VARCHAR(500) NOT NULL,
    file_size BIGINT NOT NULL,
    generated_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status VARCHAR(20) DEFAULT 'GENERATED'
);

-- Indexes for performance
CREATE INDEX idx_accounts_customer_id ON accounts(customer_id);
CREATE INDEX idx_accounts_status ON accounts(status) WHERE status = 'ACTIVE';
CREATE INDEX idx_transactions_account_date ON transactions(account_id, transaction_date);
CREATE INDEX idx_statement_metadata_account ON statement_metadata(account_id, statement_period_start);
```

### 4. Scalability and Future Functionality

#### Scalability Strategies

**Horizontal Scaling**
```java
@Configuration
@EnableAsync
public class ScalingConfiguration {
    
    @Bean(name = "statementExecutor")
    public TaskExecutor statementTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(100);
        executor.setQueueCapacity(500);
        executor.setThreadNamePrefix("statement-");
        executor.initialize();
        return executor;
    }
    
    @Bean
    public CacheManager cacheManager() {
        RedisCacheManager.Builder builder = RedisCacheManager
            .RedisCacheManagerBuilder
            .fromConnectionFactory(redisConnectionFactory())
            .cacheDefaults(cacheConfiguration());
        return builder.build();
    }
}
```

**Database Scaling Approach**
- **Read Replicas**: 3-5 read replicas for transaction queries
- **Partitioning Strategy**: 
  - Accounts: Partition by account_id ranges (1-1M, 1M-2M, etc.)
  - Transactions: Partition by date (monthly/quarterly partitions)
  - Automatic partition creation via pg_partman
- **Connection Pooling**: HikariCP with 20-50 connections per instance
- **Query Optimization**: Prepared statements, batch operations

**Microservices Scaling**
```yaml
# Kubernetes Deployment Example
apiVersion: apps/v1
kind: Deployment
metadata:
  name: statement-service
spec:
  replicas: 5
  selector:
    matchLabels:
      app: statement-service
  template:
    metadata:
      labels:
        app: statement-service
    spec:
      containers:
      - name: statement-service
        image: statement-service:latest
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "production"
```

#### Future Functionality Enhancements

**1. AI-Powered Features**
```java
@Service
public class AIInsightsService {
    
    public List<SpendingInsight> generateSpendingInsights(List<Transaction> transactions) {
        // ML model integration for spending pattern analysis
        // Fraud detection based on transaction patterns
        // Personalized financial recommendations
    }
    
    public OptimalDeliveryTime predictBestDeliveryTime(Long customerId) {
        // ML model to predict when customer is most likely to engage
        // Based on historical open rates, click patterns
    }
}
```

**2. Real-time Statement Generation**
```java
@EventListener
public class RealTimeStatementGenerator {
    
    @Async
    public void handleTransactionEvent(TransactionCompletedEvent event) {
        // Generate mini-statements for large transactions
        // Real-time balance updates
        // Instant notifications for significant changes
    }
}
```

**3. Multi-tenant Architecture**
```java
@Component
public class TenantContextHolder {
    private static final ThreadLocal<String> tenantContext = new ThreadLocal<>();
    
    public static void setTenant(String tenantId) {
        tenantContext.set(tenantId);
    }
    
    public static String getTenant() {
        return tenantContext.get();
    }
}

@Configuration
public class MultiTenantDataSourceConfig {
    
    @Bean
    public DataSource dataSource() {
        TenantRoutingDataSource dataSource = new TenantRoutingDataSource();
        // Configure multiple database connections per tenant
        return dataSource;
    }
}
```

**4. Advanced Analytics Integration**
```java
@Service
public class StatementAnalyticsService {
    
    public void publishStatementMetrics(StatementJob job) {
        // Integration with analytics platforms (Kafka, BigQuery)
        // Customer engagement tracking
        // Performance metrics collection
        // Business intelligence data pipeline
    }
}
```

**5. Blockchain Integration for Compliance**
```java
@Component
public class BlockchainAuditTrail {
    
    public String recordStatementHash(Statement statement) {
        // Create immutable audit trail using blockchain
        // Regulatory compliance through distributed ledger
        // Tamper-proof statement verification
    }
}
```

**Performance Targets**
- **Throughput**: 100,000 statements/hour during peak
- **Latency**: 95% of statements processed within 30 seconds
- **Availability**: 99.9% uptime (8.77 hours downtime/year)
- **Error Rate**: < 0.01% statement generation failures
- **Scalability**: Handle 10x load increase with linear scaling

This comprehensive design ensures the system can handle current requirements while being flexible enough to incorporate future enhancements and scale with business growth.