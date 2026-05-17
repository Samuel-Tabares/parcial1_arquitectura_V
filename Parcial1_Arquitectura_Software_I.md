# Programa de Ingeniería de Software
## Arquitectura de Software I
### V Semestre
### Parcial 1

Docente: Santiago Jaramillo López

## Descripción
La Secretaría de Salud del Eje Cafetero está licitando el diseño arquitectónico de SaludEje, una
plataforma digital para integrar los servicios de salud de los hospitales públicos de Caldas, Risaralda y
Quindío. Actualmente cada hospital opera con sistemas aislados y sin interoperabilidad: un paciente de
Armenia que se accidenta en Manizales no tiene su historia clínica disponible, los laboratorios no
comparten resultados con los médicos en tiempo real, y la facturación a las EPS se procesa de forma
manual.
Como equipo de arquitectura de software, su tarea es proponer el diseño arquitectónico completo de
SaludEje. El sistema debe cubrir cinco dominios de negocio:

    1. Historia Clínica Electrónica (HCE): Registro, consulta y actualización del historial médico
    completo de los pacientes. Debe cumplir la normativa colombiana de datos sensibles. Es el dominio
    más crítico del sistema: un error aquí puede costar vidas.

    2. Agendamiento y consultas: Citas médicas con especialistas, atención de urgencias y
    telemedicina. Presenta picos de demanda extremos (lunes entre 6am y 8am concentra el 40% de las
    agendas de la semana) y períodos de baja carga. La indisponibilidad en horas pico tiene impacto
    directo en la atención de pacientes.

    3. Laboratorio y diagnóstico: Solicitud de exámenes clínicos, carga de resultados por laboratorios
    externos y notificación inmediata al médico tratante. Integra múltiples laboratorios privados y públicos
    de la región con protocolos y formatos de datos heterogéneos.

    4. Farmacia hospitalaria: Dispensación de medicamentos, control de inventario de fármacos,
    alertas automáticas de stock mínimo y productos próximos a vencer. Cada hospital tiene su propio
    inventario pero deben poder consultarse entre sí para traslados de urgencia.

    5. Facturación y glosas: Generación de cuentas médicas para EPS como Sura y Nueva EPS,
    radicación electrónica ante las aseguradoras y seguimiento del estado de las glosas. Debe integrarse
    con los sistemas de información de cada EPS, que tienen APIs y formatos distintos.


Los actores externos del sistema son: pacientes, médicos, enfermeras, laboratoristas, farmacéuticos,
administradores hospitalarios, EPS (Sura, Coosalud, Nueva EPS), MINSALUD y laboratorios clínicos
externos.


## Contexto del proyecto:

    ●​ Presupuesto: La Secretaría de Salud tiene aprobado un presupuesto de $850 millones de pesos
       colombianos para el desarrollo e implementación del sistema en su primera fase (18 meses).
    ●​ Equipo de desarrollo: Se contratará un equipo de 12 ingenieros de software distribuidos en 3
       equipos. Ninguno de los ingenieros tiene experiencia previa trabajando con sistemas de salud ni
       con la normativa del sector.
    ●​ Tiempo de ejecución: 18 meses para tener el MVP en producción. Los primeros 6 meses son
       de diseño y arquitectura; los 12 restantes de desarrollo e implementación.
    ●​ Infraestructura disponible: Los hospitales cuentan con servidores locales propios pero con
       capacidad limitada y sin redundancia. El presupuesto incluye una partida para nube, pero la
       Secretaría exige que los datos clínicos de los pacientes permanezcan en territorio colombiano.
    ●​ Sistemas existentes: Cada hospital ya tiene su propio sistema de información hospitalario (HIS)
       con bases de datos propias. Estos sistemas no pueden ser reemplazados en la primera fase,
       SaludEje debe integrarse con ellos.



Pueden trabajar de forma individual, parejas, o máximo grupos de tres personas.


La entrega es un documento enviado al correo del docente antes del domingo 17 de mayo a las
11:59 PM. No hay sustentación: lo que se entrega es lo que se califica.


## Requerimientos mínimos obligatorios
### A. Identificación de drivers arquitectónicos
Antes de tomar cualquier decisión de diseño, el documento debe identificar y priorizar los drivers
arquitectónicos del sistema. Los drivers son el punto de partida que justifica todo lo demás.
   •​ Mínimo 4 drivers funcionales: los casos de uso o funcionalidades con mayor impacto en la
       arquitectura. No es un listado de funciones, son las funcionalidades cuyo cambio obligaría a
       rediseñar la arquitectura.
   •​ Mínimo 3 drivers de restricción: limitaciones técnicas, organizacionales o regulatorias que el
       diseño no puede ignorar.


