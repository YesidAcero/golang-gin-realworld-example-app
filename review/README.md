# Hallazgos de auditoría — `golang-gin-realworld-example-app`

**Repositorio:** [github.com/gothinkster/golang-gin-realworld-example-app](https://github.com/gothinkster/golang-gin-realworld-example-app)
**SHA revisado:** `a41289b2a4951e7cc09066f86742a27f85121c9b`

## Resumen

| ID | Hallazgo | Severidad | Archivo | Esfuerzo | ¿Bloquea el merge? |
|---|---|---|---|---|---|
| [REV-GO-001](#rev-go-001--secreto-jwt-hardcodeado-en-el-código-fuente) | Secreto JWT hardcodeado en el código fuente | 🔴 Crítica | `common/utils.go` | S | Sí |
| [REV-GO-002](#rev-go-002--token-válido-de-un-usuario-inexistente-queda-autenticado) | Token válido de un usuario inexistente queda autenticado | 🔴 Crítica | `users/middlewares.go` | S | Sí |
| [REV-GO-003](#rev-go-003--pánico-por-nil-en-init-y-testdbinit) | Pánico por nil en `Init()` y `TestDBInit()` | 🔴 Crítica | `common/database.go` | S | Sí |
| [REV-GO-004](#rev-go-004--aserción-de-tipo-sin-comprobar-en-claims) | Aserción de tipo sin comprobar en claims | 🟠 Alta | `users/middlewares.go` | S | No |
| [REV-GO-005](#rev-go-005--validación-jwt-sin-lista-blanca-de-algoritmos-ni-expiración-estricta) | Validación JWT sin lista blanca de algoritmos ni expiración estricta | 🟠 Alta | `users/middlewares.go`, `common/test_helpers.go` | S | No |
| [REV-GO-006](#rev-go-006--aceptación-de-token-jwt-por-query-string) | Aceptación de token JWT por query string | 🟠 Alta | `users/middlewares.go` | S | No |


---

## REV-GO-001 — Secreto JWT hardcodeado en el código fuente

| Campo | Contenido |
|---|---|
| **ID** | REV-GO-001 |
| **Severidad** | Crítica |
| **Archivo** | `common/utils.go` |
| **Línea** | 43 (es la constante `JWTSecret` en `common/utils.go`, y también se usa al firmar en `GenToken` alrededor de la línea 51) |
| **Esfuerzo (S/M/L)** | S |
| **¿Bloquea el merge?** | Sí |

### Evidencia

```go
const JWTSecret = "A String Very Very Very Strong!!@##$!@#$" // #nosec G101
```

Además, el secreto se usa directamente en:

- `GenToken` de `common/utils.go`
- `AuthMiddleware` de `users/middlewares.go`
- `VerifyTokenClaims` de `common/test_helpers.go`

### Impacto en producción

Es como dejar la llave maestra pegada en la puerta y usar la misma en todos lados: cualquiera que vea el código puede hacerse pasar por cualquier usuario, acceder a datos privados y causar un incidente grave de seguridad, pérdida de confianza y problemas legales.

### Recomendación

- Quitar el secreto hardcodeado y el `#nosec`.
- Leer `JWT_SECRET` desde variable de entorno al arrancar y abortar el inicio si falta o tiene menos de 32 caracteres.
- Generar una clave aleatoria nueva en el entorno/secret manager y rotar la clave.

### ¿Por qué bloquea el merge?

Es aceptable como atajo de arranque rápido en un repo de ejemplo local, pero inaceptable en producción.

---

## REV-GO-002 — Token válido de un usuario inexistente queda autenticado

| Campo | Contenido |
|---|---|
| **ID** | REV-GO-002 |
| **Severidad** | Crítica |
| **Archivo** | `users/middlewares.go` |
| **Línea** | 26–33 (`UpdateContextUserModel`) y 68–72 (`AuthMiddleware`) |
| **Esfuerzo (S/M/L)** | S |
| **¿Bloquea el merge?** | Sí |

### Evidencia

`db.First(&myUserModel, my_user_id)` se ejecuta sin comprobar el error en `UpdateContextUserModel`, dejando `my_user_id` asignado en el contexto con un `UserModel{}` vacío si el usuario ya no existe.

### Impacto en producción

Una cuenta ya eliminada o revocada podría seguir operando en la plataforma o incluso reaparecer en la base de datos si el token aún no ha caducado. Esto compromete la confianza en la gestión de cuentas, puede generar datos fantasma o accesos indebidos y afectar el cumplimiento de bajas de usuario.

### Recomendación

1. Comprobar el error devuelto por `db.First(&myUserModel, my_user_id)` dentro de `UpdateContextUserModel` (y distinguir `gorm.ErrRecordNotFound` de un error de base de datos).
2. Si el usuario no existe (o falla la consulta), resetear el contexto a usuario anónimo (`my_user_id = 0` y `UserModel{}`) en lugar de dejar un `my_user_id` huérfano con un modelo vacío.
3. En `AuthMiddleware`, si no se logra cargar un usuario válido y `auto401` es `true`, abortar la petición con `401 Unauthorized` y evitar que `UserUpdate` ejecute `Save` sobre un usuario inexistente.

### ¿Por qué bloquea el merge?

Permite que peticiones con un token de un usuario borrado avancen autenticadas o disparen escrituras sobre registros inexistentes, lo que es inaceptable en producción.

---

## REV-GO-003 — Pánico por nil en `Init()` y `TestDBInit()`

| Campo | Contenido |
|---|---|
| **ID** | REV-GO-003 |
| **Severidad** | Crítica |
| **Archivo** | `common/database.go` |
| **Línea** | 54–71 (`Init`) y 74–92 (`TestDBInit`) |
| **Esfuerzo (S/M/L)** | S |
| **¿Bloquea el merge?** | Sí |

### Evidencia

`db, err := gorm.Open(...)` imprime el error con `fmt.Println` pero no retorna error, y continúa llamando `db.DB()` sobre un `*gorm.DB` potencialmente nil.

### Impacto en producción

Si la base de datos falla al abrirse o no está disponible, la aplicación puede desplomarse de golpe en lugar de fallar de forma controlada o reportar el problema limpiamente. En producción esto significa caídas del servicio, interrupciones para los usuarios y diagnósticos más difíciles ante incidentes de infraestructura.

### Recomendación

1. Cambiar la firma de `Init()` y `TestDBInit()` para que retornen `(*gorm.DB, error)` y devolver el error de inmediato si `gorm.Open` (o `db.DB()`) falla, sin continuar usando `db` en nil.
2. En `hello.go` (y en los callers/pruebas correspondientes), comprobar el error devuelto por `common.Init()` y abortar el arranque con un mensaje de error claro si la conexión inicial falla.
3. Actualizar `TestDBInit()` y sus pruebas asociadas para manejar el error explícitamente y evitar pánicos en entornos de test.

### ¿Por qué bloquea el merge?

Si la base de datos falla al abrirse, el proceso sigue y provoca un nil pointer dereference (caída abrupta) en lugar de fallar limpiamente; en un repo de ejemplo local puede tolerarse, pero en producción es inaceptable.

---

## REV-GO-004 — Aserción de tipo sin comprobar en claims

| Campo | Contenido |
|---|---|
| **ID** | REV-GO-004 |
| **Severidad** | Alta |
| **Archivo** | `users/middlewares.go` |
| **Línea** | 78 y 79 |
| **Esfuerzo (S/M/L)** | S |
| **¿Bloquea el merge?** | No |

### Evidencia

Aserción de tipo sin comprobar sobre los claims: `uint(claims["id"].(float64))` hace panic si el claim falta o tiene otro tipo.

### Impacto en producción

Ante un token con datos de usuario ausentes, mal formados o manipulados, la aplicación puede sufrir caídas o errores inesperados para los usuarios (afectando la disponibilidad del servicio) o presentar comportamientos inconsistentes en la validación de acceso (afectando la seguridad).

### Recomendación

1. En `AuthMiddleware` (`users/middlewares.go`, línea 78), reemplaza `my_user_id := uint(claims["id"].(float64))` por una aserción segura: `rawID, ok := claims["id"].(float64)`.
2. Si `!ok` o `rawID <= 0` (o al convertir a `uint`), si `auto401` es `true`, aborta la petición con `c.AbortWithStatusJSON(http.StatusUnauthorized, common.NewErrorMessage("token", "is invalid"))` y haz `return` sin llamar a `UpdateContextUserModel`.
3. Si `auto401` es `false`, haz `return` dejando el contexto en anónimo (ya inicializado con `UpdateContextUserModel(c, 0)`).
4. Si la aserción es válida, convierte `my_user_id := uint(rawID)` y continúa con `UpdateContextUserModel(c, my_user_id)`.
5. Agrega pruebas unitarias en `users/` cubriendo tokens con claim `id` ausente, `null` o de tipo incorrecto (por ejemplo `string`) tanto con `auto401=true` (esperando `401`) como con `auto401=false`.

### ¿Por qué no bloquea el merge?

Si el PR se está cerrando enfocándose en una remediación específica ya probada y el cambio en `users/middlewares.go` (líneas 78–79) queda planificado o cubierto en el siguiente paso, se deja como `No` para no detener el merge por una aserción de tipo (`S` de esfuerzo), siempre que quede trazabilidad del pendiente.

---

## REV-GO-005 — Validación JWT sin lista blanca de algoritmos ni expiración estricta

| Campo | Contenido |
|---|---|
| **ID** | REV-GO-005 |
| **Severidad** | Alta |
| **Archivo** | `users/middlewares.go` y `common/test_helpers.go` |
| **Línea** | 62–68 (`users/middlewares.go`) y 25–35 (`common/test_helpers.go`) |
| **Esfuerzo (S/M/L)** | S |
| **¿Bloquea el merge?** | No |

### Evidencia

Parseo JWT sin lista blanca de algoritmos en el middleware (comprobación manual) y sin validación de método en `VerifyTokenClaims`.

### Impacto en producción

La validación de credenciales no es uniforme en todo el sistema ni exige con el mismo rigor su vigencia y firma, lo que puede provocar inconsistencias en el control de acceso y el riesgo de aceptar tokens con condiciones o vencimientos no exigidos estrictamente, afectando la seguridad del servicio.

### Recomendación

1. En `users/middlewares.go` (líneas 62–68), dentro de `AuthMiddleware`, pasa `jwt.WithValidMethods([]string{"HS256"})` y `jwt.WithExpirationRequired()` a `jwt.Parse`, y mantén la validación del método en el callback verificando `token.Method` y devolviendo `[]byte(common.JWTSecret)`.
2. En `common/test_helpers.go` (líneas 25–35), dentro de `VerifyTokenClaims`, pasa también `jwt.WithValidMethods([]string{"HS256"})` y `jwt.WithExpirationRequired()` a `jwt.ParseWithClaims`, y verifica que el método del token sea HMAC (`*jwt.SigningMethodHMAC`) antes de retornar `[]byte(JWTSecret)`.
3. Agrega o actualiza pruebas unitarias en `users/` y `common/` que verifiquen el rechazo de tokens sin claim `exp`, tokens expirados, tokens firmados con otro algoritmo distinto de `HS256` y tokens con firma inválida.

### ¿Por qué no bloquea el merge?

El PR ya tiene un foco y un commit listos para avanzar, y estas mejoras (`jwt.WithValidMethods` / `jwt.WithExpirationRequired` en `REV-GO-005`, y retirar `?access_token=` junto con su prueba en `REV-GO-006`) quedan documentadas para resolverse en una iteración o *follow-up* inmediato sin frenar la fusión actual.

---

## REV-GO-006 — Aceptación de token JWT por query string

| Campo | Contenido |
|---|---|
| **ID** | REV-GO-006 |
| **Severidad** | Alta |
| **Archivo** | `users/middlewares.go` |
| **Línea** | 13–27 (extracción por query string en líneas 20–24) |
| **Esfuerzo (S/M/L)** | S |
| **¿Bloquea el merge?** | No |

### Evidencia

Aceptación de token por query string (`?access_token=...`) en `extractToken`.

### Impacto en producción

Enviar tokens de sesión en la URL expone credenciales sensibles en los registros de actividad del servidor, el historial del navegador y las solicitudes salientes hacia terceros, aumentando el riesgo de filtración y uso indebido de cuentas de usuario.

### Recomendación

1. En `users/middlewares.go` (líneas 13–27, específicamente líneas 20–24), elimina la lectura de `c.Query("access_token")` en `extractToken` para que solo se extraiga el token desde la cabecera `Authorization` (`Token <jwt>`).
2. Actualiza el comentario de `extractToken` (línea 12) para reflejar que extrae únicamente desde la cabecera `Authorization`.
3. En `users/unit_test_2.go` (líneas 350–369), ajusta o reemplaza `TestExtractTokenFromQueryParameter` para comprobar que enviar `?access_token=...` ya no autentica al usuario (con `auto401=true` devuelve `401 Unauthorized`; con `auto401=false` queda como anónimo con `user_id: 0`).
4. Verifica en el código que ningún cliente o flujo interno dependa de `?access_token=` en la URL.

### ¿Por qué no bloquea el merge?

El PR ya tiene un foco y un commit listos para avanzar, y estas mejoras (`jwt.WithValidMethods` / `jwt.WithExpirationRequired` en `REV-GO-005`, y retirar `?access_token=` junto con su prueba en `REV-GO-006`) quedan documentadas para resolverse en una iteración o *follow-up* inmediato sin frenar la fusión actual.

---
