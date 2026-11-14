# Claude.md – Memoria de decisiones · Automatización bitácora RPA (Sura MX)

> **Para qué sirve:** registro vivo de lo **aprobado** en el proyecto, con su porqué. Es la fuente para actualizar la memoria de Claude y saber en qué punto vamos.
> **Qué NO va aquí:** dudas, supuestos sin confirmar ni tareas → están en `task.md`.
> **Última actualización:** 2026-11-13 · **Versión:** 0.12

## 1. Contexto del proyecto (resumen)
- **Objetivo:** automatizar la bitácora semanal de ejecuciones RPA: generar el Excel cada semana, diligenciarlo leyendo correos y enviar un único consolidado diario de errores (que dispara el ticket).
- **Meta:** automatizar ≥ 90 % del tiempo de revisión manual y validar el 100 % de los procesos esperados (~17 RPA diarios) en los primeros 30 días de producción.
- **Cuatro etapas:** 1) Generar bitácora · 2) Leer y clasificar correos · 3) Actualizar bitácora · 4) Consolidado y correo de ticket.
- **Reglas de trabajo con Claude:** no generar código ni archivos sin autorización; preguntar antes de cualquier acción generativa; Claude actúa como compañero de trabajo, explica el porqué y sugiere mejoras o riesgos.
- **Acceso a archivos (2026-11-13):** por temas de seguridad, Claude **no** tiene acceso a las carpetas ni a los archivos del usuario; trabaja con lo que está en el proyecto. Cuando el usuario tenga el aval, agregará extensiones para que Claude ayude con los documentos.

## 2. Decisiones aprobadas

### D-01 · Herramienta: solo Power Automate (en la nube)
- **Decisión:** se proyecta usar totalmente Power Automate. Python **no se ve necesario**, pero queda **en consideración**.
- **Porqué:** las 4 etapas se resuelven con conectores de Outlook y Excel; menos piezas que mantener.
- **Riesgos a vigilar:** (a) Office Scripts habilitado en el tenant (el script actual de la Etapa 1 lo requiere); (b) lógica de clasificación del cuerpo del correo, que puede volverse engorrosa si crece el catálogo de errores. Si alguno se complica, se reabre Python.

### D-02 · Dos disparadores independientes
- **Decisión:** disparador semanal (lunes 0:00) para la Etapa 1; flujos diarios L–V para las Etapas 2 a 4.
- **Porqué:** la bitácora de la semana ya existe antes del primer correo y un fallo al generarla no tumba la lectura diaria.

### D-03 · Festivos se evalúan antes de leer correos
- **Decisión:** si el día es festivo en México, la columna queda en gris y no se evalúa ningún correo (ni se marca error). El mecanismo exacto está en D-11 y D-34.
- **Porqué:** evita falsos errores de código 0 en días inhábiles (RF-12, RN-05).

### D-04 · Camino A: procesamiento por cada correo que llega
- **Decisión:** el flujo se dispara con cada correo RPA/LOG, lo clasifica y escribe su estado en el Registro (que luego pasa a la bitácora).
- **Porqué:** cada correo se registra apenas llega, sin depender de una espera larga y frágil por cada RPA.

### D-05 · Camino B: corte a las 8:30 a. m. (hora Colombia)
- **Decisión:** se hace **una sola revisión a las 8:30 a. m.**: todo RPA sin estado se marca en **rojo con error y código 0** ("No llegó" = error) y se consolida.
- **Porqué:** da un margen de ~44 min después del último RPA (llega ~7:46). **Reemplaza** la referencia anterior de 7:32 + 40 min del doc *Definición y registro*.
- **Historial de la hora:** se había aprobado 8:25 y el usuario confirmó el 2026-11-11 que es **8:30**. Los documentos que aún dicen 8:25 (diagrama y tabla del flujo general, *Definición y registro*) deben alinearse.
- **Nota:** se descartó la idea de que el 0 fuera provisional y se sobrescribiera con un correo posterior (antes D-06). El 0 de las 8:30 es el resultado de esa revisión.
- **Precisión (2026-11-12):** el corte es un disparador por hora, la puerta B de las etapas 2 a 4; su detalle está en D-25.

### D-07 · Un solo consolidado de errores al día
- **Decisión:** se envía un único consolidado por día hábil, no correos de corrección.
- **Porqué:** el ticket es una foto estable del día y se evitan correos duplicados.

### D-08 · Consolidación inmediata si ya están todos los RPA
- **Decisión:** si los 17 RPA esperados ya tienen estado, se consolida de inmediato sin esperar las 8:30.
- **Porqué:** detecta y comunica errores más temprano. La regla se definió como "los 17 tienen estado" y **no** como "llegó el RPA 17", porque los correos pueden llegar fuera de orden.
- **Precisión (2026-11-12):** "los 17" son 17 **filas esperadas** (8 de inicio, 8 de finalización y la variación de precios; ver D-22). Además, un LOG esperado (D-24) debe tener estado para poder consolidar de inmediato.
- **Confirmada (2026-11-13):** tras revisarla, el usuario decidió **mantener la consolidación inmediata** (cierra P-18 y P-06). La Etapa 4 tiene entonces dos formas de arrancar: apenas todo tiene estado, o con su propio disparador a las 8:31 (D-38).

### D-09 · Control de "día cerrado": tabla de control en el Excel (opción 1)
- **Decisión:** el control se hace con la hoja/tabla `Control` en el mismo Excel (opción 1). Su diseño está en D-35.
- **Porqué:** es visible, queda en el mismo archivo y sirve a la vez como log de auditoría (RF-17). Evita dos consolidados.
- **Descartadas:** buscar solo en Enviados de Outlook (depende del asunto y del tiempo de indexado); lista de SharePoint/Dataverse (excesivo para 17 RPA al día).

### D-10 · Formato de entregables de diseño
- **Decisión:** los flujos se entregan en Mermaid dentro de `.md`, con PNG/SVG para compartir; primero el flujo general y luego uno por etapa. La memoria de decisiones se entrega como archivo descargable.
- **Precisión (2026-11-12):** las conexiones entre flujos se dibujan con círculos: **naranja para entrada y verde para salida**, con la variable o dependencia que viaja escrita adentro.

