# Implementación real del MVP: Power Platform + Microsoft Lists + OpenRouter

**Estado:** 1 de noviembre de 2026 · **Tipo:** prototipo académico con datos y operaciones simulados.  
**Proyecto:** Gestión Inteligente de Siniestros Automotrices.  
**Responsable de esta línea de trabajo:** Felipe Agudelo (@pipeagudelo3).  
**Fuente funcional:** backlog de GitHub Issues y entregable original de análisis/BPMN.

> **Importante:** este documento describe lo que efectivamente se ha configurado, lo que se ha decidido y lo que todavía falta. No es una descripción de un sistema de seguros en producción. Los pagos, las pólizas, el core, las aprobaciones y los avisos son exclusivamente simulados.

## 1. Decisiones y restricciones técnicas

| Tema | Decisión del MVP | Estado |
| --- | --- | --- |
| Nube e integración | No contamos con acceso ni créditos de Azure. No dependeremos de servicios Azure específicos. | Decidido |
| Suite de automatización | Microsoft Power Platform del entorno académico EAFIT. | Acceso verificado |
| Orquestación/BPMS | Flujos de nube en Power Automate. | Por implementar |
| Aplicación de captura | Power Apps (Canvas App) orientada a móvil. | Por implementar |
| Persistencia | Microsoft Lists sobre SharePoint, en vez de Dataverse, porque el usuario no pudo crear tablas en el entorno personal. | Dos listas creadas |
| Sistema core de aseguradora | **Simulación** mediante las listas y flujos; no existe core real ni integración productiva. | Diseño acordado |
| IA | OpenRouter, modelo **openai/gpt-4o-mini**, invocado desde un conector personalizado. | Conector creado; prueba de texto HTTP 200 |
| Análisis de fotos | Se prevé enviar imágenes y descripción al modelo multimodal. | **No validado aún** |
| Severidad | **Leve, Medio, Grave**; “Medio” equivale a “moderado” en la HU-09 original. | Decidido |
| Aprobación, rechazo, pagos y avisos | Solo cambios de estado y registros de simulación; **sin aprobar pólizas reales, transferir dinero ni enviar mensajes a clientes reales**. | Decidido |
| Colaboración | GitHub Issues para las HU y este repositorio para acuerdos, arquitectura, pruebas y documentación. | En curso |

**Licenciamiento observado:** Power Automate Free, Power Automate para Office 365 y prueba **Power Automate Plan 2 Trial** (en la interfaz aparece vencimiento 18/01/2027). Se muestran conectores estándar, premium y personalizados; eso no garantiza que todos los colaboradores tengan el mismo acceso. **Antes de una demo en otra cuenta se debe validar cada licencia y conexión.** El conector personalizado depende de permisos premium/trial; no asumir disponibilidad permanente.

**Alcance académico:** la arquitectura propuesta en el entregable original (Azure + Dataverse + RPA sobre core/ERP) se conserva como referencia TO-BE, pero la implementación demostrable usa las herramientas accesibles. Por tanto, los resultados del caso de negocio (70 % de aprobación automática, 7 días → ~1 día, ROI, etc.) son **supuestos de simulación, no mediciones reales**.

## 2. Arquitectura objetivo del prototipo

~~~mermaid
flowchart TD
    U["Usuario de prueba en móvil"] --> PA["Power Apps · formulario"]
    PA --> LS["Microsoft Lists · Siniestros"]
    LS --> FL["Power Automate · orquestador"]
    FL --> LP["Microsoft Lists · PolizasSimuladas"]
    FL --> OR["Conector OpenRouter-Siniestros"]
    OR --> AI["GPT-4o mini · clasificación"]
    FL --> LA["AnalisisIA · pendiente"]
    FL --> LH["HistorialSiniestros · pendiente"]
    FL --> LN["NotificacionesSimuladas · pendiente"]
    FL --> LS
~~~

