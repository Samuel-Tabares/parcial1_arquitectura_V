# Propuesta de Diseño Arquitectónico — SaludEje

**Autor:** Samuel Tabares
**Fecha:** Mayo 2026
**Versión:** 1.0 (en construcción)

---

## Introducción

SaludEje es una plataforma de integración de servicios de salud para los hospitales públicos del Eje Cafetero. El problema central no es tecnológico: es que tres departamentos, con sus propios sistemas, sus propios flujos y su propia historia, deben empezar a operar como si fueran uno solo, sin interrumpir la atención de pacientes ni reemplazar la infraestructura existente.

El diseño arquitectónico propuesto en este documento parte de un principio: **las decisiones de arquitectura deben servir al sistema real, no a un sistema ideal**. SaludEje tiene un equipo sin experiencia en salud, un presupuesto limitado y 12 meses de desarrollo. El diseño debe ser ambicioso en lo funcional y conservador en lo operacional. Cada decisión está argumentada contra alternativas técnicamente válidas, y cada trade-off está explicitado.

**5 dominios del sistema:**
1. **Historia Clínica Electrónica (HCE)** — dominio más crítico; errores pueden costar vidas
2. **Agendamiento y consultas** — picos extremos (40% de la demanda semanal el lunes 6-8am)
3. **Laboratorio y diagnóstico** — múltiples laboratorios externos con formatos heterogéneos
4. **Farmacia hospitalaria** — inventario por hospital con consulta inter-hospitalaria para urgencias
5. **Facturación y glosas** — integración con EPS (Sura, Coosalud, Nueva EPS) de APIs distintas

**Actores externos:** Pacientes, médicos, enfermeras, laboratoristas, farmacéuticos, administradores hospitalarios, EPS (Sura, Coosalud, Nueva EPS), MINSALUD, laboratorios clínicos externos.

**Estructura del documento:**
- **A. Drivers arquitectónicos** — las fuerzas que condicionan todas las decisiones
- **B. Atributos de calidad** — cómo debe comportarse el sistema y bajo qué condiciones
- **C. ADRs** — el registro de las decisiones más significativas y por qué se tomaron
- **D. Modelos arquitectónicos** — la estructura del sistema y la lógica detrás de cada elección
- **E. Diagrama de arquitectura de alto nivel** — la vista integrada del sistema

---

## A. Drivers Arquitectónicos

Los drivers arquitectónicos son los requerimientos que más impactan las decisiones de diseño. No son una lista de todo lo que debe hacer el sistema — son los requerimientos cuyo cambio obligaría a rediseñar la arquitectura. La prueba para identificar un driver es directa: *si este requerimiento desaparece o cambia sustancialmente, ¿hay que rediseñar la arquitectura?* Si la respuesta es sí, es un driver.

Se clasifican en dos categorías: **funcionales** (casos de uso con alto impacto estructural) y **de restricción** (límites que el diseño no puede ignorar).

---

### Drivers Funcionales

#### DF1 — Acceso cross-hospital a la historia clínica en tiempo real

**Descripción:** Un paciente atendido en cualquier hospital del Eje Cafetero debe tener su historia clínica completa disponible para el médico tratante, independientemente de en qué hospital fue registrada originalmente. El caso que origina SaludEje es precisamente este: un paciente de Armenia que se accidenta en Manizales llega a urgencias sin historial disponible.

**Por qué es un driver:** Este requerimiento define la naturaleza distribuida del sistema. Si solo se necesitara acceso dentro del mismo hospital, cada HIS existente sería suficiente y SaludEje no tendría razón de existir como plataforma de integración. El hecho de que el acceso sea cross-hospital obliga a tener un modelo de datos centralizado para la HCE (o al menos federado con sincronización en tiempo real), una capa de API unificada que abstraiga el origen del dato, y una estrategia de identidad del paciente que funcione entre hospitales (el mismo paciente puede tener IDs distintos en el HIS de Manizales y en el de Armenia). Cualquier cambio en este requerimiento — por ejemplo, si se acepta que la HCE solo esté disponible con 24 horas de retraso — cambiaría radicalmente la estrategia de sincronización y el modelo de consistencia del sistema.

**Decisiones que condiciona:** Modelo centralizado de HCE con adaptadores por HIS local, arquitectura hexagonal en el módulo HCE para abstraer el origen del dato, API Gateway como punto de entrada único, estrategia de identidad federada del paciente.

---

#### DF2 — Gestión de picos extremos de demanda en agendamiento

**Descripción:** El lunes entre 6am y 8am concentra el 40% de las agendas de toda la semana. En ese intervalo de dos horas, el sistema debe manejar una carga equivalente a la de dos días y medio de operación normal. La indisponibilidad en ese horario tiene impacto directo en la atención de pacientes.

**Por qué es un driver:** La distribución de carga no uniforme cambia la estrategia de escalado. Si la carga fuera uniforme durante la semana, un dimensionamiento fijo sería suficiente. El pico extremo y predecible del lunes obliga a una estrategia de escalado elástico que responda rápido y a un diseño del módulo que separe el cuello de botella real (lecturas de disponibilidad) de las escrituras. Si este requerimiento desapareciera — si la Secretaría distribuyera los turnos de agendamiento para evitar el pico — el CQRS en el módulo de Agendamiento y el auto-scaling serían sobrediseño innecesario.

**Decisiones que condiciona:** CQRS en el módulo de Agendamiento (lectura y escritura con modelos independientes), auto-scaling horizontal del monolito, caché de disponibilidad en el modelo de lectura.

---

#### DF3 — Notificación inmediata de resultados de laboratorio al médico tratante

