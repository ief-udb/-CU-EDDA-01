# Documentación de Caso de Uso

---

## 1. Identificación y control del caso de uso

| **Campo** | **Descripción** |
|---|---|
| **ID del Caso de Uso** | CU-EDDA-01 |
| **Nombre del Caso de Uso** | Analítica de desempeño docente asistida por IaG y gestión de evaluaciones institucionales |
| **Fecha de creación** | 10/06/2026 |
| **Autor(es)** | Yeison Daniel Molina Monsalve — Universidad de Boyacá - IEF |
| **Prioridad del proceso** | Alta |
| **Estado** | En definición |

---

## 2. Descripción general del proceso

**Resumen:**
La Universidad de Boyacá aplica periódicamente la evaluación de desempeño por funcionario (formulario GRH-F-13) a sus docentes de tiempo completo, medio tiempo y hora cátedra. Actualmente el proceso termina con la firma del documento PDF generado por el SIIUB, sin que exista un mecanismo sistemático de análisis histórico, comparación entre pares ni generación de retroalimentación personalizada.

Este caso de uso describe la automatización del ciclo completo: desde la ingesta de los datos de evaluación provenientes del SIIUB, pasando por el cálculo de métricas derivadas (ETL), hasta la generación de informes narrativos de retroalimentación, detección de alertas tempranas y consolidados por unidad organizacional, todo ello asistido por Inteligencia Artificial Generativa (IaG), con validación humana obligatoria antes de cualquier publicación.

### 2.1. Diagrama de flujo del proceso (propuesto)

<!-- ```
[SIIUB — módulo de evaluación GRH-F-13 (Por definir)]
               │
               ▼
[ETL: ingesta, validación y normalización]
               │
               ▼
[Cálculo de métricas derivadas por registro]
   (promedios, delta, dispersión, flags)
               │
        ┌──────┴──────┐
        ▼             ▼
[Métricas          [Métricas
 por registro]      comparativas de grupo]
        │             │
        └──────┬───────┘
               ▼
[Motor IaG — generación de retroalimentación]
   (análisis de tendencias, plan de mejora,
    detección de inconsistencias texto/puntaje)
               │
               ▼
     [Revisión y validación del evaluador
      o jefe inmediato — OBLIGATORIA]
               │
        ┌──────┴──────┐
        ▼             ▼
[Informe          [Consolidado
 individual        por unidad /
 por docente]      programa / facultad]
        │             │
        └──────┬───────┘
               ▼
  [Alertas tempranas de riesgo docente]
               │
               ▼
   [Publicación según control de acceso
    por rol y unidad organizacional]
``` -->

![<Diagrama Bnmp>](<assets/bnmp.png>)
---

## 3. Actores

- **Actor primario:** Jefe inmediato / evaluador — inicia el procesamiento al cierre de cada periodo evaluativo; revisa, valida y publica los informes generados por la IaG. No modifica los puntajes registrados (eso lo hace el SIIUB).

- **Actores secundarios:**
  - **SIIUB** — fuente de datos de evaluaciones, datos maestros de docentes y estructura organizacional. El nuevo sistema consume sus datos en modo solo lectura, sin duplicar las funciones de registro.
  - **Servicio de autenticación del SIIUB** — provee roles e identidad de usuario al nuevo sistema. Credenciales locales provisionales durante desarrollo.
  - **Motor ETL** — extrae, transforma y carga los datos del SIIUB; calcula métricas derivadas determinísticas antes de enviarlas a la IaG.
  - **Motor de IaG** — genera retroalimentación narrativa, detecta patrones, sugiere planes de mejora y produce consolidados cualitativos a partir de las métricas precalculadas por el ETL.
  - **Docente evaluado** — receptor del informe individual; puede consultarlo según los permisos de su rol; no puede modificarlo.
  - **Decano / Director de programa / Vicerrector Académico** — receptores de consolidados y alertas a través del dashboard, dentro del alcance de su unidad organizacional.
  - **Administrador del sistema** — gestiona usuarios, roles, umbrales y sincronización con el SIIUB.

