# Entregable de la práctica — Parte 2 y Parte 3

**Estudiante:** Johan Carvajal · **PFC:** ARTISYNC
**Endpoint trazado:** `GET /api/v1/pedidos/mis-pedidos` (listado, equivalente al
`GET /api/v1/productos` del ejemplo del docente)
**Paquete base:** `uteq.edu.ec.artisync`

---

# Parte 2 — Tabla de trazado de flujo

**PFC: ARTISYNC — Plataforma de comisiones de arte digital**

| # | Componente | Clase Java (nombre exacto) | Método (nombre exacto) | Paquete |
|---|---|---|---|---|
| 1 | Cliente Angular | `PedidoService` (TypeScript, no Java) | `listarMisPedidos()` | `Frontend/src/app/features/pedido/services` |
| 2 | Tomcat embebido | (Automático, no editable) | Gestionado por Spring Boot | Spring interno |
| 3 | DispatcherServlet | `DispatcherServlet` | `doDispatch()` | Spring interno |
| 4 | **Filtro JWT** | **`JwtAuthenticationFilter`** | **`doFilterInternal()`** | **`uteq.edu.ec.artisync.security`** |
| 5 | HandlerMapping | `RequestMappingHandlerMapping` | `getHandler()` | Spring interno |
| 6 | **Controlador** | **`PedidoControlador`** | **`listarMisPedidos()`** | **`uteq.edu.ec.artisync.controller.pedido`** |
| 7 | **Servicio** | **`PedidoServicioImpl`** (implementa `IPedidoServicio`) | **`listarMisPedidos()`** | **`uteq.edu.ec.artisync.service.pedido.impl`** |
| 8 | **Repositorio** | **`PedidoRepository`** (extiende `JpaRepository<Pedido, Long>`) | **`findByUsuarioClienteIdUsuario()`** | **`uteq.edu.ec.artisync.repository.pedido`** |
| 9 | Serialización JSON | `MappingJackson2HttpMessageConverter` | `write()` | Spring interno |

**Entidad y DTO asociados (capa M del patrón):**

| Elemento | Clase | Paquete | Tabla / uso |
|---|---|---|---|
| Entidad JPA | `Pedido` | `uteq.edu.ec.artisync.entity.pedido` | tabla `pedidos` |
| DTO de salida | `RespuestaPedidoResumido` | `uteq.edu.ec.artisync.dto.respuesta.pedido` | lo que viaja en el JSON |

**Nota sobre el paso 4:** el PDF llama `JwtAuthFilter` a la clase genérica. En ARTISYNC su
nombre real es **`JwtAuthenticationFilter`** y sí extiende `OncePerRequestFilter`, como indica
la pista del documento.

**Nota sobre el paso 7:** ARTISYNC separa interfaz e implementación. El controlador inyecta la
interfaz `IPedidoServicio`; quien tiene la anotación `@Service` y el código real es
`PedidoServicioImpl`.

---

# Parte 3 — Preguntas de análisis

## Pregunta 1

> *En el panel Frames del depurador, cuando la ejecución está dentro del `JwtAuthFilter`,
> aparecen clases de Spring y Tomcat encima de la suya. ¿Qué representa esa pila de llamadas?
> ¿En qué orden se ejecutaron esas clases?*

Esa pila es la **cadena de filtros de servlet** (`javax.servlet.Filter`) que Tomcat y Spring
Security ejecutan **antes** de que la petición llegue a Spring MVC. No es código que yo llame:
cada eslabón invoca al siguiente mediante `filterChain.doFilter(request, response)`, así que la
pila crece hacia abajo conforme la petición avanza.

El panel Frames se lee **de abajo hacia arriba** en orden cronológico: lo de más abajo se
ejecutó primero, y `doFilterInternal` de mi filtro es lo último (arriba del todo).

Orden real en ARTISYNC:

```
1. Conector NIO de Tomcat            ← recibe el TCP, parsea el HTTP
2. StandardWrapperValve              ← pipeline de Tomcat
3. ApplicationFilterChain            ← cadena de filtros del contenedor
4. FilterChainProxy                  ← el "delegado" único de Spring Security
5. FilterChainProxy$VirtualFilterChain
6.   CorsFilter                      ← .cors(Customizer.withDefaults())
7.   HeaderWriterFilter              ← CSP, HSTS, X-Frame-Options
8.   LoginRateLimitFilter            ← propio; sale enseguida (la URI no es /api/auth/login)
9.   JwtAuthenticationFilter         ← AQUÍ está detenido el depurador
```

