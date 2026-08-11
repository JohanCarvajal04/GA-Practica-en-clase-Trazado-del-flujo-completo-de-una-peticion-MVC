# 01 — Trazado paso a paso del flujo MVC en ARTISYNC

**Petición trazada:** `POST /api/v1/pedidos`
**Actor:** usuario autenticado con rol `CLIENTE`
**Respuesta esperada:** `201 Created` + `RespuestaPedido` en JSON

Todas las rutas Java son relativas a
`artisync/Backend/src/main/java/uteq/edu/ec/artisync/`.
Las rutas TypeScript son relativas a `artisync/Frontend/src/`.

---

## Paso 0 — Origen: el componente Angular

**Archivo:** `app/features/pedido/pages/pedido-crear/pedido-crear.component.ts`
**Método:** `onSubmit()` (línea 26)

El usuario envía el formulario. El componente valida que exista `idServicio` y delega en el servicio.

```typescript
onSubmit(): void {
  if (!this.pedido.idServicio) {
    this.error = 'Debes indicar el ID del servicio';
    return;
  }
  this.loading = true;
  this.error = '';

  this.pedidoService.crearPedido(this.pedido).subscribe({
    next: (res) => {
      this.loading = false;
      this.router.navigate(['/pedido', res.idPedido]);
    },
    error: (err) => { ... }
  });
}
```

---

## Paso 1 — Emisión de la petición HTTP

**Archivo:** `app/features/pedido/services/pedido.service.ts`
**Método:** `crearPedido(peticion: PeticionCrearPedido)` (línea 21)

```typescript
private readonly API = `${environment.apiUrl}/v1/pedidos`;   // '/api/v1/pedidos'

crearPedido(peticion: PeticionCrearPedido): Observable<RespuestaPedido> {
  return this.http.post<RespuestaPedido>(this.API, peticion);
}
```

`environment.apiUrl` vale `'/api'` (`src/environments/environment.ts`), y `proxy.conf.json`
redirige `/api` hacia `http://localhost:8080`. Por eso el navegador no hace CORS en desarrollo.

---

## Paso 2 — Interceptor: se adjunta el JWT

**Archivo:** `app/core/interceptors/auth.interceptor.ts`
**Función:** `authInterceptor` (interceptor funcional registrado en `app/app.config.ts`)

```typescript
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);
  const token = authService.accessToken();

  if (!token || req.url.includes('/auth/refresh') || req.url.includes('/auth/login')
      || req.url.includes('/auth/registro')) {
    return next(req);
  }

  const authReq = req.clone({
    setHeaders: { Authorization: `Bearer ${token}` }
  });

  return next(authReq);
};
```

Aquí nace la cabecera `Authorization: Bearer eyJhbGciOi...` que el backend leerá en el paso 5.

Registro de los interceptores — `app/app.config.ts`:

```typescript
provideHttpClient(
  withFetch(),
  withInterceptors([authInterceptor, errorInterceptor])
)
```

---

## Paso 3 — Entrada al contenedor: Tomcat y el DispatcherServlet

**Clase:** `org.springframework.web.servlet.DispatcherServlet` (framework, no es código propio)
**Punto de entrada del proyecto:** `ArtisyncApplication.java` → `main()`

```java
@SpringBootApplication
@EnableAsync
@EnableScheduling
public class ArtisyncApplication {
    public static void main(String[] args) {
        SpringApplication.run(ArtisyncApplication.class, args);
    }
}
```

`@SpringBootApplication` activa la autoconfiguración de Spring MVC, que registra el
`DispatcherServlet` en `/` sobre el Tomcat embebido (puerto `8080`, definido en
`application.properties` con `server.port=8080`).

> **Importante:** el `DispatcherServlet` **todavía no procesa nada**. Antes de él se ejecuta
> la cadena de filtros de servlet (paso 4), porque `FilterChainProxy` de Spring Security
> está registrado como filtro del contenedor, no como parte del `DispatcherServlet`.

---

## Paso 4 — Cadena de filtros de Spring Security

**Archivo:** `config/SecurityConfig.java`
**Método:** `filterChain(HttpSecurity http)` (línea 40)

Aquí se declara el orden real de los filtros del proyecto:

