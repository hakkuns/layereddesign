# Spring AOP（アスペクト指向プログラミング）

## AOPとは

**AOP（Aspect-Oriented Programming）** は、横断的関心事を効率的に管理するデザインパターンです。

### 横断的関心事の例
- ログ出力
- セキュリティチェック  
- パフォーマンス測定
- トランザクション管理

## AOP用語

- **Aspect**: 横断的関心事をまとめたモジュール
- **Advice**: 実際に実行される処理（@Before、@Afterなど）
- **Pointcut**: どのメソッドにAdviceを適用するかの条件
- **JoinPoint**: プログラム実行中の特定ポイント（メソッド呼び出しなど）

## AOPの利点

### 1. 関心事の分離
ビジネスロジックから横断的関心事を分離し、コードの可読性を向上

### 2. 再利用性
一度作成したAspectは複数のクラス・メソッドで再利用可能

### 3. 保守性
横断的機能の変更時、Aspectクラスのみを修正すればよい

## Spring Bootでのアノテーション実装

### コントローラー
```java
@RestController
public class UserController {
    
    @PostMapping("/user")
    public String createUser(@RequestBody String name) {
        return "ユーザー作成: " + name;
    }
}
```

### Aspect
```java
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.*;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class LoggingAspect {
    
    @Before("execution(* com.example.UserController.createUser(..))")
    public void logBefore(JoinPoint joinPoint) {
        System.out.println("メソッド開始: " + joinPoint.getSignature().getName());
    }
    
    @After("execution(* com.example.UserController.createUser(..))")
    public void logAfter(JoinPoint joinPoint) {
        System.out.println("メソッド終了: " + joinPoint.getSignature().getName());
    }
}
```

### 設定クラス（AOPを有効化）
```java
@Configuration
@EnableAspectJAutoProxy  // @Aspectアノテーションを自動認識してプロキシを生成
public class AopConfig {
}
```
この設定により、Spring Boot が `@Aspect` アノテーションを認識し、自動的にAOPプロキシを作成します。

## XMLベースの設定

### コントローラー
```java
@Controller  // 自動的にBean化される
public class UserController {
    
    public String createUser(String name) {
        return "ユーザー作成: " + name;
    }
}
```

### Adviceクラス
```java
import org.aopalliance.intercept.MethodInterceptor;
import org.aopalliance.intercept.MethodInvocation;

@Component  // Bean化
public class LoggingInterceptor implements MethodInterceptor {
    
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        System.out.println("メソッド開始: " + invocation.getMethod().getName());
        Object result = invocation.proceed();
        System.out.println("メソッド終了: " + invocation.getMethod().getName());
        return result;
    }
}
```

### XML設定（aop:config方式）
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="
           http://www.springframework.org/schema/beans 
           http://www.springframework.org/schema/beans/spring-beans.xsd
           http://www.springframework.org/schema/aop
           http://www.springframework.org/schema/aop/spring-aop.xsd
           http://www.springframework.org/schema/context
           http://www.springframework.org/schema/context/spring-context.xsd">

    <!-- アノテーションによるBean自動検出を有効化 -->
    <context:component-scan base-package="com.example" />

    <!-- AOP設定 -->
    <aop:config>
        <aop:pointcut id="createUserPointcut" 
                      expression="execution(* com.example.UserController.createUser(..))" />
        
        <aop:aspect ref="loggingAdvice">
            <aop:before pointcut-ref="createUserPointcut" method="logBefore" />
            <aop:after pointcut-ref="createUserPointcut" method="logAfter" />
        </aop:aspect>
    </aop:config>
</beans>
```

### XML設定（ProxyFactoryBean方式）
```xml
<beans xmlns:context="http://www.springframework.org/schema/context">
    
    <!-- アノテーションによるBean自動検出 -->
    <context:component-scan base-package="com.example" />

    <!-- Pointcut定義 -->
    <bean id="methodPointcut" class="org.springframework.aop.support.NameMatchMethodPointcut">
        <property name="mappedName" value="createUser" />
    </bean>

    <!-- Advisor -->
    <bean id="loggingAdvisor" class="org.springframework.aop.support.DefaultPointcutAdvisor">
        <property name="pointcut" ref="methodPointcut" />
        <property name="advice-ref" value="loggingInterceptor" />
    </bean>

    <!-- プロキシ -->
    <bean id="userControllerProxy" class="org.springframework.aop.framework.ProxyFactoryBean">
        <property name="target-ref" value="userController" />
        <property name="interceptorNames" value="loggingAdvisor" />
    </bean>
</beans>
```