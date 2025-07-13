# 第章：結合度とは何か - 疎結合の重要性

## はじめに：なぜ疎結合が重要なのか

### 現代開発に結合の価値

現代のソフトウェア開発では、**疎結合**な設計を重要視。疎結合とは、コンポーネント間の依存関係を最小限に抑える設計原則で、保守性、テスタビリティ、拡張性の向上に直結する。

#### 疎結合が求められる背景

1. **頻繁な変更要求**：ビジネス要件の変化に素早く対応する必要
2. **複雑化するシステム**：多くのコンポーネントが相互作用する大規模システム
3. **テスト自動化**：CI/CD パイプラインでの自動テストが必須
4. **チーム開発**：複数の開発者が並行して作業する環境
5. **環境の多様化**：開発、ステージング、本番での動作環境の違い

#### この資料で学ぶこと

この資料では、疎結合を実現する具体的な技術として**依存性注入（Dependency Injection）**を学ぶ：

- **疎結合の基本概念**：なぜ疎結合が重要なのか
- **DI の原理と実装**：依存関係をどう管理するか
- **実践的なパターン**：TypeScript から Java/Spring Boot まで
- **実際のプロジェクトでの活用**：テスト戦略や設計指針

---

## 3.1 結合度の基本概念

### 密結合 vs 疎結合

**結合度（Coupling）**とは、モジュールやクラス、関数間の依存関係の強さを表す概念である。

#### 密結合（Tight Coupling）の例

```typescript
// 密結合な設計例：具象クラスへの直接依存
class FileLogger {
  log(message: string): void {
    console.log(`[FILE] ${message}`);
  }
}

class Database {
  save(name: string, email: string): void {
    console.log(`Saving to database: ${name}, ${email}`);
  }
}

class UserService {
  private logger = new FileLogger(); // ← 密結合！具象クラスに直接依存
  private db = new Database(); // ← 密結合！具象クラスに直接依存

  createUser(name: string, email: string): void {
    this.logger.log(`User creating: ${name}`);
    this.db.save(name, email);
    this.logger.log(`User created: ${name}`);
  }
}

// 使用例
const userService = new UserService();
userService.createUser('田中太郎', 'tanaka@example.com');
```

#### この設計の問題点

1. **テストが困難**：実際のファイルや DB が必要
2. **変更の波及**：ログ出力先を変更するには`UserService`のソースコードを修正
3. **再利用性の低さ**：異なる環境で使い回せない
4. **並行開発が困難**：依存関係の実装完了を待つ必要

#### 疎結合（Loose Coupling）への改善

```typescript
// 疎結合な設計例：抽象への依存
interface Logger {
  log(message: string): void;
}

interface Database {
  save(name: string, email: string): void;
}

// 具象実装
class FileLogger implements Logger {
  log(message: string): void {
    console.log(`[FILE] ${message}`);
  }
}

class ConsoleLogger implements Logger {
  log(message: string): void {
    console.log(`[CONSOLE] ${message}`);
  }
}

class RealDatabase implements Database {
  save(name: string, email: string): void {
    console.log(`Saving to database: ${name}, ${email}`);
  }
}

class MockDatabase implements Database {
  save(name: string, email: string): void {
    console.log(`Mock save: ${name}, ${email}`);
  }
}

class UserService {
  constructor(
    private logger: Logger, // 抽象に依存
    private database: Database // 抽象に依存
  ) {}

  createUser(name: string, email: string): void {
    this.logger.log(`User creating: ${name}`);
    this.database.save(name, email);
    this.logger.log(`User created: ${name}`);
  }
}

// 使用例：依存関係を外部から注入（これがDI！）
const userService = new UserService(
  new FileLogger(), // 依存関係を外部から注入
  new RealDatabase() // 依存関係を外部から注入
);

const testUserService = new UserService(
  new ConsoleLogger(), // テスト用の実装を注入
  new MockDatabase() // テスト用の実装を注入
);
```

### ここで実現されている重要な概念

#### 1. 依存関係逆転の原則（DIP）

**核心原則：「具体ではなく抽象に依存せよ」**

**従来の依存関係（密結合）**：