```java
.sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
.exceptionHandling(ex -> ex.authenticationEntryPoint(customAuthenticationEntryPoint))
.authorizeHttpRequests(auth -> auth
    .requestMatchers("/api/auth/registro", "/api/auth/login", ...).permitAll()
    .requestMatchers(HttpMethod.GET, "/api/v1/servicios/**", ...).permitAll()
    ...
    .anyRequest().authenticated()          // ← línea 74: /api/v1/pedidos cae aquí
)
.addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)   // línea 76
.addFilterBefore(loginRateLimitFilter, JwtAuthenticationFilter.class);                  // línea 77
```

Orden efectivo para esta petición:

```
CorsFilter → ... → LoginRateLimitFilter → JwtAuthenticationFilter
→ UsernamePasswordAuthenticationFilter (no actúa) → ... → AuthorizationFilter
```

### 4.a — `LoginRateLimitFilter` (se salta)

**Archivo:** `security/LoginRateLimitFilter.java`
**Método:** `doFilterInternal(...)` (línea 35)

```java
if (!"POST".equalsIgnoreCase(request.getMethod()) || !RUTA_LOGIN.equals(request.getRequestURI())) {
    // RUTA_LOGIN = "/api/auth/login"  →  esta petición no es login: pasa de largo
```

Como la URI es `/api/v1/pedidos`, el filtro delega inmediatamente en el siguiente.

---

## Paso 5 — Validación del JWT (el filtro clave del PFC)

**Archivo:** `security/JwtAuthenticationFilter.java`
**Método:** `doFilterInternal(HttpServletRequest, HttpServletResponse, FilterChain)` (línea 32)
**Clase padre:** `OncePerRequestFilter`

Este es el `JwtAuthFilter` que pide la directriz (3) de la práctica. Su nombre real en
ARTISYNC es **`JwtAuthenticationFilter`**. Hace cuatro cosas en orden:

**1) Lee la cabecera y aborta si no hay token:**

```java
String authHeader = request.getHeader("Authorization");
if (authHeader == null || !authHeader.startsWith("Bearer ")) {
    filterChain.doFilter(request, response);
    return;
}
String token = authHeader.substring(7);
```

**2) Extrae los claims y rechaza los *refresh tokens* usados como acceso:**

```java
Claims claims = jwtService.extraerTodosLosClaims(token);   // → security/JwtService.java
if ("refresh".equals(claims.get("type"))) { ... return; }
```

**3) Consulta la *blacklist* de JTI en Redis (política *fail-closed*):**

```java
String jti = claims.getId();
if (jti != null && Boolean.TRUE.equals(redisTemplate.hasKey("jti:" + jti))) {
    request.setAttribute("JWT_ERROR", "Token revocado u obsoleto");
    filterChain.doFilter(request, response);
    return;
}
```

Si Redis no responde, la petición se corta con `503` en vez de dejar pasar el token.

**4) Carga el usuario y puebla el `SecurityContext`** (línea 71 y línea 81):

```java
UserDetails userDetails = userDetailsService.loadUserByUsername(username);

if (username.equals(userDetails.getUsername()) && expiration.after(new Date())) {
    UsernamePasswordAuthenticationToken authToken =
        new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
    authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
    SecurityContextHolder.getContext().setAuthentication(authToken);   // ← línea 81
}
```

### 5.a — Primer viaje a PostgreSQL (dentro del filtro)

**Archivo:** `security/CustomUserDetailsService.java`
**Método:** `loadUserByUsername(String correo)` (línea 32)

```java
Usuario usuario = usuarioRepository.findByCorreo(correo)
        .orElseThrow(() -> new UsernameNotFoundException("Usuario no encontrado con correo: " + correo));

List<UsuarioRol> rolesUsuario = usuarioRolRepository.findByUsuarioIdUsuario(usuario.getIdUsuario());
```

Convierte roles y permisos en `GrantedAuthority` con prefijo `ROLE_` y devuelve un
**`CustomUserDetails`** (`security/CustomUserDetails.java`), que extiende `User` de Spring
Security añadiendo el campo `idUsuario`. Ese campo es el que el controlador usará después.

> Detalle observable en el depurador: **antes de llegar al controlador ya se ejecutaron
> dos SELECT** contra las tablas `usuarios` y `usuarios_roles`.

---

## Paso 6 — HandlerMapping: se resuelve el método destino

**Clase:** `org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping`
(framework). Recorre el registro de rutas construido al arrancar a partir de las anotaciones
del proyecto y encuentra la correspondencia:

| Elemento | Valor en ARTISYNC |
|---|---|
| `@RequestMapping` de clase | `/api/v1/pedidos` — `controller/pedido/PedidoControlador.java:23` |
| `@PostMapping` de método | (sin ruta adicional) — línea 29 |
| Handler resuelto | `PedidoControlador#crearPedido` |

