# Laboratorio 5 — Respuestas y retrospectiva

> **Base de las respuestas:** enunciado oficial del Laboratorio 5 (v3, apartados 8–10) y evidencias de la ejecución del 1 de noviembre de 2026. Las cifras marcadas como medidas proceden de los registros aportados. Donde no existe una medición conservada, se indica expresamente: no se han inventado tiempos ni SCN. Los errores simulados no se presentan como incidencias actuales.

## 8. Preguntas de comprobación

### A–B. Punto de partida y banco de medida

**1. Copia lab4-final frente a lab3-final.** En la copia de Lab 4 vimos que los archivos necesarios para la base estaban dentro del volumen: el script indicó «ninguno: el volumen contiene toda la base de datos». Esto indica que la reubicación de los datafiles realizada durante Lab 4 funcionó. El PDF del profesor muestra como contraste un Lab 3 con **cinco datafiles fuera del volumen**, mientras que nuestra copia de Lab 4 registra explícitamente **ninguno**. No conservamos en la información aportada la salida de una copia `lab3-final` propia; por tanto, **el dato de cinco es del ejemplo docente y no se presenta como una medida personal**. Evidencia: `docs/bitacora/lab5/evidencia/script/*_00-backup-lab4.script.log` y copia del Lab 3.

**2. Factor de bloqueo (datos medidos el 01/11/2026).** Se calcula como `bfr = NUM_ROWS / BLOCKS`, redondeado a filas por bloque. En nuestra evidencia, **CLIENTE** tiene **100.000 filas**, **1.517 bloques** y **102 bytes por fila** de media: `100000 / 1517 = 65,92`, es decir, **bfr ≈ 66**. **PEDIDO** tiene **1.000.000 de filas**, **5.320 bloques** y **32 bytes por fila**: `1000000 / 5320 = 187,97`, es decir, **bfr ≈ 188**. Por tanto, PEDIDO almacena **188 / 66 ≈ 2,85 veces** más filas por bloque, principalmente porque sus filas son más cortas. Mis cifras no son exactamente las del ejemplo del PDF: allí CLIENTE tenía 1.467 bloques; aquí son **1.517**. Evidencia: `docs/bitacora/lab5/evidencia/script/20261101T155428Z_01-perfil-tiendasur.script.log`, sección 1.

**Perfil complementario del banco (medida propia, evidencia 01).** `CLIENTE.PAIS` tiene **40 valores distintos**, histograma `FREQUENCY` y `PAIS='ES'` representa **60,1 %**; `CIUDAD='Lorca'` representa **0,1 %** y `APELLIDOS LIKE 'Gonz%'`, **2,1 %**. `PEDIDO.ESTADO` tiene **4 valores distintos**, histograma `FREQUENCY` y el estado `P` representa **2 %** (sin histograma, la estimación uniforme sería **25 %**). El último año representa **33,3 %** de los pedidos. Estos sesgos justifican medir las consultas con el plan real, no decidir un índice solo por su existencia. Evidencia: `docs/bitacora/lab5/evidencia/script/20261101T155428Z_01-perfil-tiendasur.script.log`, secciones 2 y 3.

**3. Dos ejecuciones por consulta.** La primera permite que Oracle analice la sentencia y caliente la caché. Se mide la segunda, para que el coste de análisis y la primera lectura no distorsionen la comparación. La herramienta `LAB_TOOLS.MEDIR` registra filas, lecturas lógicas y físicas, tiempo y plan. Evidencia: `script/*_01-herramienta-medir.script.log` y `scripts/lab5/01-herramienta-medir.sql`.

### C–D. Anatomía y tipos de índice