- **Nota:** Los textos libres de la entrevista (aspectos destacados, aspectos por mejorar, compromisos) no están estructurados en el SIIUB como campos separados. Se requiere que el SIIUB los exponga como texto plano; de lo contrario, el ETL los extrae del PDF mediante parsing. Este punto debe resolverse en la negociación de integración.

---

## 4. Precondiciones

1. La evaluación del periodo está registrada y firmada en el SIIUB (estado: completado) y disponible para consulta por el nuevo sistema vía conexión directa a BD o API REST, según el adaptador configurado.
2. Los datos maestros del docente (código, departamento, programa, tipo de vinculación) están sincronizados desde el módulo de RR.HH. del SIIUB.
3. La estructura jerárquica de unidades organizacionales y la relación docente–unidad (`es_jefe`) están cargadas y actualizadas en el sistema.
4. Los umbrales de clasificación de riesgo (p. ej. criterio crítico < 4.0, flag atípico z-score < −1.5) están parametrizados por el administrador.
5. El evaluador tiene credenciales válidas y el rol correspondiente con el alcance (`scope`) de su unidad organizacional.
6. **Precondición adicional para datos históricos:** si se procesan periodos anteriores al despliegue, el sistema los marca como `es_historico = true`. Las acciones de mejora generadas por IaG no se publican para datos históricos, pero sí se calculan clasificaciones y deltas (útil para construir series de tiempo y consolidados de tendencia desde el periodo 2023-10).

---

## 5. Flujo de eventos

| **Paso** | **Acción del actor** | **Respuesta del sistema / lógica de IaG** |
|---|---|---|
| 1 | El administrador o el sistema (según programación) inicia la sincronización al cierre del periodo evaluativo. | El ETL consulta el SIIUB, extrae los 23 criterios numéricos (**Pendiente definir**), los metadatos del registro y los tres textos libres de la entrevista. Valida que todos los campos obligatorios estén presentes y dentro del rango 1–5; reporta registros faltantes o fuera de rango sin abortar el proceso. |
| 2 | — (proceso automático) | El ETL calcula y persiste en `METRICAS_ETL` las métricas derivadas del registro: promedios por categoría (competencias, académico, SST, investigativo), criterio mínimo y máximo, desviación interna, conteo de criterios bajo umbral. |
| 3 | — (proceso automático, requiere ≥ 2 periodos) | El ETL calcula métricas de evolución: delta de puntaje entre periodos, delta por criterio, pendiente de tendencia (regresión lineal), racha de descenso consecutivo. |
| 4 | — (proceso automático, requiere datos del grupo completo) | El ETL calcula métricas comparativas de grupo: percentil por periodo, z-score por criterio, flag de criterio atípico (z < −1.5). |
| 5 | — (proceso automático, periodos activos) | El motor de IaG recibe las ~8 señales clave precalculadas (no los 23 valores crudos **Pendiente por Definir**) más los textos libres de la entrevista. Genera: (a) análisis de coherencia entre puntajes y texto libre, (b) identificación de los 2–3 criterios prioritarios de atención, (c) borrador de plan de mejora personalizado alineado con los compromisos declarados por el evaluado, (d) conclusión narrativa del periodo. La generación es asíncrona (< 15 s) y no bloquea el dashboard. Para datos históricos este paso se omite. |
| 6 | El jefe inmediato / evaluador revisa el borrador generado por la IaG en el dashboard. Puede aceptar, editar o rechazar cada sección antes de publicar. | El sistema registra la acción (aceptado / editado / rechazado) con marca de tiempo y usuario. El informe no se publica hasta que el evaluador valide explícitamente. |
| 7 | — (proceso automático) | El sistema genera el **informe individual** del docente: historial de niveles por criterio y periodo, métricas ETL, retroalimentación IaG validada, plan de mejora y compromisos. |
| 8 | — (proceso automático) | El sistema genera el **consolidado por unidad organizacional**: distribución de escalas (BUENO / EXCELENTE / etc.), tendencias por programa y facultad, ranking de criterios con mayor variación, alertas activas. Visible según el alcance del rol del solicitante. |
| 9 | — (proceso automático) | El sistema evalúa condiciones de alerta temprana: racha de descenso ≥ 2 periodos consecutivos en cualquier criterio, o flag atípico activo en criterio investigativo o académico. Genera alerta con estado `NUEVA`. Canal primario: badge en dashboard al iniciar sesión. Canal secundario: correo institucional para alertas de nivel crítico. Ciclo de vida de la alerta: `NUEVA → REVISADA → ATENDIDA`; el jefe debe registrar la acción tomada para cerrarla. |
| 10 | El evaluador (o el sistema, según configuración) publica o distribuye los informes a los destinatarios según los permisos y canales definidos. | El sistema distribuye los informes respetando la matriz de roles: el docente accede solo a su propio informe; el jefe inmediato a los de sus evaluados; el decano a los de su facultad; el directivo a los consolidados agregados. |

