# Flujo Etapa 1 · Generar bitácora

> v0.1 · Detalle de la Etapa 1 ·
> Parte del flujo general v0.4 (en actualización). Esta etapa corre una vez por semana; las etapas 2, 3 y 4 son diarias y están fuera de este documento.

```mermaid
flowchart TD
    classDef disparador fill:#dbeafe,stroke:#1d4ed8,color:#111
    classDef etapa fill:#e0f2fe,stroke:#0369a1,color:#111
    classDef decision fill:#fef9c3,stroke:#a16207,color:#111
    classDef salida fill:#dcfce7,stroke:#15803d,color:#111
    classDef gris fill:#e5e7eb,stroke:#6b7280,color:#111
    classDef nota fill:#fff,stroke:#9ca3af,color:#444,stroke-dasharray:4 3

    T1(["⏰ Disparador semanal<br/>Lunes 0:00 (zona horaria configurada)"]):::disparador
    CFG[("⚙️ Configuración<br/>Ubicación del Excel (un solo punto,<br/>compartido por las 4 etapas)")]:::nota
    ABRIR["Abrir el Excel de bitácoras<br/>según la configuración"]:::etapa
    EX{"¿Ya existe la hoja con<br/>la fecha ISO del lunes?"}:::decision
    FIN0(["Fin: nada que hacer<br/>(no se duplica la hoja)"]):::gris
    CREAR["<b>ETAPA 1 · Generar bitácora</b><br/>Crear hoja nueva desde PLANTILLA<br/>Nombre = fecha ISO del lunes (AAAA-MM-DD)"]:::etapa
    FECHAS["Calcular y escribir las fechas L–V<br/>(la semana puede cruzar de mes)"]:::etapa
    FEST[("📋 Hoja Festivos<br/>Lista del cliente")]:::nota
    GRIS["Marcar en gris la columna de cada día<br/>que esté en Festivos<br/>(solo visual para el humano)"]:::etapa
    OUT[("📗 Hoja de la semana lista<br/>en el Excel de OneDrive")]:::salida
    FALLA["⚠️ Si falla algún paso:<br/>ruta de aviso a un responsable<br/>(pendiente de diseño · por ahora: juan....@seti.com.co"]:::nota

    T1 --> ABRIR
    CFG -.-> ABRIR
    ABRIR --> EX
    EX -- Sí --> FIN0
    EX -- No --> CREAR --> FECHAS --> GRIS --> OUT
    FEST -.-> GRIS
    CREAR -.-> FALLA
    FECHAS -.-> FALLA
    GRIS -.-> FALLA
```

## Lectura rápida

| Elemento | Detalle |
|---|---|
| Disparador | Lunes 0:00, con la zona horaria configurada explícitamente (los festivos son de México) |
| Entrada | Hojas `PLANTILLA` y `Festivos` del Excel de bitácoras. Siempre se parte de `PLANTILLA`; la hoja de la semana anterior **no** es entrada |
| Qué hace | 1) Verifica si la hoja de la semana ya existe. 2) Si no existe, la crea desde `PLANTILLA`. 3) Escribe las fechas de lunes a viernes. 4) Marca en gris los festivos |
| Salida | Una hoja nueva por semana, con nombre = fecha ISO del lunes (10 caracteres, cabe en el límite de 31 de Excel) |
| Idempotencia | Si la hoja ya existe, no hace nada ni duplica (la Etapa 1 se puede repetir sin riesgo) |

## Decisiones aplicadas a esta etapa

1. **Una hoja nueva por semana**, no bloques apilados en una sola hoja. Porqué: las celdas quedan en direcciones fijas y la Etapa 3 no tiene que buscar el bloque de la semana entre miles de filas. Se descartó el formato apilado que se usa hoy a mano.
2. **Siempre desde `PLANTILLA`.** Porqué: se parte limpio, sin arrastrar colores ni valores de la semana anterior.
3. **Verificar que la hoja exista sin duplicar.** Porqué: el margen del lunes es corto (hoja a las 0:00 y primer correo hacia la 1:30). La Etapa 3 también verifica que la hoja exista antes de escribir.
4. **Festivos: lista entregada por el cliente**, hoy dentro del mismo Excel (hoja `Festivos`). La automatización decide con la lista, no con el color.
5. **El gris es solo visual** para quien lee la bitácora (se envía al cliente en otro correo, fuera de este alcance). No interviene en ninguna decisión del flujo.
6. **Archivo Excel configurable.** La ubicación del archivo vive en un solo punto de configuración que leen las 4 etapas. El archivo nuevo debe respetar la misma estructura (hojas `PLANTILLA`, `Festivos`, tabla de control y sus nombres).

## Fuera de esta etapa: guarda de festivos

La guarda no pertenece a la Etapa 1. Va al inicio de las etapas 2, 3 y 4, en sus dos puertas de entrada: la llegada de un correo RPA/LOG y el corte de las 8:30. Consulta la lista `Festivos` con la fecha de hoy; si es festivo, la automatización finaliza ahí. Los RPA sí pueden enviar correo en festivo (se descartan en la guarda); ninguno llega sábado ni domingo, así que la guarda solo pregunta por festivos.