**Descripción:** Cuando un laboratorio externo carga el resultado de un examen clínico, el médico tratante debe ser notificado inmediatamente, sin necesidad de que el médico consulte activamente el sistema. El flujo es: laboratorio carga resultado → sistema detecta el evento → médico recibe la notificación.

**Por qué es un driver:** La palabra "inmediata" y la inversión del flujo (push, no pull) obligan a un modelo de comunicación asíncrono basado en eventos. Si el requerimiento fuera "el médico puede consultar los resultados cuando lo desee", bastaría con un endpoint REST que el médico llama manualmente. La notificación push implica que el sistema debe saber quién es el médico tratante del paciente, debe tener un canal de notificación activo (websocket, push notification, SMS), y debe procesar el evento del laboratorio en tiempo real. Esto es lo que justifica la existencia del bus de eventos interno y del módulo de Notificaciones.

**Decisiones que condiciona:** Bus de eventos interno con el evento `ResultadoDisponible`, módulo de Notificaciones como consumidor, ACL en el módulo de Laboratorio para normalizar los formatos heterogéneos de los laboratorios externos antes de emitir el evento.

---

#### DF4 — Integración con sistemas heterogéneos externos sin reemplazarlos

**Descripción:** SaludEje debe integrarse con: los HIS existentes de cada hospital (distintos entre sí, con bases de datos propias que no pueden modificarse), las APIs de las EPS (Sura, Coosalud y Nueva EPS tienen formatos y protocolos distintos), y los laboratorios clínicos externos (mezcla de HL7, CSV, APIs propias, y otros formatos). Ninguno de estos sistemas puede ser reemplazado en la primera fase.

**Por qué es un driver:** La heterogeneidad de los sistemas externos define la necesidad del patrón ACL y de los adaptadores. Si todos los sistemas externos hablaran el mismo protocolo (por ejemplo, HL7 FHIR estándar), los adaptadores serían triviales o innecesarios. El hecho de que cada EPS tenga su propia API y que cada hospital tenga su propio HIS obliga a que SaludEje tenga una capa de integración explícita y extensible: agregar un nuevo hospital o una nueva EPS debe ser posible sin modificar la lógica de negocio central. Si este requerimiento cambiara — si se pudiera exigir a todos los actores que adoptaran un estándar común — la arquitectura de integración simplificaría radicalmente.

**Decisiones que condiciona:** Anti-Corruption Layer en los módulos de Facturación y Laboratorio, adaptadores por HIS en la capa de integración, diseño extensible que permita añadir nuevos adaptadores sin modificar el core.

---

#### DF5 — Portabilidad y acceso auditado a la historia clínica (Resolución 2654)

**Descripción:** La Resolución 2654 de 2019 establece que la Historia Clínica Electrónica debe garantizar auditabilidad (cada acceso queda registrado con quién, cuándo y desde dónde), inmutabilidad (los registros clínicos no pueden modificarse ni eliminarse, solo adicionarse) y portabilidad (el paciente puede solicitar su historia en un formato interoperable). Estos no son requerimientos opcionales: son obligaciones legales.

**Por qué es un driver:** Estos tres atributos — auditoría, inmutabilidad y portabilidad — no pueden implementarse como una capa superficial encima de una arquitectura cualquiera. Requieren decisiones estructurales: el log de auditoría debe ser un puerto obligatorio en la arquitectura hexagonal (no una opción que alguien puede omitir), el modelo de datos de la HCE no puede usar operaciones UPDATE sobre registros clínicos (solo INSERT), y debe existir un puerto de exportación en formato estándar. Si la Resolución 2654 no existiera, la arquitectura hexagonal del módulo HCE seguiría siendo una buena práctica, pero ya no sería una necesidad estructural impuesta por la normativa.

**Decisiones que condiciona:** Arquitectura hexagonal con puerto de auditoría obligatorio en HCE, modelo de datos append-only para registros clínicos, puerto de exportación en formato HL7 FHIR o equivalente.

---

### Drivers de Restricción

#### DR1 — Soberanía del dato: datos clínicos solo en territorio colombiano

**Descripción:** La Secretaría de Salud exige que los datos clínicos de los pacientes permanezcan en servidores ubicados en territorio colombiano. Esta restricción está respaldada por la Ley 1581 de 2012 y es una condición no negociable de la licitación.

**Por qué es una restricción arquitectónica:** Esta restricción elimina directamente las opciones de cloud público internacional (AWS us-east-1, Azure West Europe, GCP us-central1) para cualquier dato clínico. Las opciones quedan limitadas a: proveedores cloud con región certificada en Colombia (AWS anunció región en Colombia; también existen proveedores locales como ETB Cloud o Claro Cloud), servidores on-premise en los hospitales o en datacenter colombiano certificado, o una arquitectura híbrida donde los datos clínicos van a infraestructura colombiana y los datos no clínicos (logs de acceso, analytics, configuración) pueden ir a cloud internacional. Esta restricción también impacta la estrategia de backup y recuperación ante desastres: el backup también debe estar en territorio colombiano.

**Decisiones que condiciona:** Infraestructura híbrida (datos clínicos en nube colombiana o on-premise; servicios de cómputo pueden estar en cloud), estrategia de backup en datacenter colombiano, exclusión de opciones de cloud internacional para la BD de HCE.

---

#### DR2 — No reemplazar los HIS existentes en fase 1

**Descripción:** Cada hospital del Eje Cafetero ya tiene su propio Sistema de Información Hospitalario (HIS) con bases de datos propias y flujos operacionales activos. Estos sistemas no pueden ser reemplazados durante la primera fase de 18 meses. SaludEje debe integrarse con ellos como una capa adicional.

