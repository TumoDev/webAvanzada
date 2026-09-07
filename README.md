## Respuestas

**Pregunta 1** ¿Por qué no se recomienda desarrollar directamente sobre main en este laboratorio?

Porque main debe mantenerse siempre en un estado estable y desplegable. Si se trabaja directamente sobre main, cualquier error o cambio incompleto queda expuesto de inmediato en la rama principal, sin posibilidad de revisión previa. Usar una rama de trabajo (devops/ci-cd) permite desarrollar, probar y validar los cambios mediante CI antes de integrarlos, y hacerlo a través de un Pull Request agrega una instancia de revisión y control de calidad.

**Pregunta 2.** ¿Qué problema se evita al utilizar --skip-git al crear el proyecto Angular?

Se evita que Angular CLI inicialice un segundo repositorio Git (.git) dentro de la carpeta frontend/. Si eso ocurriera, se generaría un repositorio anidado ("repo dentro de repo"), lo que provoca conflictos al hacer git add, hace que Git trate frontend/ como un submódulo no rastreado y complica el versionado del proyecto completo desde la raíz.

**Pregunta 3.** ¿Qué verifica npm run build en esta etapa del laboratorio?

Verifica que el proyecto Angular compile correctamente en modo producción: que no existan errores de TypeScript, de plantillas o de configuración, y que se puedan generar los artefactos finales (bundles JS/CSS/HTML) en la carpeta dist/. Confirma que la aplicación está lista para ser desplegada, no solo que "funciona" en modo desarrollo.

**Pregunta 4.** ¿Qué utilidad tiene revisar git status o git diff --cached antes de realizar un commit?

Permiten confirmar exactamente qué archivos y qué contenido se van a versionar antes de comprometerlos definitivamente. git status muestra qué archivos están modificados, preparados o sin rastrear, y git diff --cached muestra el contenido exacto que quedará en el commit. Esto evita incluir por error archivos sensibles (credenciales, .env, node_modules, artefactos de build) o cambios no deseados.

**Pregunta 5.** ¿Qué evento activa el workflow ci.yml?

Se activa cuando se crea o actualiza un Pull Request cuya rama base es main (pull_request: branches: [main]). Es decir, corre en cada push hacia una rama que tiene abierto un PR contra main.

**Pregunta 6.** En runs-on: ubuntu-latest, ¿qué representa ubuntu-latest?

Es el tipo de máquina virtual (runner) que GitHub Actions provisiona para ejecutar el job: una imagen de Ubuntu Linux con la versión más reciente soportada por GitHub. Define el sistema operativo y el entorno base (herramientas preinstaladas) sobre el cual se ejecutan los pasos del workflow.

**Pregunta 7.** Ordene las etapas de validación que ejecuta el job frontend y explique por qué npm ci se ejecuta antes que las pruebas.

Orden: 1) Obtener código (checkout) → 2) Configurar Node.js → 3) Instalar dependencias (npm ci) → 4) Ejecutar pruebas (npm test) → 5) Construir Angular (npm run build).

npm ci debe ejecutarse antes que las pruebas porque instala exactamente las dependencias indicadas en package-lock.json, incluyendo los frameworks de testing (Karma/Jasmine) y las librerías de Angular necesarias para compilar y ejecutar los specs. Sin esas dependencias instaladas, los comandos npm test y npm run build fallarían por no encontrar los módulos requeridos.

**Pregunta 8.** Después del push, indique qué etapa del pipeline falla y qué ocurre con las etapas siguientes.

Falla la etapa "Ejecutar pruebas" (npm test -- --watch=false), ya que el test debe mostrar el título del catálogo espera el texto Catálogo de Recursos pero el h1 no lo contiene tras el cambio a Título incorrecto. Al fallar este paso, GitHub Actions detiene la ejecución del job: la etapa siguiente ("Construir Angular") no se ejecuta, y el workflow completo se marca como fallido (❌) en el Pull Request.

**Pregunta 9.** ¿Debería integrarse este Pull Request a main mientras el pipeline está fallando? Justifique.

