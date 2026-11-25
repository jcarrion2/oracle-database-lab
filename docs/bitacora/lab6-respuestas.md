# Laboratorio 6 - Preguntas de comprobación
Alumno: E016
Fecha: 25/11/2026

## Partes A-C: Rol operativo, parámetros y estados

### 1. Errores ORA del alert log
- Resultado: 6 errores en las últimas 24 horas.
- Hay que distinguir los errores conocidos de las incidencias nuevas.
- Un error registrado no significa necesariamente que exista un problema activo.
- Evidencia: 20261125T183942Z_01-turno.script.log.
- Pendiente: confirmar los tipos de error de ese periodo.

### 2. Ventana de mantenimiento del miércoles
- Comienza a las 22:00 y dura 4 horas.
- Tareas automáticas: recopilación de estadísticas del optimizador, asesor de espacio y asesor de ajuste SQL. Las tres tareas aparecen como ENABLED.
- Evidencia: 20261125T184503Z_01-automatizacion.spool.log.

### 3. Diferencia entre V$SPPARAMETER y V$PARAMETER
- V$SPPARAMETER muestra la configuración persistente.
- V$PARAMETER muestra los valores utilizados actualmente.
- SCOPE=SPFILE establece db_files en 250 para el siguiente arranque, sin modificar el valor actual en memoria.
- Sin RESET, el cambio habría entrado en vigor al reiniciar.
- Valor en memoria: 200.
- Evidencia: 20261125T185437Z_02-parametros.script.log.

### 4. RESET de db_files y processes
- db_files no estaba configurado explícitamente.
- Después del RESET aparece ISSPECIFIED=FALSE.
- processes sí estaba configurado con el valor 200.
- Línea del PFILE: *.processes=200
- RESET devuelve db_files a su configuración original. Para processes habría que conservar el valor explícito anterior.
- Evidencia: 20261125T185800Z_02-init_desde_spfile.ora.

### 5. Reinicio sin estado guardado
- La instancia puede arrancar correctamente mientras FREEPDB1 permanece en MOUNTED.
- La apertura del CDB no implica la apertura de todas sus PDB.
- SAVE STATE permite recuperar el estado de apertura.
- Error obtenido: ORA-01109: database not open.
- Evidencia: 20261125T185930Z_02-estados-pdb.script.log.

## Partes D-E: Servicios, sesiones y diccionario

### 6. Identificación de una sesión
- SID: 22.
- SERIAL#: 22292.
- SPID: 9990.
- Evento: PL/SQL lock timer.
- MODULE identifica el módulo de la aplicación.
- ACTION identifica la operación realizada.
- CLIENT_IDENTIFIER identifica al usuario final.
- USERNAME solamente identifica la cuenta de conexión a Oracle.
- Evidencia: 20261125T191111Z_03-sesiones.script.log.

### 7. CANCEL SQL frente a KILL SESSION
- CANCEL SQL cancela la operación, pero conserva la sesión y la transacción abierta.
- Error observado: ORA-01013.
- KILL SESSION termina la sesión y deshace las modificaciones pendientes.
- Error observado: ORA-00028.
- La fila 101 debe quedar ACTIVO al deshacerse el cambio no confirmado.
- Tras CANCEL SQL queda 1 transacción abierta. Tras KILL SESSION quedan 0 sesiones de la aplicación.
- Evidencia: 20261125T191111Z_03-sesiones.script.log.

### 8. Prefijo de usuarios comunes
- Valor de common_user_prefix: C##.
- Las cuentas comunes creadas por el administrador deben respetar este prefijo.
- Ejemplo: C##OPS_RO.
- APP_ACADEMIA es una cuenta local de FREEPDB1, con COMMON=NO.
- Evidencia: 20261125T191023Z_03-cuenta-app.script.log.

### 9. Las tres miradas

