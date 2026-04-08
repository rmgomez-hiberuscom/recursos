# IMPLEMENTACIÓN DE LOGGING CORPORATIVO - BANCO PICHINCHA

## CONTEXTO
Actúa como Desarrollador Java Senior experto en la arquitectura de microservicios. Tu objetivo es implementar código Java siguiendo estrictamente los estándares de observabilidad y arquitectura corporativa (Banco Pichincha).

---

## 1. REGLAS DE ARQUITECTURA

### Nomenclatura
- **Idioma:** TODO en **INGLÉS** (clases, métodos, variables, comentarios)
- **Estilo:** Nombres descriptivos y completos, evitar acrónimos (excepto RUC/ADM)
- **Límites de Código:**
  - Máximo 20 líneas por método
  - Máximo 5 parámetros por método
  - Máximo 500 líneas por archivo
  - Máximo 120 caracteres por línea

### Estructura de Capas
```
Proyecto
├── application/
│   ├── service/     ← Logging con @BpLogger ✅
│   ├── input/port
│   └── output/port
├── domain/          ← Sin @BpLogger ❌
│   ├── model
│   ├── exception
│   └── value
├── infrastructure/
│   ├── input/adapter/soap
│   └── output/adapter/
│       ├── repository/  ← Logging con @BpLogger ✅
│       └── api/
└── resources/
    └── application.yml
```

---

## 2. LIBRERÍA TRACE LOGGER

### Dependencia Gradle
```gradle
implementation "com.pichincha.common:lib-trace-logger:latest"
```

### Dependencias de Logging Base
```gradle
implementation "ch.qos.logback:logback-classic:1.5.13"
implementation "net.logstash.logback:logstash-logback-encoder:7.4"
```

---

## 3. CONFIGURACIÓN APPLICATION.YML

### 3.1 Configuración Básica de Trace Logger
```yaml
trace-logger:
  enabled: ${CGI_TRACE_LOGGER_ENABLED:true}
  
  # Configuración de Payload
  payload:
    mode: FULL  # Opciones: NONE, FULL, PARTIAL
    fallback-mode: EMPTY  # Opciones: FULL, EMPTY
    request:
      json-paths:  # Campos a EXCLUIR en modo FULL
        - "password"
        - "token"
        - "creditCardNumber"
        - "accountNumber"
        - "pin"
        - "secretKey"
      xpaths:  # Para requests XML/SOAP
        - "//password"
        - "//creditCardNumber"
    response:
      json-paths:  # Campos a EXCLUIR en modo FULL
        - "authToken"
        - "sessionId"
        - "refreshToken"
      xpaths:  # Para responses XML/SOAP
        - "//token"
        - "//sessionId"
  
  # Configuración de Metadata Dinámica
  metadata:
    enabled: true
    fields:
      - key: "transactionId"
        json-path: "data.transactionId"
        source: REQUEST
      - key: "userId"
        json-path: "user.id"
        source: REQUEST
      - key: "responseStatus"
        json-path: "status"
        source: RESPONSE
      - key: "confirmationNumber"
        json-path: "confirmationNumber"
        source: RESPONSE

# Configuración de Logging General
logging:
  level:
    root: DEBUG
    org.springframework: INFO
    com.pichincha: DEBUG
```

### 3.2 Configuración por Ambiente

#### application-development.yml (DEV)
```yaml
trace-logger:
  enabled: true
  payload:
    mode: FULL
    fallback-mode: EMPTY

logging:
  level:
    root: DEBUG
    com.pichincha: DEBUG
```

#### application-production.yml (PROD)
```yaml
trace-logger:
  enabled: ${CGI_TRACE_LOGGER_ENABLED:true}
  payload:
    mode: FULL
    fallback-mode: EMPTY

logging:
  level:
    root: INFO
    com.pichincha: INFO
```

---

