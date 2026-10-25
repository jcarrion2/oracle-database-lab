# Laboratorio 4 — Respuestas de comprobación y repaso

> Respuestas basadas en el enunciado del **Laboratorio 4 — Oracle DBA Workflow I (versión 3)** y en las pruebas ejecutadas el 25/10/2026. He utilizado las medidas reales de los logs; donde falta una medición, lo indico expresamente.

**Fuentes de evidencia:** `docs/bitacora/lab4/evidencia/script/`, `.../spool/`, `.../config/`, y respaldo local `backups/20261025T170825Z_lab3-final/`. Los ficheros llevan marca de tiempo; se cita el sufijo en cada respuesta para localizar el nombre exacto.

## 1. Preguntas de comprobación (1–38)

### 1. ¿Por qué la copia de la Parte A es consistente? ¿Qué pasaría si se tomara con la base abierta?

La copia es consistente porque antes se hizo `SHUTDOWN IMMEDIATE`, se cerró la base correctamente y después se copiaron los archivos con Oracle parado. Si los copiáramos con la base abierta y sin un procedimiento de backup en caliente, podríamos guardar archivos de momentos distintos y necesitar recuperación. **Evidencia:** `00-backup-lab3.script.log`, apartado «Parada limpia».

### 2. ¿Para qué sirven manifiesto.txt y SHA256SUMS? ¿Qué comprueba restaurar-estado.sh antes de tocar nada?

`manifiesto.txt` deja constancia de qué archivos había y dónde estaban. `SHA256SUMS` permite detectar si una copia se ha alterado o corrompido. En mi caso `sha256sum -c SHA256SUMS` devolvió cuatro `OK` y código `0`. Antes de restaurar, el script debe comprobar que el respaldo es válido y que puede recrearse el entorno, para no borrar primero la base y descubrir después que no hay copia utilizable. **Evidencia:** `00-backup-lab3.script.log`, carpeta local de backup, verificación SHA-256.

### 3. Define RPO y RTO con los datos de tu propia copia.

**RPO** es la cantidad máxima de datos que aceptaríamos perder: con esta copia puntual, serían los cambios posteriores al respaldo del 25/10/2026 a las 17:08:25 UTC, si no tuviéramos otra recuperación. **RTO** es cuánto tardamos en recuperar el servicio. No medí una restauración real, así que no debo inventar un RTO; tendría que cronometrar `restaurar-estado.sh` hasta volver a `healthy`.

### 4. ¿Dónde creó Oracle los datafiles de V000 y por qué? ¿Qué parámetro habría evitado el problema?

V000 usaba una ruta relativa y Oracle resolvió los datafiles inicialmente bajo `/opt/oracle/product/26ai/dbhomeFree/dbs/` (ruta enlazada hacia `/opt/oracle/oradata/dbconfig/FREE/dbs/`), en vez de la carpeta normal de datos. Fijar `DB_CREATE_FILE_DEST` a una ruta del volumen, o usar rutas absolutas correctas, habría evitado esa ambigüedad. **Evidencia:** `01-diagnostico-inicial` y `03-reubicar-datafiles`.

### 5. ¿Por qué ALTER TABLESPACE ... RENAME DATAFILE no basta para mover un archivo?

`RENAME DATAFILE` cambia en el control file la ruta que Oracle espera encontrar; **no copia físicamente el archivo**. Por eso el procedimiento del profesor deja el tablespace fuera de línea, copia conservando propietarios/permisos, actualiza la ruta y comprueba que vuelve a `ONLINE` antes de retirar el original. **Evidencia:** `03-reubicar-datafiles.script.log`.

### 6. ¿Por qué no se edita V000 y se crea V003? ¿Por qué invalidar una contraseña es mejor que borrarla del archivo?

Una migración aplicada no se reescribe porque perderíamos la trazabilidad y distintos entornos podrían tener «la misma versión» con contenido diferente. V003 hace un cambio nuevo y verificable con `ALTER USER ... NO AUTHENTICATION`. Quitar la contraseña del archivo original no elimina la autenticación que Oracle ya había configurado ni borra el secreto del historial Git. **Evidencia:** log de V003: cinco cuentas `AUTHENTICATION_TYPE=NONE`.