**Por qué es una restricción arquitectónica:** Esta restricción descarta la opción de una base de datos centralizada única como punto de partida. Si se pudiera reemplazar los HIS, se podría migrar todos los datos a un modelo unificado y el problema de integración desaparecería. Al no poder reemplazarlos, el sistema debe mantener los HIS como fuentes de verdad para sus datos históricos y construir una capa de sincronización o federación encima. Esto también significa que SaludEje no puede asumir nada sobre el modelo de datos de los HIS existentes — puede cambiar sin previo aviso, puede tener inconsistencias, puede tener formatos no estándar. El adaptador por hospital en la capa de integración es la consecuencia directa de esta restricción.

**Decisiones que condiciona:** Adaptadores por HIS en la capa de integración, diseño de la HCE como capa federada sobre los datos existentes (no como reemplazo), estrategia de sincronización incremental de datos históricos.

---

#### DR3 — Equipo sin experiencia en el dominio de salud y presupuesto limitado

**Descripción:** Los 12 ingenieros contratados no tienen experiencia previa en sistemas de salud ni en la normativa del sector colombiano. El presupuesto total para desarrollo e implementación de la primera fase es de $850 millones de pesos colombianos (~200.000 USD), distribuidos en 18 meses.

**Por qué es una restricción arquitectónica:** Esta restricción pone un techo a la complejidad operacional aceptable. Un diseño técnicamente superior pero que requiera un nivel de expertise que el equipo no tiene al inicio del proyecto es un diseño fallido para este contexto. Fue el argumento definitivo para descartar microservicios en fase 1: el overhead operacional de una arquitectura distribuida madura (service mesh, distributed tracing, sagas) habría consumido la capacidad del equipo en infraestructura en lugar de en dominio. El presupuesto limitado también descarta opciones de licenciamiento costoso (ESB comerciales como MuleSoft, soluciones SaaS de integración de salud) y favorece tecnologías open source mantenibles por el equipo.

**Decisiones que condiciona:** Monolito Modular sobre microservicios, preferencia por tecnologías open source y ecosistemas con amplia documentación, arquitectura diseñada para ser comprensible por un equipo que está aprendiendo el dominio simultáneamente.

---

#### DR4 — Continuidad operacional: el sistema no puede interrumpir la atención en curso

**Descripción:** Los hospitales del Eje Cafetero atienden pacientes las 24 horas. La implantación de SaludEje no puede causar períodos de indisponibilidad que interrumpan la atención médica en curso. Los HIS existentes deben seguir operando durante toda la fase de integración.

**Por qué es una restricción arquitectónica:** Esta restricción descarta las migraciones big-bang (apagar el sistema viejo, encender el nuevo). La integración debe ser incremental: SaludEje se agrega como capa adicional mientras los HIS siguen operando, la sincronización de datos históricos debe hacerse en paralelo sin bloquear los sistemas existentes, y el rollout debe poder revertirse por hospital sin afectar a los demás. Esto refuerza el patrón Strangler Fig para la migración y obliga a que los adaptadores de HIS sean tolerantes a fallos del HIS subyacente (si el HIS de un hospital falla, los otros hospitales no se ven afectados).

**Decisiones que condiciona:** Estrategia de rollout incremental por hospital, adaptadores HIS con manejo de fallos independiente, diseño de la sincronización de datos históricos como proceso background no bloqueante.

---

### Tabla resumen de drivers y su impacto

| Driver | Tipo | Decisión arquitectónica que condiciona |
|---|---|---|
| DF1 — Acceso cross-hospital a HCE | Funcional | HCE centralizada, API Gateway, identidad federada del paciente |
| DF2 — Picos extremos de agendamiento | Funcional | CQRS en Agendamiento, auto-scaling, caché de disponibilidad |
| DF3 — Notificación inmediata de laboratorio | Funcional | Bus de eventos interno, módulo Notificaciones, ACL en Laboratorio |
| DF4 — Integración con sistemas heterogéneos | Funcional | ACL en Facturación y Laboratorio, adaptadores por HIS |
| DF5 — Auditoría, inmutabilidad y portabilidad (Res. 2654) | Funcional | Hexagonal con puerto de auditoría obligatorio, modelo append-only en HCE |
| DR1 — Soberanía del dato | Restricción | Infraestructura colombiana para datos clínicos, arquitectura híbrida |
| DR2 — No reemplazar HIS existentes | Restricción | Adaptadores por HIS, integración como capa adicional, rollout incremental |
| DR3 — Equipo sin experiencia + presupuesto limitado | Restricción | Monolito Modular, tecnologías open source, complejidad operacional mínima |
| DR4 — Continuidad operacional | Restricción | Rollout incremental por hospital, adaptadores tolerantes a fallos del HIS |

---

## B. Atributos de Calidad con Escenarios

Los atributos de calidad describen *cómo* debe comportarse el sistema, no *qué* debe hacer. Para SaludEje se identificaron cuatro atributos críticos, cada uno justificado en el contexto específico del sistema y desarrollado con un escenario formal de seis componentes. Al final de la sección se explicitan los trade-offs entre atributos — porque optimizar uno siempre tiene un costo en otro.

---

### AQ1 — Disponibilidad

**Por qué es crítico para SaludEje:** La indisponibilidad de la HCE durante una urgencia médica puede derivar en una decisión clínica incorrecta. Un médico de urgencias que no puede acceder al historial de alergias de un paciente inconsciente puede administrar un medicamento que lo mate. A diferencia de la mayoría de los sistemas de software, donde la indisponibilidad es un inconveniente económico, en SaludEje la indisponibilidad de la HCE es un riesgo de vida. El módulo de Agendamiento tiene un perfil distinto: su indisponibilidad en el pico del lunes impacta directamente el acceso a la atención, pero el riesgo es operacional, no clínico.

