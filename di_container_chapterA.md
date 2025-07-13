# 付録 A：DI コンテナの仕組みと実装

## はじめに

第 1 章では疎結合の重要性と依存性注入（DI）の基本概念を学んだ。この章では、依存関係の管理を自動化する**DI コンテナ**の仕組みを理解し、TypeScript でシンプルな DI コンテナを実装する。

第 3 章の手動での依存性注入：

```typescript
// 手動で依存関係を注入
const userService = new UserService(
  new FileLogger(), // 毎回手動で作成
  new RealDatabase() // 毎回手動で作成
);
```

この「手動での依存関係管理」を自動化するのが DI コンテナです。そして、この自動化機能を提供するのが Spring のようなフレームワークなのです。

---

## A.1 DI コンテナとは何か

### DI コンテナの役割

**DI コンテナ**は、以下の責任を持つオブジェクト管理システムです：

1. **オブジェクトの作成**: 必要なインスタンスを生成
2. **依存関係の解決**: どのオブジェクトが何を必要としているかを把握
3. **依存関係の注入**: 適切なオブジェクトを適切な場所に渡す
4. **ライフサイクル管理**: オブジェクトの生成から破棄までを管理

### 手動管理の問題：new する順番と組み立ての複雑さ

```typescript
// 手動管理の問題例
function createUserService(): UserService {
  // 依存関係がある場合、newする順番が重要
  const logger = new FileLogger(); // まずLoggerを作成
  const database = new RealDatabase(); // 次にDatabaseを作成
  const repository = new UserRepository(database); // DatabaseがないとUserRepositoryは作れない
  const emailService = new EmailService(logger); // LoggerがないとEmailServiceは作れない

  // 最後にUserServiceを作成（すべての依存関係が必要）
  return new UserService(logger, repository, emailService);
}

// 複雑な依存関係の場合
function createOrderService(): OrderService {
  // さらに複雑な順番管理が必要
  const logger = new FileLogger();
  const database = new RealDatabase();
  const userRepository = new UserRepository(database);
  const emailService = new EmailService(logger);
  const userService = new UserService(logger, userRepository, emailService);
  const inventoryService = new InventoryService(database, logger);
  const paymentService = new PaymentService(logger);

  // OrderServiceの作成には多くの依存関係が必要
  return new OrderService(
    userService,
    inventoryService,
    paymentService,
    logger
  );
}

// 毎回この複雑な組み立て処理が必要
const userService = createUserService();
const orderService = createOrderService();
```

```typescript
// DIコンテナによる自動管理（これから実装）
const container = new DIContainer();

// 依存関係を登録
container.register('Logger', FileLogger);
container.register('Database', RealDatabase);
container.register('UserRepository', UserRepository);
container.register('EmailService', EmailService);
container.register('UserService', UserService);

// 自動的に依存関係が解決される
const userService = container.resolve<UserService>('UserService');
```

## A.2 シンプルな DI コンテナの実装

### 基本的な DI コンテナ

```typescript
// 基本的なDIコンテナ
class SimpleDIContainer {
  private services = new Map<string, any>();
  private instances = new Map<string, any>();

  // サービスを登録
  register<T>(name: string, definition: new (...args: any[]) => T): void {
    this.services.set(name, definition);
  }

  // サービスを解決（取得）
  resolve<T>(name: string): T {
    // 既にインスタンスがある場合は再利用（Singleton）
    if (this.instances.has(name)) {
      return this.instances.get(name);
    }

    const serviceDefinition = this.services.get(name);
    if (!serviceDefinition) {
      throw new Error(`Service '${name}' not found`);
    }

    // インスタンスを作成
    const instance = new serviceDefinition();
    this.instances.set(name, instance);

    return instance;
  }
}

// 使用例
interface Logger {
  log(message: string): void;
}

class ConsoleLogger implements Logger {
  log(message: string): void {
    console.log(`[LOG] ${message}`);
  }
}

class UserService {
  constructor(private logger: Logger) {}

  createUser(name: string): void {
    this.logger.log(`Creating user: ${name}`);
  }
}

// DIコンテナの使用
const container = new SimpleDIContainer();
container.register('Logger', ConsoleLogger);
container.register('UserService', UserService);

const userService = container.resolve<UserService>('UserService');
```

### 問題：依存関係の自動解決ができない

上記の実装では、`UserService`のコンストラクタに`Logger`を自動的に渡すことができません。依存関係を手動で解決する必要があります。

## A.3 依存関係自動解決機能の実装

### メタデータベースの DI コンテナ

```typescript
// 依存関係のメタデータを管理するDIコンテナ
class DIContainer {
  private services = new Map<string, ServiceDefinition>();
  private instances = new Map<string, any>();

  // サービス定義を登録
  register<T>(
    name: string,
    constructor: new (...args: any[]) => T,
    dependencies: string[] = []
  ): void {
    this.services.set(name, {
      constructor,
      dependencies,
      singleton: true,
    });
  }

  // 依存関係を自動解決してサービスを取得
  resolve<T>(name: string): T {
    // Singletonの場合、既存インスタンスを返す
    if (this.instances.has(name)) {
      return this.instances.get(name);
    }

    const serviceDefinition = this.services.get(name);
    if (!serviceDefinition) {
      throw new Error(`Service '${name}' not found`);
    }

    // 依存関係を再帰的に解決
    const dependencies = serviceDefinition.dependencies.map((dep) =>
      this.resolve(dep)
    );

    // インスタンスを作成
    const instance = new serviceDefinition.constructor(...dependencies);

    if (serviceDefinition.singleton) {
      this.instances.set(name, instance);
    }

    return instance;
  }
}

interface ServiceDefinition {
  constructor: new (...args: any[]) => any;
  dependencies: string[];
  singleton: boolean;
}
```

### 使用例：依存関係の自動解決

```typescript
// サービスクラスの定義
interface Logger {
  log(message: string): void;
}

interface Database {
  save(data: any): void;
}

interface EmailService {
  sendEmail(to: string, subject: string): void;
}

class FileLogger implements Logger {
  log(message: string): void {
    console.log(`[FILE] ${message}`);
  }
}

class PostgreSQLDatabase implements Database {
  save(data: any): void {
    console.log(`Saving to PostgreSQL: ${JSON.stringify(data)}`);
  }
}

class SMTPEmailService implements EmailService {
  private logger: Logger;

  constructor(logger: Logger) {
    this.logger = logger;
  }

  sendEmail(to: string, subject: string): void {
    this.logger.log(`Sending email to ${to}: ${subject}`);
    console.log(`Email sent via SMTP`);
  }
}

class UserService {
  constructor(
    private logger: Logger,
    private database: Database,
    private emailService: EmailService
  ) {}

  createUser(name: string, email: string): void {
    this.logger.log(`Creating user: ${name}`);

    const userData = { name, email, createdAt: new Date() };
    this.database.save(userData);

    this.emailService.sendEmail(email, 'Welcome!');

    this.logger.log(`User created successfully: ${name}`);
  }
}

// DIコンテナの設定と使用
const container = new DIContainer();

// 依存関係を明示的に登録
container.register('Logger', FileLogger, []);
container.register('Database', PostgreSQLDatabase, []);
container.register('EmailService', SMTPEmailService, ['Logger']);
container.register('UserService', UserService, [
  'Logger',
  'Database',
  'EmailService',
]);

// 使用 - 依存関係が自動的に解決される
const userService = container.resolve<UserService>('UserService');
userService.createUser('田中太郎', 'tanaka@example.com');

// 出力:
// [FILE] Creating user: 田中太郎
// Saving to PostgreSQL: {"name":"田中太郎","email":"tanaka@example.com","createdAt":"..."}
// [FILE] Sending email to tanaka@example.com: Welcome!
// Email sent via SMTP
// [FILE] User created successfully: 田中太郎
```

## A.4 フレームワークが提供する価値

### 手動 DI コンテナの限界

私たちが実装した DI コンテナでも基本的な依存関係管理はできますが、実際のプロダクション環境では以下の課題があります：

```typescript
// 手動での依存関係登録が必要
container.register('Logger', FileLogger, []);
container.register('Database', PostgreSQLDatabase, []);
container.register('EmailService', SMTPEmailService, ['Logger']);
container.register('UserService', UserService, [
  'Logger',
  'Database',
  'EmailService',
]);

// 問題点:
// 1. 依存関係を手動で書く必要がある
// 2. タイプセーフでない
// 3. 循環依存の検出が困難
// 4. 設定管理が複雑
```

### フレームワークによる自動化

**Spring、NestJS、Angular**などのフレームワークは、私たちが実装したような基本機能に加えて、以下の自動化を提供します：

#### アノテーション/デコレータベースの自動発見

```typescript
// NestJSの例 - 依存関係が自動的に解決される
@Injectable()
export class UserService {
  constructor(
    private readonly logger: Logger,      // 自動的に注入される
    private readonly database: Database,  // 自動的に注入される
    private readonly emailService: EmailService // 自動的に注入される
  ) {}
}

// Springの例（Java）
@Service
public class UserService {
    public UserService(Logger logger, Database database, EmailService emailService) {
        // Spring が自動的に依存関係を注入
    }
}
```

#### 型安全性

```typescript
// TypeScriptフレームワークでの型安全な注入
@Injectable()
export class UserService {
  constructor(
    private database: Database, // 型から自動推論
    private logger: Logger // 型チェック
  ) {}
}
```

### 手動 DI コンテナでの開発

```typescript
// 依存関係の手動登録
const container = new DIContainer();
container.register('Logger', FileLogger, []);
container.register('Database', PostgreSQLDatabase, []);
container.register('EmailService', SMTPEmailService, ['Logger']);
container.register('UserService', UserService, [
  'Logger',
  'Database',
  'EmailService',
]);
container.register('UserController', UserController, ['UserService']);

// 手動での設定管理
const config = {
  database: { host: 'localhost', port: 5432 },
  email: { smtp: 'smtp.gmail.com' },
};

// 手動でのアプリケーション起動
const userController = container.resolve<UserController>('UserController');
```

### フレームワークでの開発（NestJS 例）

```typescript
// アノテーションで自動登録
@Injectable()
export class UserService {
  constructor(
    private logger: Logger, // 自動注入
    private database: Database, // 自動注入
    private emailService: EmailService // 自動注入
  ) {}
}

@Controller('users')
export class UserController {
  constructor(private userService: UserService) {} // 自動注入
}

@Module({
  controllers: [UserController],
  providers: [UserService, Logger, Database, EmailService], // 自動発見
})
export class AppModule {}

// フレームワークが自動起動
const app = await NestFactory.create(AppModule);
await app.listen(3000);
```

## まとめ

この章では、DI コンテナの仕組みを理解し、TypeScript で実装することで以下を学びました：

### DI コンテナの本質

1. **オブジェクト管理の自動化**: 手動でのインスタンス作成から解放
2. **依存関係の自動解決**: 複雑な依存関係を自動的に解決
3. **ライフサイクル管理**: Singleton、Transient、Scoped の管理

### 手動実装での学び

```typescript
// 私たちが実装したDIコンテナの価値
const container = new DIContainer();
container.register('UserService', UserService, ['Logger', 'Database']);

// 複雑な依存関係も自動的に解決
const userService = container.resolve<UserService>('UserService');
```

### フレームワークの価値

手動実装を通じて、**フレームワークがいかに多くの複雑さを隠蔽してくれているか**が理解できました：

- **自動発見**: アノテーションによる Bean 自動登録
- **型安全性**: TypeScript の型システムとの統合
- **高度な機能**: AOP、設定管理、プロファイル管理
- **開発体験**: IDE 支援、デバッグ機能、ホットリロード

**フレームワークとは、このような依存関係管理を高度に自動化し、開発者がビジネスロジックに集中できるようにする仕組み**なのです。

次の章では、実際の Spring フレームワークがどのようにこれらの概念を実装しているかを詳しく見ていきます。

---

> **重要なポイント**  
> DI コンテナは「魔法」ではありません。私たちが実装したように、オブジェクトの作成と依存関係の解決を自動化する仕組みです。フレームワークは、この基本概念をより洗練された形で提供し、開発者の生産性を大幅に向上させているのです。
