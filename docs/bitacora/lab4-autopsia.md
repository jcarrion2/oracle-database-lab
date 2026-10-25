# Laboratorio 4 — Autopsia de la instancia

Esta revisión se ha hecho sobre la instancia `FREE` de Oracle con el contenedor `oralab-26ai`, después de completar los experimentos del Laboratorio 4. Los hallazgos se apoyan en los registros de las partes 11 y 15, y se comparan con la verificación final.

| Hallazgo observado | Riesgo | Acción propuesta | Prioridad |
|---|---|---|---|
| Los dos control files están dentro del mismo volumen persistente. | Si falla o se pierde ese volumen, se perderían las dos copias a la vez. | Separar las copias de control files en ubicaciones o discos realmente independientes y verificar la restauración. | Alta |
| Tres grupos de online redo (tres miembros en total, 600 MB), sin duplicación de miembros por grupo en el inventario. | La pérdida del único miembro de un grupo puede comprometer la recuperación. | Multiplexar los miembros de redo entre discos/volúmenes independientes. | Alta |
| En la autopsia figuran 6 entradas ORA en 24 horas: `ORA-00800` (4), `ORA-65025` (1) y `ORA-959` (1). | Mezclar errores previstos del laboratorio con errores sin clasificar genera ruido y puede ocultar incidencias. | Registrar `ORA-65025` como esperado por la prueba de eliminación de PDB; investigar el contexto exacto de `ORA-959` y los cuatro `ORA-00800`, sin asumir que son inocuos. | Media |
| El diagnóstico ADR aparece en `/opt/oracle/diag`, fuera del volumen de datos. | Las trazas podrían perderse al reemplazar el contenedor. | Persistir o exportar alert logs y trazas a un destino independiente. | Media |

## Estado verificado al final

- Base `FREE` y `FREEPDB1` en `READ WRITE`.
- Cinco tablespaces de negocio disponibles y datafiles dentro de `/opt/oracle/oradata/FREE/FREEPDB1/`.
- Los 20.000 alumnos, 40 asignaturas y 60.000 matrículas de V002 están presentes.
- Usuarios `ADMIN_*` de negocio con `AUTHENTICATION_TYPE=NONE`.
- Usuario común `C##OPS_RO` de observabilidad; verificación final sin privilegios `ANY` de escritura.
- `PDB_LAB4`, usuarios temporales y tablespace temporal del ejercicio eliminados.
- Verificación `16-verificacion-final.sh`: **todo OK**, código de salida `0`.

**Evidencias de referencia:** logs de `11-archivos`, `15-autopsia-instancia` y `16-verificacion-final`, con sus marcas de tiempo en `docs/bitacora/lab4/evidencia/`.

## Datos adicionales del diagnóstico

En la autopsia se contabilizaron **16 datafiles (3.924 MB)**, **2 control files (36 MB)**, **3 miembros online redo (600 MB)**, **3 tempfiles (60 MB)** y **2 archived logs (258 MB)**. Son cifras del registro real de la parte 15, no estimaciones. En los tablespaces de negocio, `TBS_ACADEMIA` tenía aproximadamente **23 MB utilizados de un máximo de 1.024 MB (2,3 %)** y los otros cuatro unos **6 MB cada uno (0,6 %)**.

También aparecen esquemas con estadísticas de índices incompletas (altura sin informar). Sería conveniente recopilar estadísticas mediante `DBMS_STATS` para que el optimizador disponga de mejores datos; esto se propone como mejora, **no se ha ejecutado en esta práctica**.

**Límite de esta autopsia:** no se ha realizado una restauración completa para medir RTO, ni una investigación exhaustiva de los mensajes `ORA-00800` y `ORA-959`. Que el script de verificación final marque todo `OK` significa que pasó sus comprobaciones, no que el sistema esté libre de cualquier riesgo.
