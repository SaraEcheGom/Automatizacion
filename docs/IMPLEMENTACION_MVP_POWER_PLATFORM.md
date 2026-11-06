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
| Persistencia | Microsoft Lists sobre SharePoint, en vez de Dataverse, porque el usuario no pudo crear tablas en el entorno personal. | Cinco listas creadas en el sitio SharePoint del curso |
| Sistema core de aseguradora | **Simulación** mediante las listas y flujos; no existe core real ni integración productiva. | Diseño acordado |
| IA | OpenRouter, modelo **openai/gpt-4o-mini**, invocado desde un conector personalizado. | Operación `ClasificarDanio` (texto + imágenes) usada por FL01 (ver §16) |
| Análisis de fotos | Hasta 3 fotos y la descripción se envían al modelo multimodal. | **Probado con fotos sintéticas (2026-11-03, ver §16)** |
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

Almacenada en el sitio SharePoint del curso (Section_SI3017). Contiene registros ficticios de prueba (ver §12).

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

También en el sitio SharePoint del curso (Section_SI3017), con pólizas ficticias de prueba.

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

### C. AnalisisIA — **creada el 2026-11-02, 9 columnas + Título**

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

Guardar una fila por intento para preservar el historial de IA, no sobrescribir el anterior. Predeterminado de Severidad: SinClasificar; ConfianzaIA y RiesgoFraude limitados a 0–100 sin decimales.

### D. HistorialSiniestros — **creada el 2026-11-02, 9 columnas + Título**

Esquema recomendado: **Título** (SiniestroId), **FechaEvento** (fecha/hora), **Accion** (texto), **EstadoAnterior** (texto), **EstadoNuevo** (texto), **Responsable** (texto), **Origen** (Automatizacion/Usuario), **Motivo** (varias líneas), **AsignadoAntes** (texto), **AsignadoDespues** (texto). Una fila por transición o reasignación. Origen es elección (Automatizacion/Usuario), sin valor predeterminado.

### E. NotificacionesSimuladas — **creada el 2026-11-02, 6 columnas + Título**

Esquema recomendado: **Título** (SiniestroId), **FechaEvento** (fecha/hora), **Canal** (Correo/SMS), **DestinatarioSimulado** (texto ficticio), **TipoEvento** (texto), **Mensaje** (varias líneas), **EstadoSimulacion** (Generada/Error). **No usar conectores de SMS ni enviar mensajes a destinatarios reales**. Canal es elección sin predeterminado; EstadoSimulacion tiene predeterminado Generada.

### F. ConfigReglasSimuladas — **creada el 2026-11-03 (usada por FL01)**

Fila única `VIGENTE` con columnas MontoMaxAutoCOP (2.000.000), RiesgoMaxAuto (20), ConfianzaMinAuto (85), RiesgoFraudeRevision (60), Version (`v1-propuesta-2026-11-02`) y Notas. FL01 lee esta fila en cada ejecución; si no existe, usa esos mismos valores por defecto. Los umbrales siguen siendo una **propuesta pendiente de ratificación**.

Nota original:

