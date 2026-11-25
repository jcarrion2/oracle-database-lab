# Laboratorio 6 - Simulacro de guardia
Alumno: E016
Fecha: 25/11/2026
Horas expresadas en UTC.

## INC-601 - Bloqueo de una sesión

- Detección: 20:47:45. Aviso de secretaría mediante un ticket de prioridad alta.
- Impacto: Ana García no podía guardar la ficha del alumno 101. La incidencia se validó como resuelta a las 20:53:03.
- Diagnóstico: La sesión 211 de ANA_GARCIA estaba bloqueada por la sesión 22 de APP_ACADEMIA, con SERIAL# 23361. El evento era enq: TX - row lock contention. La sesión bloqueadora tenía un bloque y un registro de UNDO.
- Causa raíz: Una transacción abierta del módulo matriculas mantenía bloqueada una fila de ACA_ESTUDIANTES.
- Intervención: ALTER SYSTEM KILL SESSION '22,23361' IMMEDIATE;
Se terminó únicamente la sesión bloqueadora, sin reiniciar la base de datos.
- Validación: El alumno 101 quedó en estado ACTIVO y el número de sesiones bloqueadas fue 0.
- Acción preventiva: Controlar la duración de las transacciones y establecer alertas ante bloqueos prolongados.

## INC-602 - Servicio de aplicación detenido

- Detección: 20:54:21. Ticket de prioridad crítica del equipo de matrículas.
- Impacto: La aplicación no podía conectarse a Oracle y devolvía ORA-12514. Se validó la recuperación a las 20:59:27.
- Diagnóstico: FREEPDB1 estaba disponible, pero academia_app aparecía como no arrancado. El listener funcionaba correctamente en el puerto 1521.
- Causa raíz: El servicio academia_app estaba detenido y no aparecía anunciado en el listener. No se determinó el motivo original de su parada.
- Intervención:
  EXEC DBMS_SERVICE.START_SERVICE('academia_app');
  ALTER PLUGGABLE DATABASE FREEPDB1 SAVE STATE;
  ALTER SYSTEM REGISTER;
Se recuperó el servicio sin reiniciar Oracle.
- Validación: APP_ACADEMIA volvió a conectarse, obtuvo un recuento de 17600 estudiantes y realizó una actualización seguida de ROLLBACK.
- Acción preventiva: Supervisar la disponibilidad del servicio y guardar el estado de la PDB con el servicio arrancado.

## INC-603 - Cuenta de aplicación bloqueada

- Detección: 21:01:25. Ticket de prioridad alta del equipo de matrículas.
- Impacto: La aplicación no podía iniciar sesión debido al error ORA-28000.
- Diagnóstico: APP_ACADEMIA aparecía en estado LOCKED, con perfil DEFAULT y un límite de 10 intentos fallidos. El servicio academia_app estaba arrancado.
- Causa raíz: Según el contexto del simulacro, el script trimestral de bajas no distinguía correctamente entre cuentas personales y cuentas de aplicación.
- Intervención: ALTER USER APP_ACADEMIA ACCOUNT UNLOCK;
La cuenta se desbloqueó a las 21:05:19 sin modificar su contraseña ni sus permisos.
- Validación: La cuenta quedó en estado OPEN. La aplicación volvió a conectarse, consultar los datos y actualizar una fila correctamente.
- Acción preventiva: Excluir las cuentas de aplicación de los procedimientos de baja de usuarios personales y revisar las cuentas afectadas antes de ejecutar dichos procedimientos.