---

## 6. Flujos alternativos y excepciones

### 6.1. Flujos alternativos (variaciones)

- **A1 — Consulta histórica:** Un directivo, decano o director de programa solicita un consolidado de periodos anteriores. El sistema recupera los datos almacenados y genera el informe sin recalcular métricas ETL ya persistidas.
- **A2 — Carga manual de PDF:** Si la integración con el SIIUB no está disponible, el administrador carga el PDF del formulario GRH-F-13 directamente. El ETL extrae los datos mediante parsing del documento. Este modo se registra en el log como origen `PDF_MANUAL` para trazabilidad.
- **A3 — Procesamiento de periodo histórico:** El administrador marca el periodo como `es_historico = true`. El sistema calcula métricas y construye series de tiempo, pero omite la generación de planes de mejora por IaG y no genera alertas nuevas.
- **A4 — Revalidación del informe:** Si el evaluador rechaza el borrador de IaG en su totalidad, puede solicitar una nueva generación con instrucciones adicionales (campo de contexto libre). El sistema registra ambas versiones con su estado.

### 6.2. Excepciones (errores)

- **E1 — Datos maestros del docente no sincronizados:** Si el código del docente no existe en la tabla `DOCENTE`, el ETL no puede vincular el registro. El sistema notifica al administrador y mantiene el registro en estado `PENDIENTE_SINCRONIZACION`.
- **E2 — Textos libres ausentes o en formato imagen:** Si los campos de entrevista no están disponibles como texto plano, el sistema marca los campos como `NO_DISPONIBLE` y genera el informe numérico sin componente narrativo, notificando al evaluador.
- **E3 — Fallo del motor de IaG:** Si la Claude API no responde dentro del tiempo esperado o retorna un error, el sistema registra el fallo en el log, notifica al administrador técnico y mantiene disponible el informe numérico con las métricas ETL. La retroalimentación narrativa queda en estado `PENDIENTE_GENERACION` para reintento manual.
- **E4 — Fallo de generación del consolidado:** Si el motor de informes no puede generar el documento (datos insuficientes, error de plantilla), el sistema registra el error, notifica al administrador técnico y mantiene el último consolidado válido disponible.
- **E5 — Umbral de confianza no alcanzado (componente predictivo — fase 2):** Si el modelo de detección de riesgo docente retorna un score < 0.50, no se genera alerta visible. Score 0.50–0.74: indicador suave "Señal de atención — revisión recomendada". Score ≥ 0.75: alerta visible "Criterio en riesgo sostenido". En todos los casos la alerta es informativa y requiere revisión humana; nunca desencadena acción automática sobre el docente ni sobre su evaluación.

---

## 7. Postcondiciones

**Éxito:**
- Los 23 criterios **Pendiente por Definir** y las métricas ETL derivadas están calculados y almacenados para el periodo procesado.
- El informe individual está disponible para el docente evaluado y su jefe inmediato según la matriz de acceso.
- El consolidado por unidad organizacional está disponible para el directivo correspondiente.
- Las alertas de riesgo (si aplica) han sido enviadas a los destinatarios definidos y tienen estado `NUEVA` en el dashboard.
- El estado del proceso queda registrado como `COMPLETADO` o `VALIDADO` (si el evaluador revisó el borrador de IaG).

**Fallo:**
- Los datos procesados parcialmente se marcan como `BORRADOR_INCOMPLETO`; no se publican.
- El sistema mantiene el último estado válido disponible.
- Se genera un registro de auditoría con la causa del fallo, el usuario y la marca de tiempo.