El orden 8 → 9 no es casual: está declarado en `SecurityConfig.filterChain()`, líneas 76-77:

```java
.addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)
.addFilterBefore(loginRateLimitFilter, JwtAuthenticationFilter.class);
```

**La observación más importante:** en esa pila **todavía no aparece
`DispatcherServlet.doDispatch()`**. Eso demuestra que la seguridad es anterior a Spring MVC —
el `DispatcherServlet` es un *servlet*, y en la especificación de Servlet todos los filtros
corren antes que el servlet destino. Una petición sin token válido puede morir sin que Spring
MVC llegue a enterarse de que existió.

---

## Pregunta 2

> *Cuando el depurador entra en el servicio, encima del método del servicio aparecen clases como
> `TransactionInterceptor` y `CglibAopProxy`. ¿Qué está haciendo Spring en ese momento? ¿Por qué
> es importante que esas clases se ejecuten antes del método del servicio?*

Spring está **abriendo la transacción de base de datos**.

El controlador cree que llama a `PedidoServicioImpl`, pero en realidad el bean inyectado **no es
mi clase**: es una **subclase generada en tiempo de ejecución por CGLIB** que hereda de
`PedidoServicioImpl` y sobrescribe sus métodos públicos. (Spring Boot fuerza CGLIB por defecto
—`spring.aop.proxy-target-class=true`— incluso cuando la clase implementa una interfaz como
`IPedidoServicio`.) Por eso aparece `CglibAopProxy.intercept()` en el panel Frames.

Ese proxy ejecuta una cadena de interceptores antes de delegar en mi código. El relevante es
`TransactionInterceptor`, que:

1. Lee la anotación `@Transactional` de `PedidoServicioImpl` (línea 42).
2. Pide una conexión al pool (HikariCP) y le hace `setAutoCommit(false)`.
3. La ata al hilo actual mediante `TransactionSynchronizationManager`, para que **todos** los
   repositorios que se usen dentro del método compartan esa misma conexión.
4. Al terminar el método hace `commit()`; si sale una `RuntimeException`, hace `rollback()`.

**Por qué tiene que ejecutarse antes:** si el método empezara sin transacción abierta, cada
`save()` iría en su propia transacción implícita. En `crearPedido()` eso rompería la
atomicidad: se insertaría el registro en `pedidos` y, si fallara el `INSERT` en
`historial_estados_pedido`, quedaría un pedido huérfano sin estado inicial. El
`TransactionInterceptor` es lo que garantiza que ambos INSERT sean una sola unidad.

**Consecuencia práctica que se ve en el depurador:** como la transacción la abre el *proxy*, si
un método del servicio llama a otro método `@Transactional` de la misma clase (`this.otroMetodo()`),
la llamada **no pasa por el proxy** y la anotación se ignora silenciosamente. Es el error clásico
de auto-invocación en Spring AOP.

---

## Pregunta 3

> *Al entrar en el repositorio, el depurador muestra la clase `SimpleJpaRepository` que nunca fue
> escrita por el equipo. ¿Cómo existe esa clase en tiempo de ejecución si no está en el código
> fuente del PFC? ¿Qué mecanismo de Spring la genera?*

`PedidoRepository` es **una interfaz sin ninguna implementación** en el proyecto:

```java
@Repository
public interface PedidoRepository extends JpaRepository<Pedido, Long> {
    List<Pedido> findByUsuarioClienteIdUsuario(Long idUsuario);
    ...
}
```

El mecanismo es **Spring Data JPA**, que actúa al arrancar la aplicación, no al compilar:

1. La autoconfiguración de Spring Boot activa `@EnableJpaRepositories`, que escanea el proyecto
   buscando interfaces que extiendan `Repository` (o `JpaRepository`).
2. Por cada una, `JpaRepositoryFactoryBean` → `RepositoryFactorySupport` crea un **proxy dinámico
   de JDK** (`java.lang.reflect.Proxy`) que implementa esa interfaz. Ese proxy es el objeto que
   Spring inyecta en `PedidoServicioImpl`.
