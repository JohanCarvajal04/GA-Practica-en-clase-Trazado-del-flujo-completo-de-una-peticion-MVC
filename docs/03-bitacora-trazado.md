# 03 — Bitácora de trazado (entregable del punto 4)

> *"Cada estudiante documenta en su cuaderno o editor de texto el nombre del archivo Java y
> el método donde se ejecuta cada paso del flujo en su propio PFC."*

**PFC:** ARTISYNC · **Petición trazada:** `POST /api/v1/pedidos`
**Paquete base:** `uteq.edu.ec.artisync` · **Raíz:** `artisync/Backend/src/main/java/uteq/edu/ec/artisync/`

---

## Tabla maestra: paso del flujo → clase Java → método

| # | Paso del flujo MVC | Archivo Java (o TS) | Método / función | Línea | ¿Código propio? |
|---|---|---|---|---|---|
| 1 | Origen de la petición (UI) | `pedido-crear.component.ts` | `onSubmit()` | 26 | Sí (Angular) |
| 2 | Emisión HTTP | `pedido.service.ts` | `crearPedido()` | 21 | Sí (Angular) |
| 3 | Inyección del token JWT | `auth.interceptor.ts` | `authInterceptor` | 5 | Sí (Angular) |
| 4 | Arranque del contenedor / registro del `DispatcherServlet` | `ArtisyncApplication.java` | `main()` | 13 | Sí |
| 5 | **DispatcherServlet** | `org.springframework.web.servlet.DispatcherServlet` | `doDispatch()` | — | No (framework) |
| 6 | Declaración de la cadena de filtros | `config/SecurityConfig.java` | `filterChain(HttpSecurity)` | 40 | Sí |
| 7 | Filtro de rate limit (se salta en esta ruta) | `security/LoginRateLimitFilter.java` | `doFilterInternal()` | 35 | Sí |
| 8 | **Filtro JWT** (el `JwtAuthFilter` de la práctica) | `security/JwtAuthenticationFilter.java` | `doFilterInternal()` | 32 | Sí |
| 8.a | Decodificación de claims | `security/JwtService.java` | `extraerTodosLosClaims()` / `extraerUsername()` | — | Sí |
| 8.b | Carga de usuario y roles desde BD | `security/CustomUserDetailsService.java` | `loadUserByUsername()` | 32 | Sí |
| 8.c | Principal autenticado | `security/CustomUserDetails.java` | constructor (`getIdUsuario()`) | 14 | Sí |
| 8.d | Publicación en el `SecurityContext` | `security/JwtAuthenticationFilter.java` | `doFilterInternal()` | 81 | Sí |
| 9 | **HandlerMapping** | `RequestMappingHandlerMapping` | `getHandlerInternal()` | — | No (framework) |
| 10 | Deserialización + Bean Validation del cuerpo | `dto/peticion/pedido/PeticionCrearPedido.java` | `@NotNull` sobre `idServicio` | 19 | Sí |
| 11 | Autorización por rol | `controller/pedido/PedidoControlador.java` | `@PreAuthorize` sobre `crearPedido` | 30 | Sí |
| 12 | **@RestController** | `controller/pedido/PedidoControlador.java` | `crearPedido()` | 31 | Sí |
| 13 | Contrato de la capa de servicio | `service/pedido/IPedidoServicio.java` | `crearPedido()` | — | Sí |
| 14 | **@Service** (regla de negocio + transacción) | `service/pedido/impl/PedidoServicioImpl.java` | `crearPedido()` | 43 | Sí |
| 15 | **JpaRepository** — escritura del pedido | `repository/pedido/PedidoRepository.java` | `save()` (heredado de `JpaRepository`) | — | Sí (interfaz) |
| 16 | **JpaRepository** — escritura del historial | `repository/pedido/HistorialEstadoPedidoRepository.java` | `save()` | — | Sí (interfaz) |
| 17 | **Entidad JPA** → tabla `pedidos` | `entity/pedido/Pedido.java` | (mapeo `@Entity`/`@Table`) | 16 | Sí |
| 18 | **Entidad JPA** → tabla `historial_estados_pedido` | `entity/pedido/HistorialEstadoPedido.java` | (mapeo `@Entity`/`@Table`) | 11 | Sí |
| 19 | **PostgreSQL** | `src/main/resources/application.properties` | `spring.datasource.url` | — | Configuración |
| 20 | Mapeo entidad → DTO | `service/pedido/impl/PedidoServicioImpl.java` | `mapToRespuesta()` | 261 | Sí |
| 21 | DTO de respuesta | `dto/respuesta/pedido/RespuestaPedido.java` | (POJO Lombok) | 16 | Sí |
| 22 | **Serialización JSON** | `MappingJackson2HttpMessageConverter` | `writeInternal()` | — | No (framework) |
| 23 | Manejo de errores → ProblemDetail (RFC 7807) | `exception/ManejadorGlobalExcepciones.java` | `@ExceptionHandler` varios | 31-137 | Sí |
| 24 | Respuesta HTTP `201 Created` | `controller/pedido/PedidoControlador.java` | `ResponseEntity.status(HttpStatus.CREATED)` | 34 | Sí |
| 25 | Recepción en el cliente | `error.interceptor.ts` → `pedido-crear.component.ts` | `errorInterceptor` → `subscribe({next})` | 35 | Sí (Angular) |

