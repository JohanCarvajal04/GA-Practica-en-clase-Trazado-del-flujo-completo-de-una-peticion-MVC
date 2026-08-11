# 02 — Puntos de ruptura en IntelliJ IDEA

Guía para reproducir en vivo el trazado de `POST /api/v1/pedidos` con el depurador,
tal como lo mostró el docente en el proyector (directrices 2 y 3 de la práctica).

---

## Preparación

1. Abrir `artisync/Backend` como proyecto Maven en IntelliJ IDEA.
2. Levantar la infraestructura desde la raíz del PFC:

```bash
docker compose up -d db redis
```

3. En la configuración de ejecución de `ArtisyncApplication`, definir las variables de entorno:

```
JWT_SECRET=<clave de 256 bits en base64>
DB_URL=jdbc:postgresql://localhost:5432/artisyncbd
DB_APP_USER=artisync_app
DB_APP_PASSWORD=12345
```

4. Para ver el SQL real en la consola durante el trazado, cambiar temporalmente en
   `src/main/resources/application.properties`:

```properties
spring.jpa.show-sql=true
```

5. Arrancar en modo **Debug** (`Shift + F9`).
6. Levantar el frontend (`cd artisync/Frontend && npm start`) e iniciar sesión con un usuario
   de rol `CLIENTE`.

---

## Los 6 puntos de ruptura

| # | Archivo | Línea | Qué demuestra |
|---|---|---|---|
| BP-1 | `security/JwtAuthenticationFilter.java` | 32 (`doFilterInternal`) | La petición pasa por los filtros **antes** que por el controlador |
| BP-2 | `security/JwtAuthenticationFilter.java` | 81 (`setAuthentication`) | El momento exacto en que el usuario queda autenticado |
| BP-3 | `security/CustomUserDetailsService.java` | 32 (`loadUserByUsername`) | Primer acceso a PostgreSQL, antes del controlador |
| BP-4 | `controller/pedido/PedidoControlador.java` | 31 (`crearPedido`) | Entrada al `@RestController` con el DTO ya deserializado y validado |
| BP-5 | `service/pedido/impl/PedidoServicioImpl.java` | 43 (`crearPedido`) | Capa `@Service`: reglas de negocio y transacción |
| BP-6 | `service/pedido/impl/PedidoServicioImpl.java` | 261 (`mapToRespuesta`) | Conversión entidad → DTO justo antes de la serialización JSON |

> En IntelliJ, colocar un punto de ruptura es hacer clic en el margen izquierdo junto al
> número de línea (o `Ctrl + F8` con el cursor en la línea).

---

## Qué inspeccionar en cada parada

### BP-1 — `JwtAuthenticationFilter.doFilterInternal` (línea 32)

En la ventana **Variables**:

| Expresión | Valor esperado |
|---|---|
| `request.getRequestURI()` | `/api/v1/pedidos` |
| `request.getMethod()` | `POST` |
| `request.getHeader("Authorization")` | `Bearer eyJhbGciOiJIUzI1NiJ9...` |
| `SecurityContextHolder.getContext().getAuthentication()` | `null` ← **todavía nadie está autenticado** |

En la ventana **Frames** se ve la pila completa de filtros de servlet por debajo:
`FilterChainProxy` → `OncePerRequestFilter.doFilter` → `doFilterInternal`. Bajando por los
frames se comprueba que `DispatcherServlet.doDispatch` **aún no aparece**: la seguridad se
ejecuta antes que Spring MVC.

Expresión útil para *Evaluate Expression* (`Alt + F8`):

```java
jwtService.extraerTodosLosClaims(request.getHeader("Authorization").substring(7))
```

### BP-2 — `SecurityContextHolder...setAuthentication` (línea 81)

| Expresión | Qué observar |
|---|---|
| `userDetails` | Instancia de `CustomUserDetails` |
| `((CustomUserDetails) userDetails).getIdUsuario()` | El ID que el controlador usará después |
| `userDetails.getAuthorities()` | `[ROLE_CLIENTE, VER_PEDIDOS, ...]` |
| `expiration` | Fecha de caducidad del token |

Ejecutar la línea con `F8` y volver a evaluar `SecurityContextHolder.getContext().getAuthentication()`:
ahora ya **no** es `null`. Ese es el instante en que la petición pasa de anónima a autenticada.

### BP-3 — `CustomUserDetailsService.loadUserByUsername` (línea 32)

Se entra aquí **desde** BP-2 (`F7`, *Step Into*, sobre la llamada de la línea 71).

- `correo` → el `sub` del JWT.
- Al pasar por `usuarioRepository.findByCorreo(correo)`, mirar la consola: aparece el
  `select ... from usuarios where correo=?`.
- `rolesUsuario` → lista de `UsuarioRol` traída de la tabla `usuarios_roles`.
- `authoritiesSet` → conjunto con los `ROLE_*` y los permisos ya normalizados a mayúsculas.

Punto didáctico: **antes de tocar el controlador ya hubo 2 consultas a PostgreSQL.**

### BP-4 — `PedidoControlador.crearPedido` (línea 31)