3. Cuando se invoca un método, el proxy decide a dónde enviarlo:
   - **Métodos heredados** (`save()`, `findById()`, `findAll()`, `delete()`) → los delega a una
     instancia de **`SimpleJpaRepository`**, que es la implementación por defecto que trae la
     librería `spring-data-jpa` en su JAR. Existe en el *classpath*, no en mi código fuente:
     por eso el depurador la muestra pero no puedo editarla.
   - **Métodos derivados del nombre** (`findByUsuarioClienteIdUsuario`) → los resuelve
     `PartTreeJpaQuery`, que parte el nombre del método (`findBy` + `UsuarioCliente` + `IdUsuario`),
     lo traduce a JPQL y lo ejecuta.

Es decir: `SimpleJpaRepository` no se *genera*, ya venía escrita dentro de Spring Data. Lo que sí
se genera dinámicamente es el **proxy** que conecta mi interfaz con ella. Por eso una interfaz
vacía funciona sin que yo escriba una sola línea de SQL.

---

## Pregunta 4

> *Si se envía la solicitud GET **sin** el encabezado `Authorization: Bearer [token]`, ¿qué paso
> del flujo detiene la solicitud y qué código HTTP devuelve? ¿En qué clase Java del proyecto
> ocurre esa decisión?*

Se detiene en el **paso 4 (la cadena de filtros de seguridad)** y devuelve **`401 Unauthorized`**.

Pero hay un matiz que sólo se ve con el depurador, y es lo interesante de la pregunta:
**`JwtAuthenticationFilter` NO es quien la bloquea.** Al no haber cabecera, sale por su primera
comprobación y deja pasar la petición sin autenticar (líneas 38-41):

```java
String authHeader = request.getHeader("Authorization");
if (authHeader == null || !authHeader.startsWith("Bearer ")) {
    filterChain.doFilter(request, response);   // ← sigue, pero sin autenticación
    return;
}
```

El corte real ocurre después, en tres tiempos:

| Orden | Clase | Qué hace |
|---|---|---|
| 1 | `AuthorizationFilter` (Spring Security) | Evalúa la regla `.anyRequest().authenticated()` de `SecurityConfig` (línea 74). El `SecurityContext` está vacío → lanza `AccessDeniedException` |
| 2 | `ExceptionTranslationFilter` (Spring Security) | Ve que el usuario es anónimo, así que no responde `403` sino que delega en el `AuthenticationEntryPoint` |
| 3 | **`CustomAuthenticationEntryPoint`** ← *clase propia del PFC* | Escribe la respuesta `401` |

**La clase del proyecto donde ocurre la decisión final es
`uteq.edu.ec.artisync.security.CustomAuthenticationEntryPoint`**, método `commence()`,
registrada en `SecurityConfig` línea 62:

```java
response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);   // 401
ProblemDetail problemDetail = ProblemDetail.forStatusAndDetail(HttpStatus.UNAUTHORIZED, message);
problemDetail.setType(URI.create("https://artisync.dev/errors/autenticacion"));
objectMapper.writeValue(response.getWriter(), problemDetail);
```

Respuesta que recibe Postman:

```json
{
  "type": "https://artisync.dev/errors/autenticacion",
  "title": "Unauthorized",
  "status": 401,
  "detail": "Autenticación requerida",
  "instance": "/api/v1/pedidos/mis-pedidos"
}
```

**Dos detalles propios de ARTISYNC:**

- El campo `detail` es más específico si el token existía pero falló: `JwtAuthenticationFilter`
  guarda el motivo en `request.setAttribute("JWT_ERROR", ...)` (`"Token expirado"`,
  `"Token revocado u obsoleto"`) y el *entry point* lo reutiliza. Sin cabecera, el mensaje
  genérico es `"Autenticación requerida"`.
- **No todos los GET dan 401.** Rutas públicas como `GET /api/v1/servicios/**` están en
  `.permitAll()` (línea 66-68) y responden `200` sin token. El `401` sólo aparece en rutas que
  caen en `.anyRequest().authenticated()`, como `/api/v1/pedidos/**`.

Y si el token es válido pero el **rol** no alcanza, el flujo es distinto: llega hasta
`@PreAuthorize`, lanza `AccessDeniedException` con usuario ya autenticado y el
`ManejadorGlobalExcepciones` (línea 100) devuelve **`403`**.

---

## Pregunta 5