**4. IX_PED_CLIENTE (medida propia).** En nuestra evidencia de anatomía, el índice contiene **1.000.000 entradas**, **2.216 hojas** y **6 bloques de rama**; su **altura es 3**. El factor de ramificación real es `(hojas + ramas - 1) / ramas = (2216 + 6 - 1) / 6 = 370,17`, es decir **≈ 370 hijos por bloque de rama**. La capacidad máxima estimada es **543** (`8028 / 14,8`). Cada hoja contiene unas **451 entradas**, frente a un máximo de **503**, de modo que la ocupación es aproximadamente **90 %**, coherente con `PCTFREE 10`. La altura comprobada por `INDEX_STATS` coincide con **3**. Si llegáramos a **1.000 millones de filas** de tamaño similar, habría del orden de **2,2 millones de hojas**; el crecimiento logarítmico del B-tree y el fan-out observado llevan a una **altura estimada de 4 niveles**. No se requieren 1.000 veces más niveles al multiplicar las filas por 1.000. Evidencia: `docs/bitacora/lab5/evidencia/script/20261101T155821Z_02-anatomia-indice.script.log`, apartado del resumen `IX_PED_CLIENTE` y líneas «Factor de ramificación real» / «Factor de ramificación máximo».

**5. ROWID de PK_CLIENTE.** Una entrada de hoja guarda la clave más la dirección de la fila: **`data:(6): 00 00 03 1b 00 00`**. Los primeros cuatro bytes (`00 00 03 1b`) representan el bloque **0x031B = 795** y los dos siguientes (`00 00`) identifican la ranura **0**. La clave se muestra como `Typ=2 Len=2: c1,2`, correspondiente a `ID_CLIENTE = 1`. En el tablespace BIGFILE el ROWID corto del índice codifica el bloque (4 bytes) y la ranura de fila (2 bytes), sin un campo independiente para el número de archivo. Evidencia: `config/*_02-volcado-indice.trc.txt` o el log 02.

**6. Tres pruebas de tipos de índice (mediciones propias).** Elegimos las siguientes:

- **Q13 — índice UNIQUE por DNI.** Sin índice se registraron **1.486 lecturas lógicas**, **3,3 ms**, para una fila; con `UNIQUE`, **3 lecturas**, **0,1 ms**. La búsqueda se dirige a la clave y el índice, además, impide duplicados: unas **495 veces menos lecturas**.
- **Q7 — índices por cliente y fecha.** Con `IX_PED_CLIENTE` fueron **15 lecturas**, **0,3 ms** (4 filas); el compuesto `(id_cliente, fecha)` bajó a **7 lecturas**, **0,2 ms**; el cubriente quedó en **3 lecturas**, **0,1 ms**. El compuesto ayuda a filtrar y ordenar por fecha, y el cubriente puede satisfacer la proyección sin accesos adicionales a los bloques de la tabla: **5 veces menos lecturas** respecto a la primera variante.
- **Q1 — índice basado en función para `UPPER(email)`.** Sin índice funcional fueron **1.486 lecturas**, **15,9 ms** para una fila; con él, **4 lecturas**, **0,1 ms**: unas **372 veces menos lecturas**. La expresión de la consulta impide aprovechar directamente un índice ordinario sobre `email`, mientras que el índice funcional guarda valores comparables a `UPPER(email)`. Hay que valorar su coste de mantenimiento frente a normalizar el dato o reescribir la consulta.

**Contraste adicional Q12, no incluido entre las tres pruebas elegidas:** índice `(estado, fecha)` **246 lecturas y 18,6 ms**, frente a índice parcial **239 lecturas y 20,2 ms** (20.000 filas). Aunque el índice parcial realizó algo menos de lecturas, **no fue más rápido en esta ejecución**. Por tanto, la decisión no se puede basar solo en un único tiempo. Todas las pruebas citadas registraron **0 lecturas físicas**. Evidencia: `docs/bitacora/lab5/evidencia/script/20261101T160006Z_03-tipos-indice.script.log`, líneas Q13, Q7, Q1 y Q12.

### E–F. Agrupamiento, equilibrio y sargabilidad

