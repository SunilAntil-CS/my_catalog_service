Why Both are required ?

> 🔹 `@EnableJpaAuditing`  
> 🔹 `@EntityListeners(AuditingEntityListener.class)` on the entity

Let’s break it down — it's a **team effort** between your **Spring config** and your **JPA entity**.
---

### ✅ Why is `@EnableJpaAuditing` required?

- This tells Spring Boot to **activate the auditing feature**.
- It registers the beans (including `AuditingEntityListener`) needed to hook into JPA lifecycle events like `@PrePersist` and `@PreUpdate`.

Without this, **Spring Data JPA won't even look for auditing annotations** like `@CreatedDate`.

---

### ✅ Why is `@EntityListeners(AuditingEntityListener.class)` required?

- This attaches the **actual listener** (`AuditingEntityListener`) to your entity.
- It hooks into JPA lifecycle callbacks for **that specific entity class** and 
applies the logic for setting `createdDate`, `lastModifiedDate`, etc.

Without this, even if auditing is enabled globally, **your `Book` entity won’t get its fields updated automatically**.

---

### 📦 Analogy: Auditor Activation

| Think of it like this                        | What it maps to             |
|---------------------------------------------|-----------------------------|
| 🔌 Turning on the auditing system globally   | `@EnableJpaAuditing`        |
| 🎯 Registering the auditor for each entity   | `@EntityListeners(...)`     |

You need both to:
1. Turn the system **on**
2. Register the **entities** that will use it

---

### 🔎 TL;DR:

| Annotation                              | Needed For                                           |
|-----------------------------------------|------------------------------------------------------|
| `@EnableJpaAuditing` (in config class)  | Globally enabling auditing features in Spring        |
| `@EntityListeners(AuditingEntityListener.class)` | Telling JPA: “Apply auditing logic to this entity”     |