---

## 8. Esquema de base de datos del aplicativo

El sistema gestiona las siguientes entidades principales. Las tablas marcadas como **[SIIUB]** se alimentan exclusivamente desde el sistema institucional; las marcadas **[ETL]** son calculadas por el proceso de transformación; las marcadas **[IaG]** son generadas por el motor de inteligencia artificial.

### 8.1. Entidades principales

| Tabla | Origen | Descripción |
|---|---|---|
| `DOCENTE` | SIIUB | Datos maestros del funcionario: código, nombre, departamento, programa, tipo de vinculación, fecha de ingreso. |
| `EVALUADOR` | SIIUB | Datos del jefe inmediato que realiza la evaluación. |
| `PERIODO` | SIIUB | Ciclo académico evaluado: código (ej. 202610), fecha inicio y fin. |
| `EVALUACION` | SIIUB | Registro central: vincula docente, evaluador y periodo; contiene puntaje total y escala. |
| `PUNTAJES_CRITERIOS` | SIIUB | Los 23 criterios individuales en DECIMAL(3,2) por evaluación. |
| `TEXTO_ENTREVISTA` | SIIUB | Tres campos TEXT: aspectos destacados, aspectos por mejorar, compromisos. Deben entregarse como texto plano. |
| `METRICAS_ETL` | ETL | Métricas derivadas calculadas automáticamente: promedios por categoría, criterio mínimo/máximo, desviación interna, delta entre periodos, tendencia (pendiente), racha de descenso, percentil de grupo, z-score, flag atípico. |
| `INFORME_IAG` | IaG | Borrador narrativo generado: análisis de coherencia, criterios prioritarios, plan de mejora, conclusión. Registra estado (BORRADOR / VALIDADO / RECHAZADO) y la acción del evaluador. |
| `UNIDAD_ORGANIZACIONAL` | SIIUB | Jerarquía de facultades, departamentos y programas con relación padre–hijo y nivel. |
| `DOCENTE_UNIDAD` | SIIUB | Relación docente–unidad con indicador `es_jefe` y vigencia temporal. |
| `ROL` | Sistema | Roles disponibles: Administrador, Directivo, Decano/Director, Jefe inmediato, Docente. Incluye `nivel_jerarquico`. |
| `USUARIO` | Sistema | Cuenta de acceso vinculada al docente y a un rol. |
| `SCOPE_ACCESO` | Sistema | Delimita a qué unidad organizacional tiene acceso cada usuario según su rol. |
| `ROL_PERMISO` | Sistema | Tabla de unión entre roles y permisos granulares (recurso + acción). |
| `ALERTA` | ETL / IaG | Alertas generadas: tipo, estado (`NUEVA → REVISADA → ATENDIDA`), destinatario, acción registrada. |
| `AUDITORIA` | Sistema | Log de accesos y acciones: usuario, recurso consultado, acción, marca de tiempo. Retención mínima 5 años. |

### 8.2. Métricas calculadas por el ETL (no se solicitan al SIIUB)

Las siguientes variables se derivan de los datos recibidos y se persisten en `METRICAS_ETL` para reducir la carga computacional de la IaG:

**Por registro (requiere 1 periodo):**
- `prom_competencias`, `prom_academico`, `prom_sst`, `prom_investigativo` — AVG de criterios por categoría.
- `criterio_min_valor` / `criterio_min_nombre` — criterio con puntaje más bajo.
- `criterio_max_valor` / `criterio_max_nombre` — criterio con puntaje más alto.
- `desviacion_intra` — STDEV de los 23 criterios **Pendiente por Definir**; alta dispersión indica perfil mixto.
- `criterios_bajo_umbral` — COUNT de criterios con valor < umbral configurable (default 4.0).

**De evolución (requiere ≥ 2 periodos):**
- `delta_puntaje` — diferencia del puntaje global respecto al periodo anterior (`LAG()`).
- `delta_por_criterio` — vector de diferencias por cada uno de los 23 criterios **Pendiente por Definir**.
- `tendencia_pendiente` — pendiente de regresión lineal del puntaje a lo largo de N periodos.
- `racha_descenso` — periodos consecutivos con delta negativo; señal de alerta temprana.