**7. Clustering factor (CF, resultados propios).** La tabla `PEDIDO` tiene **5.320 bloques** y **1.000.000 de filas**. Los índices **`PK_PEDIDO`** e **`IX_PED_FECHA`** tienen **CF = 5.220**, cercano al número de bloques: al recorrer las claves, se accede a bloques de tabla próximos entre sí. En cambio, **`IX_PED_IMPORTE`** tiene **CF = 998.078** e **`IX_PED_CLIENTE`** **CF = 999.041**, casi tantos como filas: sus ROWID están dispersos y el recorrido provoca numerosos saltos entre bloques. El cálculo manual del experimento confirmó **5.220** para `IX_PED_FECHA` y **999.041** para `IX_PED_CLIENTE`. El CF mide los **cambios de bloque al seguir el orden del índice**; no equivale necesariamente al número de bloques distintos que existen en la tabla. Un `REBUILD` del índice no reordena físicamente las filas de la tabla y, por sí solo, no corrige ese factor. Evidencia: `docs/bitacora/lab5/evidencia/script/20261101T160600Z_04-agrupamiento-equilibrio.script.log`, secciones 1 y 2.

**8. Punto de equilibrio (curva medida).** Para una tabla de **1.000.000 filas** y **5.320 bloques** (`bfr ≈ 188`), el recorrido completo medido consumió **5.232 lecturas lógicas**. Con **0,1 %** de filas (1.000), el índice con **CF malo** necesitó **975 lecturas**, frente a **11** del índice con CF bueno. Al **0,2 %** (2.000 filas), el índice malo aún registró **2.004** lecturas, menos que las **5.232** del full scan, pero el **optimizador ya eligió recorrido completo**: su decisión incorpora otros costes, no solo las lecturas lógicas observadas. Al **0,5 %** (5.000 filas), el índice malo consumió **4.875** lecturas; al **1 %**, **9.920**, superando ya las **5.232** de la tabla completa. El índice con CF bueno, en cambio, tuvo **3.938** lecturas al **50 %** y solo superó el full scan al **100 %** (**7.875** frente a **5.232**). Como orientación teórica (Unidad 3, 3.1.8), el índice con mal CF compensa si `k/r < (1/bfr) × (c_sec/c_aleat)`: el umbral es **≈ 0,53 %** con costes lógicos iguales y **≈ 0,11 %** si una lectura secuencial cuesta una quinta parte de una aleatoria. El cambio observado de plan a **0,2 %** es coherente con que el optimizador estime costes físicos y no solamente buffers. Evidencia: `docs/bitacora/lab5/evidencia/script/20261101T160600Z_04-agrupamiento-equilibrio.script.log`, sección 3.

**9. Q4b (`LIKE 'Gonz%'`) y el coste pese a usar el índice (medición propia).** El B-tree está ordenado por el **principio** de la cadena, por lo que un prefijo como `'Gonz%'` delimita un rango de claves y el índice puede utilizarse; `LIKE '%ez'` no tiene un prefijo conocido y no puede hacer esa misma búsqueda por rango. En nuestro experimento, **Q4b** devolvió **2.056 filas**, realizó **2.024 lecturas lógicas**, **0 físicas** y tardó **4,4 ms**. La prueba **Q4a**, con `LIKE '%ez'`, hizo un **recorrido completo**, obtuvo **18.902 filas**, consumió **1.523 lecturas lógicas**, **0 físicas** y tardó **26,7 ms**. Por tanto, **Q4b leyó más buffers (2.024 > 1.523)** aun usando índice: localizar ROWID mediante el índice y visitar bloques de tabla dispersos puede sumar más visitas a buffers que recorrer secuencialmente una tabla. **Matiz importante:** Q4a y Q4b tienen filtros y números de filas distintos; la comparación ilustra lecturas de acceso, no demuestra que sustituir una consulta por la otra produzca el mismo resultado ni que el índice sea más lento en tiempo. De hecho, Q4b terminó antes. En la misma evidencia, Q1 reescrita bajó de **1.486 a 4 lecturas**, Q2 de **5.284 a 320**, Q3 de **5.232 a 106** y Q5 al añadir índices de **1.486 a 6**; Q11, aunque se reescribió, mantuvo **5.232 lecturas** y el full scan, porque el rango era poco selectivo. Evidencia: `docs/bitacora/lab5/evidencia/script/20261101T160725Z_05-sargabilidad.script.log`.

### G–H. Escritura y mantenimiento

**10. Coste de los índices: 4, 9 y 5 sobre PEDIDO (mediciones propias, evidencia 06).** Se midieron **2.000 INSERT** por escenario:

