# Flujo Etapa 2 · A · Llegada de correo

> v0.1 · Puerta de llegada de correo RPA/LOG. 🟠 entrada · 🟢 salida (con la variable que viaja)

```mermaid
flowchart TD
    classDef entrada fill:#fdba74,stroke:#c2410c,color:#111
    classDef salida fill:#86efac,stroke:#15803d,color:#111
    classDef paso fill:#e0f2fe,stroke:#0369a1,color:#111
    classDef decision fill:#fef9c3,stroke:#a16207,color:#111
    classDef gris fill:#e5e7eb,stroke:#6b7280,color:#111

    IN(("ENTRADA<br/>id_correo · asunto<br/>cuerpo · fecha_recepción")):::entrada
    FEST{"¿Hoy es festivo?<br/>(hoja Festivos)"}:::decision
    FIN(["Fin: sin más acciones"]):::gris
    NORM["Normalizar el asunto"]:::paso
    CAT{"¿El asunto está en<br/>Catalogo_RPA?"}:::decision
    NOREC["Anotar en el log:<br/>no reconocido"]:::paso
    DUP{"¿El id_correo ya<br/>está en el log?"}:::decision
    REG["Registrar id_correo en el log"]:::paso
    TIPO{"Tipo según el catálogo"}:::decision
    INI["Estado: inicio → verde"]:::paso
    LOG["Estado: LOG → verde (llegó)"]:::paso
    PRE{"¿Es Variación día anterior<br/>precios Bimbo?"}:::decision
    CUERPO{"Cuerpo de la finalización"}:::decision
    OK["Estado: éxito"]:::paso
    ERR["Leer la fila Motivo, normalizar y comparar con<br/>Catalogo_Errores (de la frase más específica<br/>a la más general)"]:::paso
    COD["Estado: error + código 1–18<br/>si nada coincide: 17"]:::paso
    REV["Estado: error 17<br/>+ marca revisar formato"]:::paso
    TARDE{"¿Recibido después<br/>de las 8:30?"}:::decision
    SOLOLOG["Solo se registra en el log<br/>(no toca bitácora ni consolidado)"]:::paso
    ESLOG{"¿Terminó en error y su RPA<br/>tiene LOG en el catálogo?"}:::decision
    OUT(("SALIDA → Etapa 3<br/>RPA · tipo · estado<br/>código · motivo · fecha · id_correo")):::salida
    OUTLOG(("SALIDA → Flujo B (LOG)<br/>RPA · código")):::salida

    IN --> FEST
    FEST -- Sí --> FIN
    FEST -- No --> NORM --> CAT
    CAT -- No --> NOREC --> FIN
    CAT -- Sí --> DUP
    DUP -- Sí --> FIN
    DUP -- No --> REG --> TIPO
    TIPO -- Inicio --> INI --> OUT
    TIPO -- LOG --> LOG --> OUT
    TIPO -- Finalización --> PRE
    PRE -- Sí --> OK
    PRE -- No --> CUERPO
    CUERPO -- Éxito --> OK
    CUERPO -- Error con tabla --> ERR --> COD
    CUERPO -- No reconocido --> REV
    OK --> TARDE
    COD --> TARDE
    REV --> TARDE
    TARDE -- Sí --> SOLOLOG --> FIN
    TARDE -- No --> OUT
    TARDE -- No --> ESLOG
    ESLOG -- Sí --> OUTLOG
    ESLOG -- No --> FIN
```

## Lectura rápida

| Conexión | Variable | Con quién |
|---|---|---|
| 🟠 Entrada | id_correo, asunto, cuerpo, fecha_recepción | Correo nuevo en la carpeta de RPA |
| 🟢 Salida | RPA, tipo, estado, código, motivo, fecha, id_correo | Etapa 3 |
| 🟢 Salida | RPA, código | Flujo B (verificación del LOG) |