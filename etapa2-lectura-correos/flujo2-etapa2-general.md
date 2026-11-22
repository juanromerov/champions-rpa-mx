# Flujo Etapa 2 · General · Leer y clasificar correos

> v1.0 · Une los flujos A (llegada de correo), B (verificación del LOG) y C (corte de las 8:30) en una sola vista. Es más general: el detalle de cada parte está en su flujo.
> 🟠 entrada · 🟢 salida (con la variable que viaja)

```mermaid
flowchart TD
    classDef entrada fill:#fdba74,stroke:#c2410c,color:#111
    classDef salida fill:#86efac,stroke:#15803d,color:#111
    classDef paso fill:#e0f2fe,stroke:#0369a1,color:#111
    classDef decision fill:#fef9c3,stroke:#a16207,color:#111
    classDef gris fill:#e5e7eb,stroke:#6b7280,color:#111

    INA(("ENTRADA<br/>Correo nuevo en la carpeta RPA<br/>id_correo (Id de Internet) · asunto · cuerpo · fecha_recepción")):::entrada
    INC(("ENTRADA<br/>Hora 8:30 L–V<br/>(hora Colombia)")):::entrada

    subgraph GUARDAS["Reglas básicas (en las dos puertas)"]
        FEST{"¿Hoy es festivo?"}:::decision
        CIERRE{"¿El día ya está cerrado?<br/>(hoja Control)"}:::decision
    end
    FIN(["Fin: no se hace nada más"]):::gris

    subgraph CORREO["A · Llegada de un correo"]
        NORM["Quitar prefijos de reenvío, INCIDENCIA (EN)<br/>y la fecha final; normalizar el asunto"]:::paso
        CAT{"¿Está en Catalogo_RPA<br/>y no se había procesado?"}:::decision
        NOREC["Anotar en el Registro<br/>(no_reconocido · solo_registro)"]:::paso
        CLAS["Clasificar por tipo (el rpa sale del catálogo):<br/>inicio · finalización · LOG<br/>(en error: texto del Motivo o del cuerpo →<br/>patrón → código, o 17 si ninguno coincide)"]:::paso
        FILA["Crear fila en el Registro con su id_fila<br/>paso clasificado<br/>(gana el recibido más reciente)"]:::paso
        LOGQ{"¿Terminó en error y su RPA<br/>tiene LOG en el catálogo?"}:::decision
        ESP["B · Crear fila log_esperado con su id_fila<br/>paso en_espera_log<br/>(el LOG se espera hasta las 8:30)"]:::paso
    end

    subgraph CORTE["C · Corte de las 8:30"]
        PEND{"¿Faltan filas esperadas?<br/>(los 17 esperados del catálogo + LOG esperados)"}:::decision
        RELEER["Releer los correos de hoy<br/>entre 1:00 y 8:30"]:::paso
        SINLL["Lo que siga sin estado:<br/>fila sin_llegar · cod_error 0<br/>paso clasificado (con su id_fila)"]:::paso
    end

    OUT3(("SALIDA → Etapa 3<br/>filas en paso clasificado<br/>id_fila · rpa · grupo · estado · cod_error · error<br/>(el corte la llama como pasada final)")):::salida
    OUT4(("SALIDA → Etapa 4<br/>Registro del día completo<br/>(lo lee a las 8:31 o al estar todo con estado)")):::salida

    INA --> FEST
    INC --> FEST
    FEST -- Sí --> FIN
    FEST -- No --> CIERRE
    CIERRE -- Sí --> FIN

    CIERRE -- No, llegó correo --> NORM --> CAT
    CAT -- No es del catálogo --> NOREC --> FIN
    CAT -- Sí --> CLAS --> FILA --> LOGQ
    LOGQ -- Sí --> ESP --> OUT3
    LOGQ -- No --> OUT3

    CIERRE -- No, es la hora del corte --> PEND
    PEND -- No --> OUT3
    PEND -- Sí --> RELEER
    RELEER -- "cada correo hallado" --> NORM
    RELEER -- "al terminar de reprocesar" --> SINLL --> OUT3

    FILA -.-> OUT4
    SINLL -.-> OUT4
```

## Lectura rápida

| Conexión | Variable | Con quién |
|---|---|---|
| 🟠 Entrada | id_correo (Id de Internet), asunto, cuerpo, fecha_recepción | Carpeta de RPA en Outlook (puerta de llegada de correo) |
| 🟠 Entrada | Hora 8:30 L–V | Disparador por hora (puerta del corte) |
| 🟢 Salida | id_fila, rpa, grupo, estado, cod_error, error (filas en paso clasificado) | Etapa 3 |
| 🟢 Salida | Registro del día con todos los estados | Etapa 4 |

## Cómo se lee

1. **Las dos puertas** (correo nuevo y 8:30) pasan por las mismas dos reglas: si es festivo no se hace nada, y si el día ya está cerrado tampoco.
2. **Cada correo** se limpia (prefijos de reenvío, "INCIDENCIA (EN)" y fecha final), se reconoce con el catálogo, se clasifica y queda como una fila nueva en el Registro, con su propio `id_fila`. El `rpa` sale de la fila del catálogo, no del asunto. En error, el texto sale del Motivo o del cuerpo y se compara con los patrones: gana el primero que coincide; si ninguno coincide, es 17 y el texto se copia tal cual. Los RPA que terminan en error y tienen LOG dejan además una fila de LOG esperado.
3. **El corte de las 8:30** revisa si falta algún estado esperado, relee los correos de la madrugada (por si la regla de Outlook falló) y marca `sin_llegar` con código 0 lo que siga sin aparecer. Al terminar llama siempre a la Etapa 3 como pasada final, para que la bitácora quede completa antes del consolidado.
4. **Las Etapas 3 y 4** dependen de esta etapa, no una de la otra: la 3 refleja las filas `clasificado` en la bitácora y la 4 consolida con lo que hay en el Registro. El consolidado es el punto final del día: después de enviarlo la bitácora no se modifica.

## Reglas básicas de arranque

1. Si hoy es festivo, la ejecución termina sin hacer nada.
2. Si el día ya está cerrado en la hoja Control, la ejecución termina sin hacer nada.
3. Un correo no se procesa dos veces (su id_correo, el Id de Internet, ya está en el Registro).
