# Liderazgo técnico y gobierno de ingeniería

## 1. Plan de 30, 60 y 90 días (8 personas: 2 Sr, 4 Ssr, 2 Interns | Go, C#, 2 Next.js)

## Plan de los primeros 90 días

**Primer mes (días 1–30): apagar incendios, entender el terreno y poner reglas básicas**

Lo primero es mirar cómo están parados los servicios en Go y C# y las dos apps en Next.js antes de tocar reglas de negocio. Revisaría dónde hay secretos expuestos, cómo se despliega hoy, qué logs tenemos si algo falla y qué *runbooks* existen o faltan. Con los *interns*, la idea es asignarles tareas pequeñas y bien acotadas junto a un *senior* o *semi-senior* para que no se queden bloqueados ni metan ruido en flujos delicados. También armaría un Definition of Done sencillo que sí se cumpla (algo de pruebas en lo crítico, *lint* y *review* de PR) y acordaría con negocio cómo entran los requerimientos para no vivir a las carreras.

**Segundo mes (días 31–60): ordenar el trabajo entre equipos y blindar lo crítico**

Una vez que ya sabemos dónde duelen las cosas, empezaría a unificar criterios entre el *backend* y los dos Next.js—sobre todo contratos de API y tipos compartidos para que no se rompan cosas en silencio al cambiar un campo. Metería pruebas en los flujos que más duelen si fallan (autenticación, pagos o cambios de estado y escrituras pesadas). Repartiría mejor la propiedad de cada componente y les daría a los *semi-seniors* la conducción de *features* acotadas, apoyados en la revisión de los *seniors*, para no concentrar todas las decisiones en una sola persona.

**Tercer mes (días 61–90): bajar fricción y sostener el ritmo**

En este tramo el foco pasa a quitar trabajo repetitivo o manual que frena al equipo, bajar los tiempos de revisión y salida a producción, y atacar las duplicaciones que de verdad estén costando tiempo o bugs. Ajustaría con negocio qué podemos entregar de forma realista según la capacidad real del equipo y revisaría cómo van los *interns* con puntos claros y concretos, sin improvisar.

---

### Qué mirar desde el día 1 (sin llenarnos de tableros)

No tiene sentido armar un tablero gigante el primer día; me enfocaría en unas pocas cosas que muestran si el sistema y el equipo están sanos:

* Que no haya picos de errores 5xx ni fallos en la interfaz, que el servicio responda y que, si algo se cae, tardemos poco en levantarlo.
* Con qué frecuencia salimos a producción, cuánto tarda un PR en llegar ahí, qué porcentaje de cambios terminan generando fallos y si los PR relevantes salen con pruebas.
* Qué tanto soporte o incidentes le caen a cada persona y si el equipo tiene claro qué va primero y qué está trabado.

---

## 2. ADR-001: Convivencia y asignación de servicios en Go y C# con lógica de dominio duplicada

**Estado:** Aceptado

## Contexto
Hoy tenemos servicios en Go y en C# con lógica de dominio duplicada entre ambos, además de dos aplicaciones en Next.js. Intentar reescribir todo a un solo lenguaje frenaría las entregas de negocio y metería un riesgo enorme; por otro lado, dejar que sigan duplicando reglas sin control hace que cada cambio derive en comportamientos distintos y *bugs* en producción.

## Opciones
1. Unificar todo en un solo lenguaje y reescribir el resto.
2. Mantener Go y C# sin reglas nuevas ni control sobre la lógica compartida.
3. Mantener ambos lenguajes con reglas claras de dónde va cada cosa, frenar nuevas duplicaciones y consolidar de a poco la lógica compartida que ya está repetida.

## Decisión
Vamos con la **opción 3**.

* **Dónde va cada cambio o servicio nuevo:** las cosas nuevas o cambios en servicios existentes se quedan en el lenguaje donde ya vive el dominio o donde el equipo tenga mejor capacidad operativa hoy, salvo que haya una razón técnica o de negocio muy clara para migrar algo en particular.
* **Cómo frenamos la duplicación:** ninguna regla de dominio compartida se vuelve a copiar a mano en otro servicio. Primero definimos quién es el dueño o qué contrato/paquete compartido lo expone (con pruebas de contrato), y sobre eso se integra. Lo que ya está duplicado se va unificando de forma gradual cuando toquemos esos flujos[cite: 9].