**Escenario de calidad — Falla del nodo primario en horario nocturno:**

| Componente | Descripción |
|---|---|
| **Fuente del estímulo** | Falla de hardware en el servidor primario de la HCE |
| **Estímulo** | El nodo primario deja de responder a las 2:17am durante una atención de urgencias activa |
| **Artefacto** | Módulo de Historia Clínica Electrónica |
| **Entorno** | Operación normal, carga baja (horario nocturno), al menos un médico con una sesión activa consultando la HCE de un paciente en urgencias |
| **Respuesta** | El sistema detecta la falla, conmuta automáticamente al nodo réplica activo (hot standby), y mantiene la sesión del médico sin interrupción visible |
| **Medida de respuesta** | Tiempo de conmutación automática < 30 segundos; cero pérdida de datos; el médico no recibe un error — como máximo una pantalla de carga de 5-10 segundos; disponibilidad mensual de la HCE ≥ 99.9% (máximo 44 minutos de downtime al mes) |

**Táctica aplicada:** Redundancia activa (hot standby) con health checks continuos y failover automático. La réplica está en un datacenter colombiano distinto al primario para que una falla de datacenter no tumbe ambos nodos.

---

### AQ2 — Rendimiento bajo carga pico

**Por qué es crítico para SaludEje:** El lunes entre 6am y 8am concentra el 40% de las agendas de toda la semana. En ese intervalo, el sistema recibe en dos horas el equivalente a dos días y medio de carga normal. La operación dominante en ese pico no es el agendamiento en sí — es la *consulta de disponibilidad*: pacientes revisando qué citas hay libres antes de agendar. Si el sistema responde lento o falla en ese horario, los pacientes no pueden agendar sus citas, el cuello de botella se traslada a los hospitales físicamente, y el impacto en la atención es inmediato.

**Escenario de calidad — Pico de demanda en agendamiento:**

| Componente | Descripción |
|---|---|
| **Fuente del estímulo** | Pacientes y personal administrativo de los hospitales del Eje Cafetero |
| **Estímulo** | 1.200 solicitudes concurrentes de consulta de disponibilidad (estimado para el pico del lunes), de las cuales aproximadamente el 15% derivarán en un agendamiento efectivo |
| **Artefacto** | Módulo de Agendamiento — modelo de lectura (CQRS) |
| **Entorno** | Horario de máxima demanda: lunes 6:00am – 8:00am; el auto-scaling ya fue pre-activado a las 5:45am basado en el patrón histórico |
| **Respuesta** | El sistema responde a todas las consultas de disponibilidad con datos correctos; el auto-scaling agrega réplicas en menos de 60 segundos si el umbral de CPU supera el 70%; los agendamientos efectivos se procesan en orden y sin pérdida de datos |
| **Medida de respuesta** | Tiempo de respuesta p95 para consultas de disponibilidad < 2 segundos; tasa de error < 0.1%; el auto-scaling debe activar nuevas réplicas en ≤ 60 segundos; el sistema vuelve a la configuración base en ≤ 20 minutos después del pico |

**Táctica aplicada:** CQRS con modelo de lectura en caché (Redis o réplica de lectura de BD). Las consultas de disponibilidad nunca tocan el modelo de escritura. Auto-scaling horizontal del monolito basado en métricas de CPU/memoria. Pre-warming de réplicas antes del pico del lunes (activación programada a las 5:45am).

---

### AQ3 — Seguridad

**Por qué es crítico para SaludEje:** La Historia Clínica Electrónica contiene datos sensibles de salud — el tipo de dato más protegido por la Ley 1581. Un acceso no autorizado a la HCE de un paciente no es solo una violación de privacidad: puede exponer condiciones de salud que el paciente no ha revelado, diagnósticos psiquiátricos, enfermedades de transmisión sexual, o estados de salud con implicaciones laborales o de seguros. Además, el sistema tiene múltiples actores con niveles de acceso radicalmente distintos: un paciente solo puede ver su propia historia, un médico solo puede ver las historias de sus pacientes asignados, un administrador hospitalario tiene acceso a datos administrativos pero no clínicos. La granularidad del control de acceso no es un detalle de implementación — es un requerimiento normativo.

**Escenario de calidad — Ataque de credential stuffing contra cuentas médicas:**

| Componente | Descripción |
|---|---|
| **Fuente del estímulo** | Atacante externo con una lista de credenciales filtradas de otra plataforma |
| **Estímulo** | 500 intentos de login fallidos contra diferentes cuentas de médicos en un período de 5 minutos, desde múltiples IPs distribuidas en una red de bots |
| **Artefacto** | Auth Service (IAM centralizado) y módulo de HCE |
| **Entorno** | Operación normal; el ataque ocurre en horario no pico (3am) para evitar detección por volumen |
| **Respuesta** | El sistema detecta el patrón anómalo de intentos fallidos distribuidos, activa el bloqueo temporal de las cuentas afectadas después de 10 intentos fallidos, bloquea las IPs del rango de bots, genera una alerta de seguridad al administrador, y registra el evento completo en el log de auditoría inmutable |
| **Medida de respuesta** | Detección del patrón en ≤ 60 segundos desde el inicio del ataque; bloqueo de cuenta activado en el intento número 10; cero datos clínicos expuestos; alerta enviada al administrador en ≤ 2 minutos; el log de auditoría registra cada intento fallido con IP, timestamp y cuenta objetivo |