### B. Atributos de calidad con escenarios
El documento debe incluir mínimo 4 atributos de calidad desarrollados formalmente.
   •​ Cada atributo debe estar justificado: ¿por qué es crítico para SaludEje específicamente?
   •​ Cada atributo debe tener al menos un escenario de calidad completo con sus componentes
       respectivos.
   •​ Se deben identificar explícitamente los trade-offs entre los atributos elegidos: ¿qué se sacrifica al
       priorizar uno sobre otro? ¿Cómo se maneja ese trade-off en el diseño?


### C. Architecture Decision Records (ADRs)
El documento debe incluir mínimo 5 ADRs completos que documenten las decisiones arquitectónicas
más significativas del diseño.
   •​ Recuerden que las alternativas descartadas son opciones técnicamente válidas. El ADR debe
       demostrar que se evaluaron opciones reales.
   •​ Los ADRs deben estar numerados secuencialmente (ADR-001, ADR-002...) y deben conectarse
       con el diagrama de arquitectura y los atributos de calidad del documento.
   •​ Al menos un ADR debe documentar la justificación de la elección del modelo o modelos
       arquitectónicos principales del sistema.


### D. Modelos arquitectónicos y su justificación
El diseño arquitectónico debe emplear mínimo 3 modelos de arquitectura distintos, aplicados donde el
contexto lo justifique.
   •​ Tener en cuenta: no todos los dominios del sistema necesitan el mismo modelo. SaludEje tiene
       cinco dominios con características muy distintas, el diseño debe reflejar esa heterogeneidad.
   •​ Para cada modelo utilizado: explicar qué problema resuelve en el contexto de SaludEje, qué
       ventajas aporta y qué trade-offs introduce. Esta justificación puede ir en el ADR correspondiente o
       en una sección dedicada.
   •​ La combinación de modelos debe ser coherente: si se usan múltiples modelos, el documento debe
       explicar cómo se articulan entre sí y dónde están los límites entre ellos.


### E. Diagrama de arquitectura de alto nivel
El documento debe incluir un diagrama de arquitectura de alto nivel del sistema completo.
   •​ El diagrama debe mostrar: los actores externos, los dominios o módulos principales del sistema,
      las relaciones y flujos de comunicación entre ellos, las fuentes de datos principales y los sistemas
      externos con los que se integra.
   •​ El diagrama debe ser consistente con los ADRs y los modelos arquitectónicos elegidos. Si el ADR
      dice que el dominio de agendamiento tiene una base de datos propia, el diagrama debe reflejarlo.


### F. Estructura y calidad del documento
   •​ El documento debe tener una estructura clara y navegable: sección de drivers, sección de
       atributos de calidad, sección de ADRs, sección de modelos arquitectónicos y diagrama de alto
       nivel.
   •​ Las decisiones del documento deben ser coherentes entre sí: los ADRs deben responder a los
       drivers, el diagrama debe reflejar los ADRs, los atributos de calidad deben estar conectados con
       las tácticas elegidas.
   •​ El documento debe leerse como un argumento técnico coherente, no como una lista de
       respuestas independientes. El lector debe poder seguir el hilo: este es el problema, estos son los
       drivers, estas son las decisiones que tomamos y por qué.


## Criterios de evaluación

| Criterio | Qué se evalúa | Peso |
|---|---|---:|
| Drivers arquitectónicos | Identificación y priorización de drivers funcionales y de restricción. Los drivers deben ser pertinentes al dominio de SaludEje. | 15% |
| Atributos de calidad y escenarios | Mínimo 4 atributos con justificación y escenarios completos. Trade-offs identificados y argumentados. | 20% |
| ADRs | Mínimo 5 ADRs completos. | 25% |
| Modelos arquitectónicos y justificación | Mínimo 3 modelos distintos aplicados con criterio. Cada modelo está justificado en función de los drivers y atributos del sistema. La combinación de modelos debe ser coherente. | 20% |
| Diagrama de arquitectura de alto nivel | El diagrama es claro, completo y consistente con el resto del documento. Muestra actores, dominios, relaciones, datos y sistemas externos. | 15% |
| Coherencia y calidad del documento | Las decisiones del documento son coherentes entre sí. El documento se lee como un argumento técnico, no como respuestas independientes. Drivers, ADRs, modelos y diagrama cuentan la misma historia. | 5% |
| TOTAL |  | 100% |



## Notas importantes
1. No se aceptan entregas por fuera del plazo. El documento debe llegar al correo del docente antes del
domingo 17 de mayo a las 11:59 PM.
2. No hay sustentación. Lo que se entrega es lo que se califica, el documento debe ser suficientemente
claro y completo para hablar por sí solo.
3. El caso es abierto: no existe una única respuesta correcta. Se evalúa la calidad del razonamiento, la
coherencia de las decisiones y la profundidad del análisis, no si eligieron el 'modelo correcto'.
