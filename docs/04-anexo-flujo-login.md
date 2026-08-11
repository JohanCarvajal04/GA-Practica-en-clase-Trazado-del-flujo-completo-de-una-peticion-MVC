# 04 — Anexo: flujo de `POST /api/auth/login`

Este anexo traza la petición **pública** que genera el JWT usado en el flujo principal
(`docs/01-trazado-flujo-mvc.md`). Sirve de contraste: es la misma arquitectura MVC, pero la
petición **no** pasa por la validación del filtro JWT — porque todavía no hay token.

---

## Diferencias con el flujo autenticado

| Aspecto | `POST /api/v1/pedidos` | `POST /api/auth/login` |
|---|---|---|
| Regla en `SecurityConfig` | `.anyRequest().authenticated()` (línea 74) | `.permitAll()` (línea 64) |
| `LoginRateLimitFilter` | Se salta (URI no coincide) | **Se ejecuta**: limita intentos por IP |
| `JwtAuthenticationFilter` | Valida el token y puebla el `SecurityContext` | Entra, pero sale en la 1.ª comprobación: no hay cabecera `Authorization` |
| `authInterceptor` de Angular | Añade `Bearer <token>` | **Excluido explícitamente** (`req.url.includes('/auth/login')`) |
| `@PreAuthorize` | `hasAnyRole('CLIENTE','ADMIN')` | Ninguno |
| Autenticación real | Ya hecha (token) | `AuthenticationManager.authenticate()` con usuario y contraseña |
| Resultado | `201 Created` + `RespuestaPedido` | `200 OK` + `TokenResponse` + cookie `refreshToken` HttpOnly |

---

## Trazado paso a paso

| # | Paso | Archivo | Método | Línea |
|---|---|---|---|---|
| 1 | Formulario de login | `app/features/seguridad/services/auth.service.ts` | `login()` | — |
| 2 | Interceptor **no** añade token | `app/core/interceptors/auth.interceptor.ts` | `authInterceptor` | 5 |
| 3 | Cadena de filtros: ruta pública | `config/SecurityConfig.java` | `filterChain()` | 64 |
| 4 | **Rate limiting por IP** | `security/LoginRateLimitFilter.java` | `doFilterInternal()` | 35 |
| 5 | Filtro JWT: sale de inmediato | `security/JwtAuthenticationFilter.java` | `doFilterInternal()` | 38-41 |
| 6 | HandlerMapping | `RequestMappingHandlerMapping` | `getHandlerInternal()` | — |
| 7 | **Controlador** | `controller/seguridad/AuthController.java` | `login()` | 42 |
| 8 | **Servicio** | `service/seguridad/impl/AuthServiceImpl.java` | `login()` | 126 |
| 9 | Verificación de credenciales | `AuthServiceImpl` → `AuthenticationManager` → `CustomUserDetailsService` | `authenticate()` / `loadUserByUsername()` | 130 / 32 |
| 10 | Comprobación de 2FA | `repository/seguridad/AutenticacionDosFactoresRepository.java` | `findByUsuarioIdUsuario()` | 141 |
| 11 | **Generación del JWT** | `security/JwtService.java` | `generarToken()` / `generarRefreshToken()` | 153-154 |
| 12 | Registro de la sesión en BD | `repository/seguridad/SesionUsuarioRepository.java` | `save()` (vía `registrarSesion`) | 356 |
| 13 | Cookie HttpOnly del refresh token | `controller/seguridad/AuthController.java` | `setRefreshTokenCookie()` | — |
| 14 | Serialización | `dto/seguridad/response/TokenResponse.java` | (POJO Lombok) | — |

---

## Puntos clave del código

### Filtro de rate limit — `security/LoginRateLimitFilter.java:40`

```java
private static final String RUTA_LOGIN = "/api/auth/login";

if (!"POST".equalsIgnoreCase(request.getMethod()) || !RUTA_LOGIN.equals(request.getRequestURI())) {
    filterChain.doFilter(request, response);
    return;
}
// a partir de aquí sí cuenta intentos por IP
```

Este es el único filtro del proyecto que actúa **selectivamente sobre una sola ruta**.