**Tácticas aplicadas:** Rate limiting por cuenta y por IP en el Auth Service. Bloqueo progresivo de cuentas (5 intentos → warning, 10 intentos → bloqueo temporal). MFA obligatorio para roles clínicos (médicos, enfermeras). RBAC (Role-Based Access Control) granular por rol y por hospital. Log de auditoría inmutable en la HCE (append-only, no editable ni siquiera por administradores del sistema).

---

### AQ4 — Interoperabilidad

**Por qué es crítico para SaludEje:** La razón de existir de SaludEje es integrar sistemas heterogéneos que hoy no se hablan entre sí. Si el sistema no puede incorporar un nuevo laboratorio externo sin una reescritura del core, o si agregar una cuarta EPS requiere modificar la lógica de facturación, el sistema falla en su misión fundamental. La interoperabilidad no es solo conectividad técnica — es la capacidad de incorporar nuevos actores externos sin degradar lo que ya funciona.

**Escenario de calidad — Incorporación de un laboratorio externo no contemplado en el diseño inicial:**

| Componente | Descripción |
|---|---|
| **Fuente del estímulo** | La Secretaría de Salud contrata un nuevo laboratorio clínico regional (Laboratorio del Sur) que no estaba en el alcance inicial del MVP |
| **Estímulo** | Laboratorio del Sur necesita cargar resultados en su formato propietario (XML con esquema propio, sin HL7), mediante SFTP en lugar de API REST |
| **Artefacto** | Módulo de Laboratorio — capa ACL y registro de adaptadores |
| **Entorno** | Sistema en producción con otros laboratorios ya integrados y en operación; el nuevo adaptador no puede afectar las integraciones existentes |
| **Respuesta** | Se desarrolla e integra un nuevo adaptador para Laboratorio del Sur que traduce su formato XML propietario y su protocolo SFTP al modelo interno de SaludEje. El adaptador se registra en el sistema sin modificar el módulo de Laboratorio ni ningún otro módulo. Los laboratorios ya integrados continúan operando sin interrupción |
| **Medida de respuesta** | El nuevo adaptador puede desarrollarse e integrarse en ≤ 5 días hábiles; cero cambios al core del módulo de Laboratorio; cero regresiones en laboratorios ya integrados verificadas por suite de tests del adaptador; el resultado procesado por el nuevo adaptador es indistinguible del procesado por cualquier otro laboratorio desde la perspectiva del módulo HCE |

**Táctica aplicada:** Anti-Corruption Layer con registro de adaptadores pluggable. Cada adaptador implementa una interfaz común (`IAdaptadorLaboratorio`) que el módulo de Laboratorio consume sin conocer la implementación concreta. Los contratos de eventos entre módulos están versionados, lo que permite que nuevos adaptadores adopten la interfaz sin romper los existentes.

---

### Trade-offs entre atributos de calidad

Los cuatro atributos no son independientes. Optimizar uno tiene consecuencias medibles en los demás. A continuación se identifican los tres trade-offs más significativos del diseño y cómo se manejan.

#### Trade-off 1: Disponibilidad alta vs Consistencia estricta

**La tensión:** El teorema CAP establece que en un sistema distribuido con partición de red, no se puede garantizar simultáneamente consistencia perfecta y disponibilidad total. Para lograr alta disponibilidad en la HCE (réplica activa, failover automático), hay que aceptar una ventana brevísima de inconsistencia durante la conmutación entre nodos.

**Cómo se maneja:** Se aplica una estrategia diferenciada por dominio. En la HCE, las *escrituras* priorizan consistencia: una entrada clínica no se confirma hasta que está replicada en ambos nodos (sincronización síncrona). Las *lecturas* en el módulo de Agendamiento priorizan disponibilidad: el modelo de lectura de CQRS acepta eventual consistency (el médico puede ver por milisegundos un slot que acaba de ser tomado). Esta diferenciación evita aplicar el mismo nivel de consistencia a todos los dominios, que hubiera requerido sacrificar disponibilidad global o rendimiento global innecesariamente.

#### Trade-off 2: Seguridad fuerte vs Rendimiento

**La tensión:** Cada capa de seguridad añade latencia. El cifrado en reposo y en tránsito de los datos clínicos tiene un costo de CPU. La validación de tokens JWT en cada request al Auth Service agrega un round-trip. El MFA obligatorio para médicos añade fricción en el login. El log de auditoría append-only es una escritura adicional en cada acceso a la HCE.

**Cómo se maneja:** Se acepta el costo de seguridad en HCE sin compromisos: el cifrado, el MFA y el log de auditoría son no negociables dado el perfil normativo y el riesgo clínico. En módulos con menor criticidad (Farmacia, gestión administrativa), el nivel de seguridad es alto pero sin MFA obligatorio, lo que reduce la fricción para usuarios que acceden con mayor frecuencia. La validación de tokens se optimiza con caché de tokens válidos en el API Gateway (evita el round-trip al Auth Service en cada request durante la sesión activa).

#### Trade-off 3: Interoperabilidad amplia vs Superficie de ataque

**La tensión:** Cada punto de integración con un sistema externo (un laboratorio, una EPS, un HIS) es un potencial vector de ataque. Un laboratorio externo comprometido podría intentar inyectar datos maliciosos en la HCE a través del adaptador. Cuantos más adaptadores, mayor la superficie de ataque.

