# Checklist de Pull Request y *Definition of Done* (DoD)

Estándar de revisión y cierre para `golang-gin-realworld-example-app`, derivado de la revisión técnica y los hallazgos `REV-GO-001` a `REV-GO-006`.

---

## Parte 1: Checklist de Pull Request

### 1. Alcance y trazabilidad
- [ ] El PR tiene un propósito claro, acotado al menor número de archivos posible y sin modificaciones colaterales no justificadas.
- [ ] Cada cambio está vinculado a un *issue*, requerimiento o hallazgo (`REV-GO-XXX`).
- [ ] Los hallazgos no bloqueantes o diferidos (p. ej., `REV-GO-004` a `REV-GO-006`) se documentan como seguimiento futuro y no se mezclan con correcciones críticas.

### 2. Seguridad, secretos y autenticación
- [ ] No hay secretos, claves, contraseñas ni tokens *hardcodeados* en código fuente ni en *fixtures* de prueba (`REV-GO-001`).
- [ ] Los secretos y configuraciones sensibles se cargan desde variables de entorno explícitas y se validan al arrancar (fallando rápido si faltan o no cumplen requisitos mínimos, sin valores por defecto inseguros en producción).
- [ ] En *middlewares* de autenticación/autorización, se validan los errores de consulta a base de datos o almacén de sesión y se resetea el contexto cuando el usuario no existe o falla la validación (`REV-GO-002`).
- [ ] Las peticiones no autorizadas o con identidad inválida devuelven el código HTTP correspondiente (`401` / `403`) y nunca ejecutan escrituras (`POST`, `PUT`, `PATCH`, `DELETE`) ni mutaciones de estado.
- [ ] Se verifican aserciones de tipo, validación de *claims*/expiración y validación estricta de entradas externas.

### 3. Confiabilidad, base de datos y manejo de errores
- [ ] No se desreferencian punteros en `nil` sin verificación previa en inicialización, conexiones a base de datos ni pruebas.
- [ ] Los errores de base de datos, red o *IO* se comprueban explícitamente y no se ocultan sin justificación técnica clara.
- [ ] Las pruebas no dependen de permisos de archivos específicos del sistema operativo (p. ej., `os.Chmod(..., 0000)` en Windows o entornos con `root`/CI) sin contemplar su portabilidad.

### 4. Pruebas y puertas de calidad automatizadas
- [ ] Se incluyen pruebas unitarias o de regresión para la funcionalidad o vulnerabilidad corregida (evidenciando el fallo previo y el pase posterior).
- [ ] Las pruebas relevantes pasan sin debilitar aserciones ni alterar expectativas de seguridad existentes solo para forzar el verde.
- [ ] `golangci-lint` y el *workflow* de GitHub Actions pasan en verde sobre el *commit* final del PR.

### 5. Documentación y revisión
- [ ] La descripción del PR resume **qué** cambió, **por qué**, **cómo se verificó** y los fallos preexistentes o limitaciones de entorno conocidas.
- [ ] El informe de hallazgos (`/review/README.md`) y la documentación técnica están actualizados si el cambio altera contratos, configuración o hallazgos.
- [ ] La evidencia de ejecución (logs/capturas) se adjunta en la conversación del PR sin subir archivos binarios innecesarios al repositorio.

---

## Parte 2: Propuesta de *Definition of Done* (DoD)

Un ítem o historia se considera **Terminado (*Done*)** cuando cumple todos los criterios siguientes:

- **Código revisado y acotado:** Implementado en una rama de trabajo, revisado mediante *Pull Request* y acotado al alcance definido sin cambios colaterales injustificados.
- **Seguridad verificada:** Libre de secretos *hardcodeados*, fallos de validación de identidad/autorización o manejo inseguro de errores y entradas.
- **Pruebas aprobadas:** Las pruebas unitarias y de regresión aplicables pasan (incluyendo nuevas pruebas para correcciones), y cualquier fallo preexistente por entorno o portabilidad queda explícitamente documentado.
- **CI en verde:** `golangci-lint` y el *workflow* de GitHub Actions (lint y pruebas) están configurados y en estado **verde** (*pass*) sobre el último *commit*.
- **Documentación y merge completos:** La documentación aplicable (`/review/README.md`, descripción del PR y notas de seguimiento) está actualizada y el PR está fusionado en la rama objetivo.

---