### 7. ¿Qué demuestra la prueba de persistencia y por qué se ejecuta después de reubicar?

La prueba borró **el contenedor**, no el volumen, y lo recreó. Tras ello siguieron los cinco tablespaces `ONLINE`, las 14 tablas y los 20.000 estudiantes; el 101 continuó `ACTIVO`. Hay que mover los datafiles antes: si estuvieran fuera del volumen persistente, se perderían al eliminar el contenedor. **Evidencia:** `03-persistencia.script.log`.

### 8. Con tus evidencias: tres datos que pertenecen a la instancia y tres que pertenecen a la base de datos.

**Instancia:** estado de arranque, `STARTUP_TIME`, memoria SGA/PGA y procesos de background. **Base de datos:** `DBID=1509902213`, nombre `FREE`, fecha de creación `2026-07-05` y archivos físicos. El estado/memoria cambia al reiniciar la instancia; el DBID y los archivos identifican la base persistente. **Evidencia:** `05-instancia-base.spool.log`.

### 9. ¿Qué lee Oracle en NOMOUNT, en MOUNT y en OPEN? Relaciónalo con ORA-01507 y ORA- 01219.

En **NOMOUNT**, Oracle lee los parámetros (SPFILE/PFILE) y crea memoria/procesos; aún no usa el control file (`ORA-01507` si se pide información que exige montaje). En **MOUNT**, lee los control files y conoce los datafiles/redo, pero las tablas no están disponibles (`ORA-01219`). En **OPEN**, abre y recupera cuando hace falta los datafiles y redo y permite las consultas normales. **Evidencia:** `06-arranque-etapas.spool.log`.

### 10. Diferencia entre v$parameter y v$spparameter con el ejemplo de db_files de tu evidencia.

`V$PARAMETER` refleja los valores efectivos de la instancia ahora; `V$SPPARAMETER` muestra lo persistido en el SPFILE para arrancar. En el experimento de `db_files` se comprobó el cambio propuesto en SPFILE y se dejó sin línea pendiente (`FALSE`), como confirmó la verificación final. **Evidencia:** `07-parametros-lectura`, `07-parametros-cambios`, `16-verificacion-final`.

### 11. ¿Por qué PROCESSES rechaza el cambio en memoria aunque v$parameter lo marque IMMEDIATE?

`PROCESSES` es un parámetro estático: cambiarlo exige reiniciar la instancia, por eso el intento con `SCOPE=MEMORY` falla aunque la descripción que devuelve alguna vista sea confusa. Tampoco se cambia desde una PDB (`ORA-65040`). **Evidencia:** `07-parametros-cambios`.

### 12. ¿Qué modelo de gestión de memoria usa tu instancia? Justifícalo con tres parámetros.

Usamos **ASMM**, no AMM: el ajuste mostrado fue `SGA_TARGET=1536 MB`, `PGA_AGGREGATE_TARGET=512 MB` y `MEMORY_TARGET=0`. Oracle redistribuye los componentes de la SGA dentro de su objetivo, mientras la PGA se controla por separado. **Evidencia:** `07-parametros-lectura` y `08-memoria`.

### 13. Explica las cifras de tu experimento de caché: lecturas lógicas, físicas y tasa de acierto.

Primera ejecución, caché fría: 17.600 filas activas, **416 lecturas lógicas y 268 físicas**. Segunda, caché caliente: las mismas 17.600 filas, **247 lecturas lógicas y 0 físicas**. Tener bloques en memoria evita volver al disco; en el muestreo quedaron 245 bloques de la tabla en la caché. **Evidencia:** `08-experimento-cache.spool.log`.

### 14. Compara literales y variables de enlace (bind variables) con tus cifras de análisis (parse) y de V$SQL.

