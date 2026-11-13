# Flujo Etapa 2 · C · Corte de las 8:30

> v0.1 · Disparador por hora, L–V 8:30 (hora Colombia). 🟠 entrada · 🟢 salida

```mermaid
flowchart TD
    classDef entrada fill:#fdba74,stroke:#c2410c,color:#111
    classDef salida fill:#86efac,stroke:#15803d,color:#111
    classDef paso fill:#e0f2fe,stroke:#0369a1,color:#111
    classDef decision fill:#fef9c3,stroke:#a16207,color:#111
    classDef gris fill:#e5e7eb,stroke:#6b7280,color:#111

    IN(("ENTRADA<br/>Hora 8:30 L–V<br/>(hora Colombia)")):::entrada
    FEST{"¿Hoy es festivo?<br/>(hoja Festivos)"}:::decision
    FIN(["Fin: sin más acciones"]):::gris
    ABRIR["Abrir la hoja de la semana<br/>(verificar que exista)"]:::paso
    PEND{"¿Hay filas esperadas sin estado?<br/>(las 17 filas + los LOG esperados)"}:::decision
    RELEER["Releer los correos de hoy recibidos entre<br/>1:00 y 8:30 en: carpeta RPA,<br/>bandeja de entrada y no deseados"]:::paso
    OUTA(("SALIDA → Flujo A<br/>id_correo · asunto<br/>cuerpo · fecha_recepción")):::salida
    INB(("ENTRADA ← Flujo A<br/>RPA · tipo · estado · código")):::entrada
    SIGUE{"¿Alguna fila sigue<br/>sin estado?"}:::decision
    CERO["Definir estado: error con código 0<br/>(inicio, finalización, variación de<br/>precios y LOG esperado sin llegar)"]:::paso
    OUT3(("SALIDA → Etapa 3<br/>RPA · tipo · estado = 0 · fecha")):::salida
    OUT4(("SALIDA → Etapa 4<br/>fecha · corte cumplido")):::salida

    IN --> FEST
    FEST -- Sí --> FIN
    FEST -- No --> ABRIR --> PEND
    PEND -- No --> FIN
    PEND -- Sí --> RELEER --> OUTA
    OUTA -.-> INB
    INB --> SIGUE
    SIGUE -- Sí --> CERO --> OUT3
    CERO --> OUT4
    SIGUE -- No --> OUT4
```

## Lectura rápida

| Conexión | Variable | Con quién |
|---|---|---|
| 🟠 Entrada | Hora 8:30 L–V | Disparador por hora |
| 🟢 Salida | id_correo, asunto, cuerpo, fecha_recepción | Flujo A (cada correo recuperado) |
| 🟠 Entrada | RPA, tipo, estado, código | Flujo A (resultado) |
| 🟢 Salida | RPA, tipo, estado = 0, fecha | Etapa 3 |
| 🟢 Salida | fecha, corte cumplido | Etapa 4 |