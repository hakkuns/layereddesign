# Spring フレームワークにおける DI（依存性注入）と Bean 管理

## はじめに：Spring のコア技術としての DI

DI（依存性注入）は Spring フレームワークのコアと言える。DI とは、オブジェクト間の依存関係をフレームワークが自動的に管理する仕組み。

**従来の手動管理**：

```java
public class OrderController {
    public void processOrder() {
        Logger logger = new FileLogger();           // 手動でインスタンス作成
        Database db = new MySQLDatabase();          // 手動でインスタンス作成
        OrderService service = new OrderService(logger, db);  // 手動で依存関係解決

        service.createOrder("商品A", 1000);
    }
}
```

**Spring による自動管理**：

```java
@RestController
public class OrderController {
    private final OrderService orderService;

    // Spring が自動的に OrderService を注入
    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    @PostMapping("/orders")
    public void processOrder() {
        orderService.createOrder("商品A", 1000);
    }
}
```

開発者は「何が必要か」を宣言するだけで、Spring が依存関係の解決と注入を自動化してくれる。

---

## 1 Spring の DI コンテナと Bean

### DI コンテナの役割

**DI コンテナ**は、アプリケーション内のオブジェクト（Bean）を管理し、依存関係を自動的に解決する。

```java
// 密結合な設計（問題のあるコード）
public class UserService {
    private Logger logger = new FileLogger();        // 具象クラスに直接依存
    private UserRepository repository = new DBRepository();  // 具象クラスに直接依存

    public void createUser(String name, String email) {
        logger.log("Creating user: " + name);
        repository.save(name, email);
    }
}

// 疎結合な設計（Springの推奨方法）
@Service
public class UserService {
    private final Logger logger;
    private final UserRepository repository;

    // Spring が自動的に依存関係を注入
    public UserService(Logger logger, UserRepository repository) {
        this.logger = logger;
        this.repository = repository;
    }

    public void createUser(String name, String email) {
        logger.log("Creating user: " + name);
        repository.save(name, email);
    }
}
```

### Bean とは

**Bean**は、Spring の DI コンテナが管理するオブジェクト。通常の Java オブジェクトとの違いは、Spring によってライフサイクル管理と依存関係注入が行われること。

```java
// これは単なるJavaオブジェクト
Logger logger = new FileLogger();

// これはSpringによって管理されるBean
@Component
public class FileLogger implements Logger {
    @Override
    public void log(String message) {
        System.out.println("[FILE] " + message);
    }
}
```

## 2 ステレオタイプアノテーション

Spring では、役割に応じたアノテーションで Bean を定義する。

### 基本的なステレオタイプアノテーション

```java
// 1. @Component - 汎用的なコンポーネント
@Component
public class FileLogger implements Logger {
    @Override
    public void log(String message) {
        System.out.println("[FILE] " + message);
    }
}

// 2. @Service - ビジネスロジック層
@Service
public class OrderService {
    private final Logger logger;
    private final OrderRepository repository;

    public OrderService(Logger logger, OrderRepository repository) {
        this.logger = logger;
        this.repository = repository;
    }

    public void createOrder(String product, int price) {
        logger.log("注文作成開始: " + product);
        repository.save(new Order(product, price));
        logger.log("注文作成完了: " + product);
    }
}

// 3. @Repository - データアクセス層
@Repository
public class OrderRepository {
    public void save(Order order) {
        System.out.println("DB保存: " + order);
    }
}

// 4. @Controller - プレゼンテーション層
@RestController
public class OrderController {
    private final OrderService orderService;

    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    @PostMapping("/orders")
    public String createOrder(@RequestParam String product, @RequestParam int price) {
        orderService.createOrder(product, price);
        return "注文が作成されました";
    }
}
```

### なぜ使い分けるのか

```java
// アノテーションの内部実装（例：@Service）
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component  // ← すべて@Componentを内包している
public @interface Service {
    String value() default "";
}
```

**役割を明確にする**ことで：

- コードの意図が理解しやすくなる
- 将来的な拡張（AOP 適用など）が容易になる
- アーキテクチャの層分離が明確になる

## 3 依存関係注入の方法

### 1. コンストラクタインジェクション（推奨）

```java
@Service
public class UserService {
    private final Logger logger;          // final宣言可能
    private final UserRepository repository;

    // Spring Boot では @Autowired 不要
    public UserService(Logger logger, UserRepository repository) {
        this.logger = logger;
        this.repository = repository;
    }

    public void createUser(String name, String email) {
        logger.log("ユーザー作成開始: " + name);
        repository.save(new User(name, email));
        logger.log("ユーザー作成完了: " + name);
    }
}
```

**なぜコンストラクタインジェクションが推奨されるのか：**

