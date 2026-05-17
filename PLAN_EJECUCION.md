# Plan de Ejecución — Diseño Arquitectónico SaludEje

## Qué estamos construyendo

Diseño arquitectónico completo de **SaludEje**, plataforma digital para integrar servicios de salud de hospitales públicos de Caldas, Risaralda y Quindío. No es código: es un documento técnico que proponga cómo debería construirse el sistema.

**Actores externos:** Pacientes, médicos, enfermeras, laboratoristas, farmacéuticos, administradores hospitalarios, EPS (Sura, Coosalud, Nueva EPS), MINSALUD, laboratorios clínicos externos.

**5 dominios del sistema:**
1. Historia Clínica Electrónica (HCE) — dominio más crítico, datos sensibles, normativa colombiana
2. Agendamiento y consultas — picos extremos (40% de agendas lunes 6-8am), telemedicina
3. Laboratorio y diagnóstico — múltiples laboratorios externos con formatos heterogéneos
4. Farmacia hospitalaria — inventario por hospital + consulta entre hospitales para urgencias
5. Facturación y glosas — integración con EPS con APIs y formatos distintos

**Restricciones clave:**
- $850M COP, 18 meses, 12 ingenieros sin experiencia en salud
- Datos clínicos deben permanecer en territorio colombiano (soberanía de datos)
- No se reemplazan los HIS existentes — SaludEje se integra con ellos
- Servidores locales con capacidad limitada + partida para nube

---

## Pasos de ejecución

### Paso 1 — Decidir y sustentar el modelo arquitectónico principal

> **Qué es:** El modelo arquitectónico define la estructura general del sistema: cómo se divide, cómo se comunican las partes, y cómo evoluciona. Es la decisión más importante porque condiciona todo lo demás — el equipo, el despliegue, la escalabilidad y los costos.
>
> **Para qué sirve:** Establece el "esqueleto" sobre el que se construye todo. Un modelo mal elegido obliga a rediseñar el sistema entero más adelante.
>
> **Cómo hacerlo:** Listar las opciones reales con sus pros y contras en el contexto específico de SaludEje (no en general). Luego elegir y justificar con base en los drivers del sistema. La justificación debe responder: *¿por qué esta opción y no las otras dado este contexto?*

- [x] Evaluar: **Monolito modular** vs **Microservicios** vs **Arquitectura por capas** vs **Event-driven**
- [x] Sustentar por qué cada dominio puede/necesita un modelo distinto (HCE ≠ Agendamiento ≠ Laboratorio)
- [x] Definir cómo se articulan los modelos entre sí y dónde están los límites

**Decisión tentativa a sustentar:** Arquitectura de microservicios por dominio (domain-driven), con un bus de eventos para comunicación asíncrona entre dominios. HCE con arquitectura hexagonal para garantizar aislamiento de la lógica clínica. Justificación: equipos independientes por dominio, evolución sin acoplamiento, escalado diferenciado por carga.

---

### Paso 2 — Identificar y priorizar los drivers arquitectónicos

> **Qué es:** Los drivers arquitectónicos son los requerimientos que más impactan las decisiones de diseño. No es una lista de todo lo que debe hacer el sistema — son las cosas que si cambian, obligan a rediseñar la arquitectura. Se dividen en: funcionales (casos de uso críticos) y de restricción (límites que el diseño no puede ignorar).
>
> **Para qué sirven:** Son el punto de partida que justifica todo. Cada ADR, cada modelo, cada atributo de calidad debe poderse rastrear a un driver. Si una decisión no responde a ningún driver, sobra.
>
> **Cómo hacerlos:** Para cada driver funcional preguntarse: *¿si esto cambia, cambia la arquitectura?* Si la respuesta es sí, es un driver. Para restricciones: *¿qué no podemos hacer aunque quisiéramos?* Soberanía del dato, presupuesto, equipo sin experiencia — esas son restricciones reales.