**Comparativas de grupo (requiere el grupo completo del periodo):**
- `percentil_grupo` — posición relativa del docente dentro de su unidad (`PERCENT_RANK()`).
- `z_score_investigativo` — desviación del criterio más sensible respecto a la media del grupo.
- `flag_atipico` — booleano: 1 si z-score < −1.5 en cualquier criterio.

---

## 9. Control de acceso por rol y unidad organizacional

El sistema implementa un modelo RBAC (Role-Based Access Control) extendido con alcance por unidad organizacional. Un usuario tiene un **rol** (qué puede hacer) y un **scope** (sobre qué unidad puede hacerlo). El mismo rol puede tener alcances distintos según la unidad a la que está adscrito el usuario.

### 9.1. Roles definidos

| Rol | Nivel jerárquico | Descripción |
|---|---|---|
| Administrador | 1 | Acceso total. Gestiona usuarios, roles, ETL, umbrales y auditoría. |
| Directivo | 2 | Rector / Vicerrector Académico. Ve consolidados agregados y tendencias institucionales. |
| Decano / Director | 3 | Acceso a evaluaciones y consolidados de su facultad o programa. |
| Jefe inmediato | 4 | Acceso a evaluaciones de sus evaluados directos. Valida y publica informes. |
| Docente | 5 | Acceso exclusivo a su propia evaluación e informe. |

### 9.2. Matriz de permisos por recurso

| Recurso / acción | Administrador | Directivo | Decano/Dir. | Jefe inmediato | Docente |
|---|---|---|---|---|---|
| Ver evaluación completa (23 criterios) | ✓ Total | ✓ Total | Su facultad | Sus evaluados | Solo la propia |
| Ver consolidados y rankings | ✓ Total | ✓ Total | Su facultad | — | — |
| Ver texto libre (aspectos por mejorar / compromisos) | ✓ Total | — | Su facultad | Sus evaluados | Solo el propio |
| Ver informe narrativo IaG | ✓ Total | — | Su facultad | Sus evaluados | Solo el propio |
| Ver alertas y flags de riesgo | ✓ Total | ✓ Total | Su facultad | Sus evaluados | — |
| Validar / publicar informe IaG | ✓ Total | — | — | Sus evaluados | — |
| Ejecutar ETL / sincronizar SIIUB | ✓ Total | — | — | — | — |
| Gestionar usuarios y roles | ✓ Total | — | — | — | — |
| Consultar log de auditoría | ✓ Total | — | — | — | — |
| Configurar umbrales y parámetros | ✓ Total | — | — | — | — |

**Regla especial:** El Rector / Vicerrector accede a puntajes globales y tendencias agregadas, pero **no** a los textos libres de entrevistas individuales ni a los informes narrativos de docentes específicos. Esto evita el uso de la herramienta para microgestión fuera del canal jerárquico natural.

---

## 10. Datos requeridos del SIIUB

La integración con el SIIUB es de **solo lectura**. El aplicativo no escribe ningún dato en el sistema institucional.

### 10.1. Módulo de Recursos Humanos

| Campo | Variable destino | Formato **(Por definir)** | Prioridad |
|---|---|---|---|
| Código único del funcionario | `codigo_siiub` | INT / VARCHAR | Alta |
| Nombres y apellidos completos | `nombres`, `apellidos` | VARCHAR | Alta |
| Departamento / facultad | `departamento` | VARCHAR (catálogo) | Alta |
| Programa académico asignado | `programa` | VARCHAR (catálogo) | Alta |
| Tipo de vinculación (TC / MT / HC) | `tipo_vinculacion` | ENUM | Alta |
| Fecha de vinculación institucional | `fecha_ingreso` | DATE | Media |
| Estado activo / inactivo | `activo` | BOOLEAN | Media |
| Unidad organizacional de adscripción | `id_unidad` | INT (FK) | Alta |
| Indicador de jefatura en la unidad | `es_jefe` | BOOLEAN | Alta |

### 10.2. Módulo de Evaluación