A continuación `RequestMappingHandlerAdapter` resuelve los argumentos del método con sus
`HandlerMethodArgumentResolver`:

- `@AuthenticationPrincipal CustomUserDetails userDetails` → lo saca del `SecurityContext`
  poblado en el paso 5.
- `@Valid @RequestBody PeticionCrearPedido peticion` → Jackson deserializa el JSON del cuerpo
  y **Bean Validation** valida el DTO. Si falla, la petición nunca entra al controlador y se
  desvía al `@ExceptionHandler(MethodArgumentNotValidException.class)` del paso 11.

---

## Paso 7 — Autorización a nivel de método

**Archivo:** `config/SecurityConfig.java` — anotación `@EnableMethodSecurity` (línea 28)
**Punto de aplicación:** `controller/pedido/PedidoControlador.java:30`

```java
@PostMapping
@PreAuthorize("hasAnyRole('CLIENTE', 'ADMIN')")
public ResponseEntity<RespuestaPedido> crearPedido(...)
```

El proxy AOP (`AuthorizationManagerBeforeMethodInterceptor`) compara las *authorities*
cargadas en el paso 5.a. Si el usuario no tiene `ROLE_CLIENTE` ni `ROLE_ADMIN`, se lanza
`AccessDeniedException` **antes** de ejecutar una sola línea del controlador.

---

## Paso 8 — `@RestController`

**Archivo:** `controller/pedido/PedidoControlador.java`
**Método:** `crearPedido(CustomUserDetails userDetails, PeticionCrearPedido peticion)` (línea 31)

```java
@RestController
@RequestMapping("/api/v1/pedidos")
@RequiredArgsConstructor
public class PedidoControlador {

    private final IPedidoServicio pedidoServicio;

    @PostMapping
    @PreAuthorize("hasAnyRole('CLIENTE', 'ADMIN')")
    public ResponseEntity<RespuestaPedido> crearPedido(
            @AuthenticationPrincipal CustomUserDetails userDetails,
            @Valid @RequestBody PeticionCrearPedido peticion) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(pedidoServicio.crearPedido(userDetails.getIdUsuario(), peticion));
    }
}
```

Observaciones del trazado:

- El controlador **no contiene lógica de negocio**: sólo extrae el `idUsuario` de la identidad
  autenticada y delega. Es una capa de traducción HTTP ↔ dominio.
- Depende de la **interfaz** `IPedidoServicio`, no de la implementación: la inyección la
  resuelve Spring por constructor (`@RequiredArgsConstructor` de Lombok).
- El identificador del cliente **nunca viene del cuerpo de la petición**, sino del token.
  Eso evita que un cliente cree pedidos a nombre de otro.

---

## Paso 9 — `@Service`: regla de negocio y transacción

**Archivo:** `service/pedido/impl/PedidoServicioImpl.java`
**Método:** `crearPedido(Long idCliente, PeticionCrearPedido peticion)` (línea 43)
**Interfaz:** `service/pedido/IPedidoServicio.java`

```java
@Override
@Transactional
public RespuestaPedido crearPedido(Long idCliente, PeticionCrearPedido peticion) {

    Usuario cliente = usuarioRepository.findById(idCliente)
            .orElseThrow(() -> new ExcepcionRecursoNoEncontrado("Usuario cliente no encontrado"));

    Servicio servicio = servicioRepository.findById(peticion.getIdServicio())
            .orElseThrow(() -> new ExcepcionRecursoNoEncontrado("Servicio no encontrado"));

    // Regla de negocio: el cliente no puede pedir su propio servicio
    if (servicio.getPerfil().getUsuario().getIdUsuario().equals(idCliente)) {
        throw new ExcepcionReglaNegocio("No puedes crear un pedido para tu propio servicio");
    }

    List<FlujoTrabajo> flujos = flujoTrabajoRepository.findAll();
    if (flujos.isEmpty()) {
        throw new ExcepcionReglaNegocio("No hay flujos de trabajo configurados en el sistema");
    }
    FlujoTrabajo flujo = flujos.get(0);

    List<FlujoEtapaConfig> etapas = flujoEtapaConfigRepository
            .findByFlujoIdFlujoOrderByNumeroOrdenAsc(flujo.getIdFlujo());
    if (etapas.isEmpty()) {
        throw new ExcepcionReglaNegocio("El flujo de trabajo no tiene etapas configuradas");
    }

    Pedido pedido = Pedido.builder()
            .usuarioCliente(cliente)
            .servicio(servicio)
            .flujo(flujo)
            .precioPactado(peticion.getPrecioOfrecido() != null
                    ? peticion.getPrecioOfrecido()
                    : servicio.getPrecioBase())
            .fechaEntregaEstimada(peticion.getFechaEntregaEstimada())
            .build();

    pedido = pedidoRepository.save(pedido);                      // ← INSERT en pedidos

    HistorialEstadoPedido estadoInicial = HistorialEstadoPedido.builder()
            .pedido(pedido)
            .etapa(etapas.get(0).getEtapa())
            .observacion("Pedido creado")
            .build();
    historialRepository.save(estadoInicial);                     // ← INSERT en historial_estados_pedido

    log.info("Pedido {} creado por cliente {} para servicio {}", ...);

    return mapToRespuesta(pedido);
}
```