| Expresión | Qué observar |
|---|---|
| `userDetails.getIdUsuario()` | Mismo ID visto en BP-2 → la identidad viajó por el `SecurityContext` |
| `peticion.getIdServicio()` | Ya es un `Long` de Java: Jackson deserializó el JSON |
| `peticion.getPrecioOfrecido()` | Ya es un `BigDecimal` |

En **Frames** ahora sí aparece la pila de Spring MVC:

```
crearPedido:31, PedidoControlador
invoke:-1, ...            ← reflexión
invokeAndHandle:..., ServletInvocableHandlerMethod
invokeHandlerMethod:..., RequestMappingHandlerAdapter
doDispatch:..., DispatcherServlet          ← el DispatcherServlet
doFilterInternal:81, JwtAuthenticationFilter  ← nuestro filtro, más abajo en la pila
```

Esa pila es la prueba visual del orden `filtros → DispatcherServlet → HandlerAdapter → controlador`.

> **Prueba complementaria de `@PreAuthorize`:** repetir la petición con un usuario de rol
> `CREADOR`. El depurador **no se detiene** en BP-4 — la autorización lo cortó antes — y el
> cliente recibe `403`.

### BP-5 — `PedidoServicioImpl.crearPedido` (línea 43)

Es la parada más rica. Avanzar con `F8` línea a línea y observar:

| Línea | Qué ocurre |
|---|---|
| 44 | `usuarioRepository.findById(idCliente)` → SELECT sobre `usuarios` |
| 47 | `servicioRepository.findById(...)` → SELECT sobre `servicios` |
| 52 | Regla de negocio: comparación `creador == cliente` |
| 57 | `flujoTrabajoRepository.findAll()` → SELECT sobre `flujos_trabajo` |
| 64-65 | `findByFlujoIdFlujoOrderByNumeroOrdenAsc` → query derivada del nombre del método |
| 70-78 | Construcción de la entidad `Pedido` con el patrón *builder* |
| 80 | `pedidoRepository.save(pedido)` → **INSERT en `pedidos`** |
| 83-88 | `historialRepository.save(...)` → **INSERT en `historial_estados_pedido`** |

Comprobaciones interesantes:

- Justo **antes** de la línea 80, `pedido.getIdPedido()` es `null`.
  Justo **después**, tiene el valor generado por la secuencia de PostgreSQL (`IDENTITY`).
- Evaluar `TransactionSynchronizationManager.isActualTransactionActive()` → `true`.
  Confirma que `@Transactional` abrió la transacción al entrar al método.
- Evaluar `servicio.getPerfil()` obliga a Hibernate a resolver la relación `LAZY` y
  dispara un SELECT adicional visible en la consola.

**Demostración del rollback:** poner un punto de ruptura condicional en la línea 88 y, con
*Evaluate Expression*, lanzar `throw new RuntimeException("prueba")` con *Force Return*.
Al consultar `SELECT * FROM pedidos ORDER BY id_pedido DESC LIMIT 1` en PostgreSQL, el
pedido **no está**: `@Transactional` revirtió también el primer INSERT.

### BP-6 — `PedidoServicioImpl.mapToRespuesta` (línea 261)

| Expresión | Qué observar |
|---|---|
| `pedido` | Entidad JPA gestionada, con relaciones `LAZY` |
| `pedido.getServicio().getPerfil().getUsuario()` | Se resuelve porque **seguimos dentro de la transacción** |
| El objeto `RespuestaPedido` construido | DTO plano: sin `contrasenaHash`, sin proxies de Hibernate |

Punto didáctico clave del PFC: con `spring.jpa.open-in-view=false`, si este mapeo se hiciera
en el controlador (fuera del `@Transactional`) se lanzaría `LazyInitializationException`.
Por eso la conversión vive en el servicio.

Al pulsar `F9` (*Resume*) por última vez, Jackson serializa el DTO y el navegador recibe el
`201 Created`.

---

## Verificación en PostgreSQL

Tras completar el flujo, comprobar la escritura real:

```bash
docker compose exec db psql -U postgres -d artisyncbd -c "SELECT id_pedido, id_usuario_cliente, id_servicio, precio_pactado, fecha_inicio FROM pedidos ORDER BY id_pedido DESC LIMIT 3;"
```

```bash
docker compose exec db psql -U postgres -d artisyncbd -c "SELECT * FROM historial_estados_pedido ORDER BY id_historial DESC LIMIT 3;"
```

---

## Atajos de IntelliJ usados

| Atajo | Acción |
|---|---|
| `Ctrl + F8` | Alternar punto de ruptura en la línea actual |
| `Shift + F9` | Ejecutar en modo Debug |
| `F8` | *Step Over* — siguiente línea |
| `F7` | *Step Into* — entrar al método |
| `Shift + F8` | *Step Out* — salir del método |
| `F9` | *Resume* — continuar hasta el siguiente punto de ruptura |
| `Alt + F8` | *Evaluate Expression* |
| `Ctrl + Shift + F8` | Ver y editar todos los puntos de ruptura (condiciones, filtros) |
| `Alt + F9` | *Run to Cursor* |