| Escenario | Índices en PEDIDO | Bloques modificados / 2.000 INSERT | Bloques modificados por fila | REDO total INSERT | Tiempo INSERT |
|---|---:|---:|---:|---:|---:|
| Diseño inicial (`hoy`) | 4 | 20.331 | **10,2** | 2.605 KB | 76,6 ms |
| Propuesta del compañero | 9 | 40.355 | **20,2** | 5.033 KB | 128,5 ms |
| Migración final V005 | 5 | 24.214 | **12,1** | 3.144 KB | 84,5 ms |

La propuesta duplicó aproximadamente los bloques modificados por INSERT respecto al punto de partida (**40.355 frente a 20.331**) y casi duplicó el REDO. También hizo más caro el DELETE (de **2.437 KB** a **4.739 KB** de REDO). El diseño V005 bajó el coste de INSERT respecto a la propuesta (**3.144 KB** de REDO) y conservó los índices útiles para la carga, aunque **no mejora todas las operaciones**: `UPDATE estado` costó **515 KB** de REDO en el diseño inicial, **1.417 KB** con el compañero y **2.020 KB** con V005, por el mantenimiento del índice parcial de pendientes. Para `UPDATE importe` fueron **1.475 / 1.474 / 1.586 KB** y para `DELETE` **2.437 / 4.739 / 2.494 KB** respectivamente.

**Decisiones de la propuesta de cinco índices del compañero, según el protocolo y la migración del profesor:**

- `IX_PED_ESTADO`: no se mantuvo como índice general porque `ESTADO` tiene **4 valores** (media **25 % por valor**) y aun `'P'` representa **2 %**, por encima del umbral de referencia para un índice con mal CF (**≈ 0,5 %**). V005 prefiere un índice **parcial** para el estado pendiente.
- `IX_PED_CANAL`: no se mantuvo porque tiene solo **3 valores** (**≈ 33,3 %** por valor); demasiado poco selectivo.
- `IX_PED_CLI_FEC`: no se conservó tal cual; V005 optó por el **cubriente** `IX_PED_CLI_FEC_CUB`, que empieza por `id_cliente`, sirve a Q7 y sustituye también `IX_PED_CLIENTE`, redundante por prefijo.
- `IX_PED_FEC_CLI`: no se mantuvo; para el historial de un cliente es más útil comenzar por `id_cliente`, y las consultas de fecha ya tienen `IX_PED_FECHA`.
- `IX_PED_CP`: **sí se conservó** por la consulta Q3, después de reescribir el filtro de código postal para aprovechar el índice; el protocolo indica una selección aproximada del **0,01 %**.

Adicionalmente, V005 retiró `IX_PED_IMPORTE` tras la prueba `INVISIBLE` y la observación de uso realizada en el laboratorio. Esto no implica que deba borrarse sin más en producción: habría que observar una ventana suficiente para contemplar consultas poco frecuentes. **Evidencias:** `docs/bitacora/lab5/evidencia/script/20261101T160951Z_06-indices.script.log`; diseño aplicado por `database/migrations/V005__indices_tiendasur.sql`; protocolo `scripts/lab5/06-protocolo.sql` del material oficial.

**11. Divisiones y patrón cola (medidas propias).** Con **200.000 inserciones por tabla**, las claves aleatorias provocaron **595 divisiones de hoja y 0 del tipo 90/10**; las claves crecientes, **398 divisiones, las 398 del tipo 90/10**; el índice `REVERSE` sobre la secuencia registró **511 divisiones y 0 del tipo 90/10**. El mapa de `IX_AZAR` tenía una ocupación desigual, mientras `IX_CRECIENTE` quedó prácticamente uniforme y lleno (`[################################################################]`). El **patrón cola** surgió al insertar por la derecha y borrar por la izquierda: su mapa antes de reparar empezaba con muchos espacios en blanco y terminaba con `############################`. Tras los remedios el mapa mostró **64 tramos uniformes de `=`** (`[================================================================]`). Son situaciones diferentes: las divisiones por inserción no significan necesariamente fragmentación dañina; la cola concentra huecos antiguos y hojas llenas recientes. Evidencias: `script/20261101T161156Z_07a-crecimiento.script.log`, `script/*_07a-grafico*`, `script/*_07b-grafico*` y `script/*_07c-grafico*`.