- **Power Apps** captura descripción y hasta 10 fotos, y crea un ítem de Siniestros.
- **Power Automate** valida póliza, coordina la llamada IA, aplica reglas determinísticas y cambia estados; se debe evitar que las actualizaciones disparen bucles infinitos.
- **Microsoft Lists** sirve como almacenamiento relacional ligero por identificadores compartidos, sin depender de Dataverse.
- **OpenRouter** realiza la clasificación como apoyo; no tiene autoridad para aprobar por sí solo.
- **HistorialSiniestros** registra cada transición, decisión y reasignación; **NotificacionesSimuladas** registra avisos que *se habrían enviado*.

### Contrato de datos compartidos

| Identificador | Origen / uso | Regla |
| --- | --- | --- |
| **SiniestroId** | Campo Título de Siniestros, por ejemplo SIN-0001 | Único, estable; repetir en AnalisisIA, HistorialSiniestros y NotificacionesSimuladas. |
| **NumeroPoliza** | Siniestros.NumeroPoliza ↔ Título de PolizasSimuladas | Coincidencia exacta; no asumir póliza válida si hay cero o varios resultados. |
| **Severidad** | Siniestros y AnalisisIA | Valores Leve, Medio, Grave; “SinClasificar” solo como estado inicial. |
| **RiesgoFraude** | Siniestros y AnalisisIA | Escala didáctica 0–100, **no** un detector de fraude validado. |
| **ConfianzaIA** | Siniestros y AnalisisIA | Porcentaje 0–100; tratar como indicador de la respuesta, no como probabilidad calibrada sin evaluación. |
| **Estado / Decision** | Siniestros | Estado del flujo y ruta tomada; registrar siempre el motivo. |

**Relación:** un siniestro tiene un número de póliza; una póliza puede asociarse a varios siniestros. Un siniestro puede tener varios análisis, eventos de historial y avisos. En Lists se usará igualdad de códigos, sin relaciones Dataverse.

## 3. Inventario de listas: verificadas y pendientes

### A. Siniestros — **creada, 17 columnas**

Almacenada en Microsoft Lists del entorno personal de EAFIT; actualmente sin siniestros de prueba cargados.

| Columna | Tipo | Uso |
| --- | --- | --- |
| Título | Texto de Lists | Código de caso, p. ej. SIN-0001 |
| NombreAsegurado | Texto | Nombre ficticio |
| DocumentoAsegurado | Texto | Documento ficticio |
| NumeroPoliza | Texto | Póliza simulada |
| PlacaVehiculo | Texto | Vehículo simulado |
| FechaAccidente | Fecha y hora | Momento del accidente |
| DescripcionAccidente | Varias líneas de texto | Relato de prueba |
| MontoEstimadoCOP | Moneda COP, 0 decimales | Monto de prueba |
| Estado | Elección | Borrador, PendienteAnalisis, EnAnalisis, PendienteRevision, Aprobado, Rechazado, Cerrado; inicial Borrador |
| Severidad | Elección | SinClasificar, Leve, Medio, Grave; inicial SinClasificar |
| ConfianzaIA | Número 0–100 | Confianza, 0 decimales |
| RiesgoFraude | Número 0–100 | Riesgo didáctico, 0 decimales |
| CoberturaValida | Sí/No | Inicial No |
| Procesado | Sí/No | Antirreproceso; inicial No |
| Decision | Elección | Pendiente, AprobadoAutomaticamente, RevisionManual, Rechazado; inicial Pendiente |
| MotivoDecision | Varias líneas de texto | Explicación auditable |
| AjustadorAsignado | Texto | Usuario simulado o miembro del equipo |

### B. PolizasSimuladas — **creada, 9 columnas**

También en Microsoft Lists del entorno personal EAFIT; sin pólizas de prueba cargadas.