En la prueba se hicieron 1.000 consultas cambiando el literal y otras 1.000 usando `:id`. **Con literales** se registraron 1.044 análisis en total, **1.005 duros**, 840 ms de CPU de análisis y 910 ms de tiempo total. **Con la variable de enlace** hubo solamente **1 análisis, también duro**, 0 ms de CPU de análisis (redondeado) y 10 ms de tiempo total. En `V$SQL` quedaron **679 cursores y 679 ejecuciones con literales (24.993 KB de shared pool)**, frente a **1 cursor con 1.000 ejecuciones y 39 KB** usando `:id`. No aparecen 1.000 cursores con literales porque algunos fueron expulsados de la memoria. Con `bind` se reutiliza el cursor y se reduce mucho el trabajo de análisis. **Evidencia:** `20261025T175356Z_08-parse-y-pga.spool.log`, sección 1.

### 15. Relaciona el SPID de LGWR de v$process con el PID de /proc. ¿Por qué docker top muestra otro número?

En `V$PROCESS` el proceso `LGWR` aparece con **SPID 961**. La comprobación de procesos desde Linux dentro del contenedor también lo muestra como **PID 961** (`db_lgwr_FREE`), de modo que ambos datos corresponden al mismo proceso en ese espacio de nombres. `docker top` puede mostrar otro PID porque consulta los procesos desde el espacio de nombres del host Docker, y el contenedor tiene su propia numeración. **No se capturó el PID concreto del host**, así que no puedo poner una segunda cifra real. **Evidencias:** `20261025T175451Z_09-procesos.spool.log` y `20261025T175450Z_09-procesos-so.script.log`.

### 16. Explica con tu evidencia por qué SYS ve 16 datafiles desde la raíz y 9 desde FREEPDB1.

Desde `CDB$ROOT` SYS consulta la base completa y ve 16 datafiles repartidos entre raíz, semilla y PDB; conectado a `FREEPDB1`, la misma vista se limita a los 9 de ese contenedor. Es la separación multitenant de Oracle. **Evidencia:** `10-ciclo-vida-pdb` y `10-diccionario`.

### 17. ¿Por qué LAB4_ANALISTA recibe ORA-00942 y no "privilegios insuficientes" al consultar DBA_TABLES?

Oracle responde `ORA-00942` porque para ese usuario `DBA_TABLES` no es visible/autorizada; no tiene acceso al objeto del diccionario. `CREATE SESSION` permite conectarse, pero no consultar todas las vistas de DBA. **Evidencia:** `10-que-ve-cada-usuario.script.log`.

### 18. ¿Qué diferencia hay entre C##LAB4_AUDITOR y C##OPS_RO? ¿Qué sentencia los igualaría?

Los dos son usuarios comunes, pero `C##OPS_RO` tiene permisos de observación y `CONTAINER_DATA=ALL`, mientras `C##LAB4_AUDITOR` solo veía raíz y `CON_ID=0`. Para permitir que el segundo vea las filas de otras PDB desde la raíz: `ALTER USER c##lab4_auditor SET CONTAINER_DATA = ALL CONTAINER = CURRENT;` (además de los permisos SELECT apropiados). **Evidencia:** `10-usuarios.script.log`.

### 19. ¿Qué ventaja tiene GRANT READ ANY TABLE ON SCHEMA frente a conceder tabla por tabla? ¿Y qué riesgo?

`GRANT READ ANY TABLE ON SCHEMA` sirve para leer las tablas actuales y futuras de **un esquema**, sin conceder permisos tabla por tabla. Reduce mantenimiento; el riesgo es que abre de golpe el acceso a todas las tablas de ese esquema, incluso futuras tablas sensibles. Hay que limitarlo al esquema estrictamente necesario. **Evidencia:** parte 10 de permisos.

### 20. ¿Qué valor tiene COMMON_USER_PREFIX en tu base y qué consecuencia tuvo en el bloque 3a de 10-ciclo-vida-pdb?

`COMMON_USER_PREFIX=C##`. Dentro de `FREEPDB1` un usuario local no puede usar ese prefijo reservado para usuarios comunes: el intento del bloque 3a genera `ORA-65094`. **Evidencia:** `10-ciclo-vida-pdb`.

### 21. Clasifica los archivos de tu evidencia 11 en "son la base de datos" y "relacionados". ¿Dónde va el password file y por qué?