**12. Remedio en producción (medida propia).** Primero mediría consultas y ocupación. Para un patrón cola que esté penalizando el servicio escogería **`COALESCE`**, pues reorganiza hojas vecinas sin reconstruir por completo el índice y mantiene el acceso; en nuestro ensayo quedó en **201 hojas, 0 entradas borradas, segmento de 4 MB y 2 lecturas** para el siguiente mensaje. **`SHRINK SPACE`** mantuvo **201 hojas y 2 lecturas**, pero devolvió espacio: el segmento bajó a **1,69 MB**. **`REBUILD ONLINE`** creó **223 hojas**, ocupó **2 MB** y mantuvo las **2 lecturas**, sin mejorar la consulta medida; exige más espacio temporal y puede generar mucho redo. Para prevenir la recurrencia estudiaría particionar por antigüedad y depurar particiones. No reconstruiría un índice sin un beneficio medido. Evidencia: `script/20261101T161156Z_07c-remedios.script.log`.

### I–L. SCN, control files, redo y FRA

**13. SCN al abrir.** Oracle compara el SCN/checkpoint registrado en los control files con el SCN de cabecera de los datafiles. Si no son consistentes o están retrasados, hace falta aplicar redo para recuperar. En nuestro registro 08 se consultan las columnas **`CKPT_EN_CONTROL_FILE`**, **`CKPT_CONTROL_FILE`** y **`CKPT_CABECERA`** antes y después de forzar el checkpoint; el resumen exportado no incluye sus cifras SCN, por lo que **no se indica un número no verificado**. Evidencia: `script/*_08-scn-checkpoint.script.log`.

**14. Control files.** Tener tres copias permite sobrevivir a la pérdida o corrupción de un archivo individual y recuperarlo copiando otro sano. Pero si los tres están en el mismo volumen, una pérdida completa del disco inutilizaría todas las copias. En producción las pondría en discos/almacenamientos independientes. En el simulacro B recuperamos `control03.ctl` y la base volvió a `READWRITE/READWRITE`. Evidencia: `script/*_09-multiplexar-controlfile.script.log`, `script/*_14b-perdida-controlfile.script.log`.

**15. Redo CURRENT/ACTIVE/INACTIVE.** `CURRENT` es el grupo que LGWR está escribiendo ahora. `ACTIVE` contiene redo que todavía puede hacer falta para recuperar la instancia. `INACTIVE` ya no se necesita para esa recuperación. Oracle rechazó borrar el grupo CURRENT (`ORA-01623`), un miembro del grupo CURRENT (`ORA-01609`) y otro grupo aún necesario sin archivar (`ORA-00350`), porque comprometerían la recuperación. Evidencia: `script/*_10-redo.script.log`.

**16. FRA llena.** `ORA-19815` avisó del espacio casi agotado; al no poder guardar más archived redo aparecieron problemas como `ORA-19809` y errores del archivador (por ejemplo `ORA-16038`). Sin poder archivar los logs, llega la espera `log file switch (archiving needed)` y las transacciones se quedan bloqueadas aunque la base esté abierta. El script recuperó la FRA hasta 4 GB configurados. Evidencia: `script/*_11-fra-llena.script.log`.

### M–P. Undo, WAL y simulacros

**17. Lectores y escritores.** Oracle conserva versiones anteriores en undo, para que una consulta iniciada antes de un cambio lea una imagen consistente. La sesión lectora no necesita bloquear al escritor y el escritor puede confirmar; después, una consulta nueva ve el cambio ya confirmado. Este comportamiento corresponde al experimento de dos sesiones que describe el profesor. **No hemos aportado un registro completo de las sesiones 1–4**: el log disponible `12-undo-y-redo` documenta el coste de undo y redo, no demuestra por sí solo las dos sesiones. Se explica el mecanismo sin atribuirle cifras inventadas. Evidencia: `script/*_12-lectura-consistente*` o `spool/*_12-*`.

