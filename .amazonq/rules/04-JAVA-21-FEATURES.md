# 04 - JAVA 21 FEATURES


## FEATURES MODERNAS A USAR


Aprovecha las características de Java 21 para escribir código más limpio y conciso.


---


## ✅ RECORDS (Java 16+)


### Para DTOs
```java
// ✅ CORRECTO: Record para DTO inmutable
public record UserResponse(
   Long id,
   String email,
   String fullName,
   LocalDateTime createdAt
) {}


// ❌ INCORRECTO: Clase tradicional con boilerplate
public class UserResponse {
   private Long id;
   private String email;
   // ... getters, setters, equals, hashCode, toString
}
```


### Record con Validación
```java
public record CreateUserRequest(
   @NotBlank @Email String email,
   @NotBlank String password
) {
   // Validación adicional en constructor compacto
   public CreateUserRequest {
       if (password.length() < 8) {
           throw new IllegalArgumentException("Password too short");
       }
   }
}
```


### Record con Métodos
```java
public record Product(Long id, String name, BigDecimal price) {
  
   // Método de instancia
   public boolean isExpensive() {
       return price.compareTo(BigDecimal.valueOf(1000)) > 0;
   }
  
   // Método estático factory
   public static Product create(String name, BigDecimal price) {
       return new Product(null, name, price);
   }
}
```


**Cuándo usar Records:**
- ✅ DTOs (Request/Response)
- ✅ Value Objects inmutables
- ✅ Data carriers simples
- ❌ NO para entities JPA (usar @Entity class)
- ❌ NO si necesitas herencia


---


## ✅ TEXT BLOCKS (Java 15+)


### SQL Queries
```java
// ✅ CORRECTO: Text block
@Query("""
   SELECT u FROM User u
   LEFT JOIN FETCH u.orders
   WHERE u.status = :status
   AND u.createdAt > :date
   ORDER BY u.createdAt DESC
   """)
List<User> findActiveUsers(
   @Param("status") UserStatus status,
   @Param("date") LocalDateTime date
);


// ❌ INCORRECTO: String concatenación
@Query("SELECT u FROM User u " +
      "LEFT JOIN FETCH u.orders " +
      "WHERE u.status = :status " +
      "AND u.createdAt > :date " +
      "ORDER BY u.createdAt DESC")
```


### JSON Templates
```java
String jsonTemplate = """
   {
       "user": {
           "id": %d,
           "email": "%s",
           "active": %b
       }
   }
   """.formatted(userId, email, isActive);
```


### HTML/Email Templates
```java
String emailBody = """
   <html>
   <body>
       <h1>Welcome %s!</h1>
       <p>Your account has been created.</p>
   </body>
   </html>
   """.formatted(userName);
```


**Cuándo usar Text Blocks:**
- ✅ Queries SQL (JPQL, nativas)
- ✅ JSON/XML templates
- ✅ HTML/Email bodies
- ✅ Multi-line regex patterns
- ❌ NO para strings cortos (1 línea)


---


## ✅ PATTERN MATCHING (Java 21)


### Pattern Matching para instanceof
```java
// ✅ CORRECTO: Pattern matching
public String formatEntity(Object obj) {
   if (obj instanceof User user) {
       return "User: " + user.getEmail();
   } else if (obj instanceof Product product) {
       return "Product: " + product.getName();
   }
   return "Unknown";
}


// ❌ INCORRECTO: Cast manual
public String formatEntity(Object obj) {
   if (obj instanceof User) {
       User user = (User) obj;  // Cast innecesario
       return "User: " + user.getEmail();
   }
   return "Unknown";
}
```


### Switch Pattern Matching
```java
public double calculateDiscount(Customer customer) {
   return switch (customer) {
       case PremiumCustomer p -> p.getBalance() * 0.20;
       case RegularCustomer r -> r.getBalance() * 0.10;
       case GuestCustomer g -> 0.0;
       default -> throw new IllegalArgumentException("Unknown customer type");
   };
}
```


### Record Patterns
```java
public String formatResponse(Object response) {
   return switch (response) {
       case UserResponse(Long id, String email, _, _) ->
           "User #%d: %s".formatted(id, email);
       case ProductResponse(Long id, String name, BigDecimal price) ->
           "%s - $%.2f".formatted(name, price);
       default -> "Unknown";
   };
}
```


---


## ✅ VIRTUAL THREADS (Java 21)


