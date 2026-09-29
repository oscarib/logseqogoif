- # Objetivo del proyecto
  
Hacer un fork personal de **Logseq OG** (versión de ficheros Markdown, repo [`logseq/og`](https://github.com/logseq/og), rama `version/file`), montar el stack de desarrollo completo, y validar el ciclo de cambio → compilación → prueba, empezando por un cambio trivial ("Hello World" en algún punto visible de la UI) que después se revertirá.  
  
**Motivación:** Logseq (versión file-based, no la nueva versión DB) es la única herramienta que se ajusta al flujo mental del usuario. No quiere depender de sincronización en la nube (evita la versión DB y cualquier servicio de sync propietario) por control de datos y confidencialidad, y quiere mantener la posibilidad de colaborar con IA directamente sobre los ficheros Markdown del grafo.  
  
**Entorno físico:** PC personal (no equipo corporativo), con Windows como sistema anfitrión y una VM Linux (Ubuntu o Lubuntu) en VirtualBox para el desarrollo. Hardware: AMD Ryzen 9 5950X (16 núcleos / 32 hilos), 64 GB RAM — sin limitaciones reales de recursos.  
  
---
  
- ## Decisión de arquitectura clave: dónde se compila cada cosa
  
- **Desarrollo y compilación del código (ClojureScript + JS) → en la VM Linux**, en disco local (nunca directamente sobre una carpeta compartida de VirtualBox).
- **Generación del instalador/EXE final para Windows → en Windows nativo**, no por compilación cruzada desde Linux.
  
- ### Por qué el empaquetado final va en Windows y no cruzado desde Linux
  
La documentación oficial de Logseq (`docs/develop-logseq-on-windows.md`) exige explícitamente Visual Studio 2022 (Build Tools) para compilar en Windows, porque algunos módulos de Node necesitan compilación nativa vía `node-gyp`. Intentar que `electron-builder` genere el target Windows desde Linux (con Wine) es técnicamente posible pero frágil precisamente por esos módulos nativos: sin el toolchain de Visual Studio no hay garantía de que compilen correctamente para destino Windows. Por eso:  
  
	1. Todo el desarrollo, la compilación ClojureScript y la validación en caliente (hot-reload) se hacen en la VM Linux.
	2. El código ya compilado se traslada a Windows (por Git, lo más limpio; o copiando el resultado vía carpeta compartida de VirtualBox si se prefiere).
	3. El empaquetado final (`yarn release-electron` o equivalente) se ejecuta en Windows nativo, con Visual Studio instalado, donde cualquier módulo nativo se compila sin fricción.

---
  
- ## Avisos y riesgos a tener en cuenta
  
	1. **Carpetas compartidas de VirtualBox e inotify**: si en algún momento se edita o compila directamente sobre una carpeta compartida (`vboxsf`), el watcher de hot-reload (shadow-cljs/webpack) puede no detectar los cambios, porque esas carpetas no propagan eventos inotify de forma fiable. Usarlas solo para mover el resultado ya compilado entre entornos, nunca como directorio de trabajo activo.
	2. **Renderizado GPU de Electron dentro de la VM**: Chromium (motor de Electron) puede fallar al iniciar el sandbox de GPU en un entorno virtualizado (pantalla negra o crash al lanzar la app de desarrollo). Si ocurre, lanzar con el flag `--disable-gpu`. Activar "Enable 3D Acceleration" en la configuración de pantalla de la VM ayuda a mitigarlo.
	3. **Gestor de paquetes**: la rama `version/file` usa **Yarn 1 (classic)**, no pnpm: tiene `yarn.lock` y `static/yarn.lock`, el workflow de build cachea `yarn` y `docs/develop-logseq.md` usa `yarn`. Usar pnpm ignoraría los lockfiles y resolvería otras versiones de dependencias. Yarn 1 se activa con `corepack enable` (incluido en Node 22).
	4. **Versiones exactas del toolchain**: Node, Java, Clojure, Leiningen y Babashka tienen versiones concretas esperadas por el proyecto (revisar `.github/workflows/build.yml` del repo antes de instalar nada). Verificadas en `build.yml`: **Node 22**, **Java 11 (Zulu)**, **Clojure CLI 1.11.1.1413**, **Babashka 1.0.168**. Node 22 está fijado en el repo con `.nvmrc` (ejecutar `nvm use` en la raíz). Instalar versiones libres de los repositorios de Ubuntu puede dar fallos sutiles de compilación. Usar `nvm` para fijar la versión exacta de Node.
	5. **Instalador sin firmar**: el `.exe` generado (Squirrel) no estará firmado digitalmente. Windows SmartScreen puede avisar la primera vez que se ejecute. No es un error, es esperado.
	6. **Estado del repo**: verificado — `logseq/og` está activo y mantenido (rama `version/file`, +12.800 commits, actividad reciente), no es un fork abandonado.
	7. **IDE**: se usará IntelliJ IDEA con el plugin **Cursive** (para ClojureScript) y el plugin oficial de **Claude Code** para JetBrains. El plugin de Claude Code no trae el CLI incluido: hay que instalar `claude` por separado y que esté en el `PATH` de la VM.

---
  
- ## Primer plan de trabajo (para ejecutar con Claude Code en la VM Linux)
  
	1. **Preparar el entorno de la VM**

			- Instalar Ubuntu o Lubuntu con entorno gráfico en VirtualBox.
			- Instalar Guest Additions (para portapapeles, resolución, y carpetas compartidas si se usan más adelante).
			- Asignar recursos generosos a la VM: 8-16 GB RAM, 8-12+ hilos, disco de 40-50 GB mínimo.
  
				2. **Instalar el toolchain de desarrollo**

			- Node.js (versión exacta según `build.yml` del repo, gestionada con `nvm`).
			- Yarn 1 (classic), vía `corepack enable`.
			- Java (OpenJDK, versión que requiera el proyecto) + Clojure CLI.
			- Opcional: Babashka, si se quiere usar `bb dev:electron-start`.
			- IntelliJ IDEA + plugin Cursive + plugin de Claude Code (JetBrains Marketplace).
			- CLI de Claude Code (`npm install -g @anthropic-ai/claude-code` o instalador nativo), verificar que está en el `PATH`.
  
				3. **Clonar el repositorio**

			- `git clone https://github.com/logseq/og` (rama `version/file`), en disco local de la VM (no en carpeta compartida).
  
				4. **Instalar dependencias**

			- `yarn install` en la raíz.
			- `yarn install` dentro de `static/`.
  
				5. **Levantar el ciclo de desarrollo en caliente**

			- `yarn watch` y esperar a que compile `:electron` y `:app`.
			- `yarn dev-electron-app` para abrir la app en una ventana Electron dentro de la VM.
  
				6. **Hacer el cambio de prueba ("Hello World")**

			- Localizar un punto adecuado de la UI (ClojureScript) para insertar un texto de prueba visible.
			- Confirmar que aparece vía hot-reload en la ventana Electron, sin necesidad de recompilar desde cero.
  
				7. **Preparar el traslado a Windows para el empaquetado final**

			- Hacer commit del cambio en Git.
			- En Windows: clonar/pull el repo, instalar el stack equivalente (Node, Yarn, Java, Clojure, Visual Studio 2022 Community — ver `docs/develop-logseq-on-windows.md`).
			- Ejecutar el build de producción (`yarn release-electron` o el comando equivalente) en Windows nativo.
			- Verificar que el instalador generado funciona y muestra el cambio de prueba.
  
				8. **Revertir el cambio de prueba**

			- `git revert` o `git checkout` para deshacer el "Hello World" una vez validado el ciclo completo.
  
---
  
- ## Buenas prácticas de contribución (por si algún día se propone un PR)
  
El "Hello World" inicial es solo una prueba de que el ciclo funciona; se descartará antes de cualquier trabajo real. Cualquier corrección real (por ejemplo, el cierre de una vulnerabilidad/CVE concreta) se desarrollará en su propia rama, dedicada en exclusiva a ese fix puntual — nunca mezclada con la prueba inicial ni con otros cambios.  
  
Según `CONTRIBUTING.md` del repo ([enlace](https://github.com/logseq/og/blob/version/file/CONTRIBUTING.md)), si en algún momento se plantea enviar un Pull Request a `logseq/og`, conviene seguir desde el principio estas prácticas para no invalidarlo:  
  
- **Antes de programar**: buscar si ya existe un PR o issue sobre el mismo problema, y abrir un issue propio describiéndolo antes de empezar a escribir código.
- **CLA obligatorio**: para que Logseq acepte el código hay que firmar su Contributor License Agreement. El contribuidor conserva la propiedad de su código, pero cede a Logseq una licencia perpetua, irrevocable y sublicenciable (incluye la posibilidad de que ellos relicencien el código, incluso bajo una licencia distinta a la AGPL). Sin firma de CLA, no se acepta ningún PR.
- **Una rama, un propósito**: cada PR debe ser quirúrgico — solo el cambio que resuelve el problema concreto (p. ej. un CVE). Nada de refactors grandes no relacionados, cambios de formato/espacios en blanco, ni actualizaciones de dependencias "de paso".
- **Sin conflictos de merge**: mantener la rama actualizada/rebaseada contra `version/file` del repo original antes de proponer el PR.
- **Tests y linting**: los checks automáticos de PR (tests + lint) deben pasar en verde. Para una feature/mejora se esperan tests; para un fix es recomendable aunque no obligatorio.
- **Título del PR con prefijo categórico**: `fix`, `feat`/`feature`, `enhance`, `dev`, `chore` o `test`, según corresponda.
- **Habilitar "Allow edits by maintainers"** al abrir el PR, para que el equipo de Logseq pueda retocar la rama si hace falta.
  
En la práctica: cada rama de fix debe nacer limpia desde `version/file` actualizado, contener únicamente los cambios de ese CVE concreto, y no arrastrar nada de la configuración de entorno, pruebas sueltas o el "Hello World" de validación inicial.  
  
---
  
- ## Referencias
  
- Repositorio: https://github.com/logseq/og
- Guía de desarrollo (Linux/Mac): https://github.com/logseq/og/blob/version/file/docs/develop-logseq.md
- Guía de desarrollo en Windows: https://github.com/logseq/og/blob/version/file/docs/develop-logseq-on-windows.md
- Guía de contribución: https://github.com/logseq/og/blob/version/file/CONTRIBUTING.md
- Contributor License Agreement: https://gist.github.com/andelf/fdca603bcde3e7cb3a965f598f51ff9b
- Plugin JetBrains de Claude Code: https://plugins.jetbrains.com/plugin/27310-claude-code-beta-
