# 03 - DTOs & VALIDATION


## REGLA DE ORO: NUNCA EXPONER ENTITIES EN API


```
❌ Controller retorna Entity directamente
✅ Controller retorna DTO (Record)
```


**Por qué:**
- Entities tienen lógica de persistencia (lazy loading, cascades)
- Cambios en entities rompen contratos de API
- Seguridad: no exponer estructura interna de BD
- Control: decidir qué campos enviar/recibir


---


## ✅ DTO PATTERN CON RECORDS (Java 17+)


### Request DTO
```java
public record UserRequest(
   @NotBlank(message = "Email is required")
   @Email(message = "Invalid email format")
   String email,
  
   @NotBlank(message = "First name is required")
   @Size(min = 2, max = 50, message = "First name must be 2-50 characters")
   String firstName,
  
   @NotBlank(message = "Last name is required")
   @Size(min = 2, max = 50, message = "Last name must be 2-50 characters")
   String lastName,
  
   @Pattern(regexp = "^(?=.*[A-Za-z])(?=.*\\d)[A-Za-z\\d]{8,}$",
            message = "Password must be 8+ characters with letters and numbers")
   String password
) {}
```


### Response DTO
```java
public record UserResponse(
   Long id,
   String email,
   String firstName,
   String lastName,
   String status,
   LocalDateTime createdAt
) {
   // Constructor desde Entity (opcional)
   public UserResponse(User user) {
       this(
           user.getId(),
           user.getEmail(),
           user.getFirstName(),
           user.getLastName(),
           user.getStatus().name(),
           user.getCreatedAt()
       );
   }
}
```


### Lista Response DTO
```java
public record UserListResponse(
   List<UserResponse> users,
   int totalCount,
   int pageNumber,
   int pageSize
) {}
```


**Por qué Records:**
- ✅ Inmutables (thread-safe)
- ✅ Menos boilerplate (no getters/setters)
- ✅ Syntax moderna Java 17+
- ✅ `equals()`, `hashCode()`, `toString()` automáticos
- ✅ Semántica clara: "data carrier"


---


## 📋 VALIDACIONES JAKARTA


### Anotaciones Comunes
```java
public record ProductRequest(
   @NotNull(message = "Name cannot be null")
   @NotBlank(message = "Name cannot be empty")
   @Size(min = 3, max = 100, message = "Name must be 3-100 characters")
   String name,
  
   @NotNull(message = "Price is required")
   @DecimalMin(value = "0.01", message = "Price must be positive")
   @DecimalMax(value = "999999.99", message = "Price too high")
   BigDecimal price,
  
   @Min(value = 0, message = "Stock cannot be negative")
   @Max(value = 100000, message = "Stock exceeds limit")
   Integer stock,
  
   @NotBlank(message = "Category is required")
   String category,
  
   @Size(max = 500, message = "Description max 500 characters")
   String description,
  
   @Email(message = "Invalid email")
   String contactEmail,
  
   @Pattern(regexp = "^[A-Z]{2,3}-\\d{4,6}$", message = "Invalid SKU format")
   String sku,
  
   @Past(message = "Manufacture date must be in the past")
   LocalDate manufacturedAt,
  
   @Future(message = "Expiry date must be in the future")
   LocalDate expiresAt
) {}
```


### Validación de Listas
```java
public record OrderRequest(
   @NotNull
   Long customerId,
  
   @NotEmpty(message = "Order must have at least one item")
   @Valid  // Valida cada elemento
   List<OrderItemRequest> items
) {}


public record OrderItemRequest(
   @NotNull Long productId,
   @Min(value = 1, message = "Quantity must be at least 1")
   Integer quantity
) {}
```


### Validación Condicional (Grupos)
```java
public interface CreateValidation {}
public interface UpdateValidation {}


public record UserRequest(
   @Null(groups = CreateValidation.class)  // null en create
   @NotNull(groups = UpdateValidation.class)  // requerido en update
   Long id,
  
   @NotBlank(groups = {CreateValidation.class, UpdateValidation.class})
   String email
) {}


// Uso en Controller
@PostMapping
public ResponseEntity<UserResponse> create(
   @Validated(CreateValidation.class) @RequestBody UserRequest request
) { }


@PutMapping("/{id}")
public ResponseEntity<UserResponse> update(
   @PathVariable Long id,
   @Validated(UpdateValidation.class) @RequestBody UserRequest request
) { }
```


---


## 🎯 ACTIVAR VALIDACIÓN EN CONTROLLER


