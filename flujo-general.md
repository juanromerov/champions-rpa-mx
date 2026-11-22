# Flujo general · Automatización de la bitácora de ejecuciones RPA

> v1.0 · Vista de todo el proyecto: las cuatro etapas y cómo se conectan. El detalle de cada una está en su propio flujo (📄).
> 🟠 entrada · 🟢 salida (con la variable que viaja)

```mermaid
flowchart TD
    classDef entrada fill:#fdba74,stroke:#c2410c,color:#111
    classDef salida fill:#86efac,stroke:#15803d,color:#111
    classDef etapa fill:#e0f2fe,stroke:#0369a1,color:#111
    classDef decision fill:#fef9c3,stroke:#a16207,color:#111
    classDef gris fill:#e5e7eb,stroke:#6b7280,color:#111
    classDef datos fill:#ffffff,stroke:#9ca3af,color:#444,stroke-dasharray:4 3
    classDef alerta fill:#fecaca,stroke:#b91c1c,color:#111

    IN1(("ENTRADA<br/>Lunes 0:00")):::entrada
    E1["<b>ETAPA 1 · Generar bitácora</b><br/>Hoja de la semana desde PLANTILLA,<br/>fechas de lunes a viernes y festivos en gris<br/>📄 flujo1-bitacora.md"]:::etapa
    OUT1(("SALIDA<br/>Hoja de la semana<br/>nombre = fecha ISO del lunes")):::salida

    IN2A(("ENTRADA<br/>Llega un correo<br/>de RPA o LOG")):::entrada
    IN2C(("ENTRADA<br/>Corte L-V 8:30")):::entrada
    G{"Guardas de entrada<br/>¿Hoy es festivo?<br/>¿El día ya está cerrado en Control?"}:::decision
    FIN([" Fin: no se hace nada más "]):::gris
    E2["<b>ETAPA 2 · Leer y clasificar correos</b><br/>A · correo: clasifica y crea la fila en el Registro<br/>B · LOG: espera el LOG si el RPA falló<br/>C · corte: código 0 a lo que no llegó<br/>📄 flujo2-etapa2-general.md<br/>flujo2a-correo · flujo2b-log · flujo2c-corte-0830"]:::etapa

    E3["<b>ETAPA 3 · Actualizar bitácora</b><br/>Escribe valor y color en la hoja de la semana<br/>y marca las filas como en_bitacora<br/>📄 flujo3-actualizar-bitacora.md"]:::etapa
    OUT3(("SALIDA<br/>Celdas escritas en la hoja de la semana<br/>Filas del Registro en en_bitacora")):::salida

    IN4A(("ENTRADA<br/>Consolidación inmediata<br/>llamada desde el flujo A")):::entrada
    IN4B(("ENTRADA<br/>L-V 8:31")):::entrada
    E4["<b>ETAPA 4 · Consolidado y ticket</b><br/>Con todos los esperados con estado y sin filas pendientes:<br/>correo solo si hay errores y cierra el día en Control<br/>📄 flujo4-consolidado.md"]:::etapa
    OUT4(("SALIDA<br/>Correo del ticket, solo si hay errores<br/>Día cerrado en Control")):::salida

    subgraph DATOS["Datos compartidos y avisos"]
        DB1[("Excel de bitácoras<br/>PLANTILLA · Festivos · Control · Registro")]:::datos
        DB2[("Catalogo_RPA.xlsx · solo lectura<br/>tabla de RPA y patrones de error")]:::datos
        ALERTA["⚠️ Si una etapa falla: alerta por correo y por Teams<br/>no se marcan falsos errores de RPA"]:::alerta
    end

    IN1 --> E1 --> OUT1
    IN2A --> G
    IN2C --> G
    G -- Sí --> FIN
    G -- No --> E2
    E2 -- "filas clasificado<br/>tras cada correo (A) y pasada final del corte (C)" --> E3
    E3 --> OUT3
    OUT1 -.-> E3
    E2 -. "Registro del día<br/>con todos los estados" .-> E4
    IN4A --> E4
    IN4B --> E4
    OUT1 -.-> E4
    E4 --> OUT4
```

## Un día típico (hora Colombia)

| Hora | Qué pasa |
|---|---|
| Lunes 0:00 | La Etapa 1 crea la hoja de la semana |
| Desde ~1:30 | Llegan los correos de los RPA. Por cada uno, la Etapa 2 lo clasifica y la Etapa 3 lo refleja en la bitácora |
| Hacia las ~7:50 | Si todos los esperados ya tienen estado, la Etapa 4 consolida de inmediato |
| 8:30 | Corte: la Etapa 2 marca con código 0 lo que no llegó y la Etapa 3 hace su pasada final |
| 8:31 | La Etapa 4 consolida y cierra el día. Si falta algo, espera y reintenta unos minutos antes de alertar |

## Cómo se conectan las etapas

| De → A | Qué viaja |
|---|---|
| 🟢 Etapa 1 → Etapas 3 y 4 | Hoja de la semana (nombre = fecha ISO del lunes) |
| 🟢 Etapa 2 → Etapa 3 | Filas en paso `clasificado`: `id_fila`, `rpa`, `grupo`, `estado`, `cod_error`, `error`, `recibido` |
| 🟢 Etapa 2 → Etapa 4 | Registro del día con todos los estados (la Etapa 4 lee el Registro, no la bitácora) |
| 🟢 Etapa 3 → bitácora y Registro | Valor y color de cada celda; las filas pasan a `en_bitacora` |
| 🟢 Etapa 4 → Control y correo | Estado del día (`enviando`, `enviado` o `sin_errores`) y, si hubo errores, el correo del ticket |
| 🟠 Control → Etapas 2, 3 y 4 | Las guardas leen si el día ya está cerrado |

## Reglas que valen para todo el proyecto

1. **Herramienta única:** Power Automate en la nube, con Outlook y Excel. Todos los disparadores van en hora de Colombia.
2. **Dos disparadores independientes:** el semanal (Etapa 1) y los diarios de lunes a viernes (Etapas 2 a 4).
3. **Dos reglas básicas en las puertas de entrada:** si hoy es festivo, no se hace nada; si el consolidado ya se envió, no se sigue validando. La Etapa 1 corre todos los lunes sin guardas. La Etapa 3 solo lleva la de festivo. La Etapa 4 lleva la de festivo y revisa `Control` como candado contra el doble envío.
4. **El Registro es el punto de control entre etapas:** cada etapa sabe qué le toca por la columna `paso`, y se puede reintentar sin releer los correos.
5. **El consolidado es el punto final del día:** la bitácora no se modifica después de enviarlo.
6. **Excel configurable:** la ubicación de los archivos y los destinatarios viven en un solo punto de configuración que leen las cuatro etapas.
