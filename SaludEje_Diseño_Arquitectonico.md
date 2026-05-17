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

> *[En desarrollo — Paso 2]*

---

## B. Atributos de Calidad con Escenarios

> *[En desarrollo — Paso 3]*

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