```java
@RestController
@RequestMapping("/api/products")
@RequiredArgsConstructor
@Slf4j
public class ProductController {
  
   private final ProductService productService;
  
   @PostMapping
   public ResponseEntity<ProductResponse> create(
       @Valid @RequestBody ProductRequest request  // ← @Valid CRÍTICO
   ) {
       log.info("Creating product: {}", request.name());
       ProductResponse response = productService.create(request);
       return ResponseEntity.status(201).body(response);
   }
  
   @PutMapping("/{id}")
   public ResponseEntity<ProductResponse> update(
       @PathVariable Long id,
       @Valid @RequestBody ProductRequest request  // ← @Valid CRÍTICO
   ) {
       ProductResponse response = productService.update(id, request);
       return ResponseEntity.ok(response);
   }
}
```


**Sin `@Valid`:**
- ❌ Validaciones NO se ejecutan
- ❌ Datos inválidos pasan al service


**Con `@Valid`:**
- ✅ Spring valida automáticamente
- ✅ Si falla, lanza `MethodArgumentNotValidException`
- ✅ Exception handler lo captura y retorna 400


---


## ⚠️ ERROR RESPONSE DTO


```java
public record ErrorResponse(
   String message,
   int status,
   LocalDateTime timestamp,
   List<String> errors
) {
   // Constructor simple
   public ErrorResponse(String message, int status) {
       this(message, status, LocalDateTime.now(), List.of());
   }
  
   // Constructor con errores múltiples
   public ErrorResponse(String message, int status, List<String> errors) {
       this(message, status, LocalDateTime.now(), errors);
   }
}
```


### Exception Handler Global
```java
@ControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
  
   @ExceptionHandler(MethodArgumentNotValidException.class)
   public ResponseEntity<ErrorResponse> handleValidation(
       MethodArgumentNotValidException ex
   ) {
       List<String> errors = ex.getBindingResult()
           .getFieldErrors()
           .stream()
           .map(e -> e.getField() + ": " + e.getDefaultMessage())
           .toList();
      
       log.error("Validation errors: {}", errors);
      
       return ResponseEntity
           .badRequest()
           .body(new ErrorResponse("Validation failed", 400, errors));
   }
  
   @ExceptionHandler(EntityNotFoundException.class)
   public ResponseEntity<ErrorResponse> handleNotFound(
       EntityNotFoundException ex
   ) {
       log.error("Entity not found: {}", ex.getMessage());
       return ResponseEntity
           .status(404)
           .body(new ErrorResponse(ex.getMessage(), 404));
   }
  
   @ExceptionHandler(IllegalArgumentException.class)
   public ResponseEntity<ErrorResponse> handleBadRequest(
       IllegalArgumentException ex
   ) {
       log.error("Bad request: {}", ex.getMessage());
       return ResponseEntity
           .badRequest()
           .body(new ErrorResponse(ex.getMessage(), 400));
   }
  
   @ExceptionHandler(Exception.class)
   public ResponseEntity<ErrorResponse> handleGeneral(Exception ex) {
       log.error("Unexpected error", ex);
       return ResponseEntity
           .status(500)
           .body(new ErrorResponse("Internal server error", 500));
   }
}
```


---


## 🔄 MAPPERS ENTITY ↔ DTO


### Opción 1: Mapper Manual (Recomendado para proyectos simples)


```java
@Service
@RequiredArgsConstructor
@Slf4j
public class UserService {
  
   private final UserRepository userRepository;
  
   public UserResponse create(UserRequest request) {
       // Request DTO → Entity
       User user = User.builder()
           .email(request.email())
           .firstName(request.firstName())
           .lastName(request.lastName())
           .password(encodePassword(request.password()))
           .status(UserStatus.ACTIVE)
           .build();
      
       User saved = userRepository.save(user);
      
       // Entity → Response DTO
       return toResponse(saved);
   }
  
   public Optional<UserResponse> findById(Long id) {
       return userRepository.findById(id)
           .map(this::toResponse);
   }
  
   public List<UserResponse> findAll() {
       return userRepository.findAll()
           .stream()
           .map(this::toResponse)
           .toList();
   }
  
   // Método privado de mapeo
   private UserResponse toResponse(User user) {
       return new UserResponse(
           user.getId(),
           user.getEmail(),
           user.getFirstName(),
           user.getLastName(),
           user.getStatus().name(),
           user.getCreatedAt()
       );
   }
  
   private String encodePassword(String password) {
       // Lógica de encoding
       return password; // Placeholder
   }
}
```


### Opción 2: MapStruct (Para proyectos grandes con muchos DTOs)


