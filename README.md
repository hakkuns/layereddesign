# レイヤードアーキテクチャ設計ガイド

## 1. レイヤリングの意味と価値

システムをレイヤリングする主な意味は以下の通りです：

- **関心事の分離**: 各層が異なる責任を持つことで、コードの役割が明確になる
- **保守性の向上**: 特定の機能を修正する際、該当する層だけを変更すれば済む
- **テストのしやすさ**: 各層を独立してテストできるため、効率的なテストが可能
- **再利用性**: 下位層の機能を上位層の複数箇所から利用できる
- **チーム開発の効率化**: 各層を異なる開発者が担当でき、並行開発が可能
- **技術的な独立性**: 各層で異なる技術スタックを選択できる

## 2. アーキテクチャ原則

### 2.1 単一責任の原則（Single Responsibility Principle）
各層は明確に定義された単一の責任のみを持つ。混在した責任を避け、変更理由を一つに限定する。

### 2.2 依存性の方向制御（Dependency Direction Control）
上位層は下位層に依存してよいが、下位層は上位層に依存してはならない。循環依存を禁止し、一方向の依存関係を維持する。

### 2.3 抽象化による結合度最小化（Loose Coupling through Abstraction）
層間の通信はインターフェースや抽象化を通じて行い、具体的な実装への直接依存を避ける。

### 2.4 カプセル化と情報隠蔽（Encapsulation and Information Hiding）
各層の内部実装詳細を他層から隠蔽し、公開APIのみを通じて相互作用する。

### 2.5 置換可能性の確保（Substitutability）
同一層内の実装は、インターフェースを満たす限り相互に置換可能でなければならない。

### 2.6 横断的関心事の分離（Separation of Cross-cutting Concerns）
ログ、セキュリティ、トランザクション管理などの横断的関心事は、専用の仕組みで各層に適用する。

### 2.7 レイヤー透過性の禁止（No Layer Bypassing）
上位層は直下の層のみと通信し、中間層を飛び越えた通信を禁止する。

## 3. 設計ドキュメント体系

### 3.1 アーキテクチャ概要書（Architecture Overview）
**目的**: システム全体の層構造と原則を定義

```
ECサービス層構造：
- Presentation Layer (Web UI, API)
- Application Layer (注文処理、在庫管理)
- Domain Layer (商品、顧客、注文エンティティ)
- Infrastructure Layer (DB、外部決済API)
```

### 3.2 層間インターフェース定義書（Layer Interface Specification）
**目的**: 各層の責任と公開APIを明確化

```typescript
Application Layer → Domain Layer:
interface OrderService {
  createOrder(customerId: string, items: OrderItem[]): Order
  validateOrder(order: Order): ValidationResult
}
```

### 3.3 依存関係マップ（Dependency Map）
**目的**: 依存性の方向制御を可視化

```
Presentation → Application → Domain ← Infrastructure
（Infrastructure層のみDomainに依存注入で接続）
```

### 3.4 コンポーネント設計書（Component Design）
**目的**: 各層内のコンポーネント構造を定義

### 3.5 API仕様書（API Specification）
**目的**: 層間の通信プロトコルを定義

```json
POST /api/orders
Request: { 
  "customerId": "string", 
  "items": [], 
  "paymentMethod": "string" 
}
Response: { 
  "orderId": "string", 
  "status": "string", 
  "totalAmount": "number" 
}
```

### 3.6 データフロー図（Data Flow Diagram）
**目的**: 層を跨ぐデータの流れを可視化

```
注文処理フロー：
UI → OrderController → OrderApplicationService 
   → OrderDomainService → OrderRepository → DB
```

### 3.7 例外処理方針書（Exception Handling Policy）
**目的**: 各層での例外処理責任を明確化

- **Domain Layer**: ビジネスルール例外
- **Application Layer**: アプリケーション例外の統合
- **Presentation Layer**: HTTPステータス変換

## 4. 実装レベル設計：ECサービス例

### 4.1 Presentation Layer（プレゼンテーション層）
**責任**: ユーザーインターフェースと入出力制御

- `ProductController`: 商品一覧、詳細表示制御
- `OrderController`: 注文処理のHTTPリクエスト制御
- `CartController`: カート操作制御
- `AuthenticationFilter`: 認証フィルター
- `ValidationMiddleware`: 入力値検証
- `ErrorHandler`: エラーレスポンス変換

### 4.2 Application Layer（アプリケーション層）
**責任**: ビジネスプロセスの調整とトランザクション管理

- `OrderApplicationService`: 注文プロセス全体の調整
- `ProductApplicationService`: 商品管理プロセス
- `PaymentApplicationService`: 決済プロセス調整
- `InventoryApplicationService`: 在庫管理プロセス
- `NotificationService`: 通知送信調整
- `TransactionManager`: トランザクション境界管理

### 4.3 Domain Layer（ドメイン層）
**責任**: ビジネスルールとドメイン知識の表現

- `Product` (Entity): 商品の属性とビジネスルール
- `Order` (Entity): 注文の状態管理と計算ロジック
- `Customer` (Entity): 顧客情報と行動ルール
- `OrderItem` (Value Object): 注文明細の値オブジェクト
- `PricingService` (Domain Service): 価格計算ルール
- `InventoryDomainService`: 在庫ドメインロジック
- `OrderRepository` (Interface): 注文データアクセス抽象化

### 4.4 Infrastructure Layer（インフラストラクチャ層）
**責任**: 外部システムとの接続と技術的実装

- `OrderRepositoryImpl`: 注文データベースアクセス実装
- `ProductRepositoryImpl`: 商品データベースアクセス実装
- `PaymentGateway`: 外部決済サービス連携
- `EmailService`: メール送信実装
- `FileStorageService`: ファイル保存実装
- `DatabaseConfiguration`: DB接続設定
- `CacheService`: キャッシュ実装

## 5. コンポーネント間の協働例

### 注文処理の流れ
```
OrderController 
  ↓
OrderApplicationService 
  ↓
Order (Entity) + PricingService
  ↓
OrderRepositoryImpl + PaymentGateway
```

### 依存関係の実装例
```typescript
// Application Layer
class OrderApplicationService {
  constructor(
    private orderRepository: OrderRepository, // Interface依存
    private paymentService: PaymentService,   // Interface依存
    private notificationService: NotificationService
  ) {}
  
  async createOrder(orderData: CreateOrderRequest): Promise<Order> {
    const order = new Order(orderData);
    const validatedOrder = order.validate();
    const savedOrder = await this.orderRepository.save(validatedOrder);
    await this.paymentService.processPayment(savedOrder);
    await this.notificationService.sendConfirmation(savedOrder);
    return savedOrder;
  }
}
```