### D-11 · Guarda de festivos al inicio de cada ejecución (etapas 2, 3 y 4)
- **Decisión:** las etapas 2, 3 y 4 tienen dos puertas de entrada: la llegada de un correo RPA/LOG y el corte de las 8:30. Justo después de cada puerta, antes de hacer nada, una guarda consulta la **lista de festivos** con la fecha de hoy. Si es festivo, la automatización **finaliza ahí** (no clasifica, no escribe, no marca código 0, no consolida, no deja fila en el Registro). Precisa D-03.
- **Porqué:** (a) los RPA sí pueden enviar correo en festivo y no deben generar error; (b) la guarda debe estar en las dos puertas, porque si solo estuviera en la Etapa 2 el corte de las 8:30 enviaría un consolidado en festivo; (c) un disparador por día no se puede saltar fechas de forma nativa en Power Automate, la guarda es más simple y robusta.
- **Fuente de verdad:** la lista de festivos la entrega el cliente y está en la hoja `Festivos` del mismo Excel, **solo con la fecha** (D-37). La automatización decide con la lista, **no con el color**. El gris de la columna es solo visual para el humano (la bitácora se envía al cliente en otro correo, fuera de este alcance) y no interviene en ninguna decisión.
- **Alcance:** ningún RPA envía correo sábado o domingo; por eso la guarda solo pregunta por festivos, no por "día no hábil".
- **Precisión (2026-11-13):** es la primera de las dos reglas básicas de toda la automatización (D-34).

### D-12 · Etapa 1: una hoja nueva por semana, siempre desde `PLANTILLA`
- **Decisión:** cada semana se crea una **hoja nueva** con nombre = fecha ISO del lunes (AAAA-MM-DD), siempre a partir de `PLANTILLA`. La hoja de la semana anterior **no** es entrada de la Etapa 1.
- **Porqué:** con una hoja por semana las celdas quedan en direcciones fijas y la Etapa 3 no tiene que ubicar un bloque entre miles de filas. Partir de `PLANTILLA` evita arrastrar colores o valores de la semana anterior.
- **Descartada:** apilar la bitácora de cada semana como bloque nuevo debajo del anterior en una sola hoja (así se hace hoy a mano, ~3.800 filas): obligaría a ubicar el bloque, a leer rangos grandes y a usar fecha real en cada encabezado.
- **Límite:** Excel no tiene límite fijo de hojas (depende de la memoria); ~52 hojas al año no son un problema. El nombre ISO (10 caracteres) cabe en el máximo de 31.

### D-13 · Verificar existencia sin duplicar (Etapas 1 y 3)
- **Decisión:** la Etapa 1 verifica si la hoja de la semana ya existe y, si existe, termina sin hacer nada. La Etapa 3 también verifica que la hoja exista antes de escribir.
- **Porqué:** el margen del lunes es corto (hoja a las 0:00; primer correo hacia la 1:30). Si la Etapa 1 fallara, la Etapa 3 no debe escribir a ciegas. Además hace repetible la Etapa 1 sin riesgo de duplicados. La alerta por falla se define en D-19.

### D-14 · El archivo Excel debe ser configurable
- **Decisión:** la ubicación del Excel (ruta o ID en OneDrive) vive en **un solo punto de configuración** que leen las 4 etapas. Los nombres de las hojas fijas (`PLANTILLA`, `Festivos`, `Control`, `Registro`) también se tratan como contrato y no se escriben a mano en cada acción.
- **Porqué:** las 4 etapas usan el mismo archivo. Si cambia (p. ej., un archivo por año o uno nuevo del cliente), el cambio debe hacerse en un solo lugar para no dejar una etapa escribiendo en el archivo viejo sin que nadie lo note.
- **Límite:** el archivo nuevo debe respetar la misma estructura (hojas y nombres). **Cómo implementarlo** (variables de entorno o variable inicial) se define en construcción.
- **Precisión (2026-11-12):** el punto de configuración incluye también la ubicación del Excel de catálogo (D-27), que es un archivo aparte.

### D-15 · Los md de flujo van a GitHub: sin datos personales ni puntos internos
- **Decisión:** los `.md` de flujo (general, Etapa 1, etc.) se suben a GitHub, por lo que **no** incluyen la cuenta de ejecución, el destino de las alertas, los puntos abiertos ni otros datos internos o personales. Esa información vive solo en la memoria del proyecto (`Claude.md` y `task.md`).
- **Porqué:** el usuario no quiere subir esa información a GitHub.
- **Consecuencia:** al generar o actualizar un md de flujo, Claude no copia esos datos ni los puntos abiertos al md; los mantiene en la memoria.

### D-16 · Desarrollo con un script nuevo, sin amarrarse al actual
- **Decisión:** al empezar el desarrollo se hará un **script nuevo**; no se parte del script actual de la Etapa 1.
- **Contexto:** el script actual recibe `fechaIso` como parámetro y requiere Office Scripts. Queda por definir en el diseño de dónde sale la fecha del lunes en el script nuevo y si usará Office Scripts o acciones nativas de Excel (depende de lo que permita el tenant).

### D-17 · Cuenta de ejecución: la cuenta personal del usuario en SETI
- **Decisión:** el desarrollo y la ejecución de la automatización se hacen desde la **cuenta personal del usuario en SETI** (SETI es la empresa), con credenciales SETI.
- **Porqué:** es un proyecto a presentar, un "curso"; por eso se hace en cuenta personal.
- **Aclaración (2026-11-12):** "personal" significa la cuenta propia del usuario dentro de SETI, no una cuenta privada ajena a la empresa. Todo se hace con credenciales SETI. Al estar en el tenant de SETI, dependen de sus políticas: Office Scripts (D-01, D-16) y el envío de alertas (D-19); ver las tareas de verificación en `task.md`.
- **Nota:** resuelve el pendiente de qué cuenta es válida según parámetros de SETI. Si el proyecto se llevara a un entorno real del cliente, habría que revisar este punto (ver `task.md`, ideas a futuro).

### D-18 · Zona horaria de todos los disparadores: Colombia
- **Decisión:** todos los disparadores (lunes 0:00 y los diarios) se configuran en hora de Colombia. Resuelve P-05.
- **Porqué:** el corte de D-05 ya está definido en hora Colombia; una sola zona en todos los disparadores evita confusiones.
- **Nota:** los festivos son de México, pero la ventana diaria de operación (~1:30 a 8:30 a. m. Colombia) cae en el mismo día calendario en ambos países, así que la fecha que consulta la guarda coincide.