| Usuario y contenedor | ALL_TABLES | DBA_USERS | CDB_USERS | V$DATAFILE |
|---|---|---|---|---|
| C##OPS_RO en CDB$ROOT | 0 | 37 | 81 (37 ROOT y 44 PDB) | 17 |
| C##OPS_RO en FREEPDB1 | 0 | 44 | 44 | 10 |
| APP_ACADEMIA en FREEPDB1 | 1 | ORA-00942 | ORA-00942 | ORA-00942 |

- Los resultados dependen de los privilegios del usuario y del contenedor.
- APP_ACADEMIA no puede consultar las vistas administrativas indicadas.
- C##OPS_RO tiene ALL_CONTAINERS=Y.
- Evidencia: 20261125T192528Z_04-tres-miradas.script.log.

### 10. Deltas de rendimiento
- Recorridos completos de PEDIDO: 10.
- Tiempo de reloj: 1 segundo.
- DB time: 1,01 segundos.
- DB CPU: 0,95 segundos.
- Lecturas lógicas totales: 53171.
- Lecturas lógicas por recorrido: 5317.
- La cercanía entre DB time y DB CPU indica que la carga empleó principalmente tiempo de CPU y poco tiempo de espera.
- Evidencia: 20261125T193823Z_04-deltas.script.log.

## Partes F-G: Espacio y acceso

### 11. Medidas del tablespace
- Tablespace: TBS_ACADEMIA_HIST.
- Espacio asignado inicialmente: 10 MB.
- Espacio libre inicialmente: 3,9 MB.
- Tamaño máximo configurado: 40 MB.
- Umbral de aviso: 80%.
- Umbral crítico: 95%.
- Para alertar interesa el porcentaje ocupado respecto al máximo alcanzable.
- Evidencia: 20261125T194743Z_05-almacenamiento.script.log.

### 12. Errores de cuota
- La inserción la realiza SYS.
- El propietario del segmento y quien consume la cuota es CARGA_HIST.
- ORA-01950: falta de cuota.
- ORA-01536: cuota excedida.
- La cuota asignada era de 2 MB y se consumió completamente.
- Aumentar el tamaño del tablespace no soluciona una limitación de cuota del usuario.
- Evidencia: 20261125T194743Z_05-almacenamiento.script.log.

### 13. Carga reanudable
- Inicio: 19:49:07.
- Finalización: 19:49:21.
- Duración total: 14 segundos.
- Filas insertadas: 20000.
- Cuota ampliada a 30 MB.
- Espacio utilizado: 23 MB.
- El alert log registró la suspensión y posterior reanudación de la operación carga-historico.
- Oracle registró SUSPENDED a las 19:49:06, con TIMEOUT=180 y ORA-01536. La carga había finalizado a las 19:49:21. No se dispone aquí de la hora exacta de reanudación.
- Evidencia: 20261125T194906Z_05-resumable.script.log.

### 14. DELETE y SHRINK SPACE

| Momento | Filas | Bloques | Espacio |
|---|---:|---:|---:|
| Tras DELETE | 2000 | 2944 | 23 MB |
| Tras SHRINK SPACE | 2000 | 304 | 2,4 MB |

- DELETE elimina registros, pero no reduce necesariamente el espacio asignado al segmento ni su HWM.
- SHRINK SPACE compacta los datos y permite recuperar espacio.
- Espacio recuperado: aproximadamente 20,6 MB.
- Evidencia: 20261125T195054Z_05-hwm-shrink.script.log.

### 15. Perfiles de seguridad
- DEFAULT: 10 intentos fallidos permitidos, confirmado durante el incidente 603.
- Otros valores de DEFAULT:
  - INACTIVE_ACCOUNT_TIME: 365 días.
  - PASSWORD_GRACE_TIME: 7 días.
  - PASSWORD_LIFE_TIME: 180 días.
  - PASSWORD_LOCK_TIME: 1 día.
  - PASSWORD_REUSE_MAX: UNLIMITED.
  - PASSWORD_REUSE_TIME: UNLIMITED.
  - PASSWORD_ROLLOVER_TIME: 0.
  - PASSWORD_VERIFY_FUNCTION: NULL.
