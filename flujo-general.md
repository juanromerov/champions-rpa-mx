# Flujo general – Automatización bitácora RPA (Sura MX)

> Versión 0.1 · Borrador para validar · Herramienta base: **Power Automate (nube)**. Python queda *en consideración* (ver memoria de decisiones).

## Diagrama general (4 etapas)

```mermaid
flowchart TD
    classDef disparador fill:#dbeafe,stroke:#1d4ed8,color:#111
    classDef etapa fill:#e0f2fe,stroke:#0369a1,color:#111
    classDef decision fill:#fef9c3,stroke:#a16207,color:#111
    classDef salida fill:#dcfce7,stroke:#15803d,color:#111
    classDef gris fill:#e5e7eb,stroke:#6b7280,color:#111

    T1(["⏰ Disparador semanal<br/>Lunes 0:00"]):::disparador
    E1["<b>ETAPA 1 · Generar bitácora</b><br/>Detectar última semana, crear la siguiente (L–V),<br/>calcular fechas y marcar festivos en gris"]:::etapa
    B1[("📗 Excel en OneDrive<br/>Hoja nueva = fecha ISO del lunes")]:::salida

    T2(["⏰ Disparador diario L–V<br/>desde ~1:30 a. m."]):::disparador
    F{"¿Hoy es festivo<br/>en México?"}:::decision
    FG["Columna en gris<br/>No se espera ni evalúa ningún correo"]:::gris

    E2["<b>ETAPA 2 · Leer y clasificar correos</b><br/>Outlook 365: identificar cada RPA por asunto,<br/>detectar éxito / error, asociar correos LOG"]:::etapa
    W{"¿Llegó el correo del RPA<br/>(espera máx. 40 min)?"}:::decision
    C0["Registrar código 0<br/>'No llegó' = error"]:::etapa

    E3["<b>ETAPA 3 · Actualizar bitácora</b><br/>Verde = éxito · Rojo + código = error<br/>Clasificar contra catálogo (0–15, no catalogado = 17)"]:::etapa

    E4["<b>ETAPA 4 · Comunicación consolidada</b><br/>Un único correo diario con los incidentes<br/>(RPA, código, descripción)"]:::etapa
    Q{"¿Hubo<br/>incidentes?"}:::decision
    OK["Registrar día sin incidentes<br/>(sin ticket)"]:::salida
    TK["📧 Envío al buzón de tickets<br/>Ti-Infra MX"]:::salida
    RT["Recibir / registrar respuesta<br/>de creación del ticket"]:::salida

    T1 --> E1 --> B1
    B1 -. "semana disponible" .-> T2
    T2 --> F
    F -- Sí --> FG --> END1(["Fin del día"]):::gris
    F -- No --> E2 --> W
    W -- Sí --> E3
    W -- No --> C0 --> E3
    E3 --> B1
    E3 --> E4 --> Q
    Q -- Sí --> TK --> RT --> END2(["Fin del día"]):::salida
    Q -- No --> OK --> END2
```

## Lectura rápida

| Etapa | Disparador | Entrada | Salida |
|---|---|---|---|
| 1 · Generar bitácora | Lunes 0:00 | Última hoja de la bitácora + hoja `Festivos` | Hoja nueva de la semana (festivos en gris) |
| 2 · Leer y clasificar | Diario L–V | Correos de Outlook 365 | Estado por RPA (éxito / error / no llegó) |
| 3 · Actualizar bitácora | Fin de cada lectura | Estado por RPA + catálogo de incidentes | Celdas con color y código |
| 4 · Consolidado | Tras el último RPA (+ espera) | Incidentes del día | Correo de ticket + respuesta registrada |

## Puntos abiertos que afectan este flujo

1. **Hora de corte del consolidado:** el doc de definición dice 7:32 + 40 min de espera; el contexto inicial dice que el último RPA llega a las 7:46. Falta confirmar la hora real.
2. **Office Scripts en el tenant:** el script actual de la Etapa 1 lo requiere; si no está habilitado, la Etapa 1 hay que resolverla solo con acciones nativas de Excel.
3. **Cuenta de ejecución:** hoy se haría con la cuenta personal; falta saber cuál es válida según parámetros de SETI.
4. **Falla de la propia automatización (EX-06):** el diagrama aún no incluye la ruta de aviso a un responsable; se propone modelarla en el detalle por etapa.
