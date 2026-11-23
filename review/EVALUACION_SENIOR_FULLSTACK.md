# Filtro técnico para Desarrollador Senior Fullstack

## 1. Ejercicio técnico (90 minutos + 30 de sustentación)

**Por qué este ejercicio y no un algoritmo abstracto:** Un senior fullstack en esta plataforma no aporta valor resolviendo árboles binarios en pizarra, sino leyendo código existente, detectando riesgos reales en flujos críticos y entregando cambios seguros de extremo a extremo.

**Enunciado concreto:**

> Tienes acceso a una rama aislada de esta plataforma (`golang-gin-realworld-example-app` + cliente frontend). Actualmente existen dos comportamientos reportados:
> 1. Cuando un usuario es eliminado en base de datos, su token JWT aún vigente sigue permitiendo acceder a rutas protegidas o devuelve errores no controlados en el frontend.
> 2. El endpoint de creación/edición de artículos permite enviar payloads malformados o sin validación estricta de sesión, generando respuestas `500` o estados inconsistentes en la UI.
>
> Tu objetivo es:
> - **(a)** Diagnosticar la causa raíz en backend y frontend.
> - **(b)** Implementar la corrección mínima y segura en ambas capas respetando el contrato API existente.
> - **(c)** Agregar al menos una prueba automatizada que falle antes del cambio y pase después.
> - **(d)** Dejar un breve resumen de impacto y riesgos.

---

## 2. Señales de avance vs. señales de descarte (aunque el código funcione)

| Señales para avanzar (criterio Senior) | Señales para descartar (aunque pase pruebas) |
| :--- | :--- |
| **Alcance puntual:** Toca solo los archivos necesarios y respeta las convenciones del proyecto (*middlewares*, serializadores, manejo de errores HTTP). | **Sobreingeniería o ruido:** Refactoriza medio repositorio, cambia librerías o rompe contratos existentes sin justificación. |
| **Defensa en profundidad:** Valida tanto en el backend (`401`/`422` explícitos, limpieza de contexto) como en la experiencia del frontend (manejo limpio de sesión expirada). | **Parches cosméticos:** Solo oculta el error en el frontend o captura excepciones silenciando fallos críticos en Go. |
| **Observabilidad y pruebas con intención:** Escribe tests que prueban el caso borde real (usuario inexistente, token inválido) y explica qué quedó fuera de alcance. | **Falsa seguridad:** Escribe tests triviales que solo verifican `200 OK` o deja secretos/valores *hardcodeados* en código o fixtures. |

---

## 3. Cómo evaluar a un candidato que usó IA

El uso de IA está permitido y se espera; lo que se evalúa es la **autoría intelectual y la responsabilidad sobre el código generado**. En la sustentación haría cuatro preguntas directas:

* **Trade-offs:** *"¿Qué otra alternativa consideraste para validar la sesión en cada request y por qué elegiste esta frente al costo de consultar la base de datos?"*
* **Fallas en producción:** *"Si mañana este cambio despliega en producción y la base de datos tiene latencia alta o devuelve un error distinto a `ErrRecordNotFound`, ¿qué hace exactamente tu código?"*
* **Contrafactual en vivo:** *"Supón que ahora necesitamos soportar revocación inmediata de tokens sin golpear la BD en cada lectura; dime en qué archivo cambiarías el diseño."*
* **Detección de alucinaciones:** Señalar una línea concreta de su diff (especialmente algún helper o configuración generada) y preguntar por qué existe cada parámetro o bandera usada. 

---

## 4. Qué NO vale la pena evaluar en la prueba técnica y cómo cubrirlo

* **Memorización de sintaxis o APIs estándar** (flags exactos de `golangci-lint`, firmas de GORM o hooks de frontend): hoy se resuelven en segundos con documentación o IA; evaluarlo filtra memoria, no ingeniería.
* **Arquitectura distribuida desde cero a gran escala:** en 90 minutos solo produce diagramas genéricos; se cubre mejor mediante una **entrevista de diseño de sistemas** basada en incidentes reales del candidato.
* **Cultura de colaboración, ownership y manejo de conflicto bajo presión:** no se mide escribiendo código solo; se cubre mediante **revisión de un PR real pasado** (cómo recibe y da *code review*) y **referencias técnicas estructuradas** con pares anteriores.

---