---

## Correspondencia con las clases que pide la directriz (3)

| Clase que pide la práctica | Nombre real en ARTISYNC | Ruta |
|---|---|---|
| **Filtro JWT** (`JwtAuthFilter`) | `JwtAuthenticationFilter` | `security/JwtAuthenticationFilter.java` |
| **Controlador** | `PedidoControlador` | `controller/pedido/PedidoControlador.java` |
| **Servicio** | `PedidoServicioImpl` (implementa `IPedidoServicio`) | `service/pedido/impl/PedidoServicioImpl.java` |
| **Repositorio** | `PedidoRepository` | `repository/pedido/PedidoRepository.java` |
| **Entidad** | `Pedido` | `entity/pedido/Pedido.java` |

---

## Anotaciones que marcan cada capa

| Anotación | Dónde aparece | Qué provoca |
|---|---|---|
| `@SpringBootApplication` | `ArtisyncApplication` | Autoconfigura Spring MVC y el `DispatcherServlet` |
| `@Configuration` + `@EnableWebSecurity` | `SecurityConfig` | Construye la cadena de filtros |
| `@EnableMethodSecurity` | `SecurityConfig` (línea 28) | Habilita `@PreAuthorize` |
| `@Component` | `JwtAuthenticationFilter` | Lo registra como bean para inyectarlo en la cadena |
| `@RestController` | `PedidoControlador` (línea 22) | `@Controller` + `@ResponseBody`: la respuesta va al cuerpo |
| `@RequestMapping("/api/v1/pedidos")` | `PedidoControlador` (línea 23) | Prefijo de ruta del `HandlerMapping` |
| `@PostMapping` | `crearPedido` (línea 29) | Verbo HTTP |
| `@AuthenticationPrincipal` | Parámetro de `crearPedido` | Inyecta el `CustomUserDetails` del `SecurityContext` |
| `@Valid @RequestBody` | Parámetro de `crearPedido` | Deserializa con Jackson y valida el DTO |
| `@Service` | `PedidoServicioImpl` (línea 29) | Bean de la capa de negocio |
| `@Transactional` | `crearPedido` (línea 42) | Transacción que envuelve los dos INSERT |
| `@Repository` | `PedidoRepository` | Traduce excepciones de persistencia |
| `@Entity` / `@Table(name="pedidos")` | `Pedido` (líneas 15-16) | Mapeo objeto-relacional |
| `@RestControllerAdvice` | `ManejadorGlobalExcepciones` | Convierte excepciones en ProblemDetail |

---

## Observaciones anotadas durante el trazado

1. **La seguridad se ejecuta antes que Spring MVC.** En BP-1 la pila de llamadas no contiene
   todavía `DispatcherServlet.doDispatch`: los filtros de servlet son anteriores. El
   `DispatcherServlet` sólo ve peticiones que ya pasaron por `FilterChainProxy`.

2. **Hay dos consultas a PostgreSQL antes del controlador.** `CustomUserDetailsService`
   consulta `usuarios` y `usuarios_roles` dentro del filtro JWT. Es el coste de tener
   autoridades siempre frescas en lugar de leerlas del token.

3. **El `idUsuario` no viene del cuerpo de la petición.** El controlador lo toma de
   `userDetails.getIdUsuario()`, es decir, del token verificado. Si se leyera del JSON, un
   cliente podría crear pedidos a nombre de otro.

4. **El controlador no tiene lógica.** Sus tres líneas útiles sólo traducen HTTP a una llamada
   de dominio. Toda la validación de negocio vive en `PedidoServicioImpl`.

5. **Dos INSERT, una sola transacción.** `pedidos` e `historial_estados_pedido` se escriben
   atómicamente. Verificado provocando una excepción antes del *commit*: ambos se revierten.

6. **La entidad nunca sale al JSON.** `mapToRespuesta()` construye un `RespuestaPedido` plano
   dentro de la transacción. Con `spring.jpa.open-in-view=false`, hacerlo fuera lanzaría
   `LazyInitializationException`; además evita exponer `contrasenaHash` del `Usuario`.

7. **Los errores también tienen su capa.** Ninguna excepción llega cruda al cliente:
   `ManejadorGlobalExcepciones` las traduce a ProblemDetail (RFC 7807), que el
   `errorInterceptor` de Angular sabe leer (`error.error.detail`, `error.error.fieldErrors`).

---

## Resumen en una línea por capa

```
Angular  → pedido-crear.component.ts#onSubmit()
         → pedido.service.ts#crearPedido()
         → auth.interceptor.ts (Bearer)
Spring   → SecurityConfig#filterChain()
         → JwtAuthenticationFilter#doFilterInternal()
         → CustomUserDetailsService#loadUserByUsername()
         → DispatcherServlet + RequestMappingHandlerMapping
         → PedidoControlador#crearPedido()
         → PedidoServicioImpl#crearPedido()  [@Transactional]
         → PedidoRepository#save() / HistorialEstadoPedidoRepository#save()
         → Pedido (@Entity → tabla pedidos)
PostgreSQL → INSERT INTO pedidos ... ; INSERT INTO historial_estados_pedido ...
Jackson  → RespuestaPedido → JSON
HTTP     → 201 Created
Angular  → subscribe({ next }) → router.navigate(['/pedido', id])
```
