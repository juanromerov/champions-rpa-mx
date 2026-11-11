# Flujo general - automatización

> v0.4 · Automatización en 4 etapas. (En actualización)

>Proceso estandarizado por horas. 
> 8:00 am Consolidado inmediato si los 17 RPA sí ya tienen estado definido

> 8:30 am. Sí no se ha enviado consolidado y hay RPA sin estado, se coloca en 0 'No llegó" y se añade al consolidado de errores

```mermaid
flowchart TD
    classDef disparador fill:#dbeafe,stroke:#1d4ed8,color:#111
    classDef etapa fill:#e0f2fe,stroke:#0369a1,color:#111
    classDef decision fill:#fef9c3,stroke:#a16207,color:#111
    classDef salida fill:#dcfce7,stroke:#15803d,color:#111
    classDef gris fill:#e5e7eb,stroke:#6b7280,color:#111
    classDef nota fill:#fff,stroke:#9ca3af,color:#444,stroke-dasharray:4 3

    T1(["⏰ Disparador semanal<br/>Lunes 0:00"]):::disparador
    E1["<b>ETAPA 1 · Generar bitácora</b><br/>Crear bitacora semanal según fecha en curso (L–V),<br/>calcular fechas y marcar festivos en gris"]:::etapa
    B1[("📗 Excel en OneDrive<br/>Hoja nueva = fecha ISO del lunes")]:::salida
    T1 --> E1 --> B1

    F{"Día hábil:<br/>¿hoy es festivo<br/>en México?"}:::decision
    FG["Columna en gris<br/>No se evalúa nada. No hay disparador para procesos Etapa 2, 3 y 4"]:::gris
    B1 -. "semana disponible" .-> F
    F -- Sí --> FG

    subgraph PA ["Camino A · por cada correo que llega (desde ~1:30 a. m.)"]
        direction TB
        TE(["📩 Llega correo RPA / LOG"]):::disparador
        E2["<b>ETAPA 2 · Leer y clasificar</b><br/>Identificar RPA por asunto,<br/>detectar éxito / error, asociar LOG"]:::etapa
        E3["<b>ETAPA 3 · Actualizar bitácora</b><br/>Verde = éxito · Rojo + código (catálogo 0–15; otro = 17)<br/>Si la celda tenía 0, se sobrescribe y queda en el log"]:::etapa
        COMP{"¿Los 17 RPA esperados<br/>ya tienen estado?"}:::decision
        ESP(["Fin de esta ejecución<br/>(se espera el siguiente correo o el corte)"]):::gris
        ENVA{"¿Consolidado del día<br/>ya enviado?"}:::decision
        TE --> E2 --> E3 --> COMP
        COMP -- No --> ESP
        COMP -- "Sí · consolidar de inmediato" --> ENVA
    end

    subgraph PB ["Camino B · corte a las 8:25 a. m."]
        direction TB
        TC(["⏰ 8:25 a. m. L–V"]):::disparador
        ENVB{"¿Consolidado del día<br/>ya enviado?"}:::decision
        FALT["Marcar código 0 (No llegó = error)<br/>en cada RPA sin estado"]:::etapa
        TC --> ENVB
        ENVB -- No --> FALT
    end

    F -- No --> TE
    F -- No --> TC
    E3 -.-> B1

    FIN0(["Nada que hacer:<br/>ya se consolidó"]):::gris
    ENVA -- Sí --> FIN0
    ENVB -- Sí --> FIN0

    E4["<b>ETAPA 4 · Consolidado (único del día)</b><br/>RPA, código y descripción de los incidentes<br/>Marca 'consolidado enviado' en el log"]:::etapa
    Q{"¿Hubo<br/>incidentes?"}:::decision
    TK["📧 Envío al buzón de tickets<br/>Ti-Infra MX"]:::salida
    RT["Registrar respuesta de<br/>creación del ticket"]:::salida
    OK["Registrar día sin incidentes<br/>(sin ticket)"]:::salida

    ENVA -- No --> E4
    FALT --> E4
    E4 --> Q
    Q -- Sí --> TK --> RT
    Q -- No --> OK

    TARD["📩 Correo después del consolidado<br/>Se actualiza bitácora + log<br/>No se envía un segundo consolidado*"]:::nota
    TARD -.-> E2
```

## Lectura rápida

| Etapa | Disparador | Entrada | Salida |
|---|---|---|---|
| 1 · Generar bitácora | Lunes 0:00 | Última hoja de la bitácora + hojas `PLANTILLA` y `Festivos` | Hoja nueva de la semana (festivos en gris) |
| 2 · Leer y clasificar | Cada correo RPA/LOG que llega (L–V, desde ~1:30 a. m.; no se evalúa en festivo) | Correos de Outlook 365 | Estado por RPA (éxito / error + código) |
| 3 · Actualizar bitácora | Tras clasificar cada correo; y a las 8:25 a. m. para los RPA sin estado | Estado por RPA + catálogo de incidentes | Celdas en verde, o rojo + código; rojo + código 0 ("No llegó") a las 8:25 |
| 4 · Consolidado | Lo que ocurra primero: los 17 RPA ya tienen estado (consolida de inmediato) o corte de las 8:25 a. m. (hora Colombia) | Incidentes del día + tabla de control | Un único correo de ticket + respuesta registrada; marca "consolidado enviado" en la tabla de control |

## Puntos abiertos que afectan este flujo

1. **Correos LOG con consolidación inmediata (P-01):** ¿se consolida sin esperar el LOG o se espera si hay error en los 3 RPA que lo envían?
2. **Correo posterior a las 8:25 (P-02):** con la revisión única, la celda queda en rojo con 0; falta definir si el correo tardío se ignora, va solo al log o se anota aparte.
3. **Consolidación inmediata vs. revisión única (P-06):** confirmar si se mantiene la comprobación de "17 con estado" tras cada correo.
4. **Día sin incidentes (P-03) y RPA esperados por día (P-04):** ¿Ti-Infra espera correo diario sin errores? ¿Son los mismos 17 todos los días?
5. **Office Scripts en el tenant:** el script actual de la Etapa 1 lo requiere; si no está habilitado, se resuelve con acciones nativas de Excel.
6. **Cuenta de ejecución:** hoy se haría con la cuenta personal; falta saber cuál es válida según parámetros de SETI.
7. **Falla de la propia automatización (EX-06):** el diagrama aún no incluye la ruta de aviso a un responsable; se modelará en el detalle por etapa.
8. **Pendientes de diseño:** crear la tabla de control en el Excel y publicar el flujo general v0.4, sin la sobrescritura del 0.