- **不変性**: `final`宣言により、依存関係が変更されない
- **必須依存関係の明示**: コンストラクタの引数で必要な依存関係が明確
- **テストの容易さ**: モックオブジェクトを簡単に注入できる
- **早期エラー検出**: 依存関係の問題をアプリケーション起動時に発見

### 2. セッターインジェクション

```java
@Service
public class EmailService {
    private Logger logger;
    private EmailConfig config;

    @Autowired
    public void setLogger(Logger logger) {
        this.logger = logger;
    }

    @Autowired(required = false)  // オプショナルな依存関係
    public void setEmailConfig(EmailConfig config) {
        this.config = config != null ? config : new DefaultEmailConfig();
    }
}
```

**使用場面**：

- オプショナルな依存関係
- 循環依存の回避（ただし設計見直しが推奨）

### 3. フィールドインジェクション（非推奨）

```java
@Service
public class UserService {
    @Autowired
    private Logger logger;           // final宣言不可

    @Autowired
    private UserRepository repository;

    // 問題：テストでモックを注入するのが困難
    // 問題：依存関係が外部から見えない
}
```

## 3.4 Bean のスコープ

### Singleton（デフォルト）

```java
@Service  // デフォルトでsingleton
public class UserService {
    // アプリケーション全体で1つのインスタンス
}

// 確認方法
@Component
public class ServiceChecker {
    private final UserService service1;
    private final UserService service2;

    public ServiceChecker(UserService service1, UserService service2) {
        this.service1 = service1;
        this.service2 = service2;

        System.out.println(service1 == service2);  // true（同じインスタンス）
    }
}
```

### Prototype

```java
@Service
@Scope("prototype")
public class TaskProcessor {
    private String taskId;

    public TaskProcessor() {
        this.taskId = UUID.randomUUID().toString();
        System.out.println("新しいTaskProcessor作成: " + taskId);
    }

    // getBean()やインジェクション毎に新しいインスタンス
}
```

### 重要な注意点：状態を持つ Bean は避ける

```java
// 問題のあるコード例
@Service  // singletonがデフォルト
public class CounterService {
    private int count = 0;  // ← 危険！状態を持っている

    public void increment() {
        count++;  // 複数のリクエストで共有される
    }

    public int getCount() {
        return count;
    }
}

// 正しい実装
@Service
public class CounterService {
    // 状態を持たず、処理のみを提供
    public int increment(int currentValue) {
        return currentValue + 1;
    }
}
```

## 5 複数実装の処理

実際の開発では、同じインターフェースに複数の実装がある場合がある。

### すべての実装を取得

```java
public interface NotificationService {
    void notify(String message);
}

@Service
public class EmailNotificationService implements NotificationService {
    @Override
    public void notify(String message) {
        System.out.println("Email: " + message);
    }
}

@Service
public class SmsNotificationService implements NotificationService {
    @Override
    public void notify(String message) {
        System.out.println("SMS: " + message);
    }
}

@Service
public class NotificationManager {
    private final List<NotificationService> services;

    // すべての実装が List として注入される
    public NotificationManager(List<NotificationService> services) {
        this.services = services;
    }

    public void notifyAll(String message) {
        services.forEach(service -> service.notify(message));
    }
}
```

### 特定の実装を選択

```java
// 1. @Primary で優先実装を指定
@Service
@Primary
public class EmailNotificationService implements NotificationService {
    // デフォルトで選択される実装
}

// 2. @Qualifier で特定の実装を指定
@Service
public class OrderService {
    private final NotificationService emailService;
    private final NotificationService smsService;

    public OrderService(
        @Qualifier("emailNotificationService") NotificationService emailService,
        @Qualifier("smsNotificationService") NotificationService smsService) {
        this.emailService = emailService;
        this.smsService = smsService;
    }
}
```

## 6 条件付き Bean 定義

環境や設定に応じて Bean を切り替えることができます。

### プロファイルによる切り替え

```java
@Service
@Profile("development")
public class MockEmailService implements EmailService {
    @Override
    public void sendEmail(String to, String subject, String body) {
        System.out.println("MOCK: Email sent to " + to);
    }
}

@Service
@Profile("production")
public class RealEmailService implements EmailService {
    @Override
    public void sendEmail(String to, String subject, String body) {
        // 実際のメール送信処理
        System.out.println("REAL: Email sent to " + to);
    }
}

// application.properties で切り替え
// spring.profiles.active=development
```

### 条件付き Bean 作成

```java
@Service
@ConditionalOnProperty(name = "feature.notification.enabled", havingValue = "true")
public class NotificationService {
    // feature.notification.enabled=true の場合のみ作成
}

@Service
@ConditionalOnMissingBean(NotificationService.class)
public class DisabledNotificationService implements NotificationService {
    // NotificationService が存在しない場合のフォールバック
    @Override
    public void notify(String message) {
        System.out.println("通知機能は無効です");
    }
}
```