| Campo | Variable destino | Formato **(Por definir)** | Prioridad |
|---|---|---|---|
| Código del evaluador | `evaluador.codigo_siiub` | INT | Alta |
| Periodo evaluado (inicio y fin) | `periodo_inicio`, `periodo_fin` | DATE | Alta |
| Código periodo académico | `codigo_academico` | VARCHAR (ej. 202610) | Alta |
| Puntaje total | `puntaje_total` | DECIMAL(3,2) | Alta |
| Escala cualitativa | `escala` | VARCHAR (catálogo) | Alta |
| Código y versión del formulario | `formulario_codigo`, `version` | VARCHAR | Media |
| Valor de cada uno de los 23 criterios | `puntajes_criterios.*` | DECIMAL(3,2) × 23 | Alta |

### 10.3. Textos libres de la entrevista

Estos campos deben entregarse como **texto plano estructurado**. Si el SIIUB solo los expone como PDF, el ETL los extrae mediante parsing, con degradación de calidad registrada en el log.

| Campo | Variable destino | Formato requerido | Prioridad |
|---|---|---|---|
| Aspectos destacados del evaluado | `aspectos_destacados` | TEXT plano | Alta |
| Aspectos por mejorar | `aspectos_mejorar` | TEXT plano | Alta |
| Compromisos del evaluado | `compromisos` | TEXT plano | Alta |

### 10.4. Estructura organizacional

| Campo | Variable destino | Formato | Prioridad |
|---|---|---|---|
| Jerarquía de unidades (facultad → departamento → programa) | `UNIDAD_ORGANIZACIONAL` | Árbol con `id_padre` | Alta |
| Relación docente–unidad con indicador de jefatura | `DOCENTE_UNIDAD` | Tabla con vigencia temporal | Alta |

---

## 11. Reglas de negocio y requerimientos especiales

### Reglas de negocio

- **RN-01 — Escala de evaluación institucional (GRH-F-13):**
  - EXCELENTE: puntaje ≥ 4.6
  - BUENO: 4.0 ≤ puntaje < 4.6
  - ACEPTABLE: 3.5 ≤ puntaje < 4.0
  - DEFICIENTE: puntaje < 3.5

- **RN-02 — Criterio crítico:** Todo criterio con valor < 4.0 se marca como crítico en el informe individual. Si hay ≥ 3 criterios críticos en el mismo registro, se activa flag de alerta.

- **RN-03 — División de responsabilidades ETL / IaG:** El ETL calcula todas las métricas determinísticas (promedios, deltas, percentiles, z-scores). La IaG recibe únicamente las señales precalculadas (~8 variables clave) más los textos libres; nunca procesa los 23 valores crudos directamente. Esto reduce el costo de tokens, mejora la consistencia y hace el análisis auditable.

- **RN-04 — Validación humana obligatoria:** Ningún informe narrativo generado por la IaG se publica sin que el jefe inmediato lo haya revisado y validado explícitamente. El sistema registra la acción (aceptado / editado / rechazado) con marca de tiempo y usuario.

- **RN-05 — Compromisos como insumo de seguimiento:** Los compromisos declarados por el evaluado en la entrevista se almacenan en `TEXTO_ENTREVISTA.compromisos` y son utilizados por la IaG en el siguiente periodo para evaluar coherencia entre lo comprometido y los cambios observados en los criterios.

- **RN-06 — Alcance temporal del análisis:** El sistema procesa registros desde el periodo 2023-10. Los periodos anteriores al despliegue se cargan como históricos (`es_historico = true`) y sirven exclusivamente para construir series de tiempo; no generan alertas ni planes de mejora.

- **RN-07 — Comparativa entre docentes:** La vista comparativa (rankings, percentiles, distribución por criterio) es de uso exclusivo del Decano, Director de programa o Administrador dentro de su unidad. No se expone al jefe inmediato individual ni al docente, para evitar comparaciones sin contexto.

- **RN-08 — Análisis transdisciplinar por unidad:** Para criterios que involucran docentes de múltiples programas (p. ej. producción investigativa en ciencias básicas), el consolidado permite segmentar por unidad de adscripción del docente, no solo por asignatura, usando la jerarquía de `UNIDAD_ORGANIZACIONAL`.

### Requerimientos no funcionales