## 4. CONFIGURACIÓN LOGBACK - logback-spring.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <property name="LOG_FILE" value="${LOG_FILE:-${LOG_PATH:-${LOG_TEMP:-${java.io.tmpdir:-/tmp}}/}spring.log}"/>
    
    <!-- Appender para consola con Logstash -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeContext>true</includeContext>
            <includeMdc>true</includeMdc>
            <customFields>{"application":"MICROSERVICE_NAME","version":"1.0"}</customFields>
            <fieldNames>
                <timestamp>timestamp</timestamp>
                <level>level</level>
                <levelValue>level_code</levelValue>
                <logger>logger_name</logger>
                <message>message</message>
                <thread>thread_name</thread>
                <version>[ignore]</version>
            </fieldNames>
        </encoder>
    </appender>

    <!-- Logger específico para com.pichincha -->
    <logger name="com.pichincha" level="DEBUG" additivity="false">
        <appender-ref ref="CONSOLE"/>
    </logger>

    <!-- Root Logger -->
    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
    </root>
</configuration>
```

---

## 5. ANOTACIONES Y USO

### 5.1 @BpLogger - Logs de Aplicación

**Ubicación permitida:**
- ✅ Servicios de dominio (`application/service`)
- ✅ Use cases (`application/service`)
- ✅ Repositorios (`infrastructure/output/adapter/repository`)
- ✅ Adapters de integración

**Prohibido:**
- ❌ Controladores
- ❌ Domain models

**Emisión:** JSON a stdout con marcador `APPLICATION`

**Ejemplo:**
```java
@Service
@RequiredArgsConstructor
public class TransactionServiceImpl implements TransactionService {
    
    private final TransactionRepository repository;
    private final TransactionMapper mapper;
    
    @BpLogger
    public TransactionResponseDto processTransaction(TransactionRequestDto request) {
        Transaction entity = mapper.toEntity(request);
        Transaction saved = repository.save(entity);
        return mapper.toDto(saved);
    }
}
```

### 5.2 @BpObfuscatable - Ofuscación de Datos Sensibles

Marca campos que contienen PII o datos sensibles para su ofuscación automática.

**Aplica a:**
- Auth tokens
- Passwords
- Credit card numbers
- Account numbers
- Social security numbers / RUC
- Emails de clientes
- Números de teléfono

**Ejemplo:**
```java
@Getter
@Setter
@Builder(toBuilder = true)
@NoArgsConstructor
@AllArgsConstructor
public class TransactionDto {
    
    @BpObfuscatable
    private String creditCardNumber;
    
    @BpObfuscatable
    private String accountNumber;
    
    @BpObfuscatable
    private String authenticationToken;
    
    private String transactionId;
    private BigDecimal amount;
    private LocalDateTime timestamp;
}
```

### 5.3 CustomLogLevelHandler - Logs Dinámicos (SRE/Monitoreo)

Para eventos que requieran niveles dinámicos configurables (DEBUG, INFO, WARN, ERROR).

**Importar:**
```java
import com.pichincha.common.trace.logger.logger.custom.level.CustomLogLevel;
import com.pichincha.common.trace.logger.logger.custom.level.CustomLogLevelHandler;
```

**Uso:**
```java
@Service
@RequiredArgsConstructor
public class OrderServiceImpl {
    
    private final CustomLogLevelHandler customLogLevelHandler;
    private final OrderRepository repository;
    
    public OrderDto createOrder(OrderRequestDto request) {
        try {
            // Lógica...
            Order order = repository.save(mapper.toEntity(request));
            return mapper.toDto(order);
        } catch (InvalidAmountException ex) {
            customLogLevelHandler.log(
                    CustomLogLevel.ERROR,
                    Thread.currentThread().getStackTrace(),
                    "Invalid order amount detected",
                    new ErrorContext(request.getAmount(), ex.getMessage())
            );
            throw ex;
        }
    }
}
```

---

## 6. PATRONES DE IMPLEMENTACIÓN

### 6.1 DTOs con Ofuscación

```java
@Getter
@Setter
@Builder(toBuilder = true)
@NoArgsConstructor
@AllArgsConstructor
public class UserAccountDto {
    
    private String userId;
    
    @BpObfuscatable
    private String accountNumber;
    
    @BpObfuscatable
    private String password;
    
    @BpObfuscatable
    private String email;  // Si es dato sensible de cliente
    
    private String userName;
    private LocalDateTime createdAt;
}
```

### 6.2 Service con @BpLogger

```java
@Service
@RequiredArgsConstructor
public class PaymentServiceImpl implements PaymentService {
    