| Columna | Tipo | Uso |
| --- | --- | --- |
| Título | Texto de Lists | Código de póliza, p. ej. POL-1001 |
| NombreAsegurado | Texto | Asegurado ficticio |
| DocumentoAsegurado | Texto | Documento ficticio |
| PlacaVehiculo | Texto | Vehículo ficticio |
| EstadoPoliza | Elección | Activa, Suspendida, Vencida, Cancelada; sin valor predeterminado |
| VigenciaDesde | Fecha, sin hora | Inicio |
| VigenciaHasta | Fecha, sin hora | Fin |
| CoberturaDanios | Sí/No | Inicial No, por seguridad |
| LimiteCoberturaCOP | Moneda COP, 0 decimales | Máximo de cobertura simulado |

### C. AnalisisIA — **pendiente de crear**

Esquema recomendado:

| Columna | Tipo |
| --- | --- |
| Título = SiniestroId | Texto |
| FechaAnalisis | Fecha y hora |
| ModeloUsado | Texto; p. ej. openai/gpt-4o-mini |
| TipoDanio | Texto |
| Severidad | Elección: Leve, Medio, Grave, SinClasificar |
| ConfianzaIA | Número 0–100 |
| RiesgoFraude | Número 0–100 |
| ResumenAnalisis | Varias líneas |
| Justificacion | Varias líneas |
| ResultadoTecnico | Elección: Correcto, Error, NoConcluyente |

Guardar una fila por intento para preservar el historial de IA, no sobrescribir el anterior.

### D. HistorialSiniestros — **pendiente de crear**

Esquema recomendado: **Título** (SiniestroId), **FechaEvento** (fecha/hora), **Accion** (texto), **EstadoAnterior** (texto), **EstadoNuevo** (texto), **Responsable** (texto), **Origen** (Automatizacion/Usuario), **Motivo** (varias líneas), **AsignadoAntes** (texto), **AsignadoDespues** (texto). Una fila por transición o reasignación.

### E. NotificacionesSimuladas — **pendiente de crear**

Esquema recomendado: **Título** (SiniestroId), **FechaEvento** (fecha/hora), **Canal** (Correo/SMS), **DestinatarioSimulado** (texto ficticio), **TipoEvento** (texto), **Mensaje** (varias líneas), **EstadoSimulacion** (Generada/Error). **No usar conectores de SMS ni enviar mensajes a destinatarios reales**.

### F. ConfigReglasSimuladas — **opcional / siguiente fase**