No. Integrar un PR con el pipeline en rojo rompe la garantía de que main siempre está en un estado estable y validado. Además, en este flujo el CD (cd.yml) se dispara con el push a main, por lo que un merge con pruebas fallando podría propagar un artefacto roto o incompleto hacia el "staging" simulado. La buena práctica es bloquear el merge hasta que todos los checks estén en verde (idealmente reforzado con una regla de protección de rama).

**Pregunta 10.** Clasifique cada elemento:

| Elemento | Clasificación |
|---|---|
| package.json | Versionable |
| API_URL pública | Variable/configuración |
| AWS_REGION | Variable/configuración |
| DB_PASSWORD | Secreto/no versionable |
| API_TOKEN | Secreto/no versionable |
| terraform.tfstate | Secreto/no versionable (puede contener datos sensibles de infraestructura y además es estado local, no código) |

**Pregunta 11.** ¿Por qué una contraseña o token no debe escribirse directamente dentro de ci.yml, cd.yml o un archivo TypeScript del frontend?

Porque esos archivos quedan versionados en el historial de Git, visibles para cualquiera con acceso al repositorio (y de forma permanente en el historial, aunque se borren después). Escribir credenciales en texto plano expone el secreto a filtraciones, scraping automatizado de repos públicos y a cualquier colaborador, violando el principio de mínima exposición. Los secretos deben inyectarse en tiempo de ejecución mediante mecanismos como GitHub Secrets, que los cifra y los oculta en los logs.

**Pregunta 12.** Si un secreto real fue incluido en un commit y luego se agrega su archivo a .gitignore, ¿queda solucionado el problema? Explique qué acción adicional debe realizarse.

No queda solucionado. .gitignore solo evita que el archivo se vuelva a versionar en commits futuros, pero el secreto ya quedó guardado en el historial de Git y sigue siendo recuperable (por ejemplo, con git log, git show o clonando el repositorio). Es necesario: (1) revocar/rotar inmediatamente la credencial expuesta en el sistema correspondiente, y (2) eliminar el secreto del historial usando herramientas como git filter-repo o BFG Repo-Cleaner, y luego forzar la actualización del repositorio remoto.

**Pregunta 13.** ¿Qué diferencia existe entre terraform validate, terraform plan y terraform apply?

- terraform validate: verifica que la sintaxis y la configuración del código Terraform sean correctas (sin conectarse a ningún estado real ni calcular cambios).
- terraform plan: calcula y muestra qué acciones se realizarían (crear, modificar, destruir recursos) comparando el código con el estado actual, sin ejecutar ningún cambio real.
- terraform apply: ejecuta efectivamente los cambios planificados, creando/modificando/destruyendo los recursos reales y actualizando el archivo de estado (tfstate).

**Pregunta 14.** ¿Por qué ci.yml se activa con pull_request y cd.yml se activa con push sobre main?

ci.yml valida el código antes de integrarlo, por lo que debe ejecutarse en cada Pull Request contra main, dando retroalimentación temprana sin afectar la rama principal. cd.yml, en cambio, representa el despliegue a staging, que solo debe ocurrir una vez que el código ya fue revisado y fusionado en main; por eso se dispara con push sobre esa rama, asegurando que solo código integrado y validado llegue a producción/staging.

**Pregunta 15.** ¿Qué función cumple Terraform dentro de este flujo de CD?

Terraform automatiza y estandariza el proceso de "despliegue" (en este caso, copiar el build de Angular hacia una carpeta staging/ simulada) de manera declarativa y reproducible. En lugar de ejecutar comandos manuales, describe el resultado deseado como infraestructura como código (IaC), permitiendo versionar, planificar y aplicar los cambios de forma controlada y trazable dentro del pipeline.

**Pregunta 16.** ¿Por qué el workflow usa ${{ secrets.DEMO_TOKEN }} en lugar de escribir el valor directamente?

Porque GitHub Secrets almacena el valor cifrado y lo inyecta como variable de entorno solo durante la ejecución del workflow, sin exponerlo en el código fuente ni en los logs (GitHub además enmascara automáticamente su valor si aparece en la salida). Escribir el valor directamente en el YAML lo dejaría en texto plano y versionado permanentemente en el historial del repositorio.
