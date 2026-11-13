# Flujo Etapa 2 · Leer y clasificar correos

> v0.1 · Detalle de la Etapa 2 ·
> Parte del flujo general v0.4 (en actualización). Esta etapa tiene dos puertas de entrada: la llegada de un correo y el corte de las 8:30. Las etapas 3 y 4 solo se mencionan donde tocan a esta (entrada y salida).

```mermaid
flowchart TD
    classDef disparador fill:#dbeafe,stroke:#1d4ed8,color:#111
    classDef etapa fill:#e0f2fe,stroke:#0369a1,color:#111
    classDef decision fill:#fef9c3,stroke:#a16207,color:#111
    classDef salida fill:#dcfce7,stroke:#15803d,color:#111
    classDef error fill:#fee2e2,stroke:#b91c1c,color:#111
    classDef gris fill:#e5e7eb,stroke:#6b7280,color:#111
    classDef nota fill:#fff,stroke:#9ca3af,color:#444,stroke-dasharray:4 3

    CAT[("📋 Excel de catálogo (solo lectura)<br/>Catalogo_RPA · Catalogo_Errores")]:::nota
    FEST[("📋 Hoja Festivos<br/>Lista del cliente")]:::nota

    subgraph PA["PUERTA A · Llega un correo"]
        T1(["📩 Llega un correo a la carpeta de RPA<br/>(regla de servidor de Outlook)"]):::disparador
        G1{"¿Hoy es festivo?"}:::decision
        FIN1(["Fin: no se hace nada"]):::gris
        DUP{"¿Ya se procesó este correo?<br/>(identificador en el log)"}:::decision
        FIN2(["Fin: no se procesa dos veces"]):::gris
        NORM["<b>ETAPA 2 · Leer y clasificar</b><br/>Normalizar el asunto<br/>(minúsculas, sin tildes ni espacios sobrantes)"]:::etapa
        CONOC{"¿El asunto completo coincide<br/>con uno del catálogo?"}:::decision
        LOGNR["Registrar 'no reconocido' en el log<br/>(no es error de RPA)"]:::etapa
        TIPO{"¿Qué tipo es,<br/>según el catálogo?"}:::decision
        R_INI["Resultado: INICIO RECIBIDO<br/>(verde en su fila)"]:::salida
        R_LOG["Resultado: LOG RECIBIDO<br/>(verde en su fila)"]:::salida
        CUERPO["Convertir el cuerpo HTML a texto"]:::etapa
        EXITO{"¿Cuerpo de éxito?<br/>(estándar, o el de variaciones<br/>si es la excepción de precios)"}:::decision
        R_OK["Resultado: ÉXITO"]:::salida
        ERRFMT{"¿Cuerpo de error con<br/>tabla de detalle?"}:::decision
        MOTIVO["Leer la fila Motivo<br/>(Actividad como apoyo) y normalizar"]:::etapa
        COMP{"¿Coincide con un Motivo activo<br/>del catálogo de errores?<br/>(de la frase más específica a la más general)"}:::decision
        R_COD["Resultado: ERROR<br/>código del catálogo (1 a 18)"]:::error
        R_17["Resultado: ERROR código 17<br/>'Detenido'"]:::error
        R_FMT["Resultado: ERROR código 17<br/>+ marca 'revisar formato'"]:::error
        TIENELOG{"¿El RPA tiene un LOG<br/>en el catálogo?"}:::decision
        INICIAR[["Iniciar el flujo 2:<br/>verificar la llegada del LOG"]]:::etapa
        TARDE{"¿Se recibió después<br/>de las 8:30?"}:::decision
        LOGTARDE["Registrar solo en el log<br/>(no toca la celda roja ni el consolidado enviado)"]:::etapa
        FIN3(["Fin"]):::gris
        OUT[("📤 Resultado hacia la Etapa 3<br/>RPA · tipo · resultado · código · motivo<br/>fecha y hora de recepción · identificador<br/>(si hay reproceso, gana el último)")]:::salida
    end

    subgraph F2["FLUJO 2 · Verificar la llegada del LOG"]
        V1(["▶ Un RPA con LOG terminó en error"]):::disparador
        V2{"¿Ya llegó su LOG hoy?<br/>(carpeta de RPA)"}:::decision
        V3["Dejar el LOG como 'esperado'<br/>(el consolidado inmediato lo espera)"]:::etapa
        V4[("📤 LOG esperado hacia las Etapas 3 y 4")]:::salida
    end

    subgraph PB["PUERTA B · Corte de las 8:30"]
        T2(["⏰ Corte 8:30 a. m. (hora Colombia)<br/>arranca la automatización"]):::disparador
        G2{"¿Hoy es festivo?"}:::decision
        FIN4(["Fin: no se hace nada"]):::gris
        PEND{"¿Alguna fila esperada sin estado?<br/>(las 17 filas + los LOG esperados)"}:::decision
        FIN5(["Fin: nada por validar"]):::gris
        RELEER["Releer los correos recibidos entre<br/>la 1:00 y las 8:30 de hoy:<br/>carpeta de RPA, bandeja de entrada y no deseados"]:::etapa
        REUT[["Clasificar cada correo hallado<br/>(misma rutina, desde 'Normalizar el asunto')"]]:::etapa
        SINEST["Las filas que sigan sin estado<br/>pasan a la Etapa 3: rojo + código 0"]:::salida
    end

    T1 --> G1
    FEST -.-> G1
    G1 -- Sí --> FIN1
    G1 -- No --> DUP
    DUP -- Sí --> FIN2
    DUP -- No --> NORM --> CONOC
    CAT -.-> CONOC
    CONOC -- No --> LOGNR --> FIN3
    CONOC -- Sí --> TIPO
    TIPO -- Inicio --> R_INI
    TIPO -- LOG --> R_LOG
    TIPO -- "Finalización o variación de precios" --> CUERPO --> EXITO
    EXITO -- Sí --> R_OK
    EXITO -- No --> ERRFMT
    ERRFMT -- No --> R_FMT
    ERRFMT -- Sí --> MOTIVO --> COMP
    CAT -.-> COMP
    COMP -- Sí --> R_COD
    COMP -- No --> R_17
    R_COD --> TIENELOG
    R_17 --> TIENELOG
    R_FMT --> TIENELOG
    TIENELOG -- Sí --> INICIAR --> TARDE
    TIENELOG -- No --> TARDE
    R_INI --> TARDE
    R_LOG --> TARDE
    R_OK --> TARDE
    TARDE -- Sí --> LOGTARDE --> FIN3
    TARDE -- No --> OUT

    INICIAR -.-> V1
    V1 --> V2
    V2 -- Sí --> R_LOG
    V2 -- No --> V3 --> V4
    V3 -.->|"si llega después: entra por la Puerta A"| T1
    V3 -.->|"si no llega a las 8:30: Puerta B"| T2

    T2 --> G2
    FEST -.-> G2
    G2 -- Sí --> FIN4
    G2 -- No --> PEND
    PEND -- No --> FIN5
    PEND -- Sí --> RELEER --> REUT --> SINEST
    REUT -.-> NORM
```

