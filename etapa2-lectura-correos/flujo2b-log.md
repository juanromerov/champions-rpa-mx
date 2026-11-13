# Flujo Etapa 2 · B · Verificación de llegada del LOG

> v0.1 · Corre cuando un RPA con LOG asociado termina en error. 🟠 entrada · 🟢 salida

```mermaid
flowchart TD
    classDef entrada fill:#fdba74,stroke:#c2410c,color:#111
    classDef salida fill:#86efac,stroke:#15803d,color:#111
    classDef paso fill:#e0f2fe,stroke:#0369a1,color:#111
    classDef decision fill:#fef9c3,stroke:#a16207,color:#111
    classDef gris fill:#e5e7eb,stroke:#6b7280,color:#111
    classDef nota fill:#fff,stroke:#9ca3af,color:#444,stroke-dasharray:4 3

    IN(("ENTRADA ← Flujo A<br/>RPA · código")):::entrada
    TIENE{"¿El RPA tiene un LOG en<br/>el catálogo? (RPA origen)"}:::decision
    FIN(["Fin: no se espera LOG"]):::gris
    YA{"¿Su LOG ya tiene estado hoy?<br/>(pudo llegar fuera de orden)"}:::decision
    ESP["Marcar el LOG como esperado<br/>(sin sondeo: el flujo no se queda esperando)"]:::paso
    NOTA["Si el LOG llega después, lo atiende el Flujo A<br/>y lo pinta en verde. Si a las 8:30 sigue sin<br/>llegar, el corte (Flujo C) lo marca rojo con 0"]:::nota
    OUT1(("SALIDA → Etapa 4<br/>LOG recibido · RPA origen")):::salida
    OUT2(("SALIDA → Etapa 4 y Flujo C<br/>LOG esperado · RPA origen")):::salida

    IN --> TIENE
    TIENE -- No --> FIN
    TIENE -- Sí --> YA
    YA -- Sí --> OUT1
    YA -- No --> ESP --> OUT2
    ESP -.-> NOTA
```

## Lectura rápida

| Conexión | Variable | Con quién |
|---|---|---|
| 🟠 Entrada | RPA, código | Flujo A |
| 🟢 Salida | LOG recibido · RPA origen | Etapa 4 |
| 🟢 Salida | LOG esperado · RPA origen | Etapa 4 y Flujo C |