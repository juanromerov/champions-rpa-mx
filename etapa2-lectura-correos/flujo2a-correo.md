# Flujo Etapa 2 · A · Llegada de correo

> v0.2 · Puerta de llegada de correo RPA/LOG. 🟠 entrada · 🟢 salida (con la variable que viaja)

```mermaid
flowchart TD
    classDef entrada fill:#fdba74,stroke:#c2410c,color:#111
    classDef salida fill:#86efac,stroke:#15803d,color:#111
    classDef paso fill:#e0f2fe,stroke:#0369a1,color:#111
    classDef decision fill:#fef9c3,stroke:#a16207,color:#111
    classDef gris fill:#e5e7eb,stroke:#6b7280,color:#111

    IN(("ENTRADA<br/>id_correo · asunto<br/>cuerpo · fecha_recepción")):::entrada
    FEST{"¿Hoy es festivo?<br/>(hoja Festivos)"}:::decision
    ENV{"¿El día ya está cerrado?<br/>(hoja Control: enviando,<br/>enviado o sin_errores)"}:::decision
    FIN(["Fin: no se hace nada más"]):::gris
    NORM["Quitar prefijos de reenvío (RV:, FW:, RE:)<br/>y normalizar el asunto"]:::paso
    CAT{"¿El asunto está en<br/>Catalogo_RPA?"}:::decision
    NOREC["Crear fila en el Registro:<br/>observacion no_reconocido<br/>paso solo_registro"]:::paso
    DUP{"¿El id_correo ya<br/>está en el Registro?"}:::decision
    TIPO{"Tipo según el catálogo"}:::decision
    INI["Estado: inicio_recibido"]:::paso
    LOG["Estado: log_recibido<br/>(si había una fila en_espera_log,<br/>se actualiza esa)"]:::paso
    PRE{"¿Es Variación día anterior<br/>precios Bimbo?"}:::decision
    CUERPO{"Cuerpo de la finalización"}:::decision
    OK["Estado: exito"]:::paso
    ERR["Leer la fila Motivo, normalizar y comparar con<br/>Catalogo_Errores (de la frase más específica<br/>a la más general)"]:::paso
    COD["Estado: error · cod_error 1–18<br/>si nada coincide: 17"]:::paso
    REV["Estado: error · cod_error 17<br/>observacion revisar_formato"]:::paso
    FILA["Crear la fila en el Registro con paso clasificado<br/>(el texto del error se copia en la columna error)"]:::paso
    REP{"¿Ya hay una fila de hoy con<br/>el mismo rpa y grupo?"}:::decision
    GANA["Gana la de recibido más reciente;<br/>la otra pasa a solo_registro"]:::paso
    ESLOG{"¿Terminó en error y su RPA<br/>tiene LOG en el catálogo?"}:::decision
    OUT(("SALIDA → Etapa 3<br/>filas en paso clasificado<br/>rpa · grupo · estado · cod_error · error")):::salida
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
    CUERPO -- Error con tabla --> ERR --> COD
    CUERPO -- No reconocido --> REV
    OK --> FILA
    COD --> FILA
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
| 🟠 Entrada | id_correo, asunto, cuerpo, fecha_recepción | Correo nuevo en la carpeta de RPA (o correo hallado por el Flujo C) |
| 🟢 Salida | rpa, grupo, estado, cod_error, error | Etapa 3 |
| 🟢 Salida | rpa, cod_error | Flujo B (verificación del LOG) |

## Reglas básicas de arranque

1. Si hoy es festivo, la ejecución termina sin hacer nada.
2. Si el día ya está cerrado en la hoja Control, la ejecución termina sin hacer nada.
3. Un correo no se procesa dos veces (id_correo en el Registro).
