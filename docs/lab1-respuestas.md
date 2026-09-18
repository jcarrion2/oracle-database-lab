\# Laboratorio 1 - Git Fundamentals



Nombre: Javier Carrion Garcia



\## Preguntas de comprobación



\### 1. ¿Cuál es la diferencia entre Working Directory, Staging Area y Local Repository? Da un ejemplo de un archivo pasando por las tres.



El Working Directory es la carpeta del proyecto donde veo y modifico los archivos. La Staging Area es una zona intermedia donde preparo exactamente los cambios que quiero incluir en el siguiente commit. El Local Repository es el historial que Git guarda en la carpeta .git.



Por ejemplo, cuando creé docs/customer-schema.md, al principio estaba solamente en el Working Directory. Después ejecuté `git add docs/customer-schema.md` y pasó a la Staging Area. Finalmente, con `git commit -m "docs: add customer schema notes"` quedó guardado en el Local Repository.



\### 2. Si modificas un archivo pero no haces git add, ¿aparece ese cambio en tu próximo commit? Explica por qué.



No. El commit solamente guarda los cambios que están preparados en la Staging Area. Lo comprobé cuando creé docs/customer-schema.md e intenté hacer un commit sin ejecutar antes git add. Git mostró el mensaje "nothing added to commit but untracked files present" y el commit no se realizó.



\### 3. ¿Por qué git status no mostraba las carpetas vacías que creaste en la Parte C? ¿Qué truco usamos para solucionarlo?



Porque Git versiona archivos, no carpetas vacías. Para conseguir que las carpetas aparecieran en el repositorio añadimos archivos `.gitkeep` dentro de ellas. De esta forma Git sí tenía un archivo que podía seguir y guardar.



\### 4. Explica con tus palabras qué es HEAD.



HEAD es el puntero que indica en qué rama y en qué commit estoy trabajando actualmente. Por ejemplo, cuando estaba en `feature/customer-search`, el gráfico mostraba `HEAD -> feature/customer-search`. Al cambiar a main, HEAD pasó a apuntar a main.



\### 5. ¿Qué diferencia hay entre crear una branch con git switch -c y crear una carpeta nueva con mkdir? ¿Cómo lo comprobamos en la Parte G?



`mkdir` crea físicamente una carpeta nueva en el disco. En cambio, `git switch -c` crea una nueva línea de trabajo dentro del historial de Git, no una carpeta.



Lo comprobé creando `feature/customer-search` y ejecutando `ls -la`. No apareció ninguna carpeta llamada feature. También comprobé que `customer-search.md` desaparecía al cambiar a main y reaparecía al volver a `feature/customer-search`.



\### 6. Durante el conflicto de la Parte H, ¿qué representaba el contenido entre <<<<<<< HEAD y =======? ¿Y entre ======= y >>>>>>>?



Entre `<<<<<<< HEAD` y `=======` estaba la versión que ya tenía la rama actual, main. En mi caso era el título "Oracle Database Lab (Training Edition)".



Entre `=======` y `>>>>>>> fix/readme-subtitle` estaba la versión procedente de la rama que estaba intentando fusionar, que contenía "Oracle Database Lab — Academic Version".



Resolví el conflicto combinando ambas versiones y dejando finalmente "Oracle Database Lab (Training Edition — Academic Version)", eliminando todos los marcadores.



\### 7. ¿Por qué NO se debe hacer git commit --amend sobre un commit que ya se subió con git push?



Porque `git commit --amend` sustituye el último commit por otro nuevo y cambia su hash. Si el commit anterior ya se ha compartido mediante push, otras personas pueden tener la versión antigua y se producirían historiales diferentes. Por eso utilicé --amend solamente antes de subir el repositorio a GitHub.



\### 8. Si borras por accidente la carpeta .git de tu proyecto, ¿qué se pierde exactamente? ¿Se pierde también el código fuente que está en el disco?



Se pierde la información que Git utiliza para gestionar el repositorio local: historial de commits, ramas, referencias y configuración del repositorio. Los archivos normales del proyecto que están en el Working Directory no se borran, por lo que el código y la documentación seguirían físicamente en el disco.



\### 9. Explica con tus propias palabras la diferencia entre Git y GitHub, sin usar la palabra "nube".



Git es el programa de control de versiones que funciona en mi ordenador y permite crear commits, ramas, merges y consultar el historial incluso sin conexión a Internet. GitHub es una plataforma accesible por Internet donde puedo alojar un repositorio Git y compartirlo con otras personas, además de disponer de herramientas de colaboración como Pull Requests, Issues y revisiones.



\### 10. ¿Por qué no se debe subir un archivo .env con contraseñas reales a un repositorio, aunque el repositorio sea privado?



Porque las contraseñas o tokens quedarían registrados dentro del historial de Git y podrían ser accesibles para otras personas o servicios que tengan acceso al repositorio. Además, si posteriormente el repositorio se comparte o cambia su visibilidad, las credenciales podrían quedar expuestas. Por eso los secretos no deben incluirse en los commits y normalmente se excluyen mediante `.gitignore`.



\### 11. Un compañero te dice: "hice push y ahora GitHub me rechaza el segundo push con non-fast-forward". ¿Qué ha ocurrido probablemente y qué comando ejecutarías primero?



Probablemente el repositorio remoto contiene algún commit que todavía no existe en su repositorio local. Esto puede ocurrir, por ejemplo, si alguien ha modificado el repositorio desde GitHub. Lo primero que ejecutaría sería `git pull` para traer y combinar los cambios remotos. Después de resolver cualquier posible conflicto podría volver a ejecutar `git push`.



\### 12. ¿Qué tipo de Conventional Commit usarías para añadir un índice de rendimiento a una tabla, corregir una restricción mal definida y actualizar el README?



Para añadir un índice cuyo objetivo es mejorar el rendimiento utilizaría `perf`, por ejemplo:



`perf(db): add customer lookup index`



Para corregir una restricción mal definida utilizaría `fix`, por ejemplo:



`fix(db): correct customer constraint`



Para actualizar el README utilizaría `docs`, por ejemplo:



`docs: update README`



\## Lecciones aprendidas



\- Git no guarda automáticamente cualquier cambio: primero debo revisar el estado y decidir qué archivos pasan a la Staging Area.

\- Una branch no es una carpeta, sino una línea de evolución del historial.

\- `git status` es uno de los comandos más útiles para saber en qué situación se encuentra el repositorio.

\- Los commits deben representar cambios concretos y tener mensajes claros siguiendo Conventional Commits.

\- Un conflicto de merge no significa que el repositorio esté roto; Git está pidiendo que una persona decida qué versión debe conservarse.

\- Antes de confirmar un cambio conviene revisar `git diff --staged`.

\- `git pull` permite sincronizar mi copia local con los cambios realizados en el repositorio remoto.

\- No debo reescribir con `--amend` commits que ya han sido compartidos mediante push.