**Drivers funcionales (mínimo 4):**
- [ ] DF1: Acceso a HCE en tiempo real desde cualquier hospital del eje cafetero
- [ ] DF2: Gestión de picos extremos de agendamiento (lunes 6-8am = 40% semanal)
- [ ] DF3: Notificación inmediata de resultados de laboratorio al médico tratante
- [ ] DF4: Integración heterogénea con HIS existentes y EPS con formatos distintos

**Drivers de restricción (mínimo 3):**
- [ ] DR1: Soberanía del dato — datos clínicos solo en territorio colombiano (ley 1581 + resolución 2654)
- [ ] DR2: Presupuesto y equipo limitado — 12 ingenieros sin experiencia en salud, $850M COP
- [ ] DR3: No reemplazar HIS existentes — integración obligatoria como capa adicional

---

### Paso 3 — Definir atributos de calidad con escenarios formales

> **Qué es:** Los atributos de calidad (también llamados requerimientos no funcionales) describen *cómo* debe comportarse el sistema, no *qué* debe hacer. Disponibilidad, seguridad, escalabilidad, mantenibilidad, etc. Un escenario de calidad es una situación concreta que pone a prueba ese atributo.
>
> **Para qué sirven:** Sin escenarios, los atributos quedan vagos ("el sistema debe ser seguro" no dice nada). El escenario le da forma medible y permite comparar opciones de diseño. También revelan los trade-offs: mejorar disponibilidad puede afectar consistencia; mejorar seguridad puede afectar rendimiento.
>
> **Cómo hacerlos:** Usar el formato estándar de 6 componentes:
> - **Fuente del estímulo** — quién dispara la situación (usuario, sistema externo, atacante, el tiempo)
> - **Estímulo** — qué pasa exactamente
> - **Artefacto** — qué parte del sistema es afectada
> - **Entorno** — bajo qué condición ocurre (horario pico, falla de red, etc.)
> - **Respuesta** — cómo debe reaccionar el sistema
> - **Medida de respuesta** — cómo se sabe que la respuesta fue suficiente (número, tiempo, porcentaje)

- [ ] **Disponibilidad** — HCE indisponible = riesgo de vida. Escenario: urgencia nocturna sin acceso a historial
- [ ] **Escalabilidad** — picos de agendamiento. Escenario: lunes 6am, 1000 usuarios simultáneos
- [ ] **Seguridad** — datos clínicos sensibles, normativa colombiana. Escenario: acceso no autorizado a HCE
- [ ] **Interoperabilidad** — 3 departamentos, múltiples HIS, múltiples EPS. Escenario: integración laboratorio externo

**Trade-offs a explicar:**
- Disponibilidad alta vs Consistencia estricta (CAP theorem en HCE distribuida)
- Seguridad fuerte vs Rendimiento (cifrado end-to-end tiene costo en latencia)
- Escalabilidad elástica en nube vs Soberanía del dato (no todo puede ir a nube pública)

---

### Paso 4 — Redactar los ADRs (mínimo 5)

> **Qué es un ADR:** Architecture Decision Record — un registro escrito de una decisión arquitectónica importante. No documenta solo qué se decidió, sino por qué se decidió eso y no otra cosa. Es la memoria del equipo de arquitectura.
>
> **Para qué sirven:** Cuando alguien (en 6 meses, o el siguiente equipo) pregunta *¿por qué el sistema funciona así?*, el ADR responde. También obliga a evaluar alternativas reales antes de decidir — si no puedes escribir los pros/contras de las opciones descartadas, no entendiste bien el problema.
>
> **Cómo hacerlos — estructura de cada ADR:**
> 1. **Número y título** — ADR-001: [decisión en una línea]
> 2. **Estado** — Aceptado / Propuesto / Deprecado
> 3. **Contexto** — qué problema existe, qué fuerzas están en juego, por qué hay que decidir esto ahora
> 4. **Opciones consideradas** — mínimo 2-3 alternativas reales con sus pros y contras concretos (no genéricos)
> 5. **Decisión** — qué se elige y la razón principal
> 6. **Consecuencias** — qué se gana, qué se sacrifica, qué deuda técnica se asume, qué otros ADRs afecta
>
> **Clave:** Las opciones descartadas deben ser técnicamente válidas. Un ADR que descarta opciones absurdas no demuestra análisis real.