Los **archivos base** son datafiles (incluidos los de PDB), control files y online redo. **Relacionados:** SPFILE/PFILE, password file, alert log/trace y tempfiles (estos últimos se regeneran en restauraciones). El password file está en `ORACLE_HOME/dbs` y sirve para autenticar conexiones administrativas remotas sin depender del diccionario de datos. **Evidencia:** `11-archivos-so` y `11-archivos`.

### 22. Explica por qué academia01.dbf mide 104 865 792 bytes en Linux si v$datafile dice 100 MB.

100 MiB son `100 × 1.048.576 = 104.857.600` bytes. Linux muestra `104.865.792`, es decir, **8.192 bytes adicionales**, un bloque de 8 KB destinado a la estructura/cabecera de ese datafile BIGFILE. **Evidencia:** `11-archivos-so` y `11-archivos`.

### 23. ¿Qué dos puntos únicos de fallo (single points of failure) ves en los control files y en el redo de tu base?

Dos control files en el **mismo volumen** no protegen frente a la pérdida del volumen, aunque estén duplicados. Además, los tres grupos redo tienen **un miembro por grupo**: si se pierde el miembro activo no hay otra copia del grupo. Acción: separar copias y multiplexar redo/control entre almacenamientos independientes. **Evidencia:** `11-archivos` y `05-instancia-base`.

### 24. ¿Por qué el ADR fuera del volumen es un problema operativo aunque la base no pierda datos?

El ADR (alert log, trazas) está fuera del volumen persistente de datos. Recrear el contenedor podría llevarse consigo las trazas, aunque las tablas y los datafiles sigan intactos. Es un problema para diagnosticar incidentes y auditar, por lo que conviene persistir o centralizar estos logs. **Evidencia:** `06-alert-log` y `15-autopsia-instancia`.

### 25. Descompón el ROWID de tu estudiante 101 en objeto, bloque y fila, y comprueba cada trozo con DBMS_ROWID.

El estudiante **101**, con estado `ACTIVO`, tiene el **ROWID `AAAR3BAAAAAAAMOAAF`**. La comprobación con `DBMS_ROWID` dio **DATA_OBJECT_ID 73153, archivo absoluto 16, bloque 782 y fila 5**. En la tabla se usa un tablespace BIGFILE, por eso el script especifica la decodificación adecuada y convierte el número de archivo al absoluto. El volcado confirma el mismo bloque y la fila `tab 0, row 5`. **Evidencias:** `20261025T180536Z_12-bloque.script.log`, sección 4, y `20261025T180536Z_12-volcado-bloque.trc.txt` (bloque 782).

### 26. Del volcado: ¿qué significan nrow, fsbo, fseo y avsp? Demuestra que avsp = fseo - fsbo con tus números.

`nrow=90` indica 90 filas en el bloque. `fsbo=0xC6=198` es el comienzo del espacio libre; `fseo=0x427=1063` es su límite final; `avsp=0x361=865` es el espacio libre disponible. Se verifica: **1.063 − 198 = 865**. **Evidencia:** volcado `12-volcado-bloque.trc.txt`.

### 27. Explica la línea de la ITL con Lck = 90 y Flag --U-. ¿Qué es la limpieza diferida de bloques (delayed block cleanout)?

`Lck=90` indica que la transacción afectó a las 90 filas; `--U-` marca una transacción ya confirmada. En **delayed block cleanout**, Oracle no vuelve a escribir inmediatamente todas las cabeceras de bloque tras el COMMIT, sino que las limpia en lecturas posteriores cuando necesita verificar ese estado. **Evidencia:** volcado del bloque y explicación de la parte 12.

### 28. Calcula los 78 bytes de la fila 101 a partir de cabecera, bytes de longitud y datos.

La fila 101 tiene `tl: 78` y siete columnas. Los datos ocupan `3+9+9+11+19+6+11 = 68` bytes; hay **7 bytes de longitud**, uno por cada columna, y **3 bytes de cabecera**. Total: **68 + 7 + 3 = 78 bytes**. **Evidencia:** `12-volcado-bloque.trc.txt`, fila 5.

### 29. Calcula bfr y b de ACA_ESTUDIANTES con la fórmula de la Unidad 3 y explica por qué la tabla real tiene 244 bloques.