- PRF_PERSONAS:
  - FAILED_LOGIN_ATTEMPTS: 3.
  - PASSWORD_LOCK_TIME: 1 minuto.
  - PASSWORD_LIFE_TIME: 90 días.
  - PASSWORD_GRACE_TIME: 7 días.
  - INACTIVE_ACCOUNT_TIME: 90 días.
- Limitar los intentos fallidos ayuda a proteger frente a intentos repetidos de acceso.
- La caducidad de la contraseña obliga a renovarla según la política del perfil.
- Se comprobó el bloqueo LOCKED(TIMED) tras tres intentos incorrectos.
- Evidencias: 20261125T195806Z_06-gobierno-acceso.script.log y 20261125T195927Z_06-probar-acceso.script.log.

### 16. Privilegios de Ana García
- ROL_LECTURA_ACADEMIA dispone de SELECT ANY TABLE sobre el esquema ADMIN_ACADEMIA.
- Esto permite consultar ACA_MATRICULAS sin conceder acceso individualmente a esa tabla.
- ROL_SECRETARIA_ACADEMIA permite actualizar solamente la columna ESTADO de ACA_ESTUDIANTES.
- Cambiar el nombre de un alumno no está autorizado.
- Confirmar el código de error exacto en el registro.
- Evidencia: 20261125T195927Z_06-probar-acceso.script.log.

## Partes H-I: Objetos, diagnóstico y guardia

### 17. Invalidación de objetos
Resultados esperados según el ejercicio:
- Añadir una columna que ningún objeto utiliza: no se invalida ningún objeto.
- Ampliar email: se invalida V_CONTACTO_LAB6.
- Ampliar nombres: se invalidan V_ACTIVOS_LAB6 y F_TOTAL_ACTIVOS_LAB6 por sus dependencias.
- Oracle utiliza dependencias de grano fino.
- Confirmar los estados intermedios en la evidencia.
- Resultado final confirmado: 0 objetos de negocio inválidos.
- Evidencia: 20261125T203739Z_07-objetos.script.log.

### 18. Advisors de memoria
- SGA actual: 1536 MB.
- Duplicar la SGA a 3072 MB mantiene ESTD_DB_TIME_FACTOR=1.
- Lecturas físicas estimadas en ambos tamaños: 36540.
- No se aprecia una mejora al duplicar la SGA.
- PGA actual: 512 MB.
- PGA de 384 MB: 0 sobreasignaciones estimadas.
- PGA de 256 MB: 8 sobreasignaciones estimadas.
- PGA de 128 MB: 8 sobreasignaciones estimadas.
- No conviene reducirla a 256 MB o menos según los resultados del advisor.
- Evidencia: 20261125T204109Z_07-memoria.spool.log.

### 19. Traza de la sesión y TKPROF
- Lecturas lógicas: 247.
- Lecturas físicas: 0.
- Tiempo de CPU: 0,01 segundos.
- Tiempo transcurrido: 0,01 segundos.
- La línea FETCH de la traza indica cr=247 y cu=0.
- TKPROF confirma el resultado en la columna query.
- Evidencia: 20261125T204427Z_07-adr-trazas.script.log.

### 20. Simulacro de guardia
- INC-601: bloqueo TX de la sesión 211 por la sesión 22. KILL SESSION '22,23361' IMMEDIATE. Validación: alumno 101 ACTIVO y 0 sesiones bloqueadas.
- INC-602: servicio academia_app detenido y error ORA-12514. START_SERVICE, SAVE STATE y REGISTER. Validación: conexión recuperada y recuento de 17600 estudiantes.
- INC-603: APP_ACADEMIA en estado LOCKED. ALTER USER APP_ACADEMIA ACCOUNT UNLOCK. Validación: estado OPEN y conexión correcta.
- Evidencias: registros 08 de diagnóstico, resolución y validación.
- Informes detallados: docs/bitacora/lab6-guardia.md.