**18. DELETE, INSERT y ROLLBACK.** En nuestra prueba de 10.000 filas, el INSERT usó **140 bloques de undo (1120 KB)**, UPDATE **72 (576 KB)** y DELETE **427 (3416 KB)**. DELETE necesita conservar suficientes datos de la fila y sus entradas de índice para deshacer la eliminación; INSERT necesita principalmente poder borrar la fila recién añadida. El propio ROLLBACK cambia bloques, y por eso también genera redo para proteger ese deshacer. Evidencia: `script/20261101T163309Z_12-undo-y-redo.script.log`.

**19. Durabilidad COMMIT/checkpoint.** En el COMMIT LGWR garantiza en el redo online los cambios confirmados, aunque DBWn no haya escrito todavía todos los bloques de tabla. En el checkpoint DBWn vuelca bloques sucios a datafiles y CKPT actualiza información de checkpoint. En nuestra captura, el UPDATE+COMMIT aumentó el redo en **1779 KB** (estadística de instancia) y el checkpoint reportó **349 bloques escritos por DBWn**; las cifras incluyen trabajo de fondo. Evidencia: `script/20261101T163453Z_13-lgwr-dbwn.script.log`.

**20. Simulacro A.** La fila 102 se modificó y se confirmó; la 103 quedó modificada sin COMMIT. Tras `SHUTDOWN ABORT` y `STARTUP`, Oracle rehízo el redo (*roll forward*) para recuperar cambios y deshizo la transacción incompleta (*roll back*). Resultado: 102 conservó el cambio y 103 volvió al valor anterior; al final el script repuso la 102 para repetir el laboratorio. El *alert log* capturado registra `Beginning crash recovery of 1 threads`, `Recovery starting at checkpoint ... scn 4296692`, **17 KB de redo leídos y 20 bloques de datos que requerían recuperación**, seguido de `Completed crash recovery`. No se cronometró el tiempo total de arranque. Evidencia: `script/*_14a-caida-instancia.script.log`.

**21. Simulacros C y D (cifras propias).** En el C, el **RTO fue 74 segundos** y la ventana potencial de pérdida hasta la copia previa fue **2 segundos**; se restauró toda la base y reaparecieron las **60.000 matrículas**. En D, el **RTO de recuperación fue 10 segundos (11 desde el fallo)** y el **RPO real fue 0**: el estudiante 104 volvió a `EGRESADO` porque RMAN aplicó redo posterior a la copia. El D fue mucho más rápido porque solo se recuperó el datafile afectado, no el volumen completo. La PDB pasó por `MOUNTED` y volvió a `READ WRITE`. Evidencias: `script/20261101T164231Z_14c-simulacro-restauracion.script.log` y `script/20261101T165657Z_14d-perdida-datafile.script.log`.

**22. Clasificación de los errores del bloque 5 del healthcheck (datos propios).** El registro del 1/11/2026 contiene:

| Mensaje observado | Clasificación | Motivo y acción |
|---|---|---|
| `ORA-00202`, y referencia abreviada `ORA-205 signalled during ... MOUNT` (16:40) | **Efecto esperado del simulacro B** | Se eliminó `control03.ctl` para comprobar el fallo de montaje (`ORA-00205`), y después se repuso desde otro control file sano. |
| `ORA-01113` (dos entradas, 16:57) | **Efecto del simulacro D** | El datafile borrado necesitaba recuperación de medios; se aplicó `RESTORE`/`RECOVER` con RMAN. |
| `ORA-01110` (16:57), archivo `academia01.dbf` | **Efecto del simulacro D** | Identifica el archivo implicado. Se restauró y quedó accesible. |
| `ORA-63999` (16:57) | **Efecto del simulacro D** | Oracle detectó un fallo de medio y pudo cerrar `FREEPDB1`; el script contempló ese caso y la reabrió. |
| `ORA-00800` sobre `LGWR` y `VKTM` (16:46–16:47) | **Aviso del entorno a investigar / posible ruido conocido** | Mensaje `Set Priority Failed` de procesos de fondo. No demuestra por sí mismo pérdida de datos, pero tampoco debemos declararlo irrelevante sin revisar las trazas y los privilegios del contenedor. |