## 7 テストでの DI 活用

Spring の DI 機能により、テストが大幅に簡単になる。

### モックを使用したテスト

```java
@ExtendWith(SpringExtension.class)
@ContextConfiguration(classes = TestConfig.class)
class UserServiceTest {

    @Autowired
    private UserService userService;

    @MockBean
    private UserRepository mockRepository;

    @MockBean
    private Logger mockLogger;

    @Test
    void testCreateUser() {
        // Given
        User expectedUser = new User("テストユーザー", "test@example.com");
        when(mockRepository.save(any(User.class))).thenReturn(expectedUser);

        // When
        userService.createUser("テストユーザー", "test@example.com");

        // Then
        verify(mockLogger).log("ユーザー作成開始: テストユーザー");
        verify(mockRepository).save(any(User.class));
        verify(mockLogger).log("ユーザー作成完了: テストユーザー");
    }
}

@TestConfiguration
static class TestConfig {
    // テスト用の設定
}
```

### DI がテストを簡単にする理由

```java
// 密結合な設計（テストが困難）
public class UserService {
    private Logger logger = new FileLogger();        // 固定された依存関係
    private UserRepository repository = new DBRepository();  // 固定された依存関係

    // テストのためにFileLoggerとDBRepositoryが必要
}

// SpringのDI（テストが簡単）
@Service
public class UserService {
    private final Logger logger;
    private final UserRepository repository;

    public UserService(Logger logger, UserRepository repository) {
        this.logger = logger;
        this.repository = repository;
    }

    // テストではモックを注入できる
}
```

## 8 実践的な層分離設計

Spring の DI を活用した典型的な 3 層アーキテクチャの例：

```java
// プレゼンテーション層
@RestController
@RequestMapping("/api/users")
public class UserController {
    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @PostMapping
    public ResponseEntity<User> createUser(@RequestBody CreateUserRequest request) {
        User user = userService.createUser(request.getName(), request.getEmail());
        return ResponseEntity.ok(user);
    }
}

// ビジネスロジック層
@Service
@Transactional
public class UserService {
    private final UserRepository userRepository;
    private final EmailService emailService;
    private final Logger logger;

    public UserService(UserRepository userRepository, EmailService emailService, Logger logger) {
        this.userRepository = userRepository;
        this.emailService = emailService;
        this.logger = logger;
    }

    public User createUser(String name, String email) {
        logger.log("ユーザー作成開始: " + name);

        // ビジネスルールの検証
        if (userRepository.existsByEmail(email)) {
            throw new UserAlreadyExistsException("メールアドレスが既に使用されています: " + email);
        }

        // ユーザー作成
        User user = new User(name, email);
        User savedUser = userRepository.save(user);

        // 通知送信
        emailService.sendWelcomeEmail(savedUser);

        logger.log("ユーザー作成完了: " + name);
        return savedUser;
    }
}

// データアクセス層
@Repository
public class UserRepository {
    private final JdbcTemplate jdbcTemplate;

    public UserRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    public User save(User user) {
        String sql = "INSERT INTO users (name, email) VALUES (?, ?)";
        KeyHolder keyHolder = new GeneratedKeyHolder();

        jdbcTemplate.update(connection -> {
            PreparedStatement ps = connection.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS);
            ps.setString(1, user.getName());
            ps.setString(2, user.getEmail());
            return ps;
        }, keyHolder);

        user.setId(keyHolder.getKey().longValue());
        return user;
    }

    public boolean existsByEmail(String email) {
        String sql = "SELECT COUNT(*) FROM users WHERE email = ?";
        Integer count = jdbcTemplate.queryForObject(sql, Integer.class, email);
        return count != null && count > 0;
    }
}
```

## まとめ

### 重要なポイント

1. **Spring の DI = 疎結合設計の自動化**

   - 手動での依存関係管理をフレームワークが自動化
   - 開発者は「何が必要か」を宣言するだけ

2. **ステレオタイプアノテーションによる役割の明確化**

   - `@Service`: ビジネスロジック層
   - `@Repository`: データアクセス層
   - `@Controller`: プレゼンテーション層
   - `@Component`: 汎用コンポーネント

3. **コンストラクタインジェクションの重要性**

   - 不変性（`final`宣言）
   - テストの容易さ
   - 依存関係の明確化

4. **Bean の適切な設計**
   - 状態を持たない設計
   - 単一責任原則の遵守

### Spring の DI がもたらす価値

- **開発効率の向上**: 依存関係管理の自動化
- **テスタビリティの向上**: モック注入の容易さ
- **保守性の向上**: 疎結合な設計
- **拡張性の向上**: 実装の切り替えが容易

Spring の DI 機能を理解することで、疎結合で保守性の高いアプリケーションを構築できる。