- **Privacidad y Habeas Data:** El tratamiento de datos de desempeño docente debe cumplir la Ley 1581 de 2012 (Colombia) y la política de tratamiento de datos institucional. Los informes individuales son accesibles únicamente por el rol y scope autorizados.
- **Disponibilidad:** 99.5 % durante ventanas críticas (últimos 5 días de cada cierre de periodo evaluativo; máx. 3.6 h de caída/mes). 99.0 % en periodo normal. Mantenimiento programado con notificación 48 h de antelación, nunca en ventana crítica.
- **Latencia:** Cálculo de métricas ETL por registro: < 2 s. Generación de informe individual: < 5 s. Consolidado por unidad (≤ 200 docentes): < 20 s. Retroalimentación IaG por docente: < 15 s de forma asíncrona, sin bloquear el dashboard.
- **Trazabilidad:** Cada cálculo ETL, cada llamada a la IaG y cada versión de informe queda registrado con marca de tiempo, usuario que lo ejecutó y fuente de datos utilizada. Logs de acceso retenidos mínimo 5 años (Ley 1581/2012).
- **Interoperabilidad:** Patrón adaptador con dos implementaciones: (a) conexión directa a BD del SIIUB en modo solo lectura (implementación inicial); (b) API REST con OAuth2 Client Credentials cuando el SIIUB la exponga (implementación futura). La lógica de negocio no depende de cuál esté activa.

---

## 12. Identificación de riesgos y manejo

### Riesgos de IA y automatización

| **ID** | **Riesgo** | **Impacto** | **Probabilidad** | **Medida de mitigación** |
|---|---|---|---|---|
| R-01 | Clasificación errónea por puntaje mal registrado en el SIIUB | Alto | Media | Validación de rangos en ingesta; bloqueo de publicación si hay valores fuera de rango 1–5. |
| R-02 | La IaG genera retroalimentación incoherente con el perfil real del docente | Alto | Media | Validación humana obligatoria antes de publicar; registro de rechazos para auditoría. |
| R-03 | Acceso no autorizado a informes individuales | Alto | Baja | RBAC con scope por unidad; cifrado en tránsito (TLS) y en reposo; log de auditoría con retención 5 años. |
| R-04 | Textos libres no disponibles como texto plano desde el SIIUB | Medio | Alta | Parsing de PDF como fallback; registro del origen en log; notificación al administrador de integración. |
| R-05 | Sesgo del modelo de IaG en criterios con alta variabilidad contextual (ej. producción investigativa en docentes de HC) | Alto | Media | Las métricas comparativas segmentan por tipo de vinculación; la IaG recibe este contexto como campo explícito. Auditoría periódica de outputs. |
| R-06 | Dependencia de la estructura del formulario GRH-F-13; cambios de versión rompen el ETL | Medio | Media | Versionado del formulario (`formulario_version` en la BD); validación automática de estructura al importar. |
| R-07 | ⚠️ Uso del informe como insumo en procesos disciplinarios o de desvinculación | Alto | Media | Restricción explícita en política de uso; capacitación obligatoria antes del primer acceso; el sistema registra aceptación de términos con marca de tiempo. |

### Consideraciones éticas y bioéticas

- Las alertas de riesgo docente deben usarse como herramienta de acompañamiento y desarrollo profesional, **nunca como mecanismo de sanción automática**.
- Los informes individuales deben presentarse en lenguaje constructivo, evitando etiquetado estigmatizante. La IaG debe recibir instrucciones explícitas sobre tono en el prompt del sistema.
- **Protocolo de uso por parte del jefe inmediato:** (1) Los informes se usan exclusivamente en sesiones de retroalimentación con el docente, no en procesos disciplinarios. (2) El jefe informa al docente que se generó el informe y explica su propósito antes de la sesión. (3) Los datos de un docente no se comparten con otros docentes. (4) Los informes no son insumo para procesos de desvinculación ni sancionatorios. (5) Capacitación obligatoria en uso responsable del sistema antes del primer acceso.
- Los outputs de la IaG son siempre **sugerencias revisables**, no dictámenes. El sistema debe dejar claro en la interfaz que el contenido fue generado por inteligencia artificial y validado por un evaluador humano.

---