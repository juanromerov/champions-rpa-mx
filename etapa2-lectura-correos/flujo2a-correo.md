# Flujo Etapa 2 · A · Llegada de correo

> v1.0 · Puerta de llegada de correo RPA/LOG. 🟠 entrada · 🟢 salida (con la variable que viaja)

```mermaid
flowchart TD
    classDef entrada fill:#fdba74,stroke:#c2410c,color:#111
    classDef salida fill:#86efac,stroke:#15803d,color:#111
    classDef paso fill:#e0f2fe,stroke:#0369a1,color:#111
    classDef decision fill:#fef9c3,stroke:#a16207,color:#111
    classDef gris fill:#e5e7eb,stroke:#6b7280,color:#111

    IN(("ENTRADA<br/>id_correo (Id de Internet) · asunto<br/>cuerpo · fecha_recepción")):::entrada
    FEST{"¿Hoy es festivo?<br/>(hoja Festivos)"}:::decision
    ENV{"¿El día ya está cerrado?<br/>(hoja Control: enviando,<br/>enviado o sin_errores)"}:::decision
    FIN(["Fin: no se hace nada más"]):::gris
    NORM["Quitar prefijos de reenvío (RV:, FW:, RE:),<br/>el prefijo INCIDENCIA (EN) y la fecha final;<br/>normalizar el asunto"]:::paso
    CAT{"¿El asunto está en<br/>Catalogo_RPA?"}:::decision
    NOREC["Crear fila en el Registro (con id_fila):<br/>observacion no_reconocido<br/>paso solo_registro"]:::paso
    DUP{"¿El id_correo ya<br/>está en el Registro?"}:::decision
    TIPO{"Tipo y rpa según la fila<br/>del catálogo (el rpa no sale del asunto)"}:::decision
    INI["Estado: inicio_recibido"]:::paso
    LOG["Estado: log_recibido<br/>(si había una fila en_espera_log,<br/>se actualiza esa por su id_fila)"]:::paso
    PRE{"¿Es Variación día anterior<br/>precios Bimbo?"}:::decision
    CUERPO{"Cuerpo de la finalización"}:::decision
    OK["Estado: exito"]:::paso
    TXT1["Texto del error = fila Motivo"]:::paso
    TXT2["Texto del error = texto del cuerpo"]:::paso
    COMP["Normalizar y probar los patrones de TablaErrores<br/>por orden; gana el primero que coincide"]:::paso
    HAY{"¿Algún patrón<br/>coincide?"}:::decision
    COD["Estado: error · cod_error del patrón"]:::paso
    SINCAT["Estado: error · cod_error 17<br/>observacion error_sin_catalogo"]:::paso
    REV["Estado: error · cod_error 17<br/>observacion revisar_formato"]:::paso
    FILA["Crear la fila en el Registro con paso clasificado<br/>y su id_fila (un guid generado una sola vez)<br/>(el texto del error se copia en la columna error)"]:::paso
    REP{"¿Ya hay una fila de hoy con<br/>el mismo rpa y grupo?"}:::decision
    GANA["Gana la de recibido más reciente;<br/>la otra pasa a solo_registro (por id_fila)"]:::paso
    ESLOG{"¿Terminó en error y su RPA<br/>tiene LOG en el catálogo?"}:::decision
    OUT(("SALIDA → Etapa 3<br/>filas en paso clasificado<br/>id_fila · rpa · grupo · estado · cod_error · error")):::salida
    OUTLOG(("SALIDA → Flujo B<br/>rpa · cod_error")):::salida

    IN --> FEST
    FEST -- Sí --> FIN
    FEST -- No --> ENV
    ENV -- Sí --> FIN
    ENV -- No --> NORM --> CAT
    CAT -- No --> NOREC --> FIN
    CAT -- Sí --> DUP
    DUP -- Sí --> FIN
    DUP -- No --> TIPO
    TIPO -- Inicio --> INI --> FILA
    TIPO -- LOG --> LOG --> FILA
    TIPO -- Finalización --> PRE
    PRE -- Sí --> OK
    PRE -- No --> CUERPO
    CUERPO -- Éxito --> OK
    CUERPO -- Error con tabla --> TXT1 --> COMP
    CUERPO -- Error sin tabla --> TXT2 --> COMP
    CUERPO -- No reconocido --> REV
    COMP --> HAY
    HAY -- Sí --> COD --> FILA
    HAY -- No --> SINCAT --> FILA
    OK --> FILA
    REV --> FILA
    FILA --> REP
    REP -- Sí --> GANA --> ESLOG
    REP -- No --> ESLOG
    ESLOG -- Sí --> OUTLOG
    ESLOG -- Sí --> OUT
    ESLOG -- No --> OUT
```

## Lectura rápida

| Conexión | Variable | Con quién |
|---|---|---|
| 🟠 Entrada | id_correo (Id de Internet), asunto, cuerpo, fecha_recepción | Correo nuevo en la carpeta de RPA (o correo hallado por el Flujo C) |
| 🟢 Salida | id_fila, rpa, grupo, estado, cod_error, error | Etapa 3 |
| 🟢 Salida | rpa, cod_error | Flujo B (verificación del LOG) |

## Reglas básicas de arranque

1. Si hoy es festivo, la ejecución termina sin hacer nada.
2. Si el día ya está cerrado en la hoja Control, la ejecución termina sin hacer nada.
3. Un correo no se procesa dos veces (el id_correo, que es el Id de Internet del correo, ya está en el Registro).

## Cómo se lee el error

1. El asunto se limpia (prefijos de reenvío, "INCIDENCIA (EN)" y fecha final) y se busca completo en el catálogo; de esa fila salen el tipo y el `rpa`.
2. En una finalización con error, el texto sale de la fila Motivo (si el correo trae tabla) o del cuerpo (si no la trae).
3. Se prueban los patrones de la tabla de errores en orden, de lo más específico a lo más general; gana el primero que coincide.
4. Si ningún patrón coincide, el código es 17 con la marca `error_sin_catalogo`. El texto del error nunca se descarta: se copia tal cual a la columna `error`.
5. Un cuerpo que no es éxito ni error identificable queda en 17 con la marca `revisar_formato`; nunca se toma como éxito.

## Sobre las filas del Registro

- Toda fila nueva lleva un `id_fila` (un guid) que se genera una sola vez, al crearla, y no cambia nunca. Es la llave con la que se actualiza la fila.
- El `id_correo` no es una llave de actualización: solo sirve para no procesar dos veces el mismo correo.
