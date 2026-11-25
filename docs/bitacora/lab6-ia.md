He leído la política de uso de IA de la asignatura.

# Declaración de uso de IA · Lab 6 · E016

<!--
CÓMO SE RELLENA (léelo y borra este comentario si quieres; el script de evaluación lo ignora)
- La PRIMERA línea es obligatoria y no se toca: «He leído la política de uso de IA de la asignatura.»
- Copia este archivo como docs/bitacora/lab6-ia.md (junto a lab6-respuestas.md) y pon tu código en el título.
  Cada laboratorio tiene el suyo: en el Lab 7 será docs/bitacora/lab7-ia.md.
- Si NO has usado IA: deja solo la línea «No he usado IA.» y borra la entrada de ejemplo.
- Si la has usado: borra la línea «No he usado IA.» y escribe UNA entrada por cada uso, con los 7 campos.
  Un campo vacío es una entrada incompleta (D3: −0,5 por entrada).
- Preguntar a la IA como tutor o corregir la ortografía de un texto propio está permitido: decláralo también.
  Si no copiaste código, escribe «no aplica» en «Código recibido» y en «Evidencia de ejecución» (no cuentan como vacíos).
  Los demás campos son siempre obligatorios.
- Una entrada con TODOS los campos vacíos (la de ejemplo sin tocar) no cuenta: se ignora.
- El código copiado de una IA puntúa si lo EJECUTAS en tu contenedor, lo EXPLICAS con tus palabras,
  dices para qué lo usaste y adjuntas aquí el código y el prompt LITERAL.
-->



## Entrada 1
- **Herramienta:** ChatGPT 
- **Prompt (literal, copiado tal cual):**
```text
Aqui tienes todo el material que necesitamos miralo con calma y empezamos.
```
- **Código recibido (el fragmento, o la ruta del archivo donde está):**
```sql
no aplica
```
- **Dónde lo usé (parte, script o pregunta):** Parte A, revisión inicial del laboratorio. 
- **Propósito (para qué lo necesitaba):** Entender las instrucciones y organizar la práctica.
- **Qué hace (con mis palabras; línea a línea si es SQL corto):** Chat GPT me explico de forma detallada como estaba compuesta la practica y que es lo que debia hacer, separamos las cosas que teniamos que hacer por bloques.
- **Evidencia de ejecución (archivo de docs/bitacora/lab6/evidencia con mi cabecera):** no aplica.

## Entrada 2
- **Herramienta:** ChatGPT.
- **Prompt:** Solicitudes de ayuda durante la ejecución del laboratorio.
- **Código recibido:** Comandos de Ubuntu, Docker, SQL y Git para ejecutar los scripts, resolver errores y guardar evidencias.
- **Dónde lo usé:** Partes A-I.
- **Propósito:** Seguir los ejercicios del profesor y resolver los problemas encontrados.
- **Qué hace:** Permite ejecutar los ejercicios y comprobar sus resultados.
- **Evidencia de ejecución:** docs/bitacora/lab6/evidencia/

## Entrada 3
- **Herramienta:** ChatGPT.
- **Prompt:** Solicitudes de ayuda para resolver los tres incidentes de guardia.
- **Código recibido:**
  - ALTER SYSTEM KILL SESSION '22,23361' IMMEDIATE;
  - EXEC DBMS_SERVICE.START_SERVICE('academia_app');
  - ALTER PLUGGABLE DATABASE FREEPDB1 SAVE STATE;
  - ALTER SYSTEM REGISTER;
  - ALTER USER APP_ACADEMIA ACCOUNT UNLOCK;
- **Dónde lo usé:** Parte I.
- **Propósito:** Recuperar el funcionamiento de la aplicación.
- **Qué hace:** Elimina la sesión bloqueadora, recupera el servicio detenido y desbloquea la cuenta de aplicación.
- **Evidencia de ejecución:** Registros 08 de diagnóstico, resolución y validación.

## Entrada 4
- **Herramienta:** ChatGPT.
- **Prompt literal:** Necesito que me ayudes con las preguntas
- **Código recibido:** Borradores de documentación y código Python para actualizar las respuestas con los valores de las evidencias.
- **Dónde lo usé:** lab6-respuestas.md y lab6-guardia.md.
- **Propósito:** Organizar las preguntas y los informes de incidencias.
- **Qué hace:** Recopila los resultados de las pruebas y facilita su documentación.
- **Evidencia de ejecución:** Registros de las partes B-I y documentos de la bitácora.