```typescript
// 悪い例：具体クラスに依存
class UserService {
  private logger = new FileLogger(); // ← 具体クラスに依存（問題）
  private db = new RealDatabase(); // ← 具体クラスに依存（問題）
}
```

**逆転した依存関係（疎結合）**：

```typescript
// 良い例：抽象に依存
class UserService {
  constructor(
    private logger: Logger, // ← 抽象（インターフェース）に依存
    private database: Database // ← 抽象（インターフェース）に依存
  ) {}
}
```

**依存関係の方向**：

```
【従来の依存関係】
UserService ──depends on──> FileLogger (具象クラス)
UserService ──depends on──> RealDatabase (具象クラス)

【逆転した依存関係】
UserService ──depends on──> Logger (抽象)
                              ↑
                         FileLogger (具象クラスが抽象に依存)

UserService ──depends on──> Database (抽象)
                              ↑
                         RealDatabase (具象クラスが抽象に依存)
```

**なぜ「逆転」なのか**：

- **高レベルモジュール**（`UserService`）が**低レベルモジュール**（`FileLogger`）に依存するという従来の依存関係を逆転
- **両方とも抽象**（`Logger`、`Database`インターフェース）に依存するように変更
- 具象クラス（`FileLogger`）がインターフェース（`Logger`）に依存する構造に変わる

#### 2. 依存性注入（Dependency Injection - DI）

```typescript
// DIパターン：依存関係を外部から注入
const userService = new UserService(
  new FileLogger(), // ← 外部で作成した依存関係を注入
  new RealDatabase() // ← 外部で作成した依存関係を注入
);
```

- `UserService`は自分で依存関係を作成しない（`new FileLogger()`しない）
- 外部から必要な依存関係を「注入」してもらう
- これが「依存性注入（DI）」である

### 結合度がコードに与える影響

| 観点               | 密結合               | 疎結合                               |
| ------------------ | -------------------- | ------------------------------------ |
| **テスタビリティ** | 実際の依存関係が必要 | モックで簡単にテスト                 |
| **再利用性**       | 特定の実装に縛られる | 異なる実装と組み合わせ可能           |
| **保守性**         | 変更の影響範囲が広い | 変更の影響を局所化できる             |
| **並行開発**       | 依存関係の完成待ち   | インターフェースがあれば並行作業可能 |

## 3.2 疎結合がもたらすメリット

### テスタビリティの向上

```typescript
// 疎結合な設計はテストが簡単
describe('UserService', () => {
  it('should create user successfully', () => {
    // モック実装を簡単に作成
    const mockLogger = jest.fn();
    const mockDb = { save: jest.fn() };

    const userService = new UserService(mockLogger, mockDb);
    userService.createUser('Test User', 'test@example.com');

    // 呼び出しを検証
    expect(mockLogger).toHaveBeenCalledWith('User creating: Test User');
    expect(mockDb.save).toHaveBeenCalledWith('Test User', 'test@example.com');
  });
});
```

### 再利用性の向上

```typescript
// 同じUserServiceを異なる環境で再利用

// 本番環境
const productionService = new UserService(
  new FileLogger(), // ファイルにログ出力
  new RealDatabase() // 実際のDB接続
);

// 開発環境
const developmentService = new UserService(
  new ConsoleLogger(), // コンソールにログ出力
  new MockDatabase() // モックDB
);

// テスト環境
const testService = new UserService(
  new ConsoleLogger(), // コンソールにログ出力
  new MockDatabase() // モックDB
);
```

### 保守性・拡張性の向上

```typescript
// 新しい要件：複数のログ出力先に対応
class MultiLogger implements Logger {
  constructor(private loggers: Logger[]) {}

  log(message: string): void {
    this.loggers.forEach((logger) => logger.log(message));
  }
}

// 既存のUserServiceを変更せずに新機能を追加
const enhancedService = new UserService(
  new MultiLogger([new FileLogger(), new ConsoleLogger()]),
  new RealDatabase()
);
```

## 3.3 関数型プログラミングでの疎結合

関数型プログラミングでも、高階関数を使って疎結合な設計を実現できる。

### 高階関数による依存注入