Para HU-15 (#37), se podrán mantener límites y versiones de reglas sin editar flujos. No es necesaria para cerrar primero las seis HU de Felipe, pero los umbrales no deben esconderse como si ya estuvieran aprobados por el equipo.

## 4. Conector personalizado de OpenRouter

**Nombre configurado:** OpenRouter-Siniestros.  
**Plataforma:** Power Automate, conectores personalizados.  
**Método:** POST, **host:** openrouter.ai, **ruta:** /api/v1/chat/completions, **HTTPS**.  
**Seguridad:** API Key en encabezado Authorization, con formato Bearer, administrada en la conexión personal.  
**Modelo usado en prueba:** openai/gpt-4o-mini.  
**Operación de prueba:** ProbarConexion.  
**Evidencia observada:** prueba de **texto** con respuesta HTTP **200**; **NO** demuestra aún clasificación de fotografías, ni el flujo extremo a extremo.

Ejemplo conceptual de solicitud (sin claves ni datos personales):

~~~json
{
  "model": "openai/gpt-4o-mini",
  "messages": [
    {
      "role": "user",
      "content": "Clasifica este daño automotriz ficticio. Responde con una severidad Leve, Medio o Grave y una explicación breve."
    }
  ],
  "max_tokens": 200,
  "stream": false
}
~~~

Para HU-09, el siguiente paso es probar el formato multimodal de OpenRouter con contenido de texto + image_url, fotos **sintéticas** y una respuesta JSON validable. Si la llamada, el JSON o la confianza fallan, el caso debe pasar a revisión manual. No escribir claves de API, cuerpos con datos sensibles ni respuestas de terceros en GitHub.

**Referencias oficiales:**
- OpenRouter: https://openrouter.ai/docs/api/api-reference/chat/send-chat-completion-request
- Modelo: https://openrouter.ai/openai/gpt-4o-mini/overview
- Control de adjuntos Power Apps: https://learn.microsoft.com/en-us/power-apps/maker/canvas-apps/controls/control-attachments
- Licenciamiento: https://learn.microsoft.com/en-us/power-platform/admin/power-automate-licensing/types

## 5. Propuesta de aprobaciones automáticas (simuladas)

**Son umbrales iniciales propuestos para pruebas, pendientes de ratificación del equipo; no criterios de suscripción ni autorización financiera real.**

### Precondiciones estrictas

1. El siniestro tiene código, póliza y fecha de accidente válidos.
2. Existe **exactamente una** póliza con ese código.
3. Su estado es **Activa**, la fecha del accidente cae entre VigenciaDesde y VigenciaHasta (ambas incluidas), CoberturaDanios = Sí.
4. MontoEstimadoCOP es positivo y no excede LimiteCoberturaCOP.
5. El análisis de IA terminó con datos bien formados, rangos 0–100 y severidad reconocida.
6. No existe ya una decisión final para ese SiniestroId.

### Tabla de decisión

| Condición de simulación | Decisión | Estado |
| --- | --- | --- |
| Póliza inexistente, inactiva, fuera de vigencia, sin cobertura o monto mayor al límite | Rechazado con motivo verificable | Rechazado |
| Toda precondición válida **y** Severidad = Leve **y** MontoEstimadoCOP <= 2.000.000 **y** RiesgoFraude <= 20 **y** ConfianzaIA >= 85 | AprobadoAutomaticamente | Aprobado |
| Severidad = Medio o Grave; riesgo > 20; monto > 2.000.000; confianza < 85; falta evidencia o error de IA | RevisionManual, con motivo y ajustador | PendienteRevision |
| RiesgoFraude >= 60 | RevisionManual prioritaria con motivo “revisión por posible fraude” | PendienteRevision |

**Prioridad de evaluación:** validar primero póliza/cobertura; después reglas de riesgo (>= 60); después autoaprobación; lo demás se deriva a revisión. El modelo solo **sugiere** severidad; las reglas definitivas las implementa Power Automate. La decisión siempre se almacena en Siniestros y se copia a HistorialSiniestros con fecha, responsable/origen y motivo. El umbral 70/25/5 del BPMN es un **supuesto global para simular**, no una proporción que deba forzarse alterando decisiones por caso.

## 6. Flujos a construir

| Flujo sugerido | Disparador | Resultado | Estado |
| --- | --- | --- | --- |
| F01_RegistrarSiniestro | Alta desde Power Apps en Siniestros | Código de caso estable; Estado=PendienteAnalisis; historial “Creado” | Pendiente |
| F02_ValidarYAnalizar | Ítem nuevo/modificado con Estado=PendienteAnalisis y Procesado=No | Busca póliza; llama IA si procede; escribe AnalisisIA; aplica reglas; actualiza Siniestros e historial | Pendiente |
| F03_NotificacionSimulada | Cambio de Estado detectado mediante comparación con historial | Registra entrada Correo/SMS ficticia en NotificacionesSimuladas | Pendiente |
| F04_Reasignacion | Cambio aprobado de AjustadorAsignado | Registra anterior/nuevo, fecha, actor y motivo en HistorialSiniestros | Pendiente |

**Antibucle e idempotencia:** cada flujo debe comprobar estado, Procesado y si ya existe el evento/decisión para el caso antes de escribir. Actualizar un elemento desde un flujo puede activar de nuevo un disparador “cuando se crea o modifica”. Los errores de API, timeouts, registros de póliza duplicados o valores inesperados no pueden conducir a aprobación automática.

**Power Apps móvil:** usar formulario enlazado a Siniestros; aceptar 10 imágenes como máximo, validar extensión y tamaño y probar captura/carga de adjuntos en dispositivo móvil real. El control clásico de adjuntos requiere el contexto de un formulario de listas y en móvil puede agregar archivos de uno en uno. Documentación oficial: https://learn.microsoft.com/en-us/power-apps/maker/canvas-apps/controls/control-attachments

## 7. HU asignadas a Felipe: implementación y criterio demostrable

| HU / Issue | Funcionalidad | Evidencia esperada |
| --- | --- | --- |
| [HU-01](https://github.com/SaraEcheGom/Automatizacion/issues/23) | Reportar desde celular | App responsiva, hasta 10 fotos, siniestro creado con código |
| [HU-05](https://github.com/SaraEcheGom/Automatizacion/issues/27) | Registro automático en core | Alta completa en **Siniestros como core simulado** sin transcripción manual |
| [HU-09](https://github.com/SaraEcheGom/Automatizacion/issues/31) | Clasificar fotos | Imagen de prueba + salida Leve/Medio/Grave, confianza, justificación y trazabilidad |
| [HU-13](https://github.com/SaraEcheGom/Automatizacion/issues/35) | Aprobación automática | Caso simple de bajo monto/riesgo aprueba sin tarea humana; otros se enrutan según reglas |
| [HU-17](https://github.com/SaraEcheGom/Automatizacion/issues/39) | Reasignar casos | Cambiar ajustador y ver evento con asignación anterior, nueva, fecha y motivo |
| [HU-21](https://github.com/SaraEcheGom/Automatizacion/issues/43) | Avisos correo y SMS | Dos registros simulados por cambio de estado, generados en menos de 5 minutos; **sin envío real** |

**Atención al criterio de HU-21:** el texto original exige mensajes *enviados*. Para este MVP solo se demostrará la **generación de avisos simulados**. El equipo/docente debe aceptar expresamente esta adaptación; no se afirmará que se cumplió un envío real.

**HU-09:** el criterio original usa la palabra “moderado”; en la interfaz se utiliza **Medio**. El mapeo debe documentarse en la demo.

## 8. Seguridad y límites explícitos

- Solo nombres, documentos, placas, fotografías, correos y teléfonos **inventados**. Ningún siniestro o contrato real.
- El repositorio es **público**: **nunca** subir API keys, cadenas Authorization, conexiones de Power Automate exportadas con credenciales, enlaces privados con tokens, datos personales ni capturas con secretos.
- Mantener el secreto en la conexión del conector personalizado; restringir su uso a cuentas autorizadas.
- Acceso a Lists según los permisos del tenant. Las listas creadas están en **Mis listas** del espacio personal, **no se ha verificado acceso del resto del equipo**. Antes de integrar, comprobar permisos y si conviene moverlas a un sitio SharePoint de equipo.
- RNF-03 (datos personales), RNF-04 (roles), RNF-05 (trazabilidad), RNF-01 (< 2 min de IA) y RNF-07 (< 10 min en móvil) **no están certificados por la prueba HTTP 200**.
- No confundir “automático” con “aprobación real”: la decisión solo modifica registros de simulación.
- Distinguir “hecho” de “pendiente” en toda presentación.

## 9. Próximos pasos por dependencia

1. Preparar 5–8 pólizas y siniestros **completamente ficticios** que cubran aprobación, vencimiento, falta de cobertura, riesgo alto y error de IA.
2. Crear AnalisisIA, HistorialSiniestros y NotificacionesSimuladas.
3. Conectar Power Apps a Siniestros; validar desde teléfono el límite de 10 fotos.
4. Probar llamada multimodal real al conector (solo imágenes sintéticas) y salida JSON robusta.
5. Configurar F01–F04 y pruebas funcionales/negativas sin duplicación.
6. Ejecutar y registrar la matriz de pruebas de la sección 11.
7. Someter a revisión de equipo los umbrales de decisión y la equivalencia de avisos simulados de HU-21.

## 10. Acuerdos de equipo y mantenimiento

- Cada ajuste relevante se registra en un PR o commit/documentación y se vincula a la HU correspondiente.
- Las capturas de evidencia deben ocultar nombres reales de cuentas, correos y cualquier clave.
- Marcar Issues como finalizados **solo después** de que su prueba de aceptación pase.
- Las reglas propuestas de este documento pueden revisarse en equipo; registrar fecha, versión y motivo cuando se cambien.
- No modificar silenciosamente el diseño BPMN o las métricas del caso de negocio original: documentar siempre qué parte es **diseño esperado** y qué parte es **MVP ejecutado**.

## 11. Matriz mínima de pruebas y evidencias

**Regla general:** usar únicamente datos ficticios. Todas las pruebas siguientes están **pendientes** hasta que se ejecuten los flujos completos; el HTTP 200 del conector solo acredita conectividad de texto.

| ID | HU | Escenario de prueba | Resultado esperado |
| --- | --- | --- | --- |
| P01 | HU-01 | Alta móvil con datos completos y 1 foto | Se crea un siniestro con ID y datos completos. |
| P02 | HU-01 | Adjuntar exactamente 10 fotos | Se aceptan 10 archivos y quedan vinculados al caso. |
| P03 | HU-01 | Intentar adjuntar 11 fotos | La app bloquea el undécimo con explicación. |
| P04 | HU-01 | Fecha o póliza vacía y sin descripción | Valida campos y evita alta inválida. |
| P05 | HU-05 | Crear siniestro desde la app | La lista Siniestros refleja el caso sin digitación duplicada. |
| P06 | HU-05 | Reenviar el mismo evento o reintentar flujo | No duplica casos ni eventos de decisión. |
| P07 | HU-09 | Imagen sintética de daño leve | Devuelve Leve, confianza y justificación. |
| P08 | HU-09 | Imágenes sintéticas de daño medio y grave | Devuelve Medio/Grave y conserva la traza en AnalisisIA. |
| P09 | HU-09 | Imagen ilegible o error de proveedor/API | No se aprueba; se deriva a revisión y se registra error. |
| P10 | HU-13 | Póliza activa, vigente, con cobertura; Leve; COP 1.000.000; riesgo 10; confianza 90 | AprobadoAutomaticamente, Estado Aprobado; sin tarea humana ni pago real. |
| P11 | HU-13 | Póliza vencida o cancelada | Rechazado con motivo concreto y evento de auditoría. |
| P12 | HU-13 | Caso Grave o RiesgoFraude 80 | RevisionManual y Estado PendienteRevision. |
| P13 | HU-13 | Límite de cobertura inferior al monto solicitado | Rechazado con motivo de límite, sin aprobación automática. |
| P14 | HU-13 | ConfianzaIA 60 o datos IA malformados | RevisionManual, sin aprobación silenciosa. |
| P15 | HU-17 | Supervisor cambia asignado A por B | AjustadorAsignado=B; historial conserva A, B, fecha y motivo. |
| P16 | HU-17 | Reasignar sin motivo | Se bloquea o registra motivo obligatorio; no queda reasignación anónima. |
| P17 | HU-21 | Siniestro cambia de estado | Genera registros de Correo y SMS **simulados** en < 5 minutos. |
| P18 | HU-21 | El flujo reintenta el mismo cambio de estado | No duplica notificaciones. |
| P19 | RNF-05 | Auditar aprobación y rechazo | Cada transición tiene fecha, origen/responsable y motivo. |
| P20 | RNF-01 | Ejecutar lote de análisis de prueba | Medir latencia y evidenciar si < 120 s; no declararlo cumplido antes. |

**Evidencia por caso:** código de siniestro ficticio, captura del formulario, registro en Lists, historial de ejecución de flujo (sin secretos), salida estructurada de IA anonimizada, estado/decisión final, tiempo de extremo a extremo y resultado **Pasa/Falla**.

**Datos de arranque recomendados:** cinco pólizas ficticias (activa/vigente, vencida, sin cobertura, suspendida y cobertura insuficiente) y ocho siniestros ficticios (leve, medio, grave, riesgo alto, confianza baja, ausencia de póliza, vencimiento y reintento). Evitar usar identificaciones o imágenes de personas reales.