`@Transactional` abre la transacción al entrar y hace *commit* al salir. Los dos `INSERT`
son atómicos: si el segundo falla, el primero se revierte.

**Repositorios inyectados en este servicio** (declarados en las líneas 33-38):
`PedidoRepository`, `ServicioRepository`, `UsuarioRepository`, `FlujoTrabajoRepository`,
`FlujoEtapaConfigRepository`, `HistorialEstadoPedidoRepository`, `EtapaFlujoRepository`.

---

## Paso 10 — `JpaRepository` → Hibernate → PostgreSQL

**Archivo:** `repository/pedido/PedidoRepository.java`

```java
@Repository
public interface PedidoRepository extends JpaRepository<Pedido, Long> {
    List<Pedido> findByUsuarioClienteIdUsuario(Long idUsuario);
    List<Pedido> findByServicioPerfilIdPerfil(Long idPerfil);
    List<Pedido> findByServicioPerfilUsuarioIdUsuario(Long idUsuario);
}
```

Es una **interfaz sin implementación**: Spring Data genera en tiempo de arranque un proxy
(`SimpleJpaRepository`) y deriva el SQL a partir del nombre de cada método. `save()` lo
aporta `JpaRepository`.

**Entidad:** `entity/pedido/Pedido.java` → tabla `pedidos`

```java
@Entity
@Table(name = "pedidos")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class Pedido {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "id_pedido")
    private Long idPedido;

    @NotNull @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "id_usuario_cliente", nullable = false)
    private Usuario usuarioCliente;

    @NotNull @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "id_servicio", nullable = false)
    private Servicio servicio;

    @NotNull @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "id_flujo", nullable = false)
    private FlujoTrabajo flujo;

    @CreationTimestamp
    @Column(name = "fecha_inicio", updatable = false)
    private LocalDateTime fechaInicio;

    @Column(name = "fecha_entrega_estimada")
    private LocalDateTime fechaEntregaEstimada;

    @NotNull @DecimalMin("0.00")
    @Column(name = "precio_pactado", nullable = false, precision = 10, scale = 2)
    private BigDecimal precioPactado;
}
```

SQL que Hibernate emite (visible activando `spring.jpa.show-sql=true`):

```sql
select ... from usuarios u1_0 where u1_0.id_usuario=?
select ... from servicios s1_0 where s1_0.id_servicio=?
select ... from flujos_trabajo
select ... from flujo_etapas_config f1_0 where f1_0.id_flujo=? order by f1_0.numero_orden
insert into pedidos (fecha_entrega_estimada, id_flujo, id_servicio, id_usuario_cliente, precio_pactado) values (?,?,?,?,?)
insert into historial_estados_pedido (id_pedido, id_etapa, observacion) values (?,?,?)
select ... from historial_estados_pedido h1_0 where h1_0.id_pedido=? order by h1_0.fecha_transicion
```

Conexión configurada en `src/main/resources/application.properties`:

```properties
spring.datasource.url=${DB_URL:jdbc:postgresql://localhost:5432/artisyncbd}
spring.datasource.username=${DB_APP_USER:artisync_app}
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.open-in-view=false
```

La aplicación se conecta con la cuenta de **privilegios mínimos** `artisync_app`
(sólo DML, sin DDL); Flyway usa una cuenta distinta para las migraciones.

Tablas implicadas: `usuarios`, `usuarios_roles`, `servicios`, `flujos_trabajo`,
`flujo_etapas_config`, `etapas_flujo`, `pedidos`, `historial_estados_pedido`.

---

## Paso 11 — Mapeo a DTO y serialización JSON

