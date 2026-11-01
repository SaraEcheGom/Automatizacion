# Gestión Inteligente de Siniestros Automotrices

## Estado de la implementación académica (1 de noviembre de 2026)

> **MVP en construcción y 100 % simulado.** El sistema core de aseguradora, las pólizas, las aprobaciones, los pagos y las notificaciones no representan operaciones reales. Los resultados TO-BE descritos más abajo son supuestos del caso de negocio, no mediciones del prototipo.

**[Leer la documentación técnica completa del MVP, arquitectura, campos, decisiones y 20 casos de prueba](docs/IMPLEMENTACION_MVP_POWER_PLATFORM.md)**

| Componente actual | Tecnología | Estado verificado |
| --- | --- | --- |
| Base de casos | Microsoft Lists: Siniestros (17 columnas) | Creada; sin casos de prueba |
| Core/pólizas ficticias | Microsoft Lists: PolizasSimuladas (9 columnas) | Creada; sin pólizas de prueba |
| Inteligencia artificial | OpenRouter + openai/gpt-4o-mini | Conector personalizado OpenRouter-Siniestros probado con **texto**, HTTP 200 |
| Captura desde celular | Power Apps Canvas | Por construir |
| Orquestación y aprobación | Power Automate | Por construir |
| Análisis, auditoría y avisos ficticios | AnalisisIA, HistorialSiniestros y NotificacionesSimuladas | Listas por crear |

**Restricciones acordadas:** sin créditos de Azure ni tablas Dataverse en el entorno personal; se usan las herramientas disponibles de Power Platform y Microsoft Lists. La clasificación elegida es **Leve / Medio / Grave** (Medio = moderado en la HU-09 original). Las claves de OpenRouter deben mantenerse fuera de GitHub.

**Historias de Felipe:** [HU-01](https://github.com/SaraEcheGom/Automatizacion/issues/23), [HU-05](https://github.com/SaraEcheGom/Automatizacion/issues/27), [HU-09](https://github.com/SaraEcheGom/Automatizacion/issues/31), [HU-13](https://github.com/SaraEcheGom/Automatizacion/issues/35), [HU-17](https://github.com/SaraEcheGom/Automatizacion/issues/39) y [HU-21](https://github.com/SaraEcheGom/Automatizacion/issues/43). Consultar criterios de aceptación y pruebas en la guía.

---

Proyecto de automatización que combina **IA, BPMS y RPA** para reducir el tiempo de atención de un siniestro de auto: desde que el cliente lo reporta hasta que recibe el pago.

Hoy el proceso es manual y pasa por cinco roles (asesor, liquidador, supervisor, área de pagos y cliente), con un tiempo de ciclo de unos 7 días. La propuesta automatiza el registro, la lectura de la póliza, el análisis del daño y el pago, y deja a las personas solo los casos complejos o con sospecha de fraude.

## Resultados esperados

| Métrica | Antes (AS-IS) | Después (TO-BE) |
| --- | --- | --- |
| Tiempo de ciclo promedio | 7 días | ≈ 1 día |
| Trabajo humano por caso | 215 min | 17 min |
| Casos sin intervención humana | 0 % | 70 % |

Las cifras parten de supuestos de referencia y se validan con la simulación (ver la Wiki).

## Componentes de la solución

| Componente | Qué hace |
| --- | --- |
| **BPMS** | Orquesta el proceso, aplica las reglas de ruta y asigna tareas al liquidador y al supervisor. |
| **IA** | Lee la póliza, clasifica el daño a partir de las fotos y calcula un puntaje de riesgo de fraude. |
| **RPA** | Registra el caso en el sistema core, registra el pago en el ERP y notifica al cliente. |

## Dónde está cada entregable

| Entregable | Ubicación |
| --- | --- |
| Requerimientos funcionales y no funcionales | [Issues con etiqueta `req-funcional`](https://github.com/SaraEcheGom/Automatizacion/issues?q=label%3Areq-funcional) y [`req-no-funcional`](https://github.com/SaraEcheGom/Automatizacion/issues?q=label%3Areq-no-funcional) |
| Backlog (25 historias de usuario) | [Issues con etiqueta `historia-usuario`](https://github.com/SaraEcheGom/Automatizacion/issues?q=label%3Ahistoria-usuario) |
| Tablero Kanban | [Proyecto Automatización](https://github.com/users/SaraEcheGom/projects/6) |
| Simulación | [Wiki](https://github.com/SaraEcheGom/Automatizacion/wiki) |
| Caso de negocio | [Wiki](https://github.com/SaraEcheGom/Automatizacion/wiki) |
| Diagramas BPMN AS-IS y TO-BE | Referencias del entregable; archivos aún no publicados en el repositorio |
| Arquitectura del MVP | [Implementación real en Power Platform](docs/IMPLEMENTACION_MVP_POWER_PLATFORM.md) |
| Solución operativa | En construcción; detalles y pasos en la documentación técnica |

## Cómo está organizado el backlog

Cada historia de usuario es un issue con su criterio de aceptación, prioridad, puntos y el requisito que cumple (con enlace a su issue). Las historias se agrupan en 7 épicas: Reporte, Registro y póliza, Análisis IA, Orquestación, Pago, Notificación y Control.

El tablero tiene las columnas **Backlog → Por hacer → En progreso → En revisión → Finalizado**. En total son 116 puntos: 16 historias P0, 6 P1 y 3 P2.

## Herramientas de modelado y gestión (según propuesta original)

- **Bonita Studio**: modelado BPMN.
- **Simulador BPMN**: simulación AS-IS vs. TO-BE.
- **GitHub Issues y Projects**: requerimientos y backlog.

## Equipo
* David Felipe Rios
* Felipe Agudelo
* Juan Gonzalo de los Rios 
* Sara Echeverri