**Cómo se maneja:** El ACL no es solo un traductor de formatos — es también la primera línea de validación y sanitización. Cada adaptador debe validar y sanitizar todos los datos externos *antes* de que entren al modelo interno de SaludEje. Los datos que no pasen la validación son rechazados y registrados en el log de seguridad. El módulo de Laboratorio, como receptor de datos de fuentes externas, tiene validación de esquema estricta en la frontera del ACL: ningún dato malformado o fuera del esquema esperado puede propagarse al resto del sistema.

---

## C. Architecture Decision Records (ADRs)

> *[En desarrollo — Paso 4]*

---

## D. Modelos Arquitectónicos y su Justificación

### La decisión principal: Monolito Modular como estructura global

Antes de elegir un modelo, hay que responder una pregunta concreta: ¿cuál es el riesgo más alto de este proyecto? No la escalabilidad, no la disponibilidad — esos son problemas técnicos resolubles. El riesgo más alto es construir un sistema demasiado complejo para el equipo disponible, en el tiempo disponible, con el presupuesto disponible, en un dominio que el equipo no conoce.

Con ese marco, se evaluaron tres opciones principales:

#### Opción 1: Microservicios por dominio — descartada para fase 1

La arquitectura de microservicios propone descomponer el sistema en servicios independientes, cada uno con su propia base de datos, su propio pipeline de deployment y su propia capacidad de escalar. Es la opción que primero viene a la mente para un sistema con dominios tan distintos como HCE y Agendamiento.

**Por qué fue descartada para fase 1:**

Primero, el equipo. 12 ingenieros sin experiencia en salud deben aprender simultáneamente la normativa de la Resolución 2654, los flujos clínicos del dominio médico, los formatos RIPS, las APIs heterogéneas de las EPS, y encima operar una infraestructura distribuida que incluye service discovery, distributed tracing, circuit breakers y saga patterns para transacciones distribuidas. Esto es doble complejidad cognitiva en el peor momento posible: cuando el equipo apenas está adquiriendo el contexto del dominio.

Segundo, la consistencia de la HCE. En microservicios, una operación que toca múltiples servicios requiere un saga pattern o two-phase commit para mantener consistencia. Ambos patrones son complejos y tienen ventanas de inconsistencia. En la Historia Clínica Electrónica — donde un médico de urgencias toma decisiones clínicas basadas en los datos del sistema — un estado inconsistente, por pequeño que sea, puede derivar en una decisión médica incorrecta. El monolito modular usa transacciones ACID nativas: si algo falla, falla atómicamente. No hay estados intermedios.

Tercero, el presupuesto y el tiempo. La infraestructura para microservicios maduros — orquestación de contenedores, observabilidad distribuida, pipelines CI/CD por servicio, gestión de secretos por servicio — consume entre 2 y 4 meses de trabajo de infraestructura antes de escribir la primera línea de funcionalidad de dominio. Con 12 meses de desarrollo disponibles para el MVP, ese costo de arranque es inaceptable.

Los microservicios no son la respuesta incorrecta para SaludEje — son la respuesta correcta para la fase 2, cuando el equipo ya conoce el dominio, la carga real del sistema es medible, y hay un candidato claro para extraer primero (el módulo de Agendamiento, por sus picos de carga). El ADR-001 documenta formalmente esta decisión y la ruta de evolución.

#### Opción 2: Arquitectura por capas N-tier — descartada

La arquitectura por capas (presentación → lógica de negocio → datos) es la más conocida y documentada. Es válida para sistemas simples o equipos con poca experiencia en estilos más avanzados.

**Por qué fue descartada:** SaludEje tiene cinco dominios con características radicalmente distintas. La HCE necesita inmutabilidad de registros, log de auditoría por ley y aislamiento total de la lógica clínica. El Agendamiento necesita manejar picos de carga extremos con lecturas de disponibilidad intensivas. La Facturación necesita adaptarse a formatos externos incompatibles entre sí. Una arquitectura por capas trata todos estos dominios como si fueran el mismo problema, con las mismas capas, la misma base de datos compartida y el mismo modelo de datos. El resultado sería una base de datos monolítica compartida entre todos los dominios — lo que viola el principio de aislamiento de datos entre módulos, hace imposible cumplir correctamente la Resolución 2654 y crea acoplamiento accidental entre dominios que no deberían conocerse.

#### Opción elegida: Monolito Modular con fronteras de dominio explícitas

El Monolito Modular es un único deployable que internamente está organizado en módulos con fronteras claras y bien definidas: cada módulo tiene su propio schema de base de datos, sus propias interfaces de entrada y salida, y no accede directamente a la persistencia de ningún otro módulo. Los módulos se comunican a través de interfaces explícitas o de un bus de eventos interno.

**Por qué esta opción gana:**

- **Consistencia donde importa.** La HCE puede usar transacciones ACID para garantizar que un registro clínico se escriba completamente o no se escriba. No hay patrones saga, no hay estados intermedios inconsistentes.
- **Un solo pipeline de deployment.** El equipo puede enfocarse en aprender el dominio médico, no en operar infraestructura distribuida. Un bug en producción tiene un solo lugar donde buscar.
- **Escalado suficiente para el MVP.** El monolito completo puede replicarse horizontalmente detrás de un load balancer. Para la carga de una plataforma regional del Eje Cafetero, esto es suficiente. El pico del lunes de agendamiento se maneja con el módulo CQRS interno y auto-scaling de réplicas del monolito.
- **Evolución natural a microservicios.** El Monolito Modular con fronteras claras es el punto de partida del patrón Strangler Fig: en fase 2, el módulo de Agendamiento (el más estresado y el más estable funcionalmente) puede extraerse como microservicio independiente sin reescribir nada — solo cambia el mecanismo de comunicación entre módulos.

