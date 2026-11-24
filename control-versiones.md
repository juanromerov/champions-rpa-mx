# Control de versiones · Automatización bitácora RPA (SETI-CH-RPA-MX)

> Para qué sirve: registro de las versiones de la solución, con qué cambió en cada una y por qué (requisito de P-02: control de versiones desde la fase de desarrollo).
> Última actualización: 2026-11-24.

## Esquema de versiones

| Versión | Qué significa |
|---|---|
| Diseño v1.0 | Diseño de las cuatro etapas y flujo general cerrado (2026-11-21) |
| E1 v0.x, E2 v0.x, E3 v0.x, E4 v0.x | Construcción y pruebas por etapa (D-32) |
| Solución v1.0 | Las cuatro etapas integradas y probadas (versión inicial para la retroalimentación) |
| Solución v2.0 | Versión mejorada tras la retroalimentación, con su registro de cambios |

## Registro

| Versión | Fecha | Qué cambió | Por qué | Pruebas / evidencia |
|---|---|---|---|---|
| Diseño v1.0 | 2026-11-21 | Cuatro etapas y flujo general cerrados (D-54 a D-57 en la Etapa 4) | Base para empezar el desarrollo | Flujos `.md` validados por el usuario |
| Desarrollo iniciado | 2026-11-23 | Arranca la construcción por la Etapa 1 | Primera prueba en Power Automate | — |
| Prueba de viabilidad | 2026-11-24 | El script copió `PLANTILLA` y renombró la hoja desde Power Automate en 9 s | Confirma el riesgo (a) de D-01: Office Scripts funciona desde Power Automate sin licencia extra | Ejecución correcta en el historial |
| Nombres reales en `PLANTILLA` | 2026-11-24 | Filas 4 a 23 con los 20 nombres reales (17 esperadas y 3 LOG) | El correo del ticket toma el nombre de la bitácora (D-57) | Validado por el usuario con captura |
| E1 v0.1 | 2026-11-24 | Script definitivo (encabezado, fechas L–V, festivos en gris, idempotente) y flujo manual con Try/Catch que avisa por correo y Teams | Primera versión completa de la Etapa 1 | Pruebas: semana normal, festivo, cruce de mes, hoja ya existente y falla real con alerta |
| E1 v0.2 | 2026-11-24 | Flujo programado `2611-E1-Generar bitacora`: Recurrence lunes 0:00 hora Colombia, variable `fechaHoy` con zona horaria explícita, Try/Catch con alerta en paralelo | Dejar la Etapa 1 con su disparador real (D-02, D-18) | Pruebas manuales 1 a 3 correctas sobre la copia de pruebas |

## Hallazgos y correcciones de la Etapa 1

| Hallazgo | Causa | Corrección |
|---|---|---|
| Columna del primer día en gris, encabezado sin cambiar y fechas viejas | El flujo seguía ejecutando el script de prueba | Pegar el script definitivo y verificar el texto de salida |
| Ejecución de más de 2 minutos | Posible lectura de un rango "usado" muy grande en `Festivos` | `getUsedRange(true)` y cerrar el Excel durante la ejecución |
| Alerta falsa con la hoja ya existente | El flujo no tenía scopes; el correo era un aviso automático de Power Automate | Construir Try y Catch; el Catch corre solo si falla o se agota el tiempo |
| Teams no llegaba | La acción no estaba creada | Crear la acción con Flow bot y Recipient; correo y Teams en paralelo |
| `fechaIso inválida: fechaHoy` | El campo llevaba el texto "fechaHoy" y no la variable | Usar contenido dinámico (`variables('fechaHoy')`) |
| Con la hoja ya creada no falla aunque se renombre `PLANTILLA` | El script revisa primero si la hoja existe (D-13) | Sin cambio: es el comportamiento esperado; la prueba de falla requiere que la hoja no exista |

## Pendiente de versiones

- E1 v0.3: motivo del error en la alerta y resultado del script visible en el historial.
- Prueba real del disparador el lunes 2026-11-30 (borrar antes la hoja `2026-11-30` de la copia de pruebas).
- Prueba con 2 usuarios reales del equipo (P-02): se registra aparte, en la tarea "Pruebas".
