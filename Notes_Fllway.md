Here’s how Flyway integrates with Spring Boot 2—and how you get zero‑boilerplate, transactional schema migrations on app startup:

---

## 1. Add the Flyway Dependency

Spring Boot’s auto‑configuration kicks in as soon as it sees Flyway on the classpath. In your `pom.xml`:



```groovy
implementation "org.flywaydb:flyway-core"
```

---

## 2. Place Your Migration Scripts

By convention, Flyway looks under `src/main/resources/db/migration`. Each file must follow:

```
V<version>__<description>.sql
```

For example:

```
src/main/resources/db/migration/
├── V1__create_books_table.sql
├── V2__add_author_column.sql
└── V3__create_review_table.sql
```

- **`V`**: versioned migration  
- **`1, 2, 3…`**: version number (integer or dotted)  
- **`__`**: separator  
- **`description`**: free‑form (underscores ok)  

---

## 3. Configure via `application.yml` / `properties`

Spring Boot exposes all Flyway settings under the `spring.flyway.*` namespace. The most common:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/bookstore
    username: postgres
    password: secret
  flyway:
    enabled: true                    # default is true if flyway-core on classpath
    locations: classpath:db/migration
    baseline-on-migrate: true        # for existing schemas
    baseline-version: 1
    placeholder-replacement: true
    placeholders:
      schemaName: public
    sql-migration-prefix: V
    repeatable-sql-migration-prefix: R
    out-of-order: false
```

- **`enabled`**: switch Flyway on/off  
- **`locations`**: where to find migration scripts  
- **`baseline-on-migrate`**: if you’re adding Flyway to an existing schema  
- **`out-of-order`**: allows applying older versions after newer ones (use with care)  
- **`placeholders`**: for `${…}` in your SQL  

---

## 4. What Happens at Startup

1. **Spring Boot’s `FlywayAutoConfiguration`** sees your `spring.datasource` + `flyway-core` on the classpath.  
2. It creates a **`Flyway` bean** and wires in your `DataSource`.  
3. Before your application `DataSource` is used by anything else, it calls `flyway.migrate()`.  
4. Flyway checks (or creates) the `flyway_schema_history` table to see which versions have run.  
5. It runs each pending migration script **inside a transaction** (where the DB supports it), one by one, in version order.  

---

## 5. Schema History Table

By default, Flyway creates a table called `flyway_schema_history` with columns:

| installed_rank | version | description | type  | script                     | checksum | installed_by | installed_on      | execution_time | success |
|----------------|---------|-------------|-------|----------------------------|----------|--------------|-------------------|----------------|---------|

This prevents rerunning migrations and tracks everything.

---

## 6. Handling Multiple DataSources

If you have more than one `DataSource`, annotate one of them with `@FlywayDataSource` (Spring Boot 2.3+). For example:

```java
@Bean
@Primary
@ConfigurationProperties("spring.datasource")
public DataSource primaryDataSource() { … }

@Bean
@ConfigurationProperties("app.flyway.datasource")
@FlywayDataSource
public DataSource flywayDataSource() { … }
```

Then Flyway will use the one marked with `@FlywayDataSource`.

---

## 7. Programmatic Control

You can also inject the `Flyway` bean and call methods yourself:

```java
@Component
public class MigrationRunner {

    private final Flyway flyway;

    public MigrationRunner(Flyway flyway) {
        this.flyway = flyway;
    }

    @EventListener(ApplicationReadyEvent.class)
    public void onReady() {
        // e.g. repair a failed migration, or validate before migrate
        flyway.validate();
        flyway.repair();
        flyway.migrate();
    }
}
```

---

## 8. Advanced Features

- **Repeatable Migrations** (`R__…`): re‑applied whenever checksum changes (e.g. for views or stored procs).  
- **Callbacks**: place scripts under `db/migration` named `beforeMigrate.sql`, `afterMigrate.sql`, `beforeEachMigrate.sql`, etc.  
- **Java-based Migrations**: implement `JavaMigration` for complex logic.  
- **Placeholders**: use `${schemaName}` in your SQL and configure via `spring.flyway.placeholders`.  

---

### Quick Recipe

1. **Add** `flyway-core` dependency.  
2. **Drop** `V*.sql` files under `src/main/resources/db/migration`.  
3. **Configure** `spring.datasource` & optional `spring.flyway.*`.  
4. **Start** your app → Flyway runs `migrate()` before Hibernate or JPA initializes.  

That’s it—your schema is always up‑to‑date, versioned, and repeatable, with no extra plumbing. 
ßLet me know if you want examples of callbacks or Java migrations!