No consta en este extracto un fallo de recuperación **pendiente**. El *healthcheck* terminó con código **0**, se observó una copia RMAN `COMPLETED` y el control de `ORA-01555` dio **0**. En el archivo de texto exportado aparecen además los avisos SQL*Plus **`SP2-0310` y `SP2-0768`**: no son ORA- ni forman parte del bloque 5, pero deberían revisarse en el log completo antes de considerar el runbook completamente limpio. Evidencia: `script/20261101T170619Z_15-healthcheck.script.log` y extracto `Lab5_datos_final.txt`.

## 9. Simulacro de examen 1Z0-182 (S1–S26)

> Respuestas razonadas a partir del material docente. Donde se indican dos letras, el enunciado pide «elige dos». Adjuntar las evidencias/parte si la rúbrica lo requiere.

| Pregunta | Respuesta | Parte | Justificación breve |
|---|---|---|---|
| S1 | **B** | E | CF cercano a filas indica muchas visitas a bloques distintos al recorrer el índice. |
| S2 | **A, C** | D, F | Índice funcional sobre UPPER o reescribir el predicado si el dato se guarda normalizado. |
| S3 | **B** | F | La comparación numérica provoca TO_NUMBER sobre la columna VARCHAR2 indexada. |
| S4 | **B** | D | Bitmap es adecuado en almacenes con baja cardinalidad y pocas escrituras concurrentes. |
| S5 | **B** | D | INDEX SKIP SCAN permite explorar por la segunda columna si la primera tiene pocos valores. |
| S6 | **B** | G | INVISIBLE permite probar sin uso por el optimizador y revertir inmediatamente. |
| S7 | **A, C** | H | COALESCE y SHRINK SPACE permiten compactar/recuperar hojas sin reconstrucción convencional que bloquee. |
| S8 | **B** | H | REVERSE reparte las inserciones, pero rompe el orden útil para búsquedas por rango. |
| S9 | **C** | G | UPDATE de columna no indexada no necesita modificar las entradas de esos índices. |
| S10 | **B** | C | La altura depende en gran medida del fan-out y del tamaño de las claves. |
| S11 | **A, C** | D | En IOT las filas viven dentro de la estructura de la PK y no hay salto a otra tabla para buscar por PK. |
| S12 | **B** | I | Se comparan los SCN de checkpoint del control file y las cabeceras de datafiles. |
| S13 | **C** | I, N | CKPT actualiza información del checkpoint; DBWn escribe bloques sucios. |
| S14 | **B, E** | J | En NOMOUNT están disponibles V$INSTANCE y V$PARAMETER; aún no se ha montado el control file. |
| S15 | **B** | L | Para activar ARCHIVELOG: parar, montar, cambiar el modo y abrir. |
| S16 | **B** | K | CLEAR LOGFILE recrea un grupo INACTIVE archivado sin tener que recuperar toda la BD. |
| S17 | **A, C** | M | Undo permite lectura consistente y consultas flashback AS OF. |
| S18 | **B** | M | También hay que poder rehacer los bloques de undo tras una caída, antes de deshacer transacciones sin commit. |
| S19 | **A** | N | Commit complete garantiza el redo online persistido, no los datafiles ni el archivado. |
| S20 | **A** | I, N | Un MTTR objetivo más bajo exige checkpoints incrementales más intensos/frecuentes. |
| S21 | **B** | L | Si ARCn no puede archivar por una FRA llena, el switch queda esperando. |
| S22 | **B** | O | Se aísla el datafile de aplicación con OFFLINE y se recupera sin parar el resto. |
| S23 | **B** | O | Para SYSTEM se necesita cerrar, montar, restaurar, recuperar y abrir. |
| S24 | **B** | O | Sin redo archivado se recupera desde copia en frío, perdiendo lo posterior. |
| S25 | **B, D** | M, P | Se amplía undo y retención; RETENTION GUARANTEE puede proteger consultas críticas, vigilando el impacto en DML. |
| S26 | **A** | L, P | V$RECOVERY_AREA_USAGE separa el consumo de FRA por tipo de archivo. |