### D-19 · Falla de la propia automatización: alerta por correo y por Teams
- **Decisión:** si la automatización falla (por ejemplo, un paso de la Etapa 1), se genera una **alerta al correo personal del usuario y por Teams**. Precisa EX-06 y no marca falsos errores de RPA.
- **Porqué:** dos canales evitan depender de uno solo si el buzón es justo lo que falla.
- **Pendiente de diseño:** la ruta completa (qué falla se avisa en cada etapa y con qué contenido) se modela en el detalle de cada etapa.

### D-20 · Etapa 2 · Captura de correos
- **Decisión:** una carpeta dentro del correo y una regla de Outlook **de servidor** (por remitente del robot y asunto) mueven allí todos los correos de RPA. El flujo se dispara con cada correo que llega a la carpeta, con concurrencia limitada a 1, y lo primero que hace es la guarda de festivos (D-11) y la de día cerrado (D-34).
- **Porqué:** los RPA envían entre ~1:30 y 7:46 con el computador probablemente apagado; una regla de cliente dejaría el correo en la bandeja de entrada y el flujo nunca se dispararía, lo que produciría un falso "No llegó". La regla es el punto único de falla de la etapa; por eso existe la revisión acotada de D-25.

### D-21 · Etapa 2 · El asunto completo identifica el correo
- **Decisión:** el correo se reconoce comparando su **asunto completo**, normalizado (minúsculas, sin tildes, sin espacios sobrantes), contra `Catalogo_RPA`; el tipo (inicio, finalización, variación de precios o LOG) sale del catálogo. Un asunto que no coincide se anota en el Registro como "no reconocido" y no es error de RPA.
- **Porqué:** los asuntos son siempre los mismos. Algunos nombres están contenidos en otros (un LOG lleva el nombre de su RPA), así que buscar solo una parte del nombre confundiría los correos. Reemplaza la idea de buscar "notificación de inicio" dentro del asunto.
- **Tres grupos de correo:** inicio, finalización y LOG.
- **Hallazgo (2026-11-12):** por el reenvío de los correos, **todos los asuntos llegan con el prefijo `RV:`** (p. ej., `RV: Notificacion de inicio de proceso Fondo Ahorro Bimbo`). Hay que ignorarlo al comparar. **Resuelto el 2026-11-13: ver D-29.**
- **Nota (2026-11-13):** la fila de un correo no reconocido no guarda el asunto (el Registro no tiene esa columna); solo queda la marca `no_reconocido` y el `id_correo`.

### D-22 · Etapa 2 · El correo de inicio se marca en verde, con fila propia
- **Decisión:** el correo de inicio siempre debe llegar; al llegar se marca **verde** en su fila. Si no llega, a las 8:30 queda en rojo con código 0 (regla de la automatización). No define el estado final del RPA.
- **Porqué:** el cliente organiza así la información y el inicio tiene su propia fila en la bitácora. Reemplaza la propuesta de "solo registrarlo en el log".
- **Conteo de filas esperadas:** 17 = 8 de inicio + 8 de finalización + 1 variación de precios (sin inicio); más 3 filas de LOG, que se esperan solo en el caso de D-24.
- **Precisión (2026-11-13):** si el correo de inicio no llega, se le pone 0 **aunque internamente se sepa que el RPA terminó bien**, y se informa en el consolidado. El equipo correspondiente decide cómo maneja ese error (si lo valida o no); eso queda fuera del alcance de la automatización, que solo revisa si el correo llegó.

### D-23 · Etapa 2 · Éxito, error y variación de precios
- **Decisión:** el error se lee **siempre de la fila Motivo**. Se compara normalizado, con la frase completa y de la más específica a la más general (el 6 antes que el 3). Si nada coincide, es código 17 ("Detenido"). Un cuerpo de finalización que no se reconoce es código 17 con marca "revisar formato", **nunca éxito**. El Motivo es fijo según los errores registrados hasta hoy.
- **Variación día anterior precios Bimbo:** si llega, es éxito (la fecha de la bitácora se toma de la recepción, no del texto); si no llega, es 0. No tiene otro error posible.
- **Códigos:** los errores son los códigos 1 a 18. El 0 lo marca la revisión de las 8:30 y no sale de un correo. El 16 se mantiene en la bitácora (catálogo y leyenda) pero no tiene influencia: la automatización no lo asigna. Los errores también se consolidan en la bitácora.
- **Porqué:** un falso éxito esconde un problema (contra el objetivo de cero omisiones); un falso error solo cuesta una revisión.
- **Precisión (2026-11-12):** el usuario encontró que los **textos de error de los correos no están escritos igual que en la bitácora**. Decisión: si el Motivo no coincide con el catálogo, la bitácora queda en código **17**, y en el consolidado se **copia el texto del error tal como viene en el correo**. Así no se pierde información aunque el catálogo no cubra ese texto.
- **Precisión (2026-11-13):** el texto del error se copia en el consolidado **en todos los errores** (ver D-31). El supuesto de que "el Motivo es fijo" queda **en revisión**: los ejemplos reales del usuario traen datos variables (el nombre del RPA dentro de la frase, fechas y usuarios), por lo que la comparación de "frase completa" puede cambiar a comparar la parte fija del texto (P-15 en `task.md`).

### D-24 · Etapa 2 · Correos LOG
- **Decisión:** el LOG solo llega si el RPA anterior falla (hoy: Cierre Pensiones Bimbo, Cierre Fondo de Ahorro Bimbo y Cierre Bimbo Web) y por defecto su fila queda en gris. Cuando uno de esos RPA termina en error, arranca un **flujo 2** (flujo B) que verifica la llegada del LOG: si llega, su casilla queda en **verde**; mientras no llegue queda como "esperado" y el consolidado inmediato (D-08) lo espera; si a las 8:30 no llegó, queda en rojo con código 0 y se consolida. Si el RPA salió bien, su LOG no se espera.
- **Porqué:** la ausencia de un LOG solo es un problema cuando su RPA falló. Resuelve P-01 (opción 2).
- **Diseño del flujo 2:** sin sondeo; el LOG que llega entra por la puerta de llegada de correo y el corte de las 8:30 atrapa su ausencia. **Confirmado por el usuario el 2026-11-13** al aprobar los flujos A, B y C (cierra P-10).
- Los LOG no cuentan dentro de las 17 filas esperadas.
- **Cierre de P-12 (2026-11-13):** el usuario indicó que el hecho de que el LOG de Cierre Pensiones Bimbo "nunca haya llegado" es solo un hallazgo y **no una regla** de la automatización: ese LOG se trata igual que cualquier otro.
- **Precisión (2026-11-13):** para saber de qué RPA depende cada LOG se usa la columna `RPA origen` del catálogo, que lleva el asunto de la finalización del RPA. Es lo único que hace falta enlazar por ahora (cierra P-11).
- **Confirmado (2026-11-13, Etapa 3):** en la bitácora, la fila del LOG pasa de **gris a verde** cuando el LOG llega (ver D-41).

