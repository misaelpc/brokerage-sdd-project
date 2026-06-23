---
applyTo: "**"
---

# Estándares de Codificación y Guías de DDD para el Proyecto

## Principios de Domain-Driven Design (DDD)

Sigue estrictamente los principios de Domain-Driven Design (DDD) para modelar el dominio de negocio de manera efectiva. Usa un lenguaje ubicuo claro y enfócate en el contexto acotado (Bounded Context).

### Reglas Clave de Vaughn Vernon (Implementing Domain-Driven Design)

- **Modela invariantes verdaderos en límites de consistencia:** Identifica y protege las invariantes del dominio dentro de agregados, asegurando consistencia transaccional solo en ese límite.
- **Diseña agregados pequeños:** Mantén los agregados lo más pequeños posible para minimizar complejidad y mejorar rendimiento; un agregado debe caber en una sola transacción.
- **Referencia otros agregados por identidad:** Usa IDs (identidades únicas) para referenciar agregados externos, evitando acoplamiento fuerte y permitiendo eventual consistency.
- **Usa consistencia eventual fuera del límite:** Fuera de un agregado, asume consistencia eventual; no fuerces transacciones distribuidas a menos que sea estrictamente necesario.

#### Excepciones Prácticas a las Reglas

Rompe estas reglas solo con justificación documentada (e.g., en comentarios o ADR). Razones válidas incluyen:

- **Conveniencia de la Interfaz de Usuario:** Para mejorar UX sin impacto significativo en el dominio (e.g., denormalizar para UI rápida).
- **Falta de Mecanismos Técnicos:** Cuando la infra no soporta eventual consistency (e.g., legacy systems sin eventos).
- **Transacciones Globales:** En escenarios críticos donde la consistencia fuerte es exigida por regulación (e.g., finanzas con ACID total).
- **Rendimiento de Consultas:** Para queries read-heavy, usa CQRS con vistas denormalizadas, pero mantén el modelo de escritura puro.

### Conceptos Core de DDD

- **Lenguaje Ubicuo:** Todo el código debe reflejar el lenguaje del dominio. Usa nombres descriptivos para entidades, objetos de valor, agregados y servicios que mapeen directamente al negocio.
- **Agregados:** Diseña con límites claros; solo el root del agregado interactúa externamente. Aplica las reglas de Vernon aquí.
- **Entidades:** Tienen identidad única y encapsulan comportamiento relacionado con su ciclo de vida.
- **Objetos de Valor:** Inmutables, comparados por valor; úsalos para medidas, cantidades o descripciones (e.g., `Money`, `Address`).
- **Servicios de Dominio:** Para operaciones que no pertenecen a una entidad u objeto de valor.
- **Repositorios:** Interfaz collection-like para persistir/recuperar agregados; implementa en la capa exterior.
- **Manejo de Nulls (Null Object Pattern):** Nunca devuelvas `null` para entidades o componentes del dominio que requieran un comportamiento por defecto. Usa el **Null Object Pattern** para representar la ausencia con un objeto válido (e.g., retorna `NullCustomer` en lugar de `null`).
- **Manejo de Nulls (Optional):** Para valores que pueden estar verdaderamente ausentes (por ejemplo, resultados de búsquedas o propiedades opcionales), utiliza `Optional.empty()` en lugar de `null` y gestiona su presencia con métodos como `orElse` o `ifPresent`.
- **Factory Methods:** Favorece factory methods estáticos sobre constructores en aggregate roots, entidades y value objects para expresar el ubiquitous language (e.g., `OrderAggregate.createFromCustomerAndItems(...)` en lugar de `new OrderAggregate(...)`).

## Convenciones de Nomenclatura

- **Clases/Interfaces:** PascalCase (e.g., `CustomerAggregate`, `OrderService`).
- **Métodos/Propiedades:** camelCase (e.g., `createOrder()`, `customerName`).
- **Constantes:** UPPER_CASE con underscores (e.g., `MAX_ORDER_ITEMS`).
- **Paquetes:** Usa nombres en minúsculas con puntos (e.g., `com.proyecto.domain.model`).

## Estructura de Código y Paquetes

Organiza el código en capas DDD estrictas: **Interior** (Domain + Application, independientes de frameworks) y **Exterior** (Infrastructure + Presentation, adaptadores). No mezcles responsabilidades entre capas.

### Capa Interior

- **application**: Servicios de aplicación que orquestan operaciones de negocio (e.g., recuperan entidades via repositorio, ejecutan lógica, publican eventos, persisten cambios).
  - **data**: DTOs compartidos con el exterior (e.g., respuestas de API).
  - **command**: Comandos para acciones en el dominio (e.g., `CreateOrderCommand`).
- **domain**: Modelo de dominio puro, clasificado por tipo.
  - **model**: Agregados, entidades y objetos de valor (e.g., `OrderAggregate`, `CustomerEntity`, `MoneyValueObject`).
  - **event**: Eventos de dominio (e.g., `OrderCreatedEvent`).
  - **repository**: Interfaces de repositorios (e.g., `OrderRepository`).

### Capa Exterior

