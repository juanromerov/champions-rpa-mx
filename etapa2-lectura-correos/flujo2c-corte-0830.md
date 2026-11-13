# Flujo Etapa 2 · C · Corte de las 8:30

> v0.2 · Disparador por hora, L–V 8:30 (hora Colombia). 🟠 entrada · 🟢 salida

```mermaid
flowchart TD
    classDef entrada fill:#fdba74,stroke:#c2410c,color:#111
    classDef salida fill:#86efac,stroke:#15803d,color:#111
    classDef paso fill:#e0f2fe,stroke:#0369a1,color:#111
    classDef decision fill:#fef9c3,stroke:#a16207,color:#111
    classDef gris fill:#e5e7eb,stroke:#6b7280,color:#111
    classDef nota fill:#fff,stroke:#9ca3af,color:#444,stroke-dasharray:4 3

    IN(("ENTRADA<br/>Hora 8:30 L–V<br/>(hora Colombia)")):::entrada
    FEST{"¿Hoy es festivo?<br/>(hoja Festivos)"}:::decision
    ENV{"¿El día ya está cerrado?<br/>(hoja Control: enviando,<br/>enviado o sin_errores)"}:::decision
    FIN(["Fin: no se hace nada más"]):::gris
    ABRIR["Abrir la hoja de la semana<br/>(verificar que exista)"]:::paso
    PEND{"¿Hay filas esperadas sin estado?<br/>(las 17 del catálogo + los LOG en_espera_log)"}:::decision
    FIN2(["Fin: todo tiene estado"]):::gris
    RELEER["Releer los correos de hoy recibidos entre<br/>1:00 y 8:30 en: carpeta RPA,<br/>bandeja de entrada y no deseados"]:::paso
    OUTA(("SALIDA → Flujo A<br/>por cada correo hallado:<br/>id_correo · asunto · cuerpo · fecha_recepción")):::salida
    INB(("ENTRADA ← Flujo A<br/>rpa · grupo · estado · cod_error")):::entrada
    SIGUE{"¿Alguna fila esperada<br/>sigue sin estado?"}:::decision
    SINLL["Por cada una, crear su fila (o actualizar la<br/>en_espera_log): estado sin_llegar · cod_error 0<br/>observacion corte_0830 · id_correo vacío<br/>paso clasificado"]:::paso
    OUTE(("SALIDA → Etapa 3<br/>rpa · sin_llegar · cod_error 0")):::salida
    NOTA["A las 8:31 arranca la Etapa 4 con su propio disparador<br/>(con las mismas guardas: festivo y día cerrado)"]:::nota

    IN --> FEST
    FEST -- Sí --> FIN
    FEST -- No --> ENV
    ENV -- Sí --> FIN
    ENV -- No --> ABRIR --> PEND
    PEND -- No --> FIN2
    PEND -- Sí --> RELEER --> OUTA
    OUTA -.-> INB
    INB --> SIGUE
    SIGUE -- No --> FIN2
    SIGUE -- Sí --> SINLL --> OUTE
    OUTE -.-> NOTA
```

## Lectura rápida

| Conexión | Variable | Con quién |
|---|---|---|
| 🟠 Entrada | Hora 8:30 L–V | Disparador por hora |
| 🟢 Salida | id_correo, asunto, cuerpo, fecha_recepción (uno por correo hallado) | Flujo A |
| 🟠 Entrada | rpa, grupo, estado, cod_error | Flujo A |
| 🟢 Salida | rpa, sin_llegar, cod_error 0 | Etapa 3 |

## Reglas básicas de arranque

1. Si hoy es festivo, la ejecución termina sin hacer nada.
2. Si el día ya está cerrado en la hoja Control, la ejecución termina sin hacer nada.
3. La Etapa 4 arranca a las 8:31 con su propio disparador, un minuto después del corte.