**Trade-off explícito:** Se sacrifica escalado granular por dominio. Si el módulo de Farmacia necesita más capacidad, hay que escalar el monolito completo. Este trade-off es aceptable para el MVP: la carga diferencial entre dominios en una plataforma regional no justifica la complejidad operacional de microservicios independientes.

---

### Modelos específicos por dominio

No todos los dominios tienen el mismo perfil de carga, los mismos requerimientos de consistencia ni las mismas necesidades de integración externa. La heterogeneidad del sistema se refleja en el modelo interno de cada módulo.

| Dominio | Modelo interno | Problema principal que resuelve |
|---|---|---|
| Historia Clínica Electrónica | Arquitectura Hexagonal (Ports & Adapters) | Aislar la lógica clínica de la infraestructura; cumplir auditoría e inmutabilidad de la Resolución 2654 |
| Agendamiento y consultas | CQRS (Command Query Responsibility Segregation) | Escalar lecturas de disponibilidad independientemente de las escrituras durante picos de carga |
| Laboratorio y diagnóstico | Anti-Corruption Layer + Adapter por laboratorio | Integrar formatos heterogéneos de laboratorios externos sin contaminar el modelo interno |
| Farmacia hospitalaria | Arquitectura en capas (Layered) | Dominio estable con lógica de inventario clara; no requiere complejidad adicional |
| Facturación y glosas | Anti-Corruption Layer + Adapter por EPS | Integrar APIs heterogéneas de Sura, Coosalud y Nueva EPS sin acoplar el modelo interno a cada una |

#### Modelo 1 — Arquitectura Hexagonal en HCE

La Arquitectura Hexagonal, propuesta por Alistair Cockburn, organiza el dominio en tres capas concéntricas: el núcleo de dominio (la lógica clínica pura), los puertos (interfaces que definen cómo entra y sale la información del núcleo) y los adaptadores (implementaciones concretas de esos puertos: la base de datos, la API REST, el adaptador de HIS legacy, el adaptador de auditoría).

**Por qué en HCE:** La lógica clínica — qué constituye un registro válido, qué campos son obligatorios, qué operaciones están permitidas — no debe depender de si los datos vienen de un formulario web o de un HIS legacy. No debe depender de si se persiste en PostgreSQL o en otro motor. La Resolución 2654 exige que cada acceso a la historia clínica quede registrado. Con la arquitectura hexagonal, el puerto de auditoría es obligatorio por diseño: ningún caso de uso puede escribir o leer la HCE sin pasar por el adaptador de auditoría, porque el puerto lo exige. Esto convierte el cumplimiento normativo en una propiedad estructural del código, no en una convención que alguien puede olvidar.

**Trade-off:** Mayor complejidad de estructura inicial. Hay más archivos, más capas, más interfaces que en una solución procedural directa. Para un equipo sin experiencia en el dominio, la curva de aprendizaje es real. Se mitiga con documentación interna del módulo y con la ventaja de que la lógica clínica queda completamente testeable en aislamiento, sin base de datos ni HTTP.

#### Modelo 2 — CQRS en Agendamiento

CQRS (Command Query Responsibility Segregation) separa las operaciones de escritura (Commands: agendar una cita, cancelar, modificar) del modelo de lectura (Queries: ¿qué disponibilidad hay? ¿cuál es la agenda de este médico?).

**Por qué en Agendamiento:** El 40% de las agendas semanales se concentran el lunes entre 6am y 8am. En ese pico, la gran mayoría de las operaciones son lecturas de disponibilidad — pacientes consultando si hay citas con tal especialista en tal fecha, antes de agendar. Sin CQRS, cada consulta de disponibilidad compite con las escrituras de agendamiento en la misma base de datos, con los mismos índices, bajo el mismo lock. Con CQRS, el modelo de lectura es una proyección optimizada — puede ser una caché Redis actualizada por eventos de escritura, o una réplica de lectura de la BD — que absorbe el 90% del tráfico del pico sin tocar el modelo de escritura. La escritura (el agendamiento real) sigue siendo consistente y serializada.

**Trade-off:** Se introduce eventual consistency entre el modelo de escritura y el modelo de lectura. Si un paciente agenda una cita y otro usuario consulta disponibilidad al mismo tiempo, puede ver por un instante brevísimo la cita como disponible. Este lag es de milisegundos en la práctica y es un trade-off aceptable para el dominio de agendamiento: no es HCE, no hay vidas en juego por un lag de 50ms en una vista de disponibilidad.

#### Modelo 3 — Anti-Corruption Layer en Facturación y Laboratorio

El Anti-Corruption Layer (ACL), concepto de Domain-Driven Design, es una capa de traducción que se interpone entre el modelo interno del sistema y los modelos de sistemas externos. Su función es absorber la complejidad y la variabilidad de los externos sin dejarla entrar al dominio.

**Por qué en Facturación:** Sura, Coosalud y Nueva EPS tienen APIs distintas, formatos de factura distintos y vocabularios distintos para los mismos conceptos (lo que Sura llama "glosa" Coosalud puede llamar "devolución"). Sin ACL, el módulo de Facturación tendría que conocer los detalles de cada EPS: if-else por EPS, DTOs específicos por EPS, manejo de errores por EPS. Con ACL, hay un adaptador por EPS que traduce entre el modelo interno de SaludEje y el modelo externo de cada aseguradora. El módulo de Facturación habla siempre el mismo lenguaje interno; los adaptadores se encargan de la traducción. Agregar una cuarta EPS en el futuro significa escribir un nuevo adaptador, no modificar la lógica de facturación.

