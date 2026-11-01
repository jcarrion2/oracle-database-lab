# Runbook: pérdida de un datafile (FREEPDB1)

**Síntomas.** `ORA-01116` indica un problema de E/S y `ORA-01110` identifica el datafile afectado; `ORA-27041` indica que el sistema operativo no puede abrirlo. Con `ORA-63999`, Oracle puede cerrar `FREEPDB1` y dejarla `MOUNTED`. Registrar hora, archivo, impacto y *alert log*. **Avisar** al DBA de guardia y al responsable del servicio.

**Comprobar antes de restaurar:** confirmar número `n` del archivo, `ARCHIVELOG`, copia RMAN válida, archived redo disponible y estado de `FREEPDB1`. No borrar nada más ni reiniciar la CDB. Si es `SYSTEM`/undo de raíz, escalar: no usar este procedimiento.

```sql
-- Desde SQL*Plus como SYSDBA en CDB$ROOT:
SELECT log_mode FROM v$database;
SELECT name, open_mode FROM v$pdbs WHERE name='FREEPDB1';
SELECT file#, name, status FROM v$datafile ORDER BY file#;
```

```rman
-- Desde RMAN TARGET /:
LIST BACKUP OF DATAFILE n;
```

**Dos caminos.** Si `FREEPDB1` está `READ WRITE`, aislar el datafile con `OFFLINE`. Si Oracle la cerró (`MOUNTED`), esperar a que termine de cerrarse y omitir ese paso. Nunca lanzar RMAN mientras la PDB todavía está cerrándose.

```sql
-- Solo con FREEPDB1 abierta:
ALTER SESSION SET CONTAINER=FREEPDB1;
ALTER DATABASE DATAFILE n OFFLINE;
```

```rman
RESTORE DATAFILE n;
RECOVER DATAFILE n;
```

```sql
ALTER SESSION SET CONTAINER=FREEPDB1;
ALTER DATABASE DATAFILE n ONLINE;
ALTER SESSION SET CONTAINER=CDB$ROOT;
-- Solo si FREEPDB1 sigue MOUNTED:
ALTER PLUGGABLE DATABASE FREEPDB1 OPEN;
SELECT name, open_mode FROM v$pdbs WHERE name='FREEPDB1';
```

**Verificar:** `FREEPDB1 = READ WRITE`, archivo online y consultas de negocio funcionando. En el laboratorio, `estudiante_id=104` conservó `EGRESADO` tras la recuperación y después se devolvió a `ACTIVO`. Guardar *alert log* y salida de RMAN; anotar RPO/RTO. **Nuestro simulacro D:** RPO **0**, RTO **10 s** durante la recuperación (**11 s** desde el fallo); hubo que reabrir la PDB. Si falla el restore/recover, **escalar**, no improvisar cambios. Comunicar la recuperación a los responsables.

**Registro de seguimiento:** no considerar cerrado el incidente hasta contrastar el estado de RMAN, del archivo en `V$DATAFILE`, del alert log y de las consultas del esquema de negocio. En nuestro laboratorio se conservaron 20.000 estudiantes y el estudiante 104 volvió después a `ACTIVO` como limpieza de la prueba.
