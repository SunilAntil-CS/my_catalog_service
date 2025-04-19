
How Custom methods are implemented  in Repository!

[ ApplicationContext Startup ]
         │
         ├─ Scan packages for Repository interfaces
         │   → Register RepositoryFactoryBean(for BookRepository)
         │
  [ Context Refresh ]
         │
         ├─ Instantiate each FactoryBean
         │   → FactoryBean uses JpaRepositoryFactory
         │       → Creates proxy implementing BookRepository
         │       → Behind proxy: a SimpleJpaRepository<Book,Long> instance
         │
         ↓
[ Your Code Calls repo.findByIsbn("123") ]
         │
         ├─ Proxy intercepts call
         ├─ Delegates to SimpleJpaRepository
         ├─ SimpleJpaRepository builds & runs JPQL query
         └─ Returns result to your service


********************
 In Spring, `@Transactional` is the primary way to declaratively manage database transactions. 
 When you annotate a class or method with it, Spring will automatically start, commit, or roll back a transaction around your code. 
 Here’s how it works:

---

## 1. What `@Transactional` Does

- **Begin a transaction** before entering the annotated method.  
- **Commit the transaction** if the method completes normally.  
- **Roll back the transaction** if the method throws an exception that matches its rollback rules.  

Under the hood, Spring creates an AOP proxy around your bean. When you call a `@Transactional` method:

1. The proxy intercepts the call.  
2. It obtains a `PlatformTransactionManager` (e.g. `DataSourceTransactionManager` or `JpaTransactionManager`).  
3. It opens or joins a transaction according to the **propagation** setting.  
4. It invokes your method.  
5. Based on the outcome and the **rollbackFor** rules, it commits or rolls back.  
6. Finally, it closes or releases the transaction.

---

## 2. Where to Put It

- **On a class**: all `public` methods inherit the same settings.  
- **On a method**: overrides any class‑level settings for that method only.  

```java
@Service
@Transactional              // default applies to every public method
public class OrderService {
  
  public void placeOrder(Order o) { … }  // transactional
  
  @Transactional(readOnly = true)
  public Order findOrder(Long id) { … }   // read-only transactional
}
```

---

## 3. Key Attributes

| Attribute       | Purpose                                                                                  | Default         |
|-----------------|------------------------------------------------------------------------------------------|-----------------|
| `propagation`   | How to handle existing transactions:  
                   - `REQUIRED` (join or create)  
                   - `REQUIRES_NEW` (suspend + new)  
                   - `SUPPORTS`, `MANDATORY`, `NEVER`, etc.                                            | `REQUIRED`      |
| `isolation`     | SQL isolation level (`READ_COMMITTED`, `SERIALIZABLE`, etc.)                            | Use DB default  |
| `readOnly`      | Hint for optimizations (can disable dirty‑checking or use read‑only DB replicas)        | `false`         |
| `timeout`       | Maximum time in seconds before automatic rollback                                         | `-1` (no limit) |
| `rollbackFor`   | Exception classes that should trigger rollback                                           | `RuntimeException` and `Error` |
| `noRollbackFor` | Exceptions that should **not** trigger rollback                                          | none            |

---

## 4. Propagation Examples

```java
@Transactional(propagation = Propagation.REQUIRED)
public void serviceA() {
    // joins existing tx or starts a new one
    repo.save(...);
    serviceB();       // same transaction
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void serviceB() {
    // always starts its own transaction, suspending the outer one
    logAudit(...);
}
```

- **`REQUIRED`** (the default): run in the caller’s transaction or start one if none exists.  
- **`REQUIRES_NEW`**: always suspend any existing transaction and start a brand‑new one.  
- **Others** (`SUPPORTS`, `MANDATORY`, `NOT_SUPPORTED`, `NEVER`, `NESTED`) cover more specialized scenarios.

---

## 5. Rollback Rules

By default, Spring will roll back on any unchecked exception (subclass of `RuntimeException`) or `Error`, but **not** on checked exceptions. You can customize:

```java
@Transactional(rollbackFor = IOException.class)
public void importFile(...) throws IOException {
    // throws IOException → rollback
}
```

---

## 6. Common Pitfalls

- **Self‑invocation**: calling one `@Transactional` method from another in the same class bypasses the proxy, so the second method won’t start a new transaction.  
- **Non‑public methods**: by default only `public` methods are proxied—`protected` or `private` won’t be transactional unless you use class‑based (CGLIB) proxies and adjust settings.  
- **Missing transaction manager**: make sure you have the right `PlatformTransactionManager` bean on the classpath (`spring-boot-starter-data-jpa` or `spring-boot-starter-jdbc` auto‑configures one).

---

## 7. Under the Hood

1. **AOP Proxy Creation**  
   - Spring’s `@EnableTransactionManagement` (implicitly enabled by Spring Boot) registers a `TransactionInterceptor`.  
   - It wraps any bean with methods annotated by `@Transactional`.

2. **Interceptor Logic**  
   - Before method execution: the interceptor calls `transactionManager.getTransaction(definition)`.  
   - After method execution: it calls either `transactionManager.commit(status)` or `transactionManager.rollback(status)`
    based on exceptions and rollback rules.

---

### Quick Checklist

- ✅ Annotate your service or DAO methods with `@Transactional`.  
- ✅ Choose appropriate **propagation** and **isolation** levels.  
- ✅ Mark read‑only methods with `readOnly=true` for performance hints.  
- ✅ Handle self‑invocation by moving transactional methods into separate beans if needed.  
- ✅ Ensure your transaction manager is correctly configured.

---

