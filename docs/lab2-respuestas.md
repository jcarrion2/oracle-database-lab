# Laboratorio 2 - GitHub Team Workflow

## Preguntas de comprobación

### 1. ¿Por qué un Issue sin criterios de aceptación es un problema, aunque la descripción general parezca clara?

Porque una descripción puede explicar la idea general pero no define de forma objetiva cuándo el trabajo está terminado. Los criterios de aceptación convierten esa idea en comprobaciones concretas que el Autor y el Reviewer pueden verificar y evitan interpretaciones distintas sobre el alcance.

### 2. Explica la diferencia entre “Refs #N” y “Closes #N” en un mensaje de commit o en la descripción de un Pull Request.

Refs #N crea una referencia o relación con el Issue, pero no lo cierra. Closes #N indica que ese cambio resuelve el Issue y, cuando el PR se integra en la branch principal, GitHub puede cerrar automáticamente el Issue relacionado.

### 3. ¿Qué ocurre exactamente si intentas hacer git push directamente sobre una branch main protegida? ¿Es un error tuyo o un fallo del sistema?

GitHub rechaza el push y devuelve un mensaje indicando que la branch está protegida y que los cambios deben pasar por un Pull Request. No es un fallo: es la política funcionando correctamente. En esta práctica lo comprobé con el error GH006.

### 4. Un compañero te dice: “he aprobado el PR sin mirar los archivos, total ya me fío”. ¿Qué riesgo tiene esa forma de revisar?

El riesgo es convertir la aprobación en un trámite sin control real. Se pueden integrar errores, problemas de seguridad, cambios fuera de alcance o documentación incompleta. La confianza en la persona no sustituye a revisar el cambio que realmente va a entrar en main.

### 5. Si el reviewer pide un cambio y tú ya habías hecho push de tu branch, ¿tienes que abrir un Pull Request nuevo? Explica qué ocurre técnicamente con el PR existente cuando haces un nuevo commit.

No. El PR sigue abierto y apunta a la misma branch de trabajo. Al crear un nuevo commit y hacer push a esa branch, GitHub actualiza automáticamente el PR con el nuevo commit y con el diff resultante. Así se conserva toda la conversación y la trazabilidad en un único PR.

### 6. Describe con tus palabras la diferencia entre Merge commit, Squash and merge y Rebase and merge. ¿Cuál usarías para una branch con commits “wip”, “fix”, “fix2”, “ok ya”?

Merge commit conserva los commits de la branch y añade un commit de fusión. Squash and merge combina todos los commits de la branch en un único commit final. Rebase and merge reaplica los commits sobre la punta de main para dejar un historial lineal. Para una branch con mensajes como “wip”, “fix”, “fix2” y “ok ya” usaría Squash and merge, porque esos commits intermedios no aportan valor al historial principal.

### 7. ¿Por qué borrar una branch después del merge no elimina el trabajo realizado en ella?

Porque la branch es solo una referencia a una línea de commits. Una vez que el contenido se ha integrado en main, los cambios ya forman parte del historial de main. Borrar la branch elimina esa referencia de trabajo, no el commit ya integrado ni los archivos resultantes.

### 8. ¿Qué información debería contener siempre la descripción de un Pull Request, como mínimo?

Como mínimo debería explicar qué hace el cambio, qué se ha modificado, cómo se ha probado y qué Issue o necesidad resuelve. En la práctica lo estructuré como Summary, Changes, Testing y Related Issue.

### 9. Un reviewer escribe solo “esto está mal” como comentario. ¿Qué le falta a ese comentario para ser útil? Reescríbelo tú con un ejemplo inventado.

Le falta señalar qué problema observa, por qué importa y qué acción concreta espera. Por ejemplo: “issue (blocking): la contraseña aparece escrita en texto plano en este archivo. Muévela a una variable de entorno y elimina el valor del repositorio antes de hacer merge.”

### 10. ¿Qué diferencia hay entre que main esté protegida y que simplemente el equipo “se ponga de acuerdo” en no hacer push directo?

Un acuerdo depende de que todas las personas lo recuerden y lo respeten siempre. Una protección de GitHub es una regla técnica: la plataforma impide la acción aunque alguien se equivoque o intente saltarse el proceso. Por eso la protección es verificable y no depende solo de la disciplina del equipo.

### 11. Explica con un ejemplo propio la diferencia entre un comentario issue: (blocking) y uno nitpick: (if-minor) en formato Conventional Comments.

issue: (blocking) señala algo que debe corregirse antes del merge, por ejemplo: “issue (blocking): se ha incluido una clave API real en el archivo; elimínala y rota la credencial”. nitpick: (if-minor) es una mejora pequeña que no bloquea, por ejemplo: “nitpick (if-minor): cambiaría este nombre de variable por uno más descriptivo para facilitar la lectura”.

### 12. Si tu próximo commit es feat!: cambia la firma de la función principal de la API, ¿qué tipo de versión SemVer se dispara y por qué?

El signo de exclamación indica un breaking change. En SemVer eso implica incrementar la versión MAJOR, porque el cambio rompe compatibilidad con consumidores que dependían de la firma anterior de la API.

### 13. ¿Por qué abrir un Draft PR desde el primer commit estructural puede ahorrar tiempo al equipo, aunque parezca más lento para el autor individual?

Porque permite recibir feedback sobre el enfoque cuando todavía se ha invertido poco trabajo. Si la dirección es incorrecta, se corrige pronto y se evita desarrollar durante horas una solución que después tendría que rehacerse. Es una aplicación práctica de fallar rápido y reducir retrabajo.