## Lectura rápida

| Elemento | Detalle |
|---|---|
| Disparadores | 1) Llega un correo a la carpeta de RPA. 2) Corte de las 8:30 a. m., hora Colombia (arranca la automatización por hora). 3) Flujo 2, que arranca cuando un RPA con LOG termina en error |
| Entrada | El correo (asunto y cuerpo), la hoja `Festivos` y el Excel de catálogo (`Catalogo_RPA` y `Catalogo_Errores`), que solo se lee |
| Qué hace | Descarta festivos y duplicados, reconoce el correo por su asunto completo, lo clasifica por tipo (inicio, finalización, variación de precios o LOG) y, si es de finalización, decide éxito o error y su código |
| Salida | Un resultado por correo hacia la Etapa 3: RPA, tipo, resultado, código, motivo, fecha y hora de recepción e identificador. No escribe en la bitácora |
| Filas esperadas | 17 filas (8 de inicio, 8 de finalización y la variación de precios) y 3 de LOG, que solo se esperan si su RPA falló |

## Decisiones aplicadas a esta etapa

1. **El asunto completo identifica el correo.** Los asuntos son fijos y se comparan enteros, ya normalizados, contra el catálogo; el tipo sale del catálogo. Porqué: algunos nombres están contenidos en otros (un LOG lleva el nombre de su RPA), y con la coincidencia completa no se confunden.
2. **Tres grupos de correo: inicio, finalización y LOG.** El correo de inicio siempre debe llegar y se marca en verde en su fila; su único error posible es que no llegue (código 0). Porqué: tiene su propia fila en la bitácora y no define el estado final del RPA.
3. **El error se lee siempre de la fila Motivo.** Se compara normalizado, con la frase completa y de la más específica a la más general. Si nada coincide, es código 17 (Detenido); un cuerpo que no se reconoce es código 17 con marca "revisar formato", nunca éxito. Porqué: un falso éxito esconde un problema; un falso error solo cuesta una revisión.
4. **Variación de precios:** si llega, es éxito; si no llega, es 0. No tiene otro error posible.
5. **LOG: flujo 2 aparte.** Cuando un RPA con LOG termina en error, el flujo 2 revisa si su LOG ya llegó. Si no, lo deja como esperado y el consolidado inmediato lo espera. Si llega después, entra por la puerta A y queda en verde; si a las 8:30 no llegó, queda en rojo con 0. Porqué: el LOG solo llega si el RPA anterior falla, así que su ausencia solo es error en ese caso.
6. **El corte de las 8:30 es un disparador por hora.** Porqué: "No llegó" es la ausencia de un evento, y un flujo que solo se dispara al llegar un correo no corre cuando no llega ninguno.
7. **Revisión de las 8:30 solo si hay filas sin estado.** Relee, de la 1:00 a las 8:30 de hoy, la carpeta de RPA, la bandeja de entrada y los no deseados. Porqué: evita un falso "No llegó" cuando la regla de Outlook falla y el correo queda fuera de la carpeta. Lo que siga sin estado pasa a la Etapa 3 con rojo y código 0.
8. **Correo posterior a las 8:30:** se registra solo en el log, sin tocar la celda roja ni el consolidado ya enviado. Si un RPA reenvía su finalización el mismo día, gana el último y el anterior queda en el log.
9. **Guarda de festivos en las dos puertas** (llegada de correo y corte de las 8:30). Si es festivo, la ejecución termina ahí aunque llegue un correo.
10. **No procesar dos veces:** se guarda el identificador de cada correo en el log. Porqué: cubre reintentos y reprocesos.
11. **El catálogo es un archivo aparte, solo de lectura.** El Excel que genera las bitácoras nuevas es otro. Porqué: lo único que cambia con el tiempo (asuntos y errores) queda en un lugar editable sin tocar el flujo.

## Fuera de esta etapa

Pintar la bitácora, marcar el código 0 y escribir los colores es de la Etapa 3. El consolidado y el correo del ticket son de la Etapa 4, que espera a que las filas esperadas y los LOG esperados tengan estado.