### Filtro JWT — salida temprana (`JwtAuthenticationFilter.java:38`)

```java
String authHeader = request.getHeader("Authorization");
if (authHeader == null || !authHeader.startsWith("Bearer ")) {
    filterChain.doFilter(request, response);   // ← el login sale por aquí
    return;
}
```

En el depurador se comprueba que el punto de ruptura **BP-2** (línea 81) nunca se alcanza en
una petición de login: no hay `SecurityContext` que poblar.

### Controlador — `controller/seguridad/AuthController.java:42`

```java
@Operation(summary = "Iniciar sesión y obtener token JWT o requerimiento 2FA")
@PostMapping("/login")
public ResponseEntity<TokenResponse> login(@Valid @RequestBody LoginRequest request,
                                           HttpServletResponse response) {
    TokenResponse tokenResponse = authService.login(request);
    setRefreshTokenCookie(response, tokenResponse.getRefreshToken());
    return ResponseEntity.ok(tokenResponse);
}
```

### Servicio — `service/seguridad/impl/AuthServiceImpl.java:126`

```java
public TokenResponse login(LoginRequest request) {
    String ip = obtenerIpActual();

    try {
        authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(request.getCorreo(), request.getContrasena())
        );
    } catch (AuthenticationException e) {
        log.warn("evento=LOGIN resultado=FALLIDO correo={} ip={}", request.getCorreo(), ip);
        throw e;
    }

    Usuario usuario = usuarioRepository.findByCorreo(request.getCorreo())
            .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND, "Usuario no encontrado"));

    AutenticacionDosFactores dosFactores = autenticacionDosFactoresRepository
            .findByUsuarioIdUsuario(usuario.getIdUsuario()).orElse(null);

    if (dosFactores != null && Boolean.TRUE.equals(dosFactores.getEstaHabilitado())) {
        return TokenResponse.builder()
                .correo(usuario.getCorreo())
                .idUsuario(usuario.getIdUsuario())
                .requiere2fa(true)          // ← corta aquí: aún no hay token
                .build();
    }

    UserDetails userDetails = userDetailsService.loadUserByUsername(usuario.getCorreo());
    String accessToken  = jwtService.generarToken(userDetails);
    String refreshToken = jwtService.generarRefreshToken(userDetails);

    registrarSesion(usuario, accessToken,  jwtService.getExpirationMs());
    registrarSesion(usuario, refreshToken, jwtService.getRefreshExpirationMs());

    log.info("evento=LOGIN resultado=EXITOSO correo={} ip={} sub={}", ...);

    return TokenResponse.builder()
            .accessToken(accessToken)
            .refreshToken(refreshToken)
            .roles(roles)
            .permisos(permisos)
            .requiere2fa(false)
            .build();
}
```

Detalle observable en el depurador: `authenticationManager.authenticate(...)` acaba llamando
a **`CustomUserDetailsService.loadUserByUsername()`** — el mismo método del paso 8.b del flujo
principal. Es decir, esa clase se ejecuta **tanto al emitir el token como al validarlo**.

---

## Puntos de ruptura sugeridos para este anexo

| # | Archivo | Línea | Qué demuestra |
|---|---|---|---|
| BP-A | `security/LoginRateLimitFilter.java` | 40 | El único filtro que discrimina por URI |
| BP-B | `security/JwtAuthenticationFilter.java` | 38 | El login sale del filtro JWT sin autenticar |
| BP-C | `controller/seguridad/AuthController.java` | 42 | Entrada al controlador sin `SecurityContext` |
| BP-D | `service/seguridad/impl/AuthServiceImpl.java` | 130 | `authenticate()` compara el hash BCrypt |
| BP-E | `service/seguridad/impl/AuthServiceImpl.java` | 153 | Nace el `accessToken` que se usará en el flujo principal |

Copiando el valor de `accessToken` en BP-E y pegándolo en https://jwt.io se ven los claims
(`sub`, `jti`, `exp`, `type`) que `JwtAuthenticationFilter` leerá en la siguiente petición.
Ese es el cierre del círculo entre los dos flujos.