```java
@Mapper(componentModel = "spring")
public interface UserMapper {
  
   @Mapping(target = "id", ignore = true)
   @Mapping(target = "createdAt", ignore = true)
   @Mapping(target = "status", constant = "ACTIVE")
   User toEntity(UserRequest request);
  
   @Mapping(target = "status", expression = "java(user.getStatus().name())")
   UserResponse toResponse(User user);
  
   List<UserResponse> toResponseList(List<User> users);
}


// Uso en Service
@Service
@RequiredArgsConstructor
public class UserService {
  
   private final UserRepository userRepository;
   private final UserMapper userMapper;
  
   public UserResponse create(UserRequest request) {
       User user = userMapper.toEntity(request);
       User saved = userRepository.save(user);
       return userMapper.toResponse(saved);
   }
}
```


**Cuándo usar cada opción:**
- ✅ Manual: <10 DTOs, mapeos simples, control total
- ✅ MapStruct: >20 DTOs, mapeos complejos, múltiples proyecciones


---


## 📦 PACKAGE STRUCTURE


```
com.example.myapp.model/
├── dto/
│   ├── request/
│   │   ├── UserRequest.java          # Record
│   │   ├── ProductRequest.java       # Record
│   │   └── OrderRequest.java         # Record
│   └── response/
│       ├── UserResponse.java         # Record
│       ├── ProductResponse.java      # Record
│       ├── OrderResponse.java        # Record
│       └── ErrorResponse.java        # Record
├── entity/
│   ├── User.java                     # @Entity
│   ├── Product.java                  # @Entity
│   └── Order.java                    # @Entity
└── mapper/
   ├── UserMapper.java               # MapStruct (opcional)
   └── ProductMapper.java            # MapStruct (opcional)
```


---


## 🎯 EJEMPLOS COMPLETOS


### CRUD Completo con DTOs


```java
// Request
public record ProductRequest(
   @NotBlank String name,
   @NotNull @DecimalMin("0.01") BigDecimal price,
   @Min(0) Integer stock
) {}


// Response
public record ProductResponse(
   Long id,
   String name,
   BigDecimal price,
   Integer stock,
   LocalDateTime createdAt
) {}


// Service
@Service
@RequiredArgsConstructor
public class ProductService {
  
   private final ProductRepository productRepository;
  
   public ProductResponse create(ProductRequest request) {
       Product product = Product.builder()
           .name(request.name())
           .price(request.price())
           .stock(request.stock())
           .build();
      
       return toResponse(productRepository.save(product));
   }
  
   public ProductResponse update(Long id, ProductRequest request) {
       Product product = productRepository.findById(id)
           .orElseThrow(() -> new ProductNotFoundException(id));
      
       product.setName(request.name());
       product.setPrice(request.price());
       product.setStock(request.stock());
      
       return toResponse(product);  // Auto-save por @Transactional
   }
  
   private ProductResponse toResponse(Product product) {
       return new ProductResponse(
           product.getId(),
           product.getName(),
           product.getPrice(),
           product.getStock(),
           product.getCreatedAt()
       );
   }
}
```


---


## 🚫 ANTI-PATTERNS


### ❌ Exponer Entity en API
```java
@RestController
public class UserController {
  
   @GetMapping("/users/{id}")
   public User getUser(@PathVariable Long id) {  // ❌ NO!
       return userRepository.findById(id).orElseThrow();
   }
}
```


**Problemas:**
- Lazy loading exceptions (N+1)
- Expone estructura interna
- Cambios en entity rompen API
- Circular references en JSON


### ✅ Usar DTO
```java
@RestController
public class UserController {
  
   @GetMapping("/users/{id}")
   public ResponseEntity<UserResponse> getUser(@PathVariable Long id) {
       return userService.findById(id)
           .map(ResponseEntity::ok)
           .orElse(ResponseEntity.notFound().build());
   }
}
```


---


## 🎯 CHECKLIST DTOs


Antes de crear DTO:


- [ ] ¿Es un Record (Java 17+)?
- [ ] ¿Tiene validaciones apropiadas?
- [ ] ¿Campos son inmutables?
- [ ] ¿Naming claro? (UserRequest/UserResponse)
- [ ] ¿Se usa `@Valid` en controller?
- [ ] ¿Exception handler maneja errores de validación?
- [ ] ¿NO expone Entity directamente?
- [ ] ¿Service retorna DTO, no Entity?


---


## 💡 REGLAS FINALES


1. **NUNCA retornar Entity desde controller**
2. **SIEMPRE usar Records para DTOs** (Java 17+)
3. **SIEMPRE `@Valid` en request DTOs**
4. **Validaciones en DTO, NO en Entity** (preferido)
5. **ErrorResponse consistente** para todos los errores
6. **Mappers manuales primero**, MapStruct solo si hay >20 DTOs
7. **Un DTO Request + un DTO Response** por operación
8. **DTOs simples** (no lógica de negocio)


---


**Versión:** 2.0 
**Stack:** Java 21 + Jakarta Validation 
**Enfoque:** Buenas prácticas genéricas API design

