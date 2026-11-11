
# Champions - Automatización del monitoreo, validación y notificación de resultados RPA

Automatización de la actividad díaria de ejecuciones RPA Sura México: genera el Excel de cada semana, lo diligencia a partir de los correos de ejecución y envía un único consolidado diario de incidentes.

## Qué resuelve

Hoy el seguimiento de las ejecuciones RPA se hace de forma manual: crear la bitácora de la semana, revisar los correos de cada RPA y reportar los errores. Este proyecto automatiza ese ciclo en cuatro etapas:

1. **Generar la bitácora**: cada lunes se crea la hoja de la semana y se marcan los festivos según calendario Mexicano.
2. **Leer y clasificar correos**: se identifica cada RPA y si terminó con éxito o con error.
3. **Actualizar la bitácora**: éxito en verde; error en rojo con su código de incidente.
4. **Consolidar**: un único envío diario con los incidentes del día, y registro del resultado.

## Contenido del repositorio

| Archivo | Descripción |
|---|---|
| `flujo-general.md` | Diagrama del flujo general (Mermaid), resumen por etapa y puntos abiertos |
| `README.md` | Esta portada |


## Tecnologías

- Power Automate
- Microsoft 365

## Estado actual

- Flujo general en **v0.3**.
- Hay puntos de diseño abiertos (consolidación inmediata vs. corte horario, tratamiento de correos tardíos, cuenta de ejecución, aviso ante fallas de la propia automatización). 
-