> *Observando el SQL que Hibernate genera en la consola para el endpoint GET de listado:
> ¿Hibernate genera un SELECT con todos los campos de la tabla o sólo los que el DTO necesita?
> ¿Qué implicaciones de rendimiento tiene esto cuando la tabla tiene 50 columnas?*

**Hibernate trae TODOS los campos de la entidad, no los del DTO.** Hibernate no sabe que el DTO
existe: `findByUsuarioClienteIdUsuario()` está declarado como `List<Pedido>`, así que hidrata la
entidad `Pedido` completa. El recorte al DTO ocurre después, ya en memoria Java, dentro de
`mapToResumido()`.

SQL real del listado en ARTISYNC:

```sql
select p1_0.id_pedido, p1_0.fecha_entrega_estimada, p1_0.fecha_inicio,
       p1_0.id_flujo, p1_0.id_servicio, p1_0.id_usuario_cliente, p1_0.precio_pactado
from pedidos p1_0
where p1_0.id_usuario_cliente = ?
```

Trae las 7 columnas de la entidad aunque `RespuestaPedidoResumido` sólo publique parte de ellas.

### Implicaciones con una tabla de 50 columnas

1. **Sobre-lectura (*over-fetching*).** Si el DTO necesita 5 campos y la tabla tiene 50, se leen
   45 de más en cada fila. Con 1.000 filas son 45.000 valores transferidos por red, deserializados
   por el driver JDBC y guardados en objetos Java que nunca se usan.
2. **Presión de memoria y GC.** El *persistence context* mantiene una referencia a cada entidad
   cargada **y** una copia del estado original para detectar cambios (*dirty checking*). Con 50
   columnas eso duplica el coste por fila.
3. **Índices inútiles.** Un `SELECT *` no puede resolverse con un índice de cobertura; PostgreSQL
   se ve obligado a ir a la tabla (*heap fetch*) fila por fila.
4. **Columnas pesadas.** Si entre las 50 hay un `TEXT` o un `BYTEA` (una descripción larga, una
   imagen en base64), se transfiere íntegro en cada listado aunque nadie lo muestre.

### El problema que sí tiene ARTISYNC ahora mismo: N+1

Más grave que el ancho del SELECT es el **número** de consultas. `mapToResumido()` navega
relaciones `LAZY`:

```java
Usuario creador = pedido.getServicio().getPerfil().getUsuario();   // 3 SELECT extra por pedido
...
.etapaActual(obtenerEtapaActual(pedido.getIdPedido()))             // 1 SELECT más por pedido
```

Con 20 pedidos en la lista, eso son **1 consulta inicial + hasta 80 consultas adicionales**.
Ese es el patrón N+1, y en un listado real pesa mucho más que las columnas sobrantes.

### Cómo se corrige

| Técnica | Qué resuelve |
|---|---|
| **Proyección por constructor** — `@Query("select new ...RespuestaPedidoResumido(p.idPedido, ...) from Pedido p")` | Genera un SELECT con **sólo** las columnas del DTO |
| **Proyección por interfaz** de Spring Data (`interface PedidoResumen { Long getIdPedido(); ... }`) | Igual, sin escribir JPQL |
| **`@EntityGraph`** o `join fetch` | Elimina el N+1 trayendo las relaciones en una sola consulta |
| **`@Query` con `join fetch`** en `PedidoRepository` | Combina ambas cosas |

Aplicado a este caso, la corrección natural sería añadir a `PedidoRepository` una consulta con
`join fetch` sobre `servicio`, `perfil` y `usuarioCliente`, de forma que el listado pase de ~81
consultas a 1.

---

## Nota sobre el entorno (diferencias con el guion del docente)

| El PDF dice | En ARTISYNC |
|---|---|
| Spring Boot 3 | Spring Boot **4.1.0**, Java 21 — el flujo de 9 pasos es idéntico |
| `application.yml` | `application.properties`; el log de SQL se activa con `spring.jpa.show-sql=true` y `spring.jpa.properties.hibernate.format_sql=true` |
| `GET /api/v1/productos` | `GET /api/v1/pedidos/mis-pedidos` |
| Paquete `ec.edu.uteq.[proyecto]` | `uteq.edu.ec.artisync` |
| `backend/` | `artisync/Backend/` |
| Basta arrancar la app | Hace falta `docker compose up -d db redis` y las variables `JWT_SECRET`, `DB_URL`, `DB_APP_USER`, `DB_APP_PASSWORD` |