Para HU-15 (#37), se podrán mantener límites y versiones de reglas sin editar flujos. No es necesaria para cerrar primero las seis HU de Felipe, pero los umbrales no deben esconderse como si ya estuvieran aprobados por el equipo.

## 4. Conector personalizado de OpenRouter

**Nombre configurado:** OpenRouter-Siniestros.  
**Plataforma:** Power Automate, conectores personalizados.  
**Método:** POST, **host:** openrouter.ai, **ruta:** /api/v1/chat/completions, **HTTPS**.  
**Seguridad:** API Key en encabezado Authorization, con formato Bearer, administrada en la conexión personal.  
**Modelo usado en prueba:** openai/gpt-4o-mini.  
**Operación actual:** ClasificarDanio (2026-11-03). Sustituyó a la operación de prueba ProbarConexion (solo texto), porque dos operaciones no pueden compartir el mismo método y ruta. Su cuerpo se definió desde una muestra con `messages[].content` como arreglo de partes `text` e `image_url`, `temperature`, `max_tokens` y `response_format` JSON.  
**Evidencia observada:** inicialmente, prueba de **texto** con HTTP 200. Desde el 2026-11-03, clasificación de fotos sintéticas extremo a extremo dentro de FL01 (ver §16).

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
| F01_RegistrarSiniestro | Alta desde Power Apps en Siniestros | Código de caso estable; Estado=PendienteAnalisis; historial “Creado” | **Implementado como FL02 (2026-11-02, ver §13)** |
| F02_ValidarYAnalizar | Ítem nuevo/modificado con Estado=PendienteAnalisis y Procesado=No | Busca póliza; llama IA si procede; escribe AnalisisIA; aplica reglas; actualiza Siniestros e historial | **Implementado como FL01 (2026-11-03, ver §16)** |
| F03_NotificacionSimulada | Cambio de Estado detectado mediante comparación con historial | Registra entrada Correo/SMS ficticia en NotificacionesSimuladas | **Implementado como FL03 (2026-11-02, ver §15)** |
| F04_Reasignacion | Cambio aprobado de AjustadorAsignado | Registra anterior/nuevo, fecha, actor y motivo en HistorialSiniestros | **Resuelto en la app de supervisión, sin flujo (2026-11-02, ver §14)** |

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
- Acceso a Lists según los permisos del tenant. Las cinco listas están en el sitio SharePoint del curso (Section_SI3017) y heredan sus permisos; **no se ha verificado el acceso de cada miembro del equipo**. Comprobarlo antes de integrar.
- RNF-03 (datos personales), RNF-04 (roles), RNF-05 (trazabilidad), RNF-01 (< 2 min de IA) y RNF-07 (< 10 min en móvil) **no están certificados por la prueba HTTP 200**.
- No confundir “automático” con “aprobación real”: la decisión solo modifica registros de simulación.
- Distinguir “hecho” de “pendiente” en toda presentación.

## 9. Próximos pasos por dependencia

1. Preparar 5–8 pólizas y siniestros **completamente ficticios** que cubran aprobación, vencimiento, falta de cobertura, riesgo alto y error de IA.
2. ~~Crear AnalisisIA, HistorialSiniestros y NotificacionesSimuladas.~~ Hecho el 2026-11-02.
3. ~~Conectar Power Apps a Siniestros; validar desde teléfono el límite de 10 fotos.~~ Hecho el 2026-11-02 (ver sección 12).
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
| P01 | HU-01 | Alta móvil con datos completos y 1 foto | Se crea un siniestro con ID y datos completos. **Pasa (2026-11-02, ver §12).** |
| P02 | HU-01 | Adjuntar exactamente 10 fotos | Se aceptan 10 archivos y quedan vinculados al caso. **Pasa (2026-11-02, ver §12).** |
| P03 | HU-01 | Intentar adjuntar 11 fotos | La app bloquea el undécimo con explicación. **Pasa (2026-11-02, ver §12).** |
| P04 | HU-01 | Fecha o póliza vacía y sin descripción | Valida campos y evita alta inválida. **Pasa (2026-11-02, ver §12).** |
| P05 | HU-05 | Crear siniestro desde la app | La lista Siniestros refleja el caso sin digitación duplicada. **Pasa (2026-11-02, ver §13).** |
| P06 | HU-05 | Reenviar el mismo evento o reintentar flujo | No duplica casos ni eventos de decisión. **Pasa (2026-11-02, ver §13).** |
| P07 | HU-09 | Imagen sintética de daño leve | Devuelve Leve, confianza y justificación. **Pasa (2026-11-03, ver §16).** |
| P08 | HU-09 | Imágenes sintéticas de daño medio y grave | Devuelve Medio/Grave y conserva la traza en AnalisisIA. **Pasa (2026-11-03, ver §16).** |
| P09 | HU-09 | Imagen ilegible o error de proveedor/API | No se aprueba; se deriva a revisión y se registra error. **Pasa parcialmente:** imagen ilegible → revisión manual (ver §16). El error del proveedor/API está cubierto por diseño pero **no se forzó en prueba**. |
| P10 | HU-13 | Póliza activa, vigente, con cobertura; Leve; COP 1.000.000; riesgo 10; confianza 90 | AprobadoAutomaticamente, Estado Aprobado; sin tarea humana ni pago real. **Pasa (2026-11-03, ver §16);** la IA devolvió confianza 85, no 90. |
| P11 | HU-13 | Póliza vencida o cancelada | Rechazado con motivo concreto y evento de auditoría. **Pasa (2026-11-03, ver §16).** |
| P12 | HU-13 | Caso Grave o RiesgoFraude 80 | RevisionManual y Estado PendienteRevision. **Pasa para el caso Grave (ver §16).** RiesgoFraude 80 **no se pudo provocar** con fotos sintéticas; la regla está implementada pero sin prueba. |
| P13 | HU-13 | Límite de cobertura inferior al monto solicitado | Rechazado con motivo de límite, sin aprobación automática. **Pasa (2026-11-03, ver §16).** |
| P14 | HU-13 | ConfianzaIA 60 o datos IA malformados | RevisionManual, sin aprobación silenciosa. **Pasa para confianza baja (30, ver §16).** Datos de IA malformados: cubierto por Parse JSON con esquema, **no forzado en prueba**. |
| P15 | HU-17 | Supervisor cambia asignado A por B | AjustadorAsignado=B; historial conserva A, B, fecha y motivo. **Pasa (2026-11-02, vista previa y teléfono real, ver §14).** |
| P16 | HU-17 | Reasignar sin motivo | Se bloquea o registra motivo obligatorio; no queda reasignación anónima. **Pasa (2026-11-02, vista previa y teléfono real, ver §14).** |
| P17 | HU-21 | Siniestro cambia de estado | Genera registros de Correo y SMS **simulados** en < 5 minutos. **Pasa (2026-11-02, ver §15).** |
| P18 | HU-21 | El flujo reintenta el mismo cambio de estado | No duplica notificaciones. **Pasa tras corrección (2026-11-02, ver §15).** |
| P19 | RNF-05 | Auditar aprobación y rechazo | Cada transición tiene fecha, origen/responsable y motivo. |
| P20 | RNF-01 | Ejecutar lote de análisis de prueba | Medir latencia y evidenciar si < 120 s; no declararlo cumplido antes. **Medido (2026-11-03):** 9 ejecuciones de FL01 entre 2 y 14 s (las 6 que llamaron a la IA, entre 7 y 14 s). Muestra pequeña; no es una prueba de carga. |

**Evidencia por caso:** código de siniestro ficticio, captura del formulario, registro en Lists, historial de ejecución de flujo (sin secretos), salida estructurada de IA anonimizada, estado/decisión final, tiempo de extremo a extremo y resultado **Pasa/Falla**.

**Datos de arranque recomendados:** cinco pólizas ficticias (activa/vigente, vencida, sin cobertura, suspendida y cobertura insuficiente) y ocho siniestros ficticios (leve, medio, grave, riesgo alto, confianza baja, ausencia de póliza, vencimiento y reintento). Evitar usar identificaciones o imágenes de personas reales.

## 12. Avance HU-01: app móvil de reporte (2026-11-02)

**Estado:** implementada y probada en teléfono real con datos y fotos sintéticos. Pendiente solo la aceptación del equipo/docente para cerrar el issue [#23](https://github.com/SaraEcheGom/Automatizacion/issues/23).

### Qué quedó implementado

- App de lienzo **GestionSiniestros - Reportar siniestro HU01** (diseño Teléfono), **guardada, publicada e incluida en la solución GestionSiniestros**. Solo la usa el propietario; no se compartió ni se ampliaron permisos.
- Formulario clásico `Form1` enlazado a la lista **Siniestros** (`FormMode.New`).
- Etiquetas visibles en español sin cambiar los nombres internos de las columnas: Nombre del asegurado (ficticio), Documento ficticio, Número de póliza simulada, Placa ficticia, Fecha y hora del accidente, Descripción del accidente, Monto estimado (COP), Fotografías del daño (1 a 10).
- Campos internos ocultos: `Título` = código `SIN-<GUID>` y `Estado` = `PendienteAnalisis`. No se muestran al reportante campos de decisión, fraude ni confianza.
- Código de caso: se genera una vez al abrir la pantalla y solo se regenera con "Reportar otro siniestro"; un reintento tras un fallo conserva el mismo código.
- Validaciones del botón **Enviar reporte** antes de `SubmitForm`:
  - texto obligatorio sin vacíos ni solo espacios (nombre, documento, póliza, placa, descripción);
  - fecha seleccionada y fecha/hora no futura;
  - monto numérico y mayor que 0;
  - entre 1 y 10 adjuntos, con extensión JPG, JPEG, PNG o WEBP.
- Control de adjuntos: `MaxAttachments = 10`, `MaxAttachmentSize = 10` MB. Los archivos se guardan con el propio `SubmitForm`.
- Prevención de doble envío: el botón queda deshabilitado mientras `varGuardando` es verdadero.
- Valores iniciales confirmados en la lista: Procesado = No, CoberturaValida = No, Decision = Pendiente, Severidad = SinClasificar. (El predeterminado de Estado en la lista es Borrador; la app lo fija en PendienteAnalisis.)

### Resultados de pruebas (teléfono real, datos y fotos sintéticos)

| ID | Escenario | Resultado | Evidencia |
| --- | --- | --- | --- |
| H01-01 | Envío con formulario vacío (Studio) | **Pasa** | Mensaje de validación; sin registro creado. |
| H01-02 | Alta completa con 1 foto | **Pasa** | Registro con código SIN-…, Estado PendienteAnalisis, 1 adjunto. |
| H01-03 | Alta con exactamente 10 fotos | **Pasa** | Registro con 10 adjuntos JPG/PNG verificado en la lista. |
| H01-04 | Intentar agregar la foto 11 | **Pasa** | El control no la acepta (límite alcanzado). |
| H01-05 | Nombre solo con espacios | **Pasa** | Envío bloqueado. |
| H01-06 | Fecha futura | **Pasa** | Envío bloqueado. |
| H01-07 | Monto vacío, no numérico, 0 y negativo | **Pasa** | Envío bloqueado en los cuatro casos. |
| H01-08 | Archivo con extensión no permitida (.bmp) | **Pasa** | Envío bloqueado. |
| H01-09 | Archivo de más de 10 MB | **Pasa** | El control no lo acepta. |
| H01-10 | Sin conexión al enviar y reintento | **Pasa** | Error mostrado; el reintento conservó el código y no hubo duplicados en la lista. |

H01-02 y H01-03 se verificaron directamente en Siniestros (código, campos, estado, adjuntos y ausencia de duplicados). H01-04 a H01-10 los ejecutó el responsable en el teléfono; el resultado es consistente con que no se crearon registros adicionales.

### Observaciones

- La prueba H01-03 se registró con monto 0,5 por error de digitación del probador; es un valor positivo válido según la regla actual. Si el equipo quiere un mínimo realista, debe añadirse como regla nueva.
- FL01 (*Validar siniestro simulado*) sigue bloqueado con la condición de disparador `@equals(1,0)`; no procesó los registros de prueba (Procesado = No). Sus pruebas funcionales siguen pendientes (HU-05, HU-09, HU-13).
- No se enviaron mensajes ni se hicieron pagos reales.

## 13. Avance HU-05: registro automático en el core simulado (2026-11-02)

**Estado:** implementada y probada con datos sintéticos. Pendiente solo la aceptación del equipo/docente para cerrar el issue [#27](https://github.com/SaraEcheGom/Automatizacion/issues/27).

### Qué quedó implementado

- La app de HU-01 escribe cada reporte directamente en **Siniestros**, que actúa como core simulado; no hay transcripción manual.
- Flujo **FL02 - Registrar alta de siniestro**, activo e incluido en la solución **GestionSiniestros**:
  1. Disparador "Cuando se crea un elemento" en Siniestros (solo creación, sondeo cada minuto).
  2. *Get items* en HistorialSiniestros con filtro `Title eq '<código>' and Accion eq 'Creado'` (máximo 1).
  3. Condición `length(resultado) = 0`:
     - **Sí:** crea el evento con Título = código, FechaEvento = `utcNow()`, Accion = Creado, EstadoNuevo = Estado del siniestro, Responsable = FL02, Origen = Automatizacion, Motivo = "Alta registrada desde la app móvil (HU-01). Registro simulado."
     - **No:** no escribe nada (idempotencia ante reintentos).
- FL02 no modifica Siniestros, así que no puede reactivarse a sí mismo ni disparar FL01.
- Listas nuevas creadas en el sitio del curso: AnalisisIA, HistorialSiniestros y NotificacionesSimuladas (ver §3).

### Resultados de pruebas

| ID | Escenario | Resultado | Evidencia |
| --- | --- | --- | --- |
| P05 | Alta desde la app móvil con datos y foto sintéticos | **Pasa** | Siniestro creado con estado PendienteAnalisis; FL02 se ejecutó en < 1 s y registró el evento "Creado" en HistorialSiniestros unos 30 s después del alta. |
| P06 | Reenviar dos veces la misma ejecución de FL02 | **Pasa** | Tres ejecuciones correctas para el mismo siniestro y un solo evento "Creado" en HistorialSiniestros. |

### Observaciones

- Los registros de prueba de HU-01 creados antes de activar FL02 no tienen evento "Creado"; se dejaron así intencionalmente.
- FL01 sigue bloqueado con `@equals(1,0)` y no procesó ningún registro.
- No se enviaron mensajes ni se hicieron pagos reales.

## 14. Avance HU-17: reasignar casos (2026-11-02)

**Estado:** implementada, publicada y probada en vista previa y en teléfono real con datos ficticios. Incluida en la solución GestionSiniestros. Pendiente solo la aceptación del equipo/docente para cerrar el issue [#39](https://github.com/SaraEcheGom/Automatizacion/issues/39).

### Qué quedó implementado

- App de lienzo separada **GestionSiniestros - Supervision HU17** (rol supervisor, diseño Teléfono), **publicada e incluida en la solución GestionSiniestros**; solo la usa el propietario, sin compartir ni ampliar permisos, para que el reportante de HU-01 nunca vea la reasignación.
- Pantalla con: selector del caso (Siniestros, más reciente primero), liquidador actual y estado, selector del nuevo liquidador (lista fija ficticia: Liquidador Ana, Bruno y Carla, todos ficticios), motivo obligatorio y botón **Reasignar caso**.
- Al reasignar:
  1. Valida caso seleccionado, motivo no vacío ni solo espacios y que el nuevo liquidador sea distinto del actual.
  2. Actualiza `AjustadorAsignado` en Siniestros.
  3. Si lo anterior funciona, registra en HistorialSiniestros: Accion = Reasignado, AsignadoAntes, AsignadoDespues, FechaEvento, Estado (anterior y nuevo, sin cambio), Responsable = usuario que reasigna, Origen = Usuario y Motivo.
  4. Si falla la actualización, no escribe en el historial; si falla solo el historial, lo informa explícitamente.
- Decisión de diseño: el registro lo hace la app y no un flujo F04, porque el disparador de SharePoint no expone el valor anterior de `AjustadorAsignado`.

### Resultados de pruebas (vista previa, datos ficticios)

| ID | Escenario | Resultado | Evidencia |
| --- | --- | --- | --- |
| P16a | Reasignar con motivo vacío | **Pasa** | Mensaje "El motivo es obligatorio"; sin cambios ni eventos. |
| P16b | Reasignar con motivo solo de espacios | **Pasa** | Mismo bloqueo; sin cambios ni eventos. |
| H17-01 | Reasignar al mismo liquidador | **Pasa** | Advertencia; sin cambios ni eventos. |
| P15a | Sin asignar → Liquidador Ana (ficticio) | **Pasa** | Siniestro actualizado; evento con antes "Sin asignar", después "Liquidador Ana (ficticio)", fecha, responsable y motivo. |
| P15b | Liquidador Ana → Liquidador Bruno (ficticios) | **Pasa** | Siniestro con Bruno; evento con antes "Liquidador Ana", después "Liquidador Bruno". |

### Resultados de pruebas (teléfono real, app publicada, datos ficticios)

| ID | Escenario | Resultado | Evidencia |
| --- | --- | --- | --- |
| H17-M01 | Abrir la app publicada en el celular y seleccionar el caso de prueba | **Pasa** | Muestra liquidador actual "Liquidador Bruno (ficticio)" y estado. |
| H17-M02 | Reasignar sin motivo | **Pasa** | Mensaje "El motivo es obligatorio"; sin cambios ni eventos. |
| H17-M03 | Reasignar al mismo liquidador | **Pasa** | Advertencia; sin cambios ni eventos. |
| H17-M04 | Liquidador Bruno → Liquidador Carla (ficticios) con motivo | **Pasa** | Mensaje de éxito; siniestro con "Liquidador Carla (ficticio)"; evento Reasignado con antes Bruno, después Carla, fecha, responsable y motivo. |

Verificado directamente en las listas: el caso de prueba quedó con "Liquidador Carla (ficticio)" y HistorialSiniestros tiene, para ese código, un evento "Creado" y tres "Reasignado" (Sin asignar → Ana → Bruno → Carla), en orden y sin duplicados.

### Riesgos y pendientes

- **Interacción con FL01:** cada reasignación modifica el siniestro. Cuando se desbloquee FL01 (disparador "creado o modificado" con Procesado = No), una reasignación podría volver a disparar el análisis. Debe resolverse al trabajar HU-09/HU-13, por ejemplo marcando Procesado al terminar el análisis o filtrando por Estado en la condición del disparador.
- El rol supervisor no está restringido técnicamente (cualquier persona con acceso a la app podría reasignar); queda como mejora de RNF-04.
- Prueba en teléfono real: **Pasa** (ver tabla anterior).

## 15. Avance HU-21: avisos simulados de correo y SMS (2026-11-02)

**Estado:** implementada y probada con datos ficticios. **No se envía ningún correo ni SMS real**: solo se generan registros simulados. Pendiente: que el equipo/docente acepte esta adaptación del criterio original ("mensajes enviados") y cierre el issue [#43](https://github.com/SaraEcheGom/Automatizacion/issues/43).

### Qué quedó implementado

- Flujo de nube **FL03 - Generar avisos simulados**, incluido en la solución GestionSiniestros y activado.
- Disparador: SharePoint "Cuando se crea o se modifica un elemento" en Siniestros, con control de simultaneidad = 1 (una ejecución a la vez, para evitar duplicados por carreras).
- Pasos:
  1. Obtiene de NotificacionesSimuladas el último aviso de **Correo** del mismo código de caso (`Title eq '<código>' and Canal eq 'Correo'`, orden `ID desc`, máximo 1).
  2. Condición: si su `TipoEvento` (o `ninguno` si no existe) es distinto de `CambioEstado:<Estado actual>`, el estado cambió desde el último aviso.
  3. Si cambió, crea dos registros: uno con Canal = Correo y otro con Canal = SMS. Cada uno lleva el código del caso, FechaEvento = fecha de modificación del siniestro, `TipoEvento = CambioEstado:<estado>`, un destinatario marcado "simulado (no se envía)" con el nombre ficticio del asegurado, un mensaje que empieza por "[SIMULADO - NO ENVIADO]" y EstadoSimulacion = Generada.
  4. Si no cambió (reasignaciones, ediciones de otros campos o reprocesos del mismo cambio), no hace nada.
- No usa conectores de Outlook, SMS ni similares.

### Resultados de pruebas (datos ficticios, caso de prueba SIN-7bd20af3…)

| ID | Escenario | Resultado | Evidencia |
| --- | --- | --- | --- |
| P17 | Estado PendienteAnalisis → PendienteRevision | **Pasa** | Avisos 1 (Correo) y 2 (SMS) creados 34 s después del cambio. |
| P18 (1.er intento) | Guardar de nuevo el mismo estado | **Falla** | Se crearon avisos duplicados (3 y 4). Causa: el diseñador clásico guardó la condición como texto escapado (`@@{...}`), así que comparaba dos cadenas literales siempre distintas. |
| P18 (tras corrección) | Guardar de nuevo el mismo estado | **Pasa** | La condición se corrigió en el diseñador nuevo (verificada en vista de código). FL03 se ejecutó y no creó avisos nuevos. |
| H21-01 | Estado PendienteRevision → PendienteAnalisis | **Pasa** | Avisos 5 (Correo) y 6 (SMS) creados 34 s después del cambio. El caso quedó de nuevo en PendienteAnalisis. |

Los avisos 3 y 4 se conservan en la lista como evidencia del defecto encontrado y corregido.

### Riesgos y pendientes

- **Adaptación del criterio:** HU-21 pide mensajes enviados; el MVP solo demuestra la generación de avisos simulados. Debe aceptarse expresamente.
- La deduplicación se basa en el último aviso registrado: si FL03 falla entre la creación del aviso de Correo y la del SMS, el reintento no crearía el SMS. Riesgo bajo para el prototipo.
- Cuando se desbloquee FL01, cada cambio de estado que haga generará avisos, que es el comportamiento esperado.
- FL01 sigue bloqueado con `@equals(1,0)`; las pruebas de HU-21 no lo dispararon.

## 16. Avance HU-09 y HU-13: clasificación desde fotos y aprobación automática (2026-11-03)

**Estado:** implementadas y probadas con pólizas, casos y fotos **ficticios/sintéticos**. FL01 quedó **desbloqueado** (se retiró `@equals(1,0)` con autorización de Felipe). Pendiente: aceptación del equipo/docente para cerrar [#31](https://github.com/SaraEcheGom/Automatizacion/issues/31) y [#35](https://github.com/SaraEcheGom/Automatizacion/issues/35), y ratificar los umbrales.

### Qué quedó implementado (FL01 - Validar siniestro simulado)

- **Disparador:** "Cuando se crea o modifica un elemento" en Siniestros, con condiciones `Procesado = No` **y** `Estado = PendienteAnalisis`. Como FL01 deja Procesado = Sí y cambia el estado, sus propias actualizaciones y las reasignaciones de HU-17 ya no lo vuelven a disparar (resuelve el riesgo anotado en §14).
- **Póliza:** busca hasta 2 pólizas con el número del siniestro y exige exactamente una.
- **Reglas** leídas de ConfigReglasSimuladas (§3.F), evaluadas en este orden:
  1. Póliza inexistente o duplicada, no activa, fuera de vigencia, sin cobertura de daños, monto no positivo o mayor que el límite → **Rechazado** con motivo concreto (no se llama a la IA).
  2. Sin fotos o con error/respuesta no válida de la IA → **RevisionManual / PendienteRevision**.
  3. RiesgoFraude ≥ 60 → **RevisionManual** prioritaria por posible fraude.
  4. Severidad Leve, monto ≤ 2.000.000, riesgo ≤ 20 y confianza ≥ 85 → **AprobadoAutomaticamente / Aprobado**, sin tarea humana ni pago real.
  5. Cualquier otro caso → **RevisionManual / PendienteRevision**.
- **IA (HU-09):** envía la descripción y hasta 3 fotos adjuntas (como `image_url` en base64) a openai/gpt-4o-mini con la operación ClasificarDanio. Pide un JSON con tipo_danio, severidad (Leve/Medio/Grave; "Medio" = "moderado" de la HU), confianza, riesgo_fraude, resumen y justificación, y lo valida con un esquema. El modelo solo sugiere; la decisión la aplican las reglas.
- **Trazabilidad:** una fila por análisis en AnalisisIA (incluido ResultadoTecnico Correcto/Error/NoConcluyente); Siniestros actualizado con Estado, Decision, MotivoDecision, Severidad, ConfianzaIA, RiesgoFraude, CoberturaValida y Procesado; un evento `DecisionAutomatica:<decisión>` en HistorialSiniestros con Responsable FL01 y Origen Automatizacion. FL03 genera los avisos simulados del cambio de estado.
- **Construcción:** el bloque principal se armó como un ámbito ("Analizar y decidir") y se pegó en el diseñador nuevo, porque el diseñador clásico escapa las expresiones en condiciones (ver §15).

### Resultados de pruebas (datos y fotos sintéticos)

| ID | Caso de prueba | Esperado | Resultado | Evidencia |
| --- | --- | --- | --- | --- |
| P07, P10 | SIN-PRU-HU13-A · POL-1001 activa · 1.000.000 · foto de rayón | Leve y aprobación automática | **Pasa** | IA: "Rayón superficial", Leve, confianza 85, riesgo 10 → Aprobado. |
| P08 | SIN-PRU-HU09-M · POL-1001 · 1.800.000 · foto de abolladura | Medio y revisión | **Pasa** | IA: "Abolladura con pintura levantada", Medio, 85, 10 → PendienteRevision. |
| P08, P12 | SIN-PRU-HU13-D · POL-1001 · 1.500.000 · foto de choque frontal | Grave y revisión | **Pasa** | IA: "Daño estructural y vidrio roto", Grave, 85, 20 → PendienteRevision. |
| P11 | SIN-PRU-HU13-B · POL-1002 vencida | Rechazo | **Pasa** | Rechazado: "La póliza no está activa (estado: Vencida)". |
| H13-01 | SIN-PRU-HU13-C · POL-1003 sin cobertura | Rechazo | **Pasa** | Rechazado: "La póliza no incluye cobertura de daños". |
| P13 | SIN-PRU-HU13-E · POL-1005 límite 500.000 · monto 1.000.000 | Rechazo por límite | **Pasa** | Rechazado: "El monto estimado supera el límite de cobertura de la póliza". |
| H13-02 | SIN-PRU-HU13-F · POL-9999 inexistente | Rechazo | **Pasa** | Rechazado: "No se encontró una póliza única con el número indicado". |
| H09-01 | SIN-PRU-HU09-X · sin fotos | Revisión por falta de evidencia | **Pasa** | PendienteRevision; AnalisisIA con ResultadoTecnico NoConcluyente; no se llamó a la IA. |
| P09, P14 | SIN-PRU-HU09-I · foto ilegible (gris con ruido) | No aprobar | **Pasa** | IA: Leve con confianza 30 ("la imagen no es legible") → PendienteRevision. |

Los 9 casos quedaron con Procesado = Sí y un evento FL01 cada uno en HistorialSiniestros. Las 9 ejecuciones de FL01 terminaron correctamente, entre 2 y 14 s.

### Riesgos y pendientes

- **No probados por no poder forzarlos con datos sintéticos:** RiesgoFraude ≥ 60, error del proveedor/timeout y JSON malformado. El diseño los envía a revisión manual, pero no hay evidencia de ejecución.
- La confianza la reporta el propio modelo; no está calibrada. El modelo respondió 85 en varios casos, justo en el umbral, así que conviene que el equipo revise ese valor.
- Las fotos son dibujos sintéticos, no fotografías reales; el resultado no demuestra precisión con daños reales.
- Cada análisis consume saldo de la clave de OpenRouter del equipo (unas pocas imágenes pequeñas por caso).
- Al cambiar el conector se eliminó la operación ProbarConexion; cualquier flujo que la usara debe pasar a ClasificarDanio (FL01 ya lo hace).


## 17. Avance HU-15, HU-24, HU-11, HU-08 y HU-02 (2026-11-05)

**Estado:** implementadas y probadas con datos **ficticios**. Ninguna envía correos o SMS reales, hace pagos ni usa pólizas reales. Pendiente: aceptación del equipo/docente para cerrar [#37](https://github.com/SaraEcheGom/Automatizacion/issues/37), [#46](https://github.com/SaraEcheGom/Automatizacion/issues/46), [#33](https://github.com/SaraEcheGom/Automatizacion/issues/33), [#30](https://github.com/SaraEcheGom/Automatizacion/issues/30) y [#24](https://github.com/SaraEcheGom/Automatizacion/issues/24).

### Qué cambió

- **HU-15 · Umbrales sin programar (#37):** FL01 lee en cada ejecución la fila `VIGENTE` de ConfigReglasSimuladas (monto máximo, riesgo máximo, confianza mínima, riesgo de fraude y versión). Para cambiar una regla basta editar esa fila en la lista; no se toca el flujo. El motivo de cada decisión cita la versión de reglas usada. Valores vigentes: 2.000.000 / 20 / 85 / 60, versión `v1-propuesta-2026-11-02`.
- **HU-24 · Historial de decisiones (#46):** nueva vista **"Historial de decisiones"** en HistorialSiniestros, agrupada por siniestro y ordenada por fecha. Muestra acción, estado anterior y nuevo, responsable, origen, motivo y asignación antes y después. FL01, FL02 y la app de HU-17 ya escriben esos campos.
- **HU-11 · Puntaje de riesgo de fraude (#33):**
  - FL01 calcula un puntaje de 0 a 100: `riesgo_fraude de la IA + 15 × (número de señales por reglas)`, con tope en 100.
  - Señales por reglas (simuladas):
    - monto ≥ 80 % del límite de la póliza;
    - descripción de menos de 25 caracteres;
    - accidente dentro de los primeros 30 días de vigencia.
  - La IA también devuelve `senales_fraude`.
  - Las señales quedan en texto en la nueva columna **SenalesFraude** (AnalisisIA y Siniestros) para que el analista vea por qué se marcó el caso.
  - Con puntaje ≥ 60 el caso va a revisión manual prioritaria.
- **HU-08 · Aviso inmediato de no cobertura (#30):**
  - Los rechazos citan una **cláusula simulada**:
    - 1 Identificación de la póliza
    - 2 Estado de la póliza
    - 3 Vigencia, con fechas
    - 4 Cobertura de daños
    - 5 Límite asegurado, con montos
    - 6 Valor reclamado
  - El motivo tiene el formato "SIMULACIÓN: no cubierto. Cláusula N – …". Reemplaza los textos de rechazo descritos en §16.
  - FL03 agrega "Motivo: …" a los avisos simulados de correo y SMS cuando el estado es Rechazado.
- **HU-02 · Número de caso (#24):**
  - Al crear el siniestro, FL02 genera en NotificacionesSimuladas un **correo simulado** (no enviado) con el número de caso. TipoEvento = `NumeroCaso`.
  - El filtro de deduplicación de FL03 ahora solo considera avisos `CambioEstado:*`, para que el aviso de número de caso no bloquee los de cambio de estado.
  - Adaptación: el número se informa por aviso simulado y en la app (el ID del caso), no por correo real.

### Resultados de pruebas (datos ficticios)

| Caso | Esperado | Resultado | Evidencia |
| --- | --- | --- | --- |
| SIN-PRU-HU15-A · umbral cambiado temporalmente a v2 (monto máx. 800.000) | Revisión | **Pasa** | PendienteRevision citando la versión v2. |
| SIN-PRU-HU15-B · umbral restaurado a v1 | Aprobación | **Pasa** | Aprobado. Las decisiones anteriores no cambiaron. |
| Vista "Historial de decisiones" | Cada evento con fecha, responsable y motivo | **Pasa** | 22 eventos revisados, todos con esos tres campos. |
| SIN-PRU-HU11-S · 8.500.000 · descripción "Choque." | Señales y revisión | **Pasa** | Puntaje 40 = IA 10 + 2 señales × 15; PendienteRevision; SenalesFraude con el detalle. |
| SIN-PRU-HU11-N · caso normal | Sin señales | **Pasa** | Puntaje 10, "sin señales adicionales", Aprobado. |
| SIN-PRU-HU08-V · accidente antes de la vigencia | Rechazo con cláusula | **Pasa** | "Cláusula 3 – Vigencia (simulada) … (2026-01-01 a 2027-12-31)". |
| SIN-PRU-HU08-L · monto sobre el límite | Rechazo con cláusula | **Pasa** | "Cláusula 5 – Límite asegurado" con los montos. |
| SIN-PRU-HU02-R · alta + rechazo por vigencia | Número de caso y avisos de rechazo con motivo | **Pasa** | 1 aviso `NumeroCaso` (FL02); 2 avisos `CambioEstado:Rechazado` (correo y SMS) con "Motivo: … Cláusula 3"; sin duplicados. |

### Riesgos y pendientes

- **HU-15:** cambiar ConfigReglasSimuladas afecta todos los casos nuevos de inmediato, y la lista no exige aprobación del cambio. Conviene restringir quién puede editarla.
- **HU-11:** el puntaje y el peso de 15 por señal son una propuesta simulada, no un modelo de fraude calibrado. Falta probar un caso con puntaje ≥ 60.
- **HU-02:** el aviso de número de caso solo se genera al crear el elemento. Si FL02 falla, no hay reintento automático.
- La operación ProbarConexion se eliminó del conector OpenRouter (ver §16).

## 18. Avance HU-18, HU-19, HU-14, HU-16, HU-20, HU-10, HU-12, HU-06, HU-07, HU-23 y HU-22 (2026-11-05)

**Estado:** implementadas y probadas con datos **ficticios**, con las adaptaciones que se indican abajo. Ningún flujo hace pagos, envía correos o SMS reales ni reentrena modelos. Pendiente: aceptación del equipo/docente para cerrar cada issue.

### Componentes nuevos

| Elemento | Tipo | Para qué |
| --- | --- | --- |
| FL04 - Registrar pago simulado | Flujo automatizado (Siniestros) | HU-18 y HU-19. Se dispara cuando Estado = Aprobado y PagoRegistrado ≠ Sí, con concurrencia 1 y control de duplicados. |
| FL05 - Alertas de SLA simuladas | Flujo programado (cada hora) | HU-16 |
| FL06 - Reporte diario de pagos simulado | Flujo programado (lunes a viernes, 7:13, hora Colombia) | HU-20 |
| FL07 - Guardar corrección de IA | Flujo automatizado (AnalisisIA) | HU-12 |
| FL08 - Encuesta de satisfacción simulada | Flujo automatizado (Siniestros, Estado = Cerrado) | HU-22 |
| PagosSimulados | Lista ("ERP" ficticio) | Registro de pagos simulados con el número de caso y una referencia `ERP-SIM-…`. |
| DatosEntrenamientoIA | Lista | Correcciones de la IA guardadas como datos de entrenamiento. |
| EncuestasSatisfaccion | Lista | Calificación de 1 a 5, con validación en el servidor. |
| Vistas de Siniestros | Vistas | "Bandeja por prioridad" (HU-14); "Tablero - hoy", "Tablero - ultimos 30 dias" y "Tablero - fraude alto" (HU-23). |

**Columnas nuevas:**
- Siniestros:
  - Pago: MontoAprobadoCOP, DeducibleAplicadoCOP, ValorPagoCOP, PagoRegistrado.
  - SLA: FechaLimiteSLA (calculada como `=[Creado]+3`) y AlertaSLAEnviada.
  - Estimación de la IA: MontoIAMinCOP, MontoIAMaxCOP, ConfianzaMontoIA, EstimacionMontoIA.
  - Póliza: AlertaPoliza, DatosPolizaExtraidos, EnlacePoliza.
  - Tablero y encuesta: HorasHastaDecision, EncuestaEnviada.
- PolizasSimuladas: DeducibleCOP.
- AnalisisIA:
  - Estimación de la IA: MontoIAMinCOP, MontoIAMaxCOP, ConfianzaMontoIA.
  - Corrección: SeveridadCorregida, TipoDanioCorregido, MotivoCorreccion, CorreccionRegistrada.

**Cambios en FL01** (el ámbito se pegó de nuevo en el diseñador nuevo):
- La IA también devuelve `monto_estimado_min`, `monto_estimado_max` y `confianza_monto`.
- FL01 ahora escribe la ficha de la póliza con la confianza de cada campo, una alerta si no encuentra la póliza, el enlace a la póliza y las horas hasta la decisión.

FL04 a FL08 se crearon dentro de la solución GestionSiniestros y usan la referencia de conexión de SharePoint de la solución.

### Decisiones y adaptaciones

- **HU-18:** pago = monto aprobado − deducible de la póliza, con mínimo 0. Si el caso no tiene MontoAprobadoCOP, se usa MontoEstimadoCOP. Se agregó un deducible ficticio a cada póliza: 500.000, 300.000, 400.000, 600.000 y 100.000.
- **HU-19:** el "ERP" es la lista PagosSimulados. El registro lleva el número de caso (Title) y la referencia `ERP-SIM-aaaammdd-ID`. No hay integración con un ERP real.
- **HU-14 y HU-16:**
  - SLA supuesto de 72 h desde la creación. Debe confirmarse con el equipo, porque cambiarlo exige editar la columna calculada y FL05.
  - La bandeja muestra los casos en PendienteAnalisis, EnAnalisis y PendienteRevision, ordenados por vencimiento.
  - La alerta va a un supervisor ficticio cuando el caso llega al 80 % del SLA (57,6 h) y está en EnAnalisis o PendienteRevision. Se envía una sola vez por caso.
  - AjustadorAsignado es texto, no persona, así que la bandeja no filtra "mis casos". Se agrupa o filtra por ajustador.
- **HU-20:** el reporte es un aviso simulado, en NotificacionesSimuladas, con los pagos del día hábil anterior en hora Colombia; el lunes incluye el fin de semana. El total se calcula en el flujo.
- **HU-10:** el rango y la confianza del monto los reporta el modelo. La estimación es orientativa, no un avalúo, y no está calibrada con precios reales.
- **HU-12:**
  - El liquidador corrige en AnalisisIA las columnas SeveridadCorregida, TipoDanioCorregido y MotivoCorreccion.
  - FL07 copia la corrección a DatosEntrenamientoIA, con la etiqueta `Medio->Grave`, y registra el evento `CorreccionIA` en el historial.
  - No se reentrena el modelo automáticamente.
- **HU-06 y HU-07:**
  - Las pólizas son registros estructurados ficticios, no PDF, así que no hay extracción de documentos. La confianza de cada campo es 100 % porque el dato viene estructurado.
  - Si no se encuentra la póliza, AlertaPoliza = Sí y la ficha muestra una alerta; el caso se rechaza por la Cláusula 1.
  - EnlacePoliza abre la ficha de la póliza. Si no se encontró, abre la lista para buscarla manualmente.
- **HU-23:**
  - Sin Power BI, el tablero son vistas de SharePoint con totales: recuento, riesgo promedio, horas promedio hasta la decisión y suma de pagos. Las vistas agrupan por estado.
  - "Del mes" se implementó como últimos 30 días, porque las vistas de SharePoint no filtran por mes calendario.
- **HU-22:** al cerrar el caso, FL08 crea la encuesta (Pendiente) y un correo simulado con el enlace al formulario para calificar de 1 a 5.

### Resultados de pruebas (datos ficticios)

| HU | Caso | Esperado | Resultado | Evidencia |
| --- | --- | --- | --- | --- |
| HU-18/19 | SIN-PRU-HU18-A · POL-1001 · aprobado 1.200.000 | Pago 700.000 registrado con número de caso | **Pasa** | PagosSimulados: 1.200.000 − 500.000 = 700.000, ref. ERP-SIM-20261106-25; evento `PagoRegistradoSimulado` (FL04). |
| HU-18 | SIN-PRU-HU18-B · POL-1005 · aprobado 80.000 | Pago no negativo | **Pasa** | Pago 0 (deducible 100.000). |
| HU-18 | Modificar de nuevo SIN-PRU-HU18-A | Sin pago duplicado | **Pasa** | Un solo registro en PagosSimulados. |
| HU-14 | Vista "Bandeja por prioridad" | Ordenada por vencimiento del SLA | **Pasa** | Orden ascendente por FechaLimiteSLA; muestra la fecha límite de cada caso. |
| HU-16 | Ejecución manual de FL05 | Alerta al 80 % solo en casos que lo alcanzaron | **Pasa** | 4 alertas (casos creados el 3/11, al 94–95 % del SLA). Dos casos del 5/11 no se alertaron. Una segunda ejecución no generó duplicados. |
| HU-20 | Ejecución manual de FL06 con 2 pagos ficticios del 4/11 | Reporte con número de pagos y total | **Pasa** | "Reporte de pagos simulados del 2026-11-04 … Pagos: 2. Total: 750.000 COP", con el detalle; excluye el pago del 5/11. |
| HU-10 | SIN-PRU-HU10-A · foto sintética de abolladura | Monto con rango y confianza | **Pasa** | "Entre 1.500.000 y 2.100.000 COP (confianza 80/100)"; también en AnalisisIA. |
| HU-10 | SIN-PRU-HU10-R · POL-1003 sin cobertura | Rechazo sin romper el flujo | **Pasa** | Rechazado por la Cláusula 4; "Sin estimación de IA para este caso". |
| HU-12 | Corregir el análisis 11 (Medio → Grave) | Corrección guardada como dato de entrenamiento | **Pasa** | DatosEntrenamientoIA: etiqueta `Medio->Grave`; evento `CorreccionIA` (Origen Usuario). |
| HU-06/07 | SIN-PRU-HU06-A · POL-1001 | Póliza vinculada con datos y confianza | **Pasa** | Ficha con estado, vigencia, cobertura, límite y deducible (100 % c/u) y enlace a la póliza. |
| HU-06 | SIN-PRU-HU06-B · POL-9998 inexistente | Caso con alerta | **Pasa** | AlertaPoliza = Sí; "ALERTA: no se encontró una póliza única…". |
| HU-23 | Vista "Tablero - hoy" | Volumen, tiempos y fraude del día | **Pasa** | Recuento 14, riesgo promedio 7, horas hasta la decisión y pagos del día. |
| HU-22 | Cerrar SIN-PRU-HU18-A | Encuesta 1–5 al cierre | **Pasa** | Encuesta creada y correo simulado con el enlace. La respuesta 4 se guardó y un valor 7 fue rechazado por la validación. |

**Datos insertados manualmente como prueba (no generados por los flujos):**
- 2 filas de PagosSimulados con origen "Fixture de prueba HU-20".
- La corrección del análisis 11.
- La respuesta de la encuesta.

### Riesgos y pendientes

- **HU-03, HU-04 y HU-25 siguen sin implementar.** HU-03 y HU-04 requieren pantallas nuevas en la app. HU-25 implica crear grupos o cambiar permisos en el sitio, y eso necesita autorización explícita del equipo (restricción del proyecto).
- **FL05 y FL06** solo se probaron con ejecuciones manuales. Falta observar las ejecuciones programadas.
- **Casos atascados:** hay tres casos de pruebas anteriores, creados por la app el 2/11 (`SIN-…` con POL-TEST-00x), que siguen en PendienteAnalisis. FL01 solo los procesará si se modifican.
- **Cada análisis con fotos consume saldo de OpenRouter.** Las pruebas de este bloque usaron una sola foto sintética, en dos intentos: el primero falló por una expresión, que ya se corrigió.

## 19. Avance HU-03: consultar el estado del caso en línea (2026-11-05)

**Estado:** implementada, probada en la vista previa de Power Apps Studio con casos ficticios y publicada. Pendiente: probarla en el celular y que el equipo o el docente la acepten para cerrar [#25](https://github.com/SaraEcheGom/Automatizacion/issues/25).

### Qué se hizo (app "GestionSiniestros - Reportar siniestro HU01")

- **Pantalla nueva `ScrConsultaEstado`:** el cliente escribe su documento (ficticio) y, si quiere, el número de caso. La galería muestra cada caso con:
  - el número de caso;
  - el **estado**;
  - la **fecha del último cambio** (columna Modificado), en hora local.
- Los casos se ordenan del más reciente al más antiguo.
- Mensajes de apoyo:
  - "Ingrese un documento para consultar." cuando el campo está vacío;
  - "No se encontraron casos con esos datos." cuando no hay resultados.
- **Navegación:** botón "Consultar mi caso" en la cabecera de la pantalla de reporte, y botón "Volver a reportar" en la consulta.
- **Construcción:** los controles se pegaron como código YAML en Power Apps Studio (Label 2.5.1, Classic TextInput 2.3.2, Gallery 2.15.0 y Classic Button 2.2.0). No se cambiaron listas, flujos ni permisos.

### Pruebas (vista previa de Studio, datos ficticios)

| Caso | Esperado | Resultado |
| --- | --- | --- |
| Documento PRUEBA-005 | Casos del asegurado con estado y fecha | **Pasa:** SIN-PRU-HU18-B (Aprobado, 05/11/2026 19:25), SIN-PRU-HU13-E (Rechazado) y SIN-1005 (Borrador). |
| PRUEBA-005 + número SIN-PRU-HU13-E | Solo ese caso | **Pasa:** Rechazado, último cambio 03/11/2026 09:16. |
| Documento inexistente | Mensaje sin resultados | **Pasa:** "No se encontraron casos con esos datos." |
| Botones de navegación | Ir y volver entre reporte y consulta | **Pasa** |

### Riesgos y pendientes

- **Privacidad:** cualquiera que conozca un documento puede ver el estado de esos casos, porque no hay autenticación del cliente. Es aceptable con datos ficticios, pero en un sistema real debe exigirse identidad verificada (relacionado con HU-25).
- **"Último cambio" usa la fecha de modificación del registro:** también cambia con actualizaciones internas, como la marca de alerta de SLA o el pago simulado. Para mostrar solo cambios de estado habría que leer HistorialSiniestros.
- **Aviso de delegación:** Studio muestra una advertencia de delegación en el filtro. Con pocos registros ficticios no afecta, pero con más de 500 elementos la búsqueda podría quedar incompleta.
- **Falta la prueba en el celular** con la app publicada.