### D-25 · Etapa 2 · El corte de las 8:30 es un disparador; revisión acotada
- **Decisión:** el corte es una segunda puerta de entrada (puerta B), programada L–V a las 8:30 hora Colombia. Corre la revisión solo si hay filas esperadas sin estado. Relee los correos recibidos entre la **1:00 y las 8:30** de ese día en la carpeta de RPA, la bandeja de entrada y los no deseados. Lo que siga sin estado pasa a la Etapa 3 con rojo y código 0. Precisa D-05 y D-11.
- **Porqué:** "No llegó" es la ausencia de un evento, y un flujo que solo se dispara cuando llega un correo no corre cuando no llega ninguno. La ventana horaria evita revisar todo el buzón y la relectura cubre una regla de Outlook que falle.
- **Precisión (2026-11-13):** el usuario indica que, según la experiencia de la operación, el último RPA llega hacia las 7:45–7:50 y las 8:30 es un margen muy alto. Por eso se **descarta el riesgo de concurrencia** entre el flujo del corte y el flujo de llegada de correos: a las 8:30 no se esperan correos en curso.

### D-26 · Etapa 2 · Reprocesos y correos que llegan con el día cerrado
- **Decisión:** si un RPA envía dos finalizaciones el mismo día, gana el último y el anterior queda en el Registro. Un correo no se procesa dos veces (se guarda su identificador en el Registro). Resuelve P-02.
- **Porqué:** es lo más simple y deja rastro.
- **Precisión (2026-11-13):** "gana el último" se resuelve en el Registro con la columna `recibido`: la llave es `fecha` + `rpa` + `grupo` y gana el de `recibido` más reciente; la fila del que pierde pasa a `paso = solo_registro` ("ya no define lo que muestra la bitácora"). El `id_correo` es otra cosa: solo evita procesar **el mismo correo** dos veces, no decide entre dos correos distintos.
- **Cambio (2026-11-13):** antes un correo posterior a las 8:30 se anotaba en el Registro. Ahora, con la regla de "día cerrado" (D-34), **un correo que llega cuando el consolidado ya se envió o el día ya está cerrado no se procesa**: la automatización llega hasta ese envío y no valida nada más. Esto descarta P-19 (fila `sin_llegar` frente a un correo posterior).

### D-27 · Etapa 2 · Catálogo en un Excel aparte, solo de lectura
- **Decisión:** el catálogo es un archivo distinto del Excel que genera las bitácoras nuevas y el flujo solo lo lee. Contiene la tabla `TablaRPA` (hoja `Catalogo_RPA`: Asunto, Tipo, Hora aprox, RPA origen) y la tabla `TablaErrores` (hoja `Catalogo_Errores`: Codigo, Texto, Activo). El usuario pega los asuntos reales desde el histórico de correos; la hora aproximada es la que puso el cliente y es solo referencia. El texto del código 7 está por verificar: "prcoseo" parece una errata de la bitácora (hay dos filas, la incorrecta inactiva).
- **Porqué:** solo debe llevar lo que la automatización necesita (más información es ruido); lo que cambia con el tiempo queda en un lugar editable sin tocar el flujo; el conector de Excel de Power Automate lee tablas.
- **Precisión (2026-11-12):** los textos de `TablaErrores` se copiaron de la bitácora y **son provisionales**: no coinciden con los de los correos y se reemplazarán con lo que encuentre el barrido de errores del usuario (P-15 en `task.md`).
- **Precisión (2026-11-13):** el usuario ya cargó los asuntos reales en `Catalogo_RPA.xlsx` (subido al proyecto): 20 filas (8 de inicio, 8 de finalización, 1 de variación de precios y 3 de LOG), escritos sin el prefijo `RV:`. `RPA origen` solo se llena en los LOG. `Catalogo_Errores` ya tiene una columna de ejemplos reales de error. **El catálogo de errores aún no está listo:** falta validar los errores con el histórico de correos, que hoy no se puede revisar por una caída de Microsoft; el usuario subirá el archivo cuando lo tenga.
- **Precisión (2026-11-13, Etapa 3):** `TablaRPA` gana la columna `fila_bitacora` (ver D-42).

### D-28 · Las Etapas 2 y 3 se mantienen separadas
- **Decisión:** el 2026-11-12 se consideró unir las Etapas 2 y 3 en un solo flujo (llega RPA → clasifica → define estado → llena bitácora) y el usuario decidió **no hacer ningún cambio**: se conserva el diseño de cuatro etapas.
- **Porqué:** el usuario decidió no cambiar el diseño ya acordado (no indicó otro motivo). En la discusión se vio que la fusión eliminaba el traspaso entre etapas, pero perdía la frontera que permite probar la clasificación sola.
- **Consecuencia:** el traspaso de datos de la Etapa 2 a la 3 se define en D-30.

### D-29 · Etapa 2 · Se quitan los prefijos de reenvío antes de comparar el asunto (resuelve P-14)
- **Decisión (2026-11-13):** antes de normalizar y comparar el asunto, se quitan al inicio los prefijos de reenvío o respuesta, repetidos y en sus variantes (`RV:`, `FW:`, `Fwd:`, `RE:`, sin importar mayúsculas). El catálogo guarda los asuntos reales sin prefijo y la comparación sigue siendo de asunto completo (D-21).
- **Porqué:** por temas de licencias y por trabajar desde la cuenta del usuario en SETI, los correos de los RPA no llegan directo al buzón de la automatización: llegan a otro buzón que los reenvía, y por eso el asunto trae `RV:`. Normalizar el asunto del correo y el del catálogo funciona con o sin prefijo.
- **Precisión:** el `RV:` seguirá apareciendo durante las pruebas. Cómo llegan exactamente los correos reenviados (remitente, cuerpo y hora) no se ha podido verificar todavía; queda como prueba (ver `task.md`).

