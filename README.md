# Gestión Inteligente de Siniestros Automotrices

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
| Diagramas BPMN AS-IS y TO-BE | Carpeta [`bpmn/`](bpmn/) |
| Arquitectura | Carpeta [`arquitectura/`](arquitectura/) |
| Solución operativa | Carpeta [`solucion/`](solucion/) |

## Cómo está organizado el backlog

Cada historia de usuario es un issue con su criterio de aceptación, prioridad, puntos y el requisito que cumple (con enlace a su issue). Las historias se agrupan en 7 épicas: Reporte, Registro y póliza, Análisis IA, Orquestación, Pago, Notificación y Control.

El tablero tiene las columnas **Backlog → Por hacer → En progreso → En revisión → Finalizado**. En total son 116 puntos: 16 historias P0, 6 P1 y 3 P2.

## Herramientas

- **Bonita Studio**: modelado BPMN.
- **Simulador BPMN**: simulación AS-IS vs. TO-BE.
- **GitHub Issues y Projects**: requerimientos y backlog.

## Equipo
* David Felipe Rios
* Felipe Agudelo
* Juan Gonzalo de los Rios 
* Sara Echeverri
