# データベースアクセスの抽象化：RepositoryパターンとMyBatis

## 1. ORM（Object-Relational Mapping）とは

**ORM**は、リレーショナルデータベースとプログラミング言語のデータ構造をマッピングする技術です。データベースのテーブルとプログラム内のデータ構造（クラス、構造体、辞書など）を対応づけて、データベース操作を簡単にします。

### ORMの役割
- データベーステーブル ↔ プログラムのデータ構造の変換
- SQLの抽象化
- データベースとプログラムの構造の違いを吸収

## 2. Repositoryパターン

**Repositoryパターン**は、データアクセス層を抽象化するデザインパターンです。データをあたかもメモリ内のコレクションのように扱えるインターフェースを提供し、実際のデータ保存方法の詳細を隠蔽します。

### 目的
- データアクセス層の抽象化
- ビジネスロジックとデータ永続化の分離
- テスタビリティの向上
- データベース技術の切り替え可能性

## 3. MyBatisの特徴

### 主な特徴
1. **XML設定でSQL直接記述**: XMLファイルに実際のSQLを記述
2. **手動SQL定義**: `findAll()`や`save()`などの基本的なCRUD操作も手動で定義
3. **SQL完全制御**: 複雑なクエリやパフォーマンスチューニングが可能
4. **軽量**: JPA/Hibernateと比較して軽量で既存DBに適用しやすい

### 他のORMとの違い
- **JPA/Hibernate**: 自動クエリ生成、アノテーション中心
- **MyBatis**: 手動SQL定義、XML中心、SQL完全制御

## 4. Spring Boot + MyBatis実装例

### 4.1 エンティティクラス

```java
public class User {
    private Long id;
    private String email;
    private String name;
    private LocalDateTime createdAt;
    
    public User() {}
    
    public User(String email, String name) {
        this.email = email;
        this.name = name;
        this.createdAt = LocalDateTime.now();
    }
    
    // getter、setter省略
}
```

### 4.2 Repository実装（シンプルなアプローチ）

```java
// UserRepository.java
@Mapper
public interface UserRepository {
    Optional<User> findById(Long id);
    List<User> findAll();
    void insert(User user);
    void update(User user);
    void delete(Long id);
}
```

### 4.3 XML設定（実際の実装）

```xml
<!-- UserRepository.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" 
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<mapper namespace="com.example.repository.UserRepository">
    
    <select id="findById" parameterType="long" resultType="User">
        SELECT id, email, name, created_at
        FROM users 
        WHERE id = #{id}
    </select>
    
    <select id="findAll" resultType="User">
        SELECT id, email, name, created_at
        FROM users 
        ORDER BY created_at DESC
    </select>
    
    <insert id="insert" parameterType="User" useGeneratedKeys="true" keyProperty="id">
        INSERT INTO users (email, name, created_at)
        VALUES (#{email}, #{name}, #{createdAt})
    </insert>
    
    <update id="update" parameterType="User">
        UPDATE users 
        SET email = #{email}, name = #{name}
        WHERE id = #{id}
    </update>
    
    <delete id="delete" parameterType="long">
        DELETE FROM users WHERE id = #{id}
    </delete>
    
</mapper>
```

### 4.4 サービス層

```java
@Service
@Transactional
public class UserService {
    
    private final UserRepository userRepository;
    
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    public User createUser(String email, String name) {
        User user = new User(email, name);
        userRepository.insert(user);
        return user;
    }
    
    public User getUserById(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new EntityNotFoundException("User not found"));
    }
    
    public List<User> getAllUsers() {
        return userRepository.findAll();
    }
    
    public User updateUser(Long id, String email, String name) {
        User user = getUserById(id);
        user.setEmail(email);
        user.setName(name);
        userRepository.update(user);
        return user;
    }
    
    public void deleteUser(Long id) {
        userRepository.delete(id);
    }
}
```

### 4.5 コントローラー層

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    private final UserService userService;
    
    public UserController(UserService userService) {
        this.userService = userService;
    }
    
    @PostMapping
    public ResponseEntity<User> createUser(@RequestBody CreateUserRequest request) {
        User user = userService.createUser(request.getEmail(), request.getName());
        return ResponseEntity.status(HttpStatus.CREATED).body(user);
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        User user = userService.getUserById(id);
        return ResponseEntity.ok(user);
    }
    
    @GetMapping
    public ResponseEntity<List<User>> getAllUsers() {
        List<User> users = userService.getAllUsers();
        return ResponseEntity.ok(users);
    }
}
```

## 5. 実装アプローチの選択

### シンプルなアプローチ（推奨）
- `@Mapper`アノテーションを付けたインターフェースをRepositoryとして直接使用
- XMLファイルに実装を記述
- 実用性重視、理解しやすい

### 厳密なRepositoryパターン
- ドメイン層にRepositoryインターフェースを定義
- MyBatisのMapperとは別の実装クラスを作成
- 設計的には正しいが、複雑さが増す

## 6. まとめ

**MyBatisの利点**：
- SQLを直接制御できるため、複雑なクエリやパフォーマンスチューニングが可能
- 既存のデータベース構造に柔軟に対応
- 軽量で学習コストが低い

**Repositoryパターンの効果**：
- データアクセス層の抽象化により、ビジネスロジックがデータベースの詳細に依存しない
- テスト時にモックやフェイクオブジェクトを使用可能
- データベース技術の変更時も、サービス層への影響を最小限に抑制

**実用的な組み合わせ**：
- `@Mapper`インターフェースを直接Repositoryとして使用
- XMLファイルでSQL実装を記述
- シンプルで保守しやすい構造

MyBatisとRepositoryパターンの組み合わせにより、SQLの柔軟性を保ちながら、保守性とテスト性の高いアプリケーションを構築できます。