### D-30 · Hoja `Registro` como punto de control entre etapas (resuelve P-09, P-13 y P-17)
- **Decisión (2026-11-13):** la automatización lleva una hoja `Registro` (tabla `TablaRegistro`) con **una fila por correo procesado** (~20 al día, ~5.000 al año). Resuelve P-13 con la opción B (guardar primero y luego entregar). Su columna más importante es **`paso`**: marca en qué punto va cada fila para que cada etapa sepa qué hacer (valores en D-36).
- **Columnas finales (10):** `fecha`, `id_correo`, `recibido`, `rpa`, `grupo`, `estado`, `cod_error`, `error`, `observacion`, `paso`.
  - `id_correo`: imprescindible; evita procesar el mismo correo dos veces. Vacío en las filas del corte de las 8:30 (se identifican por `fecha` + `rpa`).
  - `recibido`: hora de recepción en hora Colombia (Power Automate la entrega en UTC). Sirve para "gana el último" (D-26) y para comparar la hora real con la `Hora aprox` del catálogo y afinar las horas.
  - `grupo` (antes `Tipo`): inicio, finalizacion o log. La variación de precios cuenta como `finalizacion`.
  - `estado`: inicio_recibido, exito, error, log_esperado, log_recibido o sin_llegar.
  - `cod_error` (antes Código): 0 a 18; el 0 solo lo asigna el corte de las 8:30.
  - `error` (antes Motivo): texto del error copiado del correo; de aquí lo toma el consolidado (D-31).
  - `observacion` (antes Marca): nota opcional (no_reconocido, revisar_formato, corte_0830).
- **No se guarda** la hora en que se escribió ("Escrito a las"): `paso = en_bitacora` ya lo dice.
- **Convención de nombres:** columnas y valores en minúsculas, sin espacios ni tildes y con guion bajo (`id_correo`, `solo_registro`), porque el Registro puede pasar a una base de datos.
- **Nombres:** al cuaderno interno de la automatización se le llama **Registro**. La palabra "LOG" queda solo para el correo que manda el RPA cuando falla.
- **Porqué:** el Registro es el punto de control entre etapas y guarda lo mínimo para no releer los correos. Se mantienen `cod_error` y `error` porque son lo que la Etapa 3 escribe y lo que la Etapa 4 copia; así no hay que volver a interpretar el HTML del correo para reintentar, y las filas del corte de las 8:30 (sin correo) quedan completas (cierra P-17).
- **Dónde vive:** en el Excel de las bitácoras (D-37). **Cómo se crean las filas:** una por cada correo que llega; las ausencias solo reciben fila en el corte de las 8:30 (D-36).
- **Pendiente (2026-11-13):** las filas sin correo (`sin_llegar`, `log_esperado`) no tienen `id_correo` y el conector de Excel necesita una llave única para actualizar una fila; ver P-22 en `task.md`.

### D-31 · El consolidado copia siempre el texto del error del correo (resuelve el punto 1 de P-15)
- **Decisión (2026-11-13):** en el consolidado se copia el texto del error tal como viene en el correo, en **todos** los errores, reconocidos o no. Precisa D-23.
- **Porqué:** el texto ya se lee para clasificar el error, así que copiarlo es solo pasarlo, y reduce reglas en la lógica.
- **Cómo:** el texto queda en la columna `error` del Registro y la Etapa 4 lo toma de ahí (D-30).

### D-32 · Estrategia de pruebas: se prueba al implementar, etapa por etapa
- **Decisión (2026-11-13):** primero se termina el diseño de la lógica de las cuatro etapas. Después se **implementa por etapas y cada una se prueba a medida que se construye**. No se hacen ahora pruebas de escritorio ni simulaciones de la lógica.
- **Porqué:** (a) todavía no está el catálogo de errores, así que una prueba hoy sería parcial; (b) el correo no se puede usar por una caída de Microsoft, de modo que no hay correos reales con qué probar; (c) probar cada etapa al construirla permite aislar fallas (se conserva la frontera entre etapas de D-28).
- **Consecuencia:** las pruebas con correos reales listadas en `task.md` (asuntos, conversión del HTML, textos del Motivo, correos reenviados, regla de Outlook) se hacen dentro de la construcción de la etapa que corresponda.

### D-33 · Etapa 2: lógica y flujos aprobados; el catálogo de errores queda pendiente
- **Decisión (2026-11-13):** el usuario confirmó que los tres flujos de la Etapa 2 (A llegada de correo, B verificación del LOG y C corte de las 8:30) le gustan como están y que la lógica y el funcionamiento de la etapa quedan completos. Lo único pendiente es el **catálogo de errores**; cuando esté, los flujos se complementarán para que queden tal cual el funcionamiento. Los flujos "es probable que cambien".
- **Porqué:** el catálogo no cambia la lógica, solo los textos que se comparan; se puede cerrar el diseño y avanzar a las Etapas 3 y 4 sin esperarlo.
- **Documento retirado:** `claude/etapa2-reglas.md` (v0.3) se validó contra este archivo y `task.md`, lo que solo vivía allí pasó a `task.md` (sección 4.1) y se eliminó. La única fuente de la Etapa 2 es este archivo.
- **Versión vigente (2026-11-13):** flujos A, B y C en v0.2 (con las dos reglas básicas de D-34 y los pasos de D-36) y el nuevo **flujo general de la Etapa 2 v0.1** (`flujo2-etapa2-general.md`), que los une en una sola vista más general. El usuario aprobó el estilo de los flujos v0.2 ("me encantaron").

### D-34 · Dos reglas básicas para toda la automatización
- **Decisión (2026-11-13):** toda la automatización, sin importar sus otras reglas, tiene dos reglas básicas que se aplican en cada puerta, en este orden: **(1) si hoy es festivo, no corre, no hace nada: fin; (2) si el consolidado ya se envió (el día ya está cerrado), no se sigue validando nada**. Si algo arrancara sin la primera, o siguiera corriendo después de la segunda, hay que corregirlo.
- **Cómo se sabe que el día está cerrado:** la hoja `Control` (D-35) tiene una fila de hoy con estado `enviando`, `enviado` o `sin_errores`.
- **Excepción:** la **Etapa 1 corre todos los lunes sin depender de festivos** (solo genera la hoja y marca los festivos en gris); no lleva guardas.
- **Dónde se aplican:** puerta de llegada de correo (flujo A), corte de las 8:30 (flujo C), el flujo B hereda las guardas del A, y la Etapa 4 las lleva en cada disparador propio (el de las 8:31 y el de consolidación inmediata). La Etapa 3 hereda las guardas del flujo que la invoca, a menos que tenga su propio disparador (en ese caso, también las lleva).
- **Porqué:** son las dos situaciones en las que nada debe cambiar: festivo (no hay operación) y día cerrado (el consolidado es una foto estable, D-07).
- **Consecuencia:** cierra P-08 (en festivo no se deja rastro en el Registro) y descarta P-19 (D-26).
- **Precisión (2026-11-13, Etapa 3):** la segunda regla (día cerrado) **no aplica a la Etapa 3**, y el usuario indica que **tampoco a la Etapa 4**; ver D-39.

### D-35 · Hoja `Control`: estados que cierran el día; solo la Etapa 4 escribe
- **Decisión (2026-11-13):** `TablaControl` con columnas `fecha`, `estado_consolidado`, `hora_envio` y `ticket`; una fila por día. Estados: **`enviando`** (reserva antes de enviar, evita duplicados), **`enviado`** (el consolidado salió) y **`sin_errores`** (el día no tuvo errores y se cierra sin correo; `hora_envio` queda vacía). Cualquiera de los tres cierra el día (D-34). **Solo la Etapa 4 escribe** estado y hora; las demás etapas solo leen para la regla de día cerrado.
- **`ticket`:** lo llena una persona a mano; **queda fuera del alcance** del proyecto actual registrar la respuesta del consolidado, y eso generaría otras reglas. Confirmado de nuevo el 2026-11-13.
- **Nombre del estado:** el usuario eligió `sin_errores` (propuesta inicial de Claude: `sin_incidentes`).
- **Porqué:** `enviando` evita que dos ejecuciones envíen el mismo consolidado; `sin_errores` hace que un día limpio también quede cerrado y que la automatización deje de validar.
- **Riesgo conocido:** un `enviando` que se quede atascado cerraría el día sin que salga el correo; se mitiga con la alerta de D-19.
- **En revisión:** el usuario preguntó por qué hay tres estados (ver P-20 en `task.md`).

### D-36 · Pasos del Registro (columna `paso`) y creación de filas
- **Decisión (2026-11-13):** valores de `paso` (renombrados por el usuario: pendiente→clasificado, escrito→en_bitacora, en_espera→en_espera_log):
  - **`clasificado`:** la Etapa 2 ya clasificó el correo y creó la fila; falta que la Etapa 3 lo refleje en la bitácora. Lo pone la Etapa 2.
  - **`en_bitacora`:** la Etapa 3 ya lo reflejó en la bitácora. Lo pone la Etapa 3.
  - **`en_espera_log`:** se espera el LOG de un RPA que terminó en error y todavía no llegó; la Etapa 3 lo ignora. Si el LOG llega, la fila pasa a `clasificado` (`log_recibido`); si a las 8:30 no llegó, pasa a `clasificado` como `sin_llegar` con `cod_error` 0.
  - **`solo_registro`:** solo deja rastro; la fila no define lo que muestra la bitácora (un reproceso reemplazado por uno más reciente, o un correo no reconocido). La Etapa 3 la ignora.
- **Descartado:** un paso `consolidado`, porque `Control` ya dice si el día se envió.
- **Cómo se crean las filas:** **una fila por cada correo cuando llega.** Los RPA que no llegan no tienen fila hasta las 8:30: el corte compara la lista de esperados del `Catalogo_RPA` (8 inicio + 8 finalización + 1 variación = 17, más los LOG en `en_espera_log`) con las filas del día y crea una fila `sin_llegar` con `cod_error` 0 por cada faltante.
- **Porqué no crear las 17 filas al inicio:** haría falta otro disparador, una fila por esperado se sobrescribiría y se perdería el rastro de los reprocesos (D-26), y una fila `sin_llegar` antes de las 8:30 diría algo falso (que no llegó es un hecho solo a las 8:30).

### D-37 · El Registro vive en el Excel de las bitácoras; `Festivos` solo lleva la fecha (resuelve P-16)
- **Decisión (2026-11-13):** el Excel donde se generan las bitácoras tiene **4 hojas fijas**: `PLANTILLA`, `Festivos` (tabla con solo la columna `fecha`), `Control` y `Registro`. El `Catalogo_RPA.xlsx` sigue siendo un archivo aparte y de solo lectura (D-27). Se pidió un documento de referencia con esas cuatro hojas.
- **Porqué:** las etapas leen y escriben en un solo archivo (D-14); el catálogo es de solo lectura. La descripción en `Festivos` es innecesaria: solo hace falta la fecha.
- **Entregable de referencia:** `Bitacora_Estructura_Referencia.xlsx` (datos de ejemplo, no reales; `PLANTILLA` con el formato original; listas de validación para `grupo`, `estado`, `observacion`, `paso` y `cod_error`). Ya está subido al proyecto.

### D-38 · Etapas 3 y 4 dependen de la Etapa 2; ejecución separada a las 8:30 y 8:31
- **Decisión (2026-11-13):** las Etapas 3 y 4 **dependen de la Etapa 2** (de lo que quedó en el Registro), no una de la otra. La **Etapa 4** tiene su propio disparador a las **8:31** (1 minuto de retraso sobre el corte) para evitar ejecuciones simultáneas; lleva las mismas dos guardas (D-34) en la forma que se defina en su diseño (ver D-39).
- **Porqué:** el orden entre 3 y 4 no importa porque ambas leen el Registro, que es el punto de control (D-30).
- **Precaución anotada por Claude (a diseñar):** un minuto fijo no garantiza que la Etapa 2 (corte) haya terminado de escribir los 0; la Etapa 4 debería comprobar que el corte terminó, o encadenarse al final del corte (ver P-21 en `task.md`).
- **Precisión (2026-11-13):** el usuario había dicho que "la bitácora también se crea a las 8:30"; la interpretación de Claude era que la Etapa 3 corre a las 8:30. **Esa interpretación queda reemplazada por D-40:** el disparador principal de la Etapa 3 es la llamada desde los flujos A y C; la hoja de la semana la sigue creando la Etapa 1 los lunes (D-12).

### D-39 · Etapa 3 · Guardas: solo festivo; la de día cerrado no aplica
- **Decisión (2026-11-13):** la Etapa 3 lleva solo la guarda de **festivo**; **no** lleva la de día cerrado. El usuario indica que la guarda de día cerrado **tampoco** iría en la Etapa 4. Precisa D-34.
- **Porqué:** con la consolidación inmediata (D-08) el día puede cerrarse hacia las 7:50, y con esa guarda la Etapa 3 no escribiría nada en la bitácora de ese día. La guarda existe para no validar correos nuevos; la Etapa 3 solo refleja lo que ya quedó clasificado.
- **Pendiente:** la Etapa 4 igual necesita comprobar `Control` antes de enviar, porque es su candado contra un doble envío (D-35); cómo se llama y dónde va se define al diseñarla (P-20 y P-22 en `task.md`).

### D-40 · Etapa 3 · Disparador principal: llamada desde los flujos A y C
- **Decisión (2026-11-13):** la Etapa 3 la llama el **flujo A** después de clasificar cada correo y el **flujo C** al terminar el corte (flujo hijo). Cada llamada barre **todas** las filas `clasificado` del día, sin importar quién la llamó. Precisa D-38.
- **Porqué:** quienes usan la bitácora están acostumbrados a verla llenarse para saber cómo va el proceso (el usuario lo confirmó el 2026-11-13).
- **Condición:** depende de que la licencia y las políticas de SETI permitan flujos hijo (que exigen una *solución* de Power Automate); el usuario lo consulta en la reunión del 2026-11-13. Si no es posible, hay plan B (ejecución a hora fija o recurrencia; ver P-23 en `task.md`).

### D-41 · Etapa 3 · Un script de Office Scripts escribe el valor y el color; sin formato condicional
- **Decisión (2026-11-13):** la Etapa 3 usa **un script fijo** de Office Scripts, que recibe como parámetro la lista de celdas a escribir (hoja, columna, fila, estado, código) y escribe el valor y el color. Verde `#92D050` y sin texto para `inicio_recibido`, `exito` y `log_recibido`; rojo `#FF0000` con el código en **letra blanca** para `error` y `sin_llegar` (código 0). La fila del LOG pasa de gris a verde cuando el LOG llega.
- **Porqué:** el conector de Excel de Power Automate no escribe en celdas sueltas, así que el script se necesita de todas formas; que además pinte el color es lo más simple y deja la bitácora como la ve hoy el cliente.
- **Descartado:** el formato condicional en `PLANTILLA`. Un número de estado (1, 2, 3) chocaría con los códigos de error 1 a 18 que se ven en la celda, y el verde necesitaría un texto marcador oculto.
- **Pendiente de construcción:** cómo viaja la lista al script (como arreglo o como texto JSON que el script convierte) y probarlo desde Power Automate (D-32).

### D-42 · Etapa 3 · La fila de la bitácora sale de una columna del catálogo
- **Decisión (2026-11-13):** `TablaRPA` del `Catalogo_RPA.xlsx` gana la columna **`fila_bitacora`**: la fila que ocupa cada asunto en la bitácora (las 20 filas del catálogo corresponden a las 20 de la bitácora: 17 RPA + 3 LOG). `PLANTILLA` es general y la ubicación no cambia entre hojas; solo cambia la hoja. Precisa D-27.
- **Porqué:** es lo más simple por ahora. Condición: si alguien inserta una fila en `PLANTILLA`, hay que actualizar el catálogo.
- **Mejora futura:** encontrar la fila de una forma más robusta (por ejemplo, por nombre completo del RPA) queda anotada, no se hace ahora.

## 3. Decisiones heredadas del doc *Definición y registro* (vigentes)
- Bitácora en Excel en OneDrive, conservando estructura, colores y catálogo de incidentes.
- Festivos de México (lista entregada por el cliente); RPA solo de lunes a viernes; festivo en gris.
- Colores: verde `#92D050` éxito · rojo `#FF0000` + código = error · gris = día inhábil/festivo.
- Hoja de cada semana nombrada con la fecha ISO del lunes (AAAA-MM-DD); crear una semana ya existente no hace nada.
- Catálogo de incidentes con códigos 0–18 (el doc *Definición y registro* aún dice 0–15 y debe actualizarse); error no catalogado = 17 ("se ha detenido").
- Los correos LOG solo llegan cuando hay error en: Cierre Pensiones Bimbo, Cierre Fondo de Ahorro Bimbo y Cierre Bimbo Web.

## 4. Estado actual del proyecto
- **Etapa del plan:** Diseño.
- **Hecho:** flujo general v0.3 (el usuario lo está actualizando a v0.4; ver `task.md`; el diagrama aún incluye la sobrescritura del 0, que ya no aplica, y la hora 8:25). **Flujo de la Etapa 1 aprobado como correcto (2026-11-11).** La única versión vigente es la editada por el usuario, `flujo1-bitacora.md` (raíz del proyecto), que va a GitHub y por eso no incluye puntos abiertos ni datos internos (D-15).
- **Etapa 2 (2026-11-13):** reglas acordadas (D-20 a D-27), flujos A, B y C **aprobados por el usuario** (D-33) y actualizados a v0.2 con las reglas básicas (D-34) y los pasos (D-36). Flujo general de la Etapa 2 v0.1 (`flujo2-etapa2-general.md`) y Excel de referencia (D-37) ya están en el proyecto. Falta el catálogo de errores (P-15), que el usuario subirá cuando pueda revisar el histórico de correos.
- **Etapa 3 (2026-11-13):** lógica **casi cerrada** (D-39 a D-42): solo la guarda de festivo, la llaman los flujos A y C, un script fijo escribe valor y color, y la fila sale de `fila_bitacora` del catálogo. Falta resolver la llave única del Registro (P-22), el respaldo del disparador (P-23), la respuesta sobre licencia y flujos hijo, y confirmar el borrador de lógica de `task.md`. El flujo `.md` se genera cuando el usuario lo pida.
- **Hallazgos del usuario (2026-11-12):** el LOG de Cierre Pensiones nunca llega (solo un hallazgo, no una regla; D-24); todos los asuntos llegan con `RV:` por el reenvío (D-29); los textos de error de los correos no son los de la bitácora (D-23, P-15). El barrido de errores sigue en pausa por la caída global de Microsoft.
- **Estrategia (2026-11-13):** terminar el diseño de las cuatro etapas, luego implementar y probar por etapas (D-32).
- **Siguiente:** cerrar la Etapa 3 y diseñar la **Etapa 4** (ver `task.md`), junto con P-20 (¿tres estados en `Control`?). Se sugirió continuar en un chat nuevo.