    private final PaymentRepository paymentRepository;
    private final PaymentMapper mapper;
    private final ExternalPaymentGateway gateway;
    
    @BpLogger
    public PaymentConfirmationDto processPayment(PaymentRequestDto request) {
        Payment payment = mapper.toEntity(request);
        Payment processed = paymentRepository.save(payment);
        PaymentConfirmationDto confirmation = gateway.confirm(processed);
        return mapper.toDto(confirmation);
    }
}
```

### 6.3 Repository con @BpLogger

```java
@Repository
@RequiredArgsConstructor
public class PaymentRepositoryImpl implements PaymentRepository {
    
    private final PaymentJpaRepository jpaRepository;
    private final PaymentMapper mapper;
    
    @BpLogger
    public Payment save(Payment payment) {
        PaymentEntity entity = mapper.toEntity(payment);
        PaymentEntity saved = jpaRepository.save(entity);
        return mapper.toDomain(saved);
    }
    
    @BpLogger
    public Optional<Payment> findById(String paymentId) {
        Optional<PaymentEntity> entity = jpaRepository.findById(paymentId);
        return entity.map(mapper::toDomain);
    }
}
```

---

## 7. OPCIONES DE CONFIGURACIÓN DE PAYLOAD

| Propiedad | Descripción | Valor por Defecto |
|-----------|-------------|-------------------|
| `trace-logger.payload.mode` | NONE: sin payload, FULL: completo con exclusiones, PARTIAL: solo campos incluidos | NONE |
| `trace-logger.payload.fallback-mode` | Comportamiento si no hay path: FULL o EMPTY | EMPTY |
| `trace-logger.payload.request.json-paths` | Rutas JSON a EXCLUIR en FULL o INCLUIR en PARTIAL | [] |
| `trace-logger.payload.request.xpaths` | Expresiones XPath para XML/SOAP (FULL o PARTIAL) | [] |
| `trace-logger.payload.response.json-paths` | Rutas JSON para response | [] |
| `trace-logger.payload.response.xpaths` | Expresiones XPath para response XML | [] |

---

## 8. OPCIONES DE CONFIGURACIÓN DE METADATA

| Propiedad | Descripción | Valor por Defecto | Requerido |
|-----------|-------------|-------------------|-----------|
| `trace-logger.metadata.enabled` | Habilita extracción de metadata dinámica | false | No |
| `trace-logger.metadata.fields` | Lista de campos a extraer como metadata | [] | No |
| `trace-logger.metadata.fields[].key` | Nombre de la clave en el mapa de metadata | - | **Sí** |
| `trace-logger.metadata.fields[].json-path` | JSONPath para extraer el valor (payloads JSON) | Opcional | No |
| `trace-logger.metadata.fields[].xpath` | XPath para extraer el valor (payloads XML) | Opcional | No |
| `trace-logger.metadata.fields[].source` | Origen: REQUEST o RESPONSE | REQUEST | No |

---

## 9. EJEMPLOS DE CONFIGURACIÓN COMPLETA

### Ejemplo 1: MODO FULL (Payload Completo con Exclusiones)

```yaml
trace-logger:
  enabled: true
  payload:
    mode: FULL
    fallback-mode: EMPTY
    request:
      json-paths:
        - "user.password"
        - "credentials.token"
        - "payment.creditCardNumber"
    response:
      json-paths:
        - "data.authToken"
        - "session.refreshToken"
```

**Resultado:** Envía el payload completo EXCEPTO los campos listados.

### Ejemplo 2: MODO PARTIAL (Solo Campos Específicos)

```yaml
trace-logger:
  enabled: true
  payload:
    mode: PARTIAL
    request:
      json-paths:
        - "user.id"
        - "user.name"
        - "transaction.amount"
    response:
      json-paths:
        - "status"
        - "confirmationNumber"
```

**Resultado:** Envía SOLO los campos listados.

### Ejemplo 3: MODO PARCIAL para XML/SOAP

```yaml
trace-logger:
  enabled: true
  payload:
    mode: PARTIAL
    request:
      xpaths:
        - "//soapenv:Header/security/username"
        - "//transaction/id"
    response:
      xpaths:
        - "//response/status"
        - "//response/confirmationCode"
```

### Ejemplo 4: Metadata Extraction

```yaml
trace-logger:
  metadata:
    enabled: true
    fields:
      - key: "transactionId"
        json-path: "data.transactionId"
        source: REQUEST
      - key: "userId"
        json-path: "user.id"
        source: REQUEST
      - key: "operationStatus"
        json-path: "response.status"
        source: RESPONSE
      - key: "processingTime"
        json-path: "metrics.duration"
        source: RESPONSE
```

---

## 10. GUÍA DE CAMPOS SENSIBLES A OFUSCAR

### Autenticación y Autorización
- Contraseñas (`password`, `pwd`)
- Tokens (`authToken`, `accessToken`, `refreshToken`, `sessionId`)
- Claves de API (`apiKey`, `secretKey`)
- Credenciales SOAP (`username`, `password` en header)

### Datos Financieros
- Número de tarjeta (`creditCardNumber`, `cardNumber`)
- Número de cuenta (`accountNumber`, `accountId`)
- Número de ruta (`routingNumber`)
- CVV/CVC

### Datos Personales (PII)
- Documento de identidad (`documentId`, `ruc`, `cedula`, `dni`)
- Email (`email`, `emailAddress`)
- Teléfono (`phone`, `phoneNumber`)
- Dirección (`address`, `homeAddress`)

### URLs y Endpoints
- URLs con credenciales incrustadas
- Connection strings con passwords

---

## 11. FLUJO DE LOGGING POR CAPA

```
┌─────────────────────────────────────┐
│      SOAP Controller                │
│    (SIN @BpLogger)                  │
└────────────┬────────────────────────┘
             │
             ↓
┌─────────────────────────────────────┐
│    Service (@BpLogger)              │
│  - LOG: entrada + salida            │
│  - CustomLogLevelHandler (errores)  │
└────────────┬────────────────────────┘
             │
             ↓
┌─────────────────────────────────────┐
│    Repository (@BpLogger)           │
│  - LOG: persistencia                │
│  - Query execution                  │
└────────────┬────────────────────────┘
             │
             ↓
        DATABASE
```

---

## 12. REQUERIMIENTO PLANTILLA

Para generar un servicio, completa:

```
## NUEVA FUNCIONALIDAD: [NOMBRE]

**Descripción:** [DESCRIBIR LA FUNCIONALIDAD]

**Servicios involucrados:** 
- [Servicio 1]
- [Servicio 2]

**DTOs necesarios:**
| DTO | Campos | Campos Sensibles |
|-----|--------|------------------|
| [NombreRequestDto] | [campo1, campo2] | [sensible1, sensible2] |
| [NombreResponseDto] | [campo3, campo4] | [sensible3] |

**Configuración YAML requerida:**
- Payload mode: [NONE/FULL/PARTIAL]
- Metadata fields: [listar si aplica]
- Campos a excluir: [listar]

**Casos de error a manejar:**
- [Error 1]
- [Error 2]

**Integración SOAP:** [Sí/No] - [Describir si aplica]
```

---

## 13. VERIFICACIÓN PRE-IMPLEMENTACIÓN

Antes de generar código, confirma:

- [ ] ¿Todos los campos PII tienen @BpObfuscatable?
- [ ] ¿@BpLogger solo está en Service, UseCase, Repository?
- [ ] ¿Máximo 20 líneas por método?
- [ ] ¿Máximo 5 parámetros?
- [ ] ¿Comentarios en INGLÉS?
- [ ] ¿application.yml tiene configuración de payload y metadata?
- [ ] ¿logback-spring.xml está configurado con Logstash?

---

## NOTAS IMPORTANTES

✅ **Todo debe estar en INGLÉS**  
✅ **@BpObfuscatable en TODO dato sensible**  
✅ **@BpLogger SOLO en application/service, repositorios y adapters**  
❌ **Sin @BpLogger en controllers**  
❌ **Sin CustomLogLevelHandler si no está habilitado en config**  
✅ **Payload mode FULL es recomendado para observabilidad + seguridad**  
✅ **Metadata dinamica para seguimiento de transacciones**  