## Tabla de los cuatro simulacros (Parte O)

| Simulacro | Resultado y alcance | RPO observado | RTO propio |
|---|---|---|---|
| A — caída de instancia | Se conserva el COMMIT de la fila 102 y se revierte la 103 | 0 datos confirmados perdidos | **No cronometrado en la evidencia disponible** |
| B — pérdida de control03.ctl | Se recupera la copia del control file y CDB/PDB abren | 0 datos perdidos | **No cronometrado en la evidencia disponible** |
| C — DROP TABLE PURGE | Se restaura toda la copia; 60.000 matrículas | Ventana de **2 s** desde copia; no equivale a pérdida probada de registros | **74 s** |
| D — pérdida de datafile | RMAN RESTORE+RECOVER; fila 104 conservada; PDB reabierta | **0** | **10 s** de recuperación; **11 s** desde fallo |

> **Límite de medición:** no se dispone del RTO numérico de A y B en las evidencias aportadas. No se repitieron simulacros destructivos solo para rellenar la tabla. El RPO de A y B se expresa como ausencia observada de pérdida de transacciones confirmadas. El RPO potencial de C refleja la separación entre la copia y el fallo, no una pérdida efectiva de dos segundos de operaciones.

## Retrospectiva de los simulacros

- **A.** La instancia se recuperó automáticamente; un COMMIT sobrevivió y una transacción sin COMMIT se deshizo. Mejoraría el registro automático del tiempo de arranque y de los mensajes de crash recovery.
- **B.** La multiplexación permitió reponer `control03.ctl`, pero las copias dentro de un mismo volumen no protegen ante pérdida del disco. Mejoraría la distribución de copias entre almacenamientos diferentes.
- **C.** Recuperar una sola tabla con `PURGE` obligó en el ejercicio a restaurar todo el volumen: RTO **74 s**. Antes de una restauración global en producción valoraría recuperación de tabla a un instante anterior mediante RMAN, evitando revertir el resto de datos.
- **D.** El datafile se restauró en **10 s**, con **RPO 0**, usando redo archivado; Oracle cerró temporalmente `FREEPDB1` y la dejamos nuevamente en `READ WRITE`. Mejoraría el runbook para contemplar ambos estados (abierta o montada).

### Retrospectiva adicional: FRA llena (Parte L)

- **Impacto:** bloqueo de escrituras esperando `log file switch (archiving needed)`; no se verificó pérdida de datos confirmados.
- **Causa:** la FRA quedó sin espacio para archivar redo durante el experimento.
- **Detección:** alerta `ORA-19815` y errores posteriores del archivador; consulta del estado de archivado/FRA.
- **Resolución en laboratorio:** restaurar el tamaño de la FRA a **4 GB** y liberar archivados mediante el procedimiento de laboratorio. **En producción no se deben borrar archivados indiscriminadamente sin copias y política de retención.**
- **Mejora:** alarma preventiva a partir del 80 % de ocupación de la FRA; revisar capacidad y copias antes de volver a generar cargas grandes.

## Notas de trazabilidad y control antes de la entrega

- **Datos no verificados:** salida real de la copia `lab3-final`; valores SCN numéricos del experimento 08; traza de las dos sesiones de lectura consistente; RTO de los simulacros A y B. El profesor puede penalizar esas ausencias; no deben sustituirse por ejemplos de su PDF.
- **Alertas adicionales:** comprobar `SP2-0310` y `SP2-0768` en el log completo del healthcheck y decidir si requieren corrección.
- **Seguridad (Parte Q.2):** comprobar el contenido *y el historial* de Git antes de subir, excluyendo `config/.env`, `backups/`, datafiles y ficheros de control, archivados, RMAN y trazas.
- **GitHub:** incluir este archivo y `docs/runbooks/perdida-datafile.md` junto a scripts, V004/V005 y evidencias; abrir PR del Issue #7, revisión y merge, después crear `lab5-final`.

Las referencias `script/*` señalan archivos existentes dentro de `docs/bitacora/lab5/evidencia/`; se han conservado los nombres concretos cuando estaban disponibles.