## Consecuencias
* **A favor:** no paramos el producto con reescrituras largas, aprovechamos lo que el equipo ya domina y cortamos de raíz que siga creciendo la duplicación.
* **En contra:** seguimos cargando con dos *runtimes* y *toolchains* (Go y C#), y dependemos de que en *code review* y pruebas de contrato no se cuele lógica repetida.

## Cuándo revisar o revertir esta decisión
Si durante dos trimestres seguidos vemos que los incidentes por reglas de negocio desalineadas o el costo de mantener ambos entornos superan claramente lo que costaría migrar los servicios activos, volvemos a evaluar consolidar en un solo lenguaje.


---

## 3. Conversación difícil (máx. 300 palabras)

**Al desarrollador senior:**
"Gracias por sacar esto adelante, sé que te pusieron una fecha encima sin revisarla conmigo y así es muy difícil trabajar bien. Igual, si subimos cambios en flujos críticos sin pruebas, cualquier fallo nos va a explotar en producción y al final perdemos el doble de tiempo apagando incendios. Cuando te vuelvan a pedir una fecha que no dé margen ni para los tests básicos, avísame antes de comprometerte y yo hablo con negocio para mover el alcance o el plazo. Con lo que ya tienes andando, revisemos hoy mismo qué prueba mínima le metemos antes de cerrar para no salir a ciegas."

**A negocio:**
"Tenemos que ponernos de acuerdo antes de cerrar fechas con el cliente o stakeholders, porque comprometer tiempos sin revisar capacidad ni impacto técnico nos está obligando a salir sin pruebas. Puede que hoy parezca que avanzamos más rápido, pero en la práctica terminamos con errores en producción y retrabajo que frenan lo que viene después. Yo me encargo de darles tiempos realistas y alternativas de alcance; si alguna fecha es inamovible, definamos juntos qué sacamos de esa entrega para no poner en riesgo la estabilidad"


---

## 4. Métricas de ingeniería

**Qué mediría**

Arrancaría con las cuatro métricas de DORA para ver cómo estamos entregando y qué tan estable sale todo: qué tan seguido desplegamos, cuánto tarda un cambio desde el PR hasta producción, qué porcentaje de despliegues terminan en incidente o *rollback* y cuánto nos toma levantar el servicio cuando algo falla. 

A eso le sumaría tres cosas muy prácticas para el día a día: la cobertura de pruebas sobre el código nuevo o modificado (no tanto el total histórico), cuánto tiempo se quedan los PRs esperando revisión y qué porcentaje del tiempo del equipo se va apagando incendios frente a construir *features*.

**De dónde salen los datos (sin poner a nadie a llenar planillas)**

La idea es que todo salga automático de las herramientas que ya usamos:
* **Repositorio y CI/CD:** de las marcas de tiempo de los PRs (creación y *merge*), los commits y las corridas del pipeline sacamos frecuencia, tiempos de espera y duración de cada despliegue.
* **Alertas y monitoreo:** cruzando el inicio y cierre de incidentes con la etiqueta de cada *release* o *hotfix* sabemos qué despliegue rompió algo y cuánto tardó en estabilizarse.
* **Pruebas y calidad:** los mismos reportes de tests y *linters* que corren en CI nos dan la cobertura del diff en cada PR.

**El cuidado que hay que tener**

Si nos ponemos a perseguir los números por cumplir una meta, el equipo termina haciendo trampa sin querer: partir cambios en commits artificiales, aprobar PRs sin mirar para bajar los tiempos de revisión, escribir tests vacíos solo para subir cobertura o aguantar despliegues grandes por miedo a dañar la tasa de fallos. Al final terminas con un tablero todo en verde mientras producción está cada vez más frágil.

---

## 5. Deuda técnica: visibilidad y financiación

A un comité enfocado en *features* no se le pide tiempo para "limpiar código"; se le habla de plata, riesgo operativo y velocidad futura. Si el argumento se queda en lo técnico, casi siempre pierde frente a una funcionalidad nueva.

* **Traducir el problema a impacto de negocio:** en lugar de pedir un refactor, mostrar qué nos está frenando entregas (por ejemplo, que cada cambio en un módulo tome el doble de tiempo) o qué riesgo real tenemos en producción (caídas en horas pico o brechas de seguridad).
* **Separar lo rutinario de lo que exige presupuesto propio:** la deuda menor se va limpiando de paso cuando tocamos esa zona por una *feature*; lo que es bloqueante o de alto riesgo se presenta como un ítem con cupo explícito, explicando qué mejora concreta vamos a medir antes y después.
* **Poner el *trade-off* sobre la mesa:** dejar claro qué *feature* secundaria se corre si aprobamos el arreglo hoy, versus qué costo o riesgo asume el negocio si decidimos patearlo.

**Ejemplo concreto**

Imaginemos el flujo de checkout o creación de pedidos corriendo sobre una consulta sin índices ni caché, o con una validación débil de sesión/token. El argumento frente al comité sería algo así: *"Hoy ese endpoint aguanta el tráfico normal, pero en una campaña o pico de demanda se degrada y empieza a tirar errores. Si nos tumba el flujo de compra por 2 horas en hora pico, perdemos alrededor de $X en ventas directas más las horas de todo el equipo apagando el incendio. Invertir 3 días de un desarrollador esta semana para indexar, cachear y blindar con pruebas cuesta una fracción de una sola caída y nos evita frenar el roadmap del próximo mes con hotfixes de emergencia."*

---

## 6. Estimación (Épica = Bloque 1, equipo de 5 personas)

* **Descomposición del Bloque 1:**

  *Pendiente de completar finalizado el Bloque 1.*
  
---