- **infra**: Implementaciones técnicas y adaptadores.
  - **controller**: Controladores REST (e.g., `OrderController`).
    - **model**: Contratos de entrada (e.g., request/response DTOs).
    - **media**: Hipermedia/HATEOAS (e.g., enlaces en respuestas).
    - **problem**: Manejo de errores (e.g., traducir exceptions a HTTP status).
  - **event**: Publicadores de eventos (e.g., via Kafka/JMS).
  - **messaging**: Consumidores de mensajes entrantes (e.g., handlers para comandos via queue).
  - **persistence**: Implementaciones de repositorios (e.g., JPA/Hibernate).
  - **service**: Clientes para servicios externos (REST/SOAP/gRPC).
    - **model**: DTOs para intercambio con externos.

Ejemplo de estructura de paquetes en IntelliJ:

```
src/
├── main/
│   └── java/com/proyecto/
│       ├── application/
│       │   ├── data/
│       │   └── command/
│       ├── domain/
│       │   ├── model/
│       │   ├── event/
│       │   └── repository/
│       └── infra/
│           ├── controller/
│           │   ├── model/
│           │   ├── media/
│           │   └── problem/
│           ├── event/
│           ├── messaging/
│           ├── persistence/
│           └── service/
│               └── model/
```

## Best Practices

- **Fail Fast:** Valida precondiciones (guards) al inicio de métodos para fallar temprano y evitar indentación profunda con if/else anidados. Usa exceptions descriptivas en ubiquitous language (e.g., mensajes que expliquen la regla de negocio violada). Prefiere exceptions estándar para consistencia con handlers existentes; crea customs solo si necesitas atributos extra o para reforzar el lenguaje ubicuo.
  - **IllegalArgumentException:** Para violaciones de invariantes o reglas de negocio (e.g., input inválido). Equivale a HTTP 400.
  - **IllegalStateException:** Para inconsistencias en el estado de un agregado (e.g., operación no permitida en estado actual). Equivale a HTTP 400.
  - **NoSuchElementException:** Para elementos no encontrados (e.g., aggregate no existe). Equivale a HTTP 404.
  - **Custom Exceptions:** Solo cuando requieras info adicional (e.g., campos para logging) o para expresar dominio único (e.g., `OrderCannotBeCancelledException`). Evita inflar handlers; fallback a RuntimeException para HTTP 500.

## Prácticas de Continuous Integration (CI)

Favorece la integración constante para mantener el código estable y deployable. Integra commits frecuentes (mínimo daily) y usa pipelines para builds/tests automáticos.

- **Branch by Abstraction:** Para refactorings grandes, crea una abstracción (interface) sobre el código legacy y migra gradualmente. Sugiere interfaces en domain cuando se evolucione un aggregate.
- **Hidden Features:** Desarrolla features completas pero invisibles (e.g., vía config flags); integra sin exponer hasta QA. Usa comentarios como `// Hidden until v2.0` para marcar.
- **Feature Toggles:** Implementa toggles para encender/apagar features dinámicamente (e.g., `@FeatureToggle("new-order-flow")` en métodos de app service). Limpia toggles obsoletos en cada release.

## Testing

- **Nombres de pruebas en camelCase:** Los métodos de prueba deben usar camelCase, sin guiones bajos. Ejemplo: `obtenerProductoExistenteRetornaDetalleCompleto` en lugar de `obtenerProducto_existente_retornaDetalleCompleto`.
- **Unit Tests:** Enfócate en lógica de dominio (agregados, entidades, transiciones de estado). Usa mocks mínimos; prueba comportamiento, no implementación.
- **Integration Tests:** Para interacciones de repositorios, controladores y servicios de aplicación. Cubre flujos end-to-end en capas.
- **Estructura BDD (Given-When-Then):** Estructura todos los tests con:
  - **Given:** 0 o más precondiciones/setup (e.g., fixtures o mocks).
  - **When:** Exactamente 1 acción bajo prueba (e.g., método del aggregate).
  - **Then:** Al menos 1 validación (e.g., assertions en estado o eventos emitidos).
- **Don't Repeat Yourself (DRY):** Evita redundancias en la pirámide de tests.
  - No dupliques escenarios de persistencia ya cubiertos en tests de repositorio dentro de los tests de servicios.
  - En su lugar, asegúrate de que los tests de integración a nivel de servicio verifiquen aspectos específicos de ese nivel, como límites transaccionales, lógica de orquestación y side effects, en vez de solo repetir pruebas básicas de persistencia.
- **Assertions obligatorias:** Todo método `@Test` DEBE incluir al menos una llamada a JUnit `assert*()` o `fail()`. Las verificaciones de Mockito (`verify()`, `solMock.verify()`) NO cuentan como assertions para análisis estático. Cuando el test valida que no se lance excepción, usa `assertDoesNotThrow()`; cuando valida estado, usa `assertEquals()`, `assertTrue()`, etc.
- **Herramientas:** JUnit 5, Mockito, Testcontainers para DBs externas. Usa `@DisplayName` para describir el escenario en lenguaje ubicuo.

## Security

- Valida inputs en controladores y servicios de aplicación.
- Aplica controles de acceso (e.g., roles en agregados via domain services).
- No expongas datos sensibles (e.g., usa masking en logs/respuestas).
- Sigue OWASP: sanitiza queries, usa HTTPS, maneja secrets via Vault o env vars.

## Consejos para IntelliJ/Copilot

- Siempre sugiere código que respete la separación de capas: nada de infra en domain.
- Al generar, prioriza inmutabilidad y encapsulación.
- Si hay ambigüedad, pregunta por el Bounded Context o invariantes específicas.