- [ ] **ADR-001:** Elección del modelo arquitectónico principal (microservicios por dominio)
- [ ] **ADR-002:** Estrategia de integración con HIS existentes (API Gateway + adaptadores por hospital)
- [ ] **ADR-003:** Modelo de datos para HCE (base de datos propia por dominio vs compartida)
- [ ] **ADR-004:** Estrategia de nube con soberanía del dato (AWS/Azure Colombia + on-premise para datos clínicos)
- [ ] **ADR-005:** Mecanismo de comunicación entre dominios (eventos asincrónicos con broker vs REST síncrono)

---

### Paso 5 — Diseñar el diagrama de arquitectura de alto nivel

> **Qué es:** Una representación visual del sistema completo que muestra quién usa el sistema, qué módulos lo componen, cómo se comunican entre sí y con el exterior, y dónde viven los datos. No es un diagrama de clases ni de flujo — es una vista de pájaro de la arquitectura.
>
> **Para qué sirve:** Es lo que cualquier stakeholder (técnico o no) ve primero. Debe ser suficientemente claro para que alguien que no leyó el documento igual entienda la estructura general. También actúa como verificación de coherencia: si el diagrama contradice un ADR, algo está mal.
>
> **Cómo hacerlo:** Usar el modelo C4 simplificado — nivel de contexto (sistema + actores externos) y nivel de contenedor (componentes internos principales). Cada elemento del diagrama debe aparecer mencionado en algún ADR o sección de modelos. Usar Mermaid para incluirlo directamente en el MD.

- [ ] Identificar los componentes: API Gateway, cada microservicio/dominio, event broker, bases de datos por dominio, adaptadores HIS, conectores EPS, capa de autenticación
- [ ] Definir flujos principales: paciente agenda cita → médico accede HCE → solicita examen → laboratorio carga resultado → médico notificado → se genera cuenta a EPS
- [ ] Crear el diagrama en formato Mermaid para incluir en el documento final

**Componentes clave a mostrar:**
```
Actores externos → API Gateway (punto de entrada único)
                 → Auth Service (IAM centralizado)
                 → [Dominio HCE] ↔ DB HCE (on-premise cifrada)
                 → [Dominio Agendamiento] ↔ DB Agenda
                 → [Dominio Laboratorio] ↔ Adaptadores laboratorios externos
                 → [Dominio Farmacia] ↔ DB Inventario por hospital
                 → [Dominio Facturación] ↔ Conectores EPS (Sura, Coosalud, Nueva EPS)
                 → Event Broker (comunicación asíncrona entre dominios)
                 → Adaptadores HIS (uno por hospital existente)
Sistemas externos: HIS hospitales, APIs EPS, MINSALUD (RIPS), Laboratorios externos
```

---

### Paso 6 — Ensamblar el documento final

> **Qué es:** Juntar todo lo producido en los pasos anteriores en un único documento coherente. No es solo pegar secciones — es asegurarse de que el documento se lea como un argumento técnico continuo, no como respuestas independientes.
>
> **Para qué sirve:** El evaluador (o cliente) debe poder seguir el hilo: *este es el problema → estos son los drivers → estas son las decisiones que tomamos y por qué → así quedó el sistema*. Si el diagrama muestra algo que ningún ADR menciona, o un atributo de calidad no aparece en ninguna táctica, el documento no es coherente.
>
> **Cómo hacerlo:** Al final, hacer una revisión cruzada: cada driver debe aparecer en al menos un ADR. Cada ADR debe verse reflejado en el diagrama. Cada atributo de calidad debe tener una táctica concreta en el diseño que lo soporte.

- [ ] Introducción y contexto del sistema
- [ ] Drivers arquitectónicos (funcionales + restricciones)
- [ ] Atributos de calidad y escenarios (con trade-offs)
- [ ] ADRs (ADR-001 al ADR-005)
- [ ] Modelos arquitectónicos y justificación por dominio
- [ ] Diagrama de arquitectura de alto nivel
- [ ] Revisión cruzada de coherencia — drivers → ADRs → modelos → diagrama cuentan la misma historia