Con los valores del experimento, `bfr = floor((8192−96)×0,90/76) = 95 filas/bloque` y `b = ceil(20.000/95) = 211 bloques` (modelo de la Unidad 3). Un ajuste más realista considera cabeceras de Oracle y directorio de filas (`bfr≈93`, `b≈216`); las estadísticas dan **244 bloques**, porque hay espacio reservado, sobrecostes, bloques no llenos y la marca de agua alta. **Evidencia:** `13-capacidad.spool.log`.

### 30. Con tus cifras de 5a, 5b y 5c, ¿cuánto cuesta una lectura lógica, una física aleatoria y un bloque secuencial?

- **Lectura lógica:** 400.000 lecturas en 1.010 ms de CPU, aproximadamente **2,53 µs por lectura** (5a).
- **Física aleatoria:** 5.464 lecturas de un bloque en 52,2 ms, aproximadamente **9,5 µs por lectura** (5b).
- **Secuencial:** 4.595 bloques en unos 28 ms, aproximadamente **6,1 µs por bloque**; el script estima **1.272 MB/s** (5c).

En este entorno la caché de Linux influye mucho en el tiempo observado, por lo que estas cifras no se pueden tomar como latencias de un disco de producción. **Evidencia:** `20261025T180618Z_13-capacidad.spool.log`, sección 5.

### 31. Recalcula el umbral de rentabilidad de la sección 5d suponiendo un HDD (Bw = 150 MB/s, lat = 5 ms).

Con `B=8192` bytes y `Bw=150 MB/s`, transferir un bloque secuencialmente cuesta aproximadamente `8192/(150×10^6)=0,0546 ms`. Frente a `5 ms` de latencia aleatoria, el cociente es `0,0109`. Con `bfr≈86`, el umbral es `(1/86)×0,0109≈0,000127`, es decir, **0,013 % de las filas**. Superado ese porcentaje, el recorrido completo puede salir mejor según este modelo simplificado.

### 32. ¿Por qué la "lectura física" sale en microsegundos en el contenedor? ¿Invalida eso el método?

Las lecturas físicas salen en microsegundos porque el contenedor puede recibir datos desde la **caché de páginas de Linux**, incluso después de `FLUSH BUFFER_CACHE`, que solo vacía la caché de Oracle. No invalida la comparación lógica del método, pero esos tiempos no representan los de un disco HDD/SSD real de producción. **Evidencia:** `13-capacidad`.

### 33. Diferencia migración (row migration) y encadenamiento (row chaining): causa, cómo se detecta y cómo se cura.

**Migración:** una fila antes cabía, creció con UPDATE y Oracle la movió a otro bloque dejando un puntero en el original; se puede curar con `MOVE` y espacio libre preventivo. **Encadenamiento:** una fila no cabe entera en un bloque, así que necesariamente ocupa varios; moverla al mismo tamaño de bloque no lo elimina. Se detectan con estadísticas de filas continuadas, `ANALYZE ... LIST CHAINED ROWS` y pruebas del ejercicio. **Evidencia:** `14-filas-migradas`.

### 34. Explica el 3,2 de lecturas lógicas por búsqueda de la sección 4 a partir del 20 % de filas migradas.

De 1.000 accesos, 800 normales cuestan 3 lecturas lógicas y 200 migrados cuestan 4: `(800×3 + 200×4)/1000 = 3,2` lecturas lógicas por acceso. El 20 % migrado suma la lectura extra para seguir el puntero. **Evidencia:** `14-filas-migradas`.

### 35. ¿Por qué ALTER TABLE ... MOVE deja el índice UNUSABLE? ¿Qué harías en producción?

`ALTER TABLE ... MOVE` cambia los ROWID; el índice conserva referencias antiguas y pasa a `UNUSABLE`. En el laboratorio se reconstruyó y quedó `VALID`, con `CHAIN_CNT=0`. En producción convendría planificar la operación o usar `MOVE ONLINE` y comprobar índices/impacto en sesiones. **Evidencia:** `14-filas-migradas`.

### 36. Elige tres hallazgos de tu autopsia (15) y escribe para cada uno riesgo, acción y prioridad.