**Por qué en Laboratorio:** El mismo argumento aplica. Los laboratorios externos tienen formatos de resultado distintos y protocolos de transferencia distintos (algunos usan HL7, otros CSV, otros APIs propias). El ACL absorbe esa variabilidad.

**Trade-off:** Mayor tiempo de desarrollo inicial — cada adaptador externo debe construirse y mantenerse. Si un laboratorio cambia su API, hay que actualizar su adaptador. Pero ese cambio está contenido en un único lugar del código, no disperso por todo el sistema.

#### Modelo 4 — Arquitectura en capas en Farmacia

La Farmacia hospitalaria tiene la lógica más estable y predecible del sistema: gestión de inventario, alertas de stock mínimo y vencimiento, dispensación de medicamentos. No tiene picos de carga extremos, no requiere integración con sistemas externos heterogéneos y no tiene requerimientos normativos tan exigentes como la HCE.

**Por qué Layered aquí:** Aplicar hexagonal o CQRS a un dominio de inventario estable sería complejidad innecesaria. La arquitectura en capas (controlador → servicio → repositorio) es suficiente y más fácil de mantener para el equipo. La consulta inter-hospitalaria de inventarios (para traslados de urgencia) se implementa como un servicio de integración dentro del mismo módulo, no como una arquitectura distinta.

**Trade-off:** Si el dominio de Farmacia evoluciona significativamente (por ejemplo, se integra con proveedores externos en fase 2), puede necesitar un ACL. El Layered simple hace esa evolución más costosa que si se hubiera usado Hexagonal desde el inicio. Este riesgo es bajo dado el perfil del dominio.

---

### Comunicación entre módulos: bus de eventos interno

Los módulos del Monolito Modular no se llaman directamente entre sí. La comunicación entre dominios ocurre a través de un bus de eventos interno, de forma asíncrona.

Los flujos principales mediados por eventos:

- **Laboratorio carga resultado** → emite `ResultadoDisponible(pacienteId, medicoId, resultado)` → el módulo de Notificaciones lo consume y alerta al médico tratante en tiempo real
- **Agendamiento confirma una cita** → emite `CitaConfirmada(pacienteId, medicoId, fecha)` → HCE puede pre-cargar el contexto del paciente para cuando llegue la consulta
- **Farmacia detecta stock crítico** → emite `StockMínimo(hospitalId, medicamentoId, cantidadActual)` → notificación al farmacéutico responsable
- **HCE registra una atención** → emite `AtencionRegistrada(pacienteId, codigosProcedimiento)` → Facturación inicia la generación de la cuenta médica para la EPS correspondiente

En fase 1, el bus de eventos es una implementación interna del monolito (puede ser tan simple como un sistema de observers en memoria, o una librería ligera de mensajería interna). La interfaz del bus — los eventos y sus payloads — está diseñada para que en fase 2 pueda reemplazarse por un broker externo (Kafka, RabbitMQ) sin cambiar el código de los módulos. Solo cambia el adaptador del bus.

---

### Articulación entre modelos: cómo se conectan

```
┌──────────────────────────────────────────────────────────────┐
│                     MONOLITO MODULAR                         │
│                                                              │
│  ┌─────────────┐    ┌──────────────┐    ┌───────────────┐   │
│  │  Módulo HCE │    │  Módulo      │    │  Módulo       │   │
│  │ (Hexagonal) │    │  Agendamiento│    │  Laboratorio  │   │
│  │             │    │  (CQRS)      │    │  (ACL)        │   │
│  └──────┬──────┘    └──────┬───────┘    └──────┬────────┘   │
│         │                 │                    │             │
│         └─────────────────┴────────────────────┘            │
│                           │                                  │
│                   Bus de Eventos Interno                     │
│                           │                                  │
│         ┌─────────────────┴────────────────────┐            │
│         │                                       │            │
│  ┌──────┴──────┐                     ┌──────────┴────────┐  │
│  │  Módulo     │                     │  Módulo           │  │
│  │  Farmacia   │                     │  Facturación      │  │
│  │  (Layered)  │                     │  (ACL)            │  │
│  └─────────────┘                     └───────────────────┘  │
└──────────────────────────────────────────────────────────────┘
         │                                         │
    API Gateway                          Adaptadores externos:
    (punto de entrada único)             - HIS de cada hospital
    Auth Service (IAM)                   - APIs EPS (Sura, Coosalud, Nueva EPS)
                                         - Laboratorios clínicos externos
                                         - MINSALUD (RIPS)
```

Los límites entre modelos son precisos y no se superponen:

- **El Monolito Modular** opera al nivel del sistema completo: define el deployable, el pipeline de CI/CD y cómo los módulos se despliegan juntos. No dice nada sobre la organización interna de cada módulo.
- **La Arquitectura Hexagonal** opera al nivel interno del módulo HCE: cómo organiza su código, sus capas, sus puertos. El resto del sistema la ve solo a través del bus de eventos y de la API Gateway.
- **CQRS** opera al nivel interno del módulo Agendamiento: cómo separa lecturas de escrituras. El bus de eventos consume los resultados de los Commands, no el mecanismo interno de CQRS.
- **El ACL** opera en la frontera entre los módulos de Facturación y Laboratorio con los sistemas externos. Es invisible para el bus de eventos y para los demás módulos.
- **Layered** opera al nivel interno del módulo Farmacia: organización simple de capas sin implicaciones para el sistema global.

Esta separación garantiza que cambiar el modelo interno de un módulo (por ejemplo, reemplazar CQRS por Event Sourcing en Agendamiento en fase 2) no requiera tocar ningún otro módulo ni el diseño global.

---

## E. Diagrama de Arquitectura de Alto Nivel

> *[En desarrollo — Paso 5]*