---

## Decisiones críticas a sustentarse antes de escribir

> Estas son las preguntas que hay que responder con argumentos antes de empezar a redactar. Si se empieza a escribir sin tenerlas claras, el documento queda inconsistente.

| Decisión | Opciones a evaluar | Criterio de elección |
|---|---|---|
| Modelo arquitectónico | Microservicios / Monolito modular / Layered | Independencia de equipos, escalado diferenciado |
| Infraestructura | Cloud Colombia (AWS/Azure) + on-premise | Soberanía del dato + elasticidad |
| Comunicación entre dominios | Event-driven (Kafka) vs REST vs GraphQL | Latencia vs desacoplamiento |
| Integración HIS | API Gateway + adaptadores vs ESB vs ETL | No modificar HIS existentes |
| Modelo de datos HCE | DB por dominio vs DB compartida | Aislamiento, consistencia, normativa |

---

## Checklist de normativa colombiana

> Puntos específicos que el diseño arquitectónico debe respetar. Cada uno debe verse reflejado en algún ADR, atributo de calidad o táctica del documento final.

**Ley 1581 de 2012 — Protección de datos personales:**
- [ ] Los datos clínicos son datos sensibles — requieren consentimiento explícito del paciente para su tratamiento
- [ ] El paciente tiene derecho de acceso, corrección y supresión de sus datos (habeas data)
- [ ] Debe existir un Responsable del Tratamiento identificado (el hospital o la Secretaría de Salud)
- [ ] Los datos no pueden transferirse a terceros sin autorización, salvo excepciones legales (urgencias, MINSALUD)
- [ ] Debe existir un registro de actividades de tratamiento de datos

**Resolución 2654 de 2019 — Historia Clínica Electrónica:**
- [ ] La HCE debe garantizar autenticidad, integridad, confidencialidad, disponibilidad y auditabilidad
- [ ] Cada acceso a la historia clínica debe quedar registrado (quién, cuándo, desde dónde)
- [ ] Los registros de HCE no pueden modificarse ni eliminarse — solo adicionarse (inmutabilidad del registro)
- [ ] La firma digital o mecanismo equivalente es obligatorio para validar entradas clínicas
- [ ] El sistema debe permitir portabilidad: el paciente puede pedir su historia en formato interoperable
- [ ] Tiempo mínimo de conservación: 15 años desde la última atención (implica estrategia de archivado)

**RIPS — Registro Individual de Prestación de Servicios de Salud:**
- [ ] Cada atención debe generar un reporte RIPS en el formato exigido por MINSALUD
- [ ] El sistema debe permitir radicación electrónica de RIPS ante MINSALUD
- [ ] Los RIPS son insumo directo para la facturación a EPS — debe haber trazabilidad entre atención y cuenta médica

**Implicaciones arquitectónicas directas:**
- Auditoría de accesos → necesidad de un log inmutable y centralizado (no editable por administradores)
- Inmutabilidad del registro clínico → la HCE no usa UPDATE sobre registros médicos, solo INSERT
- Soberanía del dato → ningún dato clínico puede salir de servidores en territorio colombiano
- Portabilidad → el dominio HCE debe exponer los datos en un formato estándar (HL7 FHIR es el estándar internacional; verificar si MINSALUD lo exige)

---

## Datos clave a tener siempre presentes

- Normativa: **Ley 1581 (datos personales)**, **Resolución 2654 de 2019 (historia clínica electrónica)**, **RIPS** (reporte obligatorio a MINSALUD)
- El HCE es el dominio más crítico — errores aquí pueden costar vidas
- Pico de agendamiento: lunes 6-8am = 40% de la semana
- 3 regiones: Caldas, Risaralda, Quindío — múltiples hospitales con HIS heterogéneos
- Equipo nuevo en salud → la arquitectura debe ser comprensible y mantenible
- MVP en 12 meses de desarrollo (los primeros 6 son diseño y arquitectura)