```typescript
// 関数型アプローチでの疎結合
type Logger = (message: string) => void;
type Saver = (name: string, email: string) => void;

function createUserService(logger: Logger, saver: Saver) {
  return {
    createUser: (name: string, email: string) => {
      logger(`User creating: ${name}`);
      saver(name, email);
      logger(`User created: ${name}`);
    },
  };
}

// 使用例
const fileLogger: Logger = (msg) => console.log(`[FILE] ${msg}`);
const dbSaver: Saver = (name, email) => console.log(`DB: ${name}, ${email}`);

const userService = createUserService(fileLogger, dbSaver);
userService.createUser('田中太郎', 'tanaka@example.com');

// テスト用の実装も簡単
const testLogger: Logger = (msg) => console.log(`[TEST] ${msg}`);
const mockSaver: Saver = (name, email) =>
  console.log(`MOCK: ${name}, ${email}`);

const testUserService = createUserService(testLogger, mockSaver);
```

この関数型アプローチも、実質的には**依存性注入（DI）**と同じ効果を得られている。

## 3.4 TypeScript での疎結合実装

### インターフェースによる抽象化

```typescript
// より複雑な例：バリデーションも含む
interface UserValidator {
  isValid(name: string, email: string): boolean;
}

interface UserNotifier {
  notify(message: string): void;
}

class UserService {
  constructor(
    private validator: UserValidator,
    private notifier: UserNotifier,
    private logger: Logger,
    private database: Database
  ) {}

  createUser(name: string, email: string): void {
    if (!this.validator.isValid(name, email)) {
      this.logger.log('Invalid user data');
      throw new Error('Invalid user data');
    }

    this.logger.log(`User creating: ${name}`);
    this.database.save(name, email);
    this.notifier.notify(`Welcome ${name}!`);
    this.logger.log(`User created: ${name}`);
  }
}

// 具象実装
class SimpleValidator implements UserValidator {
  isValid(name: string, email: string): boolean {
    return name.length >= 2 && email.includes('@');
  }
}

class EmailNotifier implements UserNotifier {
  notify(message: string): void {
    console.log(`Email sent: ${message}`);
  }
}

// 使用例
const userService = new UserService(
  new SimpleValidator(),
  new EmailNotifier(),
  new FileLogger(),
  new RealDatabase()
);
```

### 関数型アプローチとの組み合わせ

```typescript
// 関数型とOOPのハイブリッドアプローチ
type UserDependencies = {
  validator: UserValidator;
  notifier: UserNotifier;
  logger: Logger;
  database: Database;
};

function createUserService(deps: UserDependencies) {
  return {
    createUser: (name: string, email: string) => {
      if (!deps.validator.isValid(name, email)) {
        deps.logger.log('Invalid user data');
        throw new Error('Invalid user data');
      }

      deps.logger.log(`User creating: ${name}`);
      deps.database.save(name, email);
      deps.notifier.notify(`Welcome ${name}!`);
      deps.logger.log(`User created: ${name}`);
    },
  };
}

// 使用例
const dependencies: UserDependencies = {
  validator: new SimpleValidator(),
  notifier: new EmailNotifier(),
  logger: new FileLogger(),
  database: new RealDatabase(),
};

const userService = createUserService(dependencies);
```

## まとめ

この章では、疎結合の重要性と依存性注入の基本概念について学んだ：

1. **疎結合の価値**：テスタビリティ、再利用性、保守性の向上
2. **密結合の問題**：変更の波及、テストの困難さ、再利用の制限
3. **DIP と DI の基礎**：「具体ではなく抽象に依存せよ」という原則
4. **実装アプローチ**：オブジェクト指向でも関数型でも実現可能

依存性注入は、パラダイムに関係なく適用できる強力な設計原則である。疎結合な設計により、変化に強く、テストしやすく、保守しやすいコードを書くことができる。

次の章では、関数型プログラミングでの依存関係管理パターンをより詳しく見て、その特徴と利点を理解する。

---

> **次の章に向けて**  
> 疎結合の価値と DI の基本概念を理解したところで、次は関数型プログラミングでの具体的な依存関係管理パターンを学び、その後オブジェクト指向のアプローチと比較していく。