**Archivo:** `service/pedido/impl/PedidoServicioImpl.java`
**Método:** `mapToRespuesta(Pedido pedido)` (línea 261, privado)

```java
private RespuestaPedido mapToRespuesta(Pedido pedido) {
    List<RespuestaHistorialEstado> historial = historialRepository
            .findByPedidoIdPedidoOrderByFechaTransicionAsc(pedido.getIdPedido())
            .stream().map(this::mapHistorial).collect(Collectors.toList());

    Usuario creador = pedido.getServicio().getPerfil().getUsuario();

    return RespuestaPedido.builder()
            .idPedido(pedido.getIdPedido())
            .idServicio(pedido.getServicio().getIdServicio())
            .tituloServicio(pedido.getServicio().getTituloServicio())
            .idCliente(pedido.getUsuarioCliente().getIdUsuario())
            .nombreCliente(pedido.getUsuarioCliente().getNombres() + " " + pedido.getUsuarioCliente().getApellidos())
            .idCreador(creador.getIdUsuario())
            .nombreCreador(creador.getNombres() + " " + creador.getApellidos())
            .etapaActual(obtenerEtapaActual(pedido.getIdPedido()))
            .precioPactado(pedido.getPrecioPactado())
            .fechaInicio(pedido.getFechaInicio())
            .fechaEntregaEstimada(pedido.getFechaEntregaEstimada())
            .nombreFlujo(pedido.getFlujo().getNombreFlujo())
            .historial(historial)
            .build();
}
```

**DTO de salida:** `dto/respuesta/pedido/RespuestaPedido.java`

La conversión ocurre **dentro de la transacción**, lo cual es obligatorio en este proyecto
porque `spring.jpa.open-in-view=false`: fuera del `@Transactional` las relaciones `LAZY`
(`servicio`, `usuarioCliente`, `flujo`) ya no se pueden navegar.

Al volver al `DispatcherServlet`, `RequestResponseBodyMethodProcessor` toma el
`ResponseEntity<RespuestaPedido>` y selecciona `MappingJackson2HttpMessageConverter`
(por `Accept: application/json`), que serializa el DTO:

```json
{
  "idPedido": 17,
  "idServicio": 4,
  "tituloServicio": "Ilustración de personaje full body",
  "idCliente": 9,
  "nombreCliente": "Johan Carvajal",
  "idCreador": 3,
  "nombreCreador": "María Loor",
  "etapaActual": "Briefing",
  "precioPactado": 120.00,
  "fechaInicio": "2026-08-10T14:22:31.118",
  "fechaEntregaEstimada": "2026-08-25T00:00:00",
  "nombreFlujo": "Flujo estándar de comisión",
  "historial": [ { "idEtapa": 1, "nombreEtapa": "Briefing", "observacion": "Pedido creado", "fechaTransicion": "..." } ]
}
```

Estado devuelto: **`201 Created`**, fijado en el controlador con
`ResponseEntity.status(HttpStatus.CREATED)`.

### Ruta alternativa: si algo falla

**Archivo:** `exception/ManejadorGlobalExcepciones.java` (`@RestControllerAdvice`)

Las excepciones lanzadas en el `@Service` no llegan al cliente como *stack trace*: se
convierten a **ProblemDetail (RFC 7807)**.

| Excepción | Handler (línea) | Estado HTTP |
|---|---|---|
| `MethodArgumentNotValidException` | línea 32 | 400 |
| `ExcepcionRecursoNoEncontrado` | línea 52 | 404 |
| `ExcepcionReglaNegocio` | línea 79 | 409 / 400 |
| `AccessDeniedException` | línea 100 | 403 |
| `AuthenticationException`, `JwtException` | línea 110 | 401 |
| `Exception` (catch-all) | línea 129 | 500 |

---

## Paso 12 — Vuelta al cliente Angular

**Archivo:** `app/core/interceptors/error.interceptor.ts`

La respuesta atraviesa de nuevo la cadena de interceptores. Con `201` no se dispara ningún
`catchError`, así que llega directamente al `subscribe({ next })` del componente:

```typescript
next: (res) => {
  this.loading = false;
  this.router.navigate(['/pedido', res.idPedido]);
}
```

Si el backend hubiera devuelto `401`, `errorInterceptor` habría intentado
`auth.refreshToken()` y reintentado la petición; con `403` habría redirigido a
`/no-autorizado`; con `400` habría mostrado los `fieldErrors` del ProblemDetail.

El ciclo se cierra: el usuario ve la pantalla de detalle del pedido recién creado.