**Hallazgo 1:** los dos control files viven en el mismo volumen; riesgo de perder ambas copias si falla el volumen; acción: multiplexar en ubicaciones físicas distintas; **prioridad alta**. **Hallazgo 2:** redo sin varios miembros por grupo; riesgo de único punto de fallo; acción: multiplexar; **alta**. **Hallazgo 3:** seis ORA en las últimas 24 h (`ORA-00800` cuatro, `ORA-65025` uno esperado, `ORA-959` uno); riesgo de alertas sin clasificar; acción: investigar `ORA-00800` y `ORA-959`, excluir del aviso el experimento documentado; **media**. **Evidencia:** `11-archivos` y `15-autopsia-instancia`.

### 37. ¿Por qué C##OPS_RO y no SYSTEM para observar? ¿Qué no puede hacer C##OPS_RO?

`C##OPS_RO` da observabilidad usando mínimo privilegio, sin ejecutar todas las comprobaciones como `SYSTEM` o `SYSDBA`. Tiene lecturas de catálogo y concesiones puntuales; no debería poder modificar tablas ni hacer operaciones administrativas de escritura. La verificación mostró **0 privilegios `ANY` de escritura**. **Evidencia:** `02-cuenta-observabilidad` y `16-verificacion-final`.

### 38. ¿Qué dos vías de exposición de contraseñas corrige este laboratorio respecto al Lab 3?

Se corrigen dos vías: **(1)** contraseñas literales en los scripts iniciales (cuentas `ADMIN_*` convertidas en `NO AUTHENTICATION`; secretos reales solo en `.env` excluido); y **(2)** contraseñas en argumentos de la línea de comandos `sys/clave@...`, visibles en procesos o historial. El nuevo ejecutor usa autenticación de SO `sqlplus / as sysdba`. **Evidencia:** V003, `08-aplicar-migraciones.sh` y comprobación de seguridad O.2.

## 2. Preguntas tipo examen (P1–P46, P-D1, P-G1 y P-G2)

**P1: A, D.** La parada limpia deja una copia fría consistente, restaurable sin aplicar redo adicional.

**P2: B.** En este procedimiento se pone el tablespace OFFLINE, se copia el archivo físicamente, se renombra la ruta y se pone ONLINE.

**P3: A, C.** El arranque cambia STARTUP_TIME; la fecha CREATED pertenece a la base persistente.

**P4: A, C.** En NOMOUNT hay parámetros/instancia, pero todavía no se ha montado el control file.

**P5: B.** Para cambiar ARCHIVELOG la base debe estar montada sin estar abierta.

**P6: B.** Un parámetro dinámico con SPFILE y SCOPE omitido se aplica a MEMORY y SPFILE.

**P7: C.** PROCESSES es de instancia y el cambio desde PDB provoca ORA-65040.

**P8: A, C.** Se puede arrancar con un PFILE bueno y después regenerar el SPFILE desde él.

**P9: B.** ASMM gestiona la SGA; DB_CACHE_SIZE fija el mínimo de esa caché dentro de la SGA.

**P10: A, D.** Las áreas de ordenación y la memoria de sesión de un servidor dedicado pertenecen a su PGA.

**P11: C.** LGWR debe escribir el redo del COMMIT para darlo por confirmado.

**P12: B, D.** PMON y SMON son procesos de background básicos; ARCn depende de ARCHIVELOG.

**P13: B.** Desde una PDB solo se ven los archivos de su propio CON_ID.

**P14: B.** CONTAINER_DATA controla los CON_ID cuyas filas puede ver un usuario común desde la raíz.

**P15: A, D.** El privilegio cubre las tablas presentes y futuras del esquema indicado, no otros esquemas.

**P16: B, D.** Control files y online redo son archivos estructurales de la base, a diferencia del password file y SPFILE.

**P17: C.** Sin archivelogs ni redo disponible, los cambios posteriores a la última copia no se pueden recuperar.

**P18: B.** Para BIGFILE hay que indicar el formato BIGFILE al decodificar el ROWID.

**P19: A, B.** PCTFREE detiene nuevos inserts bajo ese umbral y reserva espacio para crecimiento por UPDATE.