### Para Operaciones Bloqueantes
```java
@Configuration
public class AsyncConfig {
  
   // Usar virtual threads para operaciones I/O intensivas
   @Bean
   public ExecutorService virtualThreadExecutor() {
       return Executors.newVirtualThreadPerTaskExecutor();
   }
}


@Service
@RequiredArgsConstructor
public class EmailService {
  
   private final ExecutorService virtualThreadExecutor;
  
   public void sendEmailAsync(String to, String subject, String body) {
       virtualThreadExecutor.submit(() -> {
           // Operación bloqueante en virtual thread
           sendEmail(to, subject, body);
       });
   }
}
```


### Con @Async
```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {
  
   @Override
   public Executor getAsyncExecutor() {
       return Executors.newVirtualThreadPerTaskExecutor();
   }
}


@Service
public class NotificationService {
  
   @Async
   public void sendNotification(User user) {
       // Se ejecuta en virtual thread automáticamente
   }
}
```


**Cuándo usar Virtual Threads:**
- ✅ Operaciones I/O bloqueantes (HTTP calls, DB queries)
- ✅ High concurrency scenarios (miles de requests)
- ✅ Simplificar código asíncrono
- ❌ NO para CPU-intensive tasks
- ❌ NO si ya usas reactive (WebFlux)


---


## ✅ SWITCH EXPRESSIONS (Java 14+)


### Como Expresión
```java
// ✅ CORRECTO: Switch expression
String message = switch (status) {
   case PENDING -> "Order is pending";
   case PROCESSING -> "Order is being processed";
   case SHIPPED -> "Order has been shipped";
   case DELIVERED -> "Order delivered";
   default -> "Unknown status";
};


// ❌ INCORRECTO: Switch statement tradicional
String message;
switch (status) {
   case PENDING:
       message = "Order is pending";
       break;
   case PROCESSING:
       message = "Order is being processed";
       break;
   // ...
}
```


### Con Yield
```java
int daysToDeliver = switch (shippingMethod) {
   case EXPRESS -> 1;
   case STANDARD -> {
       // Bloque de código complejo
       int baseDays = 3;
       if (isPriorityCustomer) {
           yield baseDays - 1;
       }
       yield baseDays;
   }
   case ECONOMY -> 7;
   default -> throw new IllegalArgumentException("Invalid shipping");
};
```


---


## ✅ SEALED CLASSES (Java 17+)


### Jerarquía Controlada
```java
public sealed interface PaymentMethod
   permits CreditCard, DebitCard, PayPal {
   BigDecimal process(BigDecimal amount);
}


public final class CreditCard implements PaymentMethod {
   private final String cardNumber;
  
   @Override
   public BigDecimal process(BigDecimal amount) {
       // Procesar con tarjeta de crédito
       return amount;
   }
}


public final class DebitCard implements PaymentMethod {
   @Override
   public BigDecimal process(BigDecimal amount) {
       // Procesar con tarjeta de débito
       return amount;
   }
}


public final class PayPal implements PaymentMethod {
   @Override
   public BigDecimal process(BigDecimal amount) {
       // Procesar con PayPal
       return amount;
   }
}
```


### Pattern Matching con Sealed
```java
public BigDecimal calculateFee(PaymentMethod method, BigDecimal amount) {
   return switch (method) {
       case CreditCard cc -> amount.multiply(BigDecimal.valueOf(0.03));
       case DebitCard dc -> amount.multiply(BigDecimal.valueOf(0.01));
       case PayPal pp -> amount.multiply(BigDecimal.valueOf(0.05));
       // No necesita default, compilador sabe que es exhaustivo
   };
}
```


**Cuándo usar Sealed:**
- ✅ Jerarquías cerradas conocidas
- ✅ Domain modeling (estados, tipos)
- ✅ Cuando quieres pattern matching exhaustivo
- ❌ NO para jerarquías extensibles por terceros


---


## 🎯 CHECKLIST JAVA 21


Antes de escribir código:


- [ ] ¿Puedo usar Record en lugar de clase?
- [ ] ¿Tengo query SQL multilinea? → Text block
- [ ] ¿Hago instanceof + cast? → Pattern matching
- [ ] ¿Uso switch tradicional? → Switch expression
- [ ] ¿Operaciones I/O bloqueantes? → Virtual threads
- [ ] ¿Jerarquía cerrada conocida? → Sealed class


---


## 💡 REGLAS FINALES


1. **Records** para todos los DTOs
2. **Text blocks** para queries y templates
3. **Pattern matching** en lugar de instanceof + cast
4. **Switch expressions** en lugar de statements
5. **Virtual threads** para I/O bloqueante intensivo
6. **Sealed classes** para jerarquías de dominio cerradas
7. Mantener compatibilidad Java 21+ (no features preview)


---


**Versión:** 1.0 
**Java:** 21 LTS 
**Enfoque:** Features estables y production-ready