## 5. Historial de cambios
| Fecha | Cambio |
|---|---|
| 2026-10-28 | Se crea el archivo. D-01 a D-10 aprobadas en la sesión de diseño del flujo general. |
| 2026-10-28 | v0.2: se retira D-06 (sobrescritura del 0). D-05 queda como revisión única a las 8:25 con rojo + código 0. D-09 pasa a tarea la creación de la tabla de control. |
| 2026-11-11 | v0.3: se agregan D-11 (guarda de festivos), D-12 (hoja nueva por semana desde `PLANTILLA`, se descarta apilar), D-13 (verificar existencia sin duplicar), D-14 (archivo Excel configurable). D-03 se precisa con D-11. Etapa 1 detallada. |
| 2026-11-11 | v0.4: se agregan D-15 (md de flujo sin datos internos por GitHub), D-16 (script nuevo), D-17 (cuenta personal, por ser un proyecto de curso), D-18 (zona horaria Colombia; resuelve P-05), D-19 (alerta de falla por correo y Teams). Etapa 1 aprobada. Se deja una sola copia del md de la Etapa 1. |
| 2026-11-11 | v0.5: el usuario confirma que el corte es a las **8:30** (antes 8:25). Se actualizan D-05, D-08, D-11 y D-18. Resuelve P-07. |
| 2026-11-12 | v0.6: Etapa 2 acordada. Se agregan D-20 (captura), D-21 (asunto completo), D-22 (inicio en verde), D-23 (éxito, error y variación de precios), D-24 (LOG y flujo 2), D-25 (8:30 como disparador y ventana 1:00–8:30), D-26 (tardíos y reprocesos), D-27 (catálogo aparte). Se precisan D-05, D-08 y D-14. Resuelven P-01 y P-02. |
| 2026-11-12 | v0.7: se agrega D-28 (las Etapas 2 y 3 se mantienen separadas; se consideró y descartó la fusión). D-10 se precisa con la convención de círculos naranja/verde en las conexiones entre flujos. Nota en D-24: el LOG de Cierre Pensiones Bimbo nunca ha llegado (P-12). Estado actualizado con el borrador de los tres flujos de la Etapa 2. |
| 2026-11-12 | v0.8: D-17 aclarada (cuenta personal del usuario en SETI, con credenciales SETI). D-21: hallazgo del prefijo `RV:` en todos los asuntos (P-14). D-23: los textos de error de los correos no coinciden con los de la bitácora; si no coinciden, la bitácora queda en 17 y el consolidado copia el texto del correo (P-15). D-27: los textos de `TablaErrores` son provisionales. El usuario hará un barrido de errores. |
| 2026-11-13 | v0.9: se agregan D-29 (prefijos de reenvío se quitan antes de comparar; resuelve P-14), D-30 (hoja `Registro` con la columna Paso como punto de control; resuelve P-09 y P-13) y D-31 (el consolidado copia siempre el texto del error; resuelve el punto 1 de P-15). Se precisan D-21, D-23, D-24 y D-27 (catálogo con los 20 asuntos reales; `RPA origen` solo para LOG, cierra P-11). Estado actualizado. |
| 2026-11-13 | v0.9.1: D-30 con las columnas finales del Registro (10): `Tipo` pasa a `grupo`, `Código` a `cod_error`, `Motivo` a `error` y `Marca` a `observacion`; se mantienen cod_error y error (cierra P-17), se conserva `recibido` y se quita "Escrito a las". Convención de nombres en minúsculas, sin espacios ni tildes, por si pasa a una base de datos. D-26 precisada: "gana el último" usa `recibido` con la llave fecha + rpa + grupo, distinta del `id_correo`. D-31 ya toma el texto de `error`. D-21 y D-26 hablan de "Registro" en vez de "log". |
| 2026-11-13 | v0.10: se agregan D-32 (se prueba al implementar, etapa por etapa) y D-33 (flujos y lógica de la Etapa 2 aprobados; catálogo de errores pendiente; se elimina `etapa2-reglas.md`). Precisiones: D-22 (inicio ausente = 0 y se informa aunque el RPA haya terminado bien), D-25 (se descarta el riesgo de concurrencia: el último RPA llega ~7:45–7:50), D-26 (nota sobre correos tardíos, P-19), D-24 (flujo 2 confirmado, cierra P-10), D-27 (catálogo de errores aún no listo), D-08 (en revisión, P-18). Sección 1: Claude sin acceso a archivos por seguridad. |
| 2026-11-13 | v0.11: se agregan D-34 (dos reglas básicas: festivo y día cerrado; Etapa 1 corre todos los lunes), D-35 (`Control` con `enviando`/`enviado`/`sin_errores`; solo la Etapa 4 escribe; `ticket` manual fuera de alcance), D-36 (pasos del Registro `clasificado`/`en_bitacora`/`en_espera_log`/`solo_registro` y cómo se crean las filas), D-37 (Registro en el Excel de las bitácoras con 4 hojas; `Festivos` solo con fecha; Excel de referencia) y D-38 (Etapas 3 y 4 dependen de la 2; Etapa 4 a las 8:31). D-08 confirmada (cierra P-18 y P-06); D-24 cierra P-12 (no es regla); D-26 cambia (con el día cerrado un correo no se procesa; descarta P-19); D-11 cierra P-08; D-33 con flujos v0.2 y flujo general de la Etapa 2 v0.1. |
| 2026-11-13 | v0.12: Etapa 3 (lógica casi cerrada). Se agregan D-39 (la Etapa 3 solo lleva la guarda de festivo; la de día cerrado no aplica a la 3 ni, según el usuario, a la 4), D-40 (la Etapa 3 la llaman los flujos A y C; depende de la licencia), D-41 (un script fijo de Office Scripts escribe valor y color, letra blanca en rojo, LOG de gris a verde; sin formato condicional) y D-42 (columna `fila_bitacora` en el catálogo; mejora futura: ubicar la fila por nombre). Se precisan D-24, D-27, D-30 (pendiente de llave única, P-22), D-34, D-35 (ticket manual, confirmado) y D-37 (el Excel de referencia ya está en el proyecto). D-38: se reemplaza la interpretación de que la Etapa 3 corre a las 8:30 por D-40. |