**P20: B.** bfr=floor((8192−96)×0,90/180)=40; b=2.000.000/40=50.000 bloques.

**P21: B.** El evento scattered read corresponde a lecturas multibloque hacia la buffer cache.

**P22: A, C.** Se identifican filas encadenadas y se puede reorganizar ONLINE con más PCTFREE.

**P23: B.** 12 KB no caben en un bloque de 8 KB; la fila sigue encadenada aunque se mueva.

**P24: B.** El usuario común de solo lectura y concesiones puntuales sigue mínimo privilegio.

**P25: B.** PDB$SEED es la plantilla de solo lectura que Oracle utiliza para crear otras PDB.

**P26: C.** C## está reservado a nombres comunes; un usuario local con ese prefijo produce ORA-65094.

**P27: B.** Primero se cierra la PDB y luego se elimina con INCLUDING DATAFILES; lo probamos en la parte 10.

**P28: A, C.** V$DATAFILE y V$LOG pueden consultarse a partir del control file en MOUNT.

**P29: A.** KCC es un componente interno relacionado con las estructuras del control file.

**P30: B.** Las vistas CDB_ están disponibles en contenedores; desde una PDB se restringen sus filas a esa PDB.

**P31: B.** Los tempfiles contienen datos temporales y se regeneran: no forman parte del backup de RMAN.

**P32: B.** La instancia conserva los parámetros en memoria y puede reconstruir SPFILE desde MEMORY.

**P33: B.** V$PWFILE_USERS informa sobre usuarios del password file y privilegios administrativos.

**P34: A, B.** La ITL y el directorio de filas son estructuras de cabecera de bloques de Oracle.

**P35: B.** AUTOALLOCATE comienza con extensiones pequeñas (64 KB) y crece a extensiones de 1 MB.

**P36: B.** ASSM es la gestión de espacio libre mediante mapas de bits por segmento.

**P37: B.** La configuración predeterminada descrita para 23ai crea un tablespace BIGFILE local.

**P38: A.** Las cuatro lecturas lógicas se atendieron en memoria, sin necesidad de lecturas físicas.

**P39: A, B.** Más PCTFREE o filas de mayor tamaño dejan sitio para menos filas por bloque.

**P40: B.** Vaciar la caché afecta a otras sesiones y puede disparar temporalmente las latencias.

**P41: B.** Una migrada se trasladó tras crecer; una encadenada nunca cabe completa en un bloque.

**P42: A.** MOVE cambia ROWID y deja índices inutilizables; se reconstruyen o se usa MOVE ONLINE.

**P43: A.** V$DIAG_ALERT_EXT deja leer mensajes del alert log mediante SQL.

**P44: A, B.** Hay que aumentar capacidad si procede e identificar los segmentos que consumen el espacio.

**P45: A, B.** Todas las PDB comparten la SGA/procesos de la instancia y los online redo de la CDB.

**P46: B.** Un usuario local solo con CREATE SESSION no puede pasar a la raíz sin permisos.

**P-D1: C.** NO AUTHENTICATION deja la cuenta como propietaria de objetos pero sin contraseña utilizable.

**P-G1: B.** Los bind variables permiten reutilizar cursores en lugar de generar SQL distinto por literal.

**P-G2: B.** La PGA disponible no bastó y la operación necesitó seis pasadas sobre datos temporales.

## Notas de las comprobaciones

- **RPO/RTO:** el respaldo frío fue verificado, pero no se ha ejecutado una restauración completa cronometrada. Se explica el RPO de la copia puntual y **no se atribuye un RTO medido**.
- **PID desde el host:** `LGWR` se identificó como PID/SPID 961 dentro del contenedor. Falta, si se quiere completar la comparación con dos números concretos, capturar la salida de `docker top` desde el host Docker.
- **Registro de errores:** en la autopsia figuran cuatro `ORA-00800`, un `ORA-65025` esperado y un `ORA-959`. La verificación final dio `OK`, pero eso **no implica que todos los mensajes del alert log sean irrelevantes**.
- Las respuestas tipo examen se han conservado con sus justificaciones. Conviene contrastarlas con el enunciado del profesor y entenderlas antes de entregar.
