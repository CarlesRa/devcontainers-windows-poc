# Propuesta: entorno de desarrollo contenerizado para aplicaciones Angular

> Borrador de trabajo. Los puntos marcados como **[PENDIENTE]** no se han medido o confirmado todavía y no deben presentarse como hechos.

## Índice

1. [Objetivo](#1-objetivo)
2. [Principios](#2-principios)
3. [Arquitectura](#3-arquitectura)
4. [Motor de contenedores en Windows](#4-motor-de-contenedores-en-windows)
5. [Configuración requerida en Windows con Podman](#5-configuración-requerida-en-windows-con-podman)
6. [Dev Container de referencia](#6-dev-container-de-referencia)
7. [Gobierno y seguridad](#7-gobierno-y-seguridad)
8. [Riesgos y mitigaciones](#8-riesgos-y-mitigaciones)
9. [Plan de adopción](#9-plan-de-adopción)
10. [Pendientes](#10-pendientes)
- [Anexo A: procedimiento de diagnóstico del hot reload](#anexo-a-procedimiento-de-diagnóstico-del-hot-reload)
- [Anexo B: notas de la prueba de concepto](#anexo-b-notas-de-la-prueba-de-concepto)

---

## 1. Objetivo

Que los equipos de desarrollo trabajen sin instalar Node, npm ni Angular CLI en el equipo del usuario. Todo el entorno vive en imágenes corporativas, versionadas y auditables, y se abre desde el editor con un único fichero de configuración (`devcontainer.json`).

## 2. Principios

- **Estándares abiertos**: imágenes OCI y especificación Dev Containers. Ninguna pieza ata la solución a un único motor.
- **Mínimo software en el equipo del usuario**: el motor de contenedores y el editor. Nada más.
- **Control centralizado**: las imágenes las mantiene el equipo de plataforma; los equipos de producto las consumen.
- **Trazabilidad**: toda configuración está en Git, con revisión obligatoria.

## 3. Arquitectura

| Capa | Qué contiene | Quién la mantiene |
|---|---|---|
| Imágenes base corporativas (`mrr-front`, `mrr-back`) | Runtime, CLI, certificados, configuración de proxy y registry, usuario no-root | Plataforma |
| Repositorio de la aplicación | `.devcontainer/devcontainer.json` y `Containerfile` de producción (multi-stage) | Equipo de producto, con revisión de plataforma |
| CI/CD | Lint, tests, build, escaneo, firma y publicación | Plantillas de plataforma |
| Producción | Imagen mínima (servidor web o JRE), sin herramientas de desarrollo | Generada por el pipeline |

La imagen de desarrollo y la de producción son distintas: la de producción nunca parte de la de desarrollo.

Estructura típica de un repositorio de aplicación:

```
mi-app/
├── .devcontainer/
│   └── devcontainer.json     # desarrollo: apunta a la imagen corporativa
├── Containerfile             # producción (multi-stage)
└── src/...
```

Si el proyecto necesita varios servicios (frontend, backend, base de datos), se añade un `compose.yaml` y el `devcontainer.json` indica en qué servicio trabaja el editor.

## 4. Motor de contenedores en Windows

Los contenedores se ejecutan igual con cualquiera de los dos motores. Esto es lo que hemos probado:

| | Podman Desktop | WSL Containers (`wslc`) |
|---|---|---|
| Hot reload sin polling | Sí, con el ajuste de la sección 5.1 | Sí |
| Compose | Sí (`docker-compose`) | No por ahora |
| macOS / Linux | Sí | No |
| Madurez | Alta | Disponibilidad general anunciada el 29 de septiembre de 2026 |
| Software adicional | Podman Desktop | Ninguno (incluido en WSL) |
| Integración con Intune y Defender | **[PENDIENTE]** | Indicada por Microsoft, no verificada por nosotros |

**Recomendación**: Podman Desktop como opción principal. `wslc` como alternativa a reevaluar cuando incorpore compose y soporte multiplataforma.

> No hemos encontrado documentación que indique que Podman pueda usar `wslc` como motor: son alternativas, no piezas combinables. **[PENDIENTE]** Revisar notas de versión de Podman Desktop antes de la reunión.

## 5. Configuración requerida en Windows con Podman

### 5.1 Archivo `.wslconfig`

Ubicación: `C:\Users\<usuario>\.wslconfig`

```ini
[wsl2]
virtiofs=true
```

Tras aplicarlo: `wsl --shutdown`, esperar unos segundos y arrancar de nuevo la máquina de Podman desde Podman Desktop.

**Por qué es necesario.** El servidor de desarrollo de Angular recarga la aplicación automáticamente cuando se guarda un cambio. Para eso necesita que el sistema le avise de que un archivo ha cambiado. Cuando el código está en una carpeta de Windows y se monta en el contenedor, ese aviso depende del tipo de montaje:

| Montaje | Tipo (`FSTYPE`) | Detección de cambios sin polling |
|---|---|---|
| Sin ajuste | `9p` | **No** |
| Con `virtiofs=true` | `virtiofs` | **Sí** |

Resultado obtenido con el mismo proyecto, el mismo motor y el mismo comando (`ng serve`); lo único que varió fue esa línea de `.wslconfig`.

Para comprobar el tipo de montaje dentro del contenedor:

```
findmnt -T . -o TARGET,FSTYPE,OPTIONS
```

**Alcance.** `.wslconfig` es un ajuste del usuario que afecta a todas sus distribuciones WSL2, no solo a la de Podman. Se puede desplegar por los mecanismos habituales de gestión de equipos.

**Versiones con las que se ha probado** (una única combinación):

| Componente | Versión |
|---|---|
| WSL | 3.0.3.0 |
| Kernel | 6.18.54.1-1 |
| WSLg | 1.0.79 |
| Windows | 10.0.26300.9711 |

**[PENDIENTE]** Confirmar la versión mínima de WSL y de Windows que necesita el ajuste `virtiofs`, y que los equipos del cliente la cumplan. No hemos probado otras versiones. El build de Windows probado (26300) es posterior al de Windows 11 24H2 (26100); averiguar si es una versión de preview y repetir la prueba en una versión estable.

### 5.2 Ajustes de VS Code (Dev Containers)

| Ajuste | Valor |
|---|---|
| `Dev › Containers: Docker Path` | `podman` |
| `Dev › Containers: Docker Compose Path` | `docker-compose` (solo si se usa compose) |
| `Dev › Containers: Execute In WSL` | Desmarcado |

Notas:

- `podman-compose` (paquete de Python) no viene con Podman Desktop. El Compose que instala su extensión es `docker-compose.exe`.
- Con `Execute In WSL` marcado, la extensión ejecuta `podman` dentro de WSL, donde no está disponible, y falla al abrir el contenedor.
- Si VS Code ofrece "instalar Docker en WSL", cancelar: no hace falta.

### 5.3 ¿Qué es el polling y cuándo se usa?

**Qué es, en sencillo.** Para recargar la aplicación al guardar un cambio, el servidor de desarrollo necesita enterarse de que el archivo ha cambiado. Hay dos formas:

- **Con aviso del sistema (lo normal)**: el servidor se queda esperando y el sistema operativo le avisa en el momento en que se guarda el archivo. Es instantáneo y no gasta recursos mientras no hay cambios. Es como esperar a que suene el timbre.
- **Con polling (`--poll`)**: el servidor no espera ningún aviso; cada cierto tiempo (por ejemplo, cada 2 segundos) revisa por su cuenta si la fecha de modificación de los archivos ha cambiado. Es como asomarse a la puerta cada 2 segundos para ver si hay alguien.

**Cuándo hace falta.** Cuando el aviso del sistema no llega al contenedor, como ocurre con el montaje `9p` de la sección 5.1. Con `virtiofs` no es necesario.

**Qué cuesta.**

- Algo más de CPU, que crece con el número de archivos vigilados.
- Hasta 2 segundos de retraso en detectar un cambio.

El polling solo mira la fecha de los archivos, no su contenido. **[PENDIENTE]** Medir el consumo real con una aplicación de tamaño representativo; hasta entonces no damos cifras.

**Plan B** si en algún equipo no se puede aplicar el ajuste de la sección 5.1:

```json
"scripts": {
  "start": "ng serve",
  "start:poll": "ng serve --poll 2000"
}
```

`npm start` no lleva polling; `npm run start:poll` es la salida cuando los cambios no se detectan. Conviene nombrar el script por lo que hace (`poll`), no por el motor.

## 6. Dev Container de referencia

`.devcontainer/devcontainer.json` probado:

```json
{
  "name": "Angular Dev",
  "image": "mcr.microsoft.com/devcontainers/javascript-node:24",
  "remoteUser": "node",
  "postCreateCommand": "npm install -g @angular/cli",
  "forwardPorts": [4200],
  "customizations": {
    "vscode": {
      "extensions": [
        "angular.ng-template",
        "dbaeumer.vscode-eslint",
        "esbenp.prettier-vscode"
      ]
    }
  }
}
```

Entorno obtenido: Node 24.22.1, npm 11.19.1, Angular CLI 22.2.5, usuario `node` sin privilegios de root. En producción, la imagen corporativa equivalente sustituye a la de Microsoft.

**Requisitos para la imagen corporativa**

1. Usuario no-root con UID 1000, compatible con Dev Containers.
2. Sin preguntas interactivas de Angular CLI (analíticas y autocompletado), configuradas de antemano (por ejemplo, `NG_CLI_ANALYTICS=false` y `ng config -g cli.completion.prompted true`).
3. Certificados corporativos y configuración de npm hacia el registry interno.
4. Versiones fijadas, sin etiqueta `latest`.
5. Chromium headless en la variante de CI.
6. Mecanismo para que los volúmenes montados en el workspace tengan como dueño al usuario `node`, sin depender de `sudo` si plataforma lo retira.

**Credenciales.** El contenedor no trae credenciales propias. VS Code reenvía la configuración de Git, las credenciales HTTPS del gestor de Windows y el agente SSH si está activo en el host. No se montan `~/.ssh`, `~/.npmrc` ni tokens; si hacen falta, se pasan por variable de entorno o mount explícito, nunca escritos en el `devcontainer.json`.

**Al arrancar la aplicación.** Con el reenvío de puertos de VS Code, `ng serve` a secas fue suficiente con `wslc`. `--host 0.0.0.0` solo es necesario si el puerto se publica directamente con el motor. Se observó que abrir la aplicación desde el aviso de VS Code, en alguna prueba con Podman, no recargaba, mientras que escribir `http://localhost:4200` a mano sí. No lo hemos diagnosticado (**[PENDIENTE]**).

## 7. Gobierno y seguridad

- **Revisión obligatoria** (CODEOWNERS) de plataforma sobre `Containerfile`, `.devcontainer/` y pipeline.
- **Validación automática en CI**: el `FROM` solo puede apuntar al registry corporativo, usuario no-root, sin `latest` (por ejemplo, con Hadolint o políticas OPA/Conftest).
- **Escaneo de vulnerabilidades** y firma de imágenes antes de publicar.
- **Reconstrucción periódica** de las imágenes base para recoger parches.
- **Protección de rama**: sin push directo a `main`, merge solo con pipeline en verde.
- **Admisión en el clúster** (si procede): solo imágenes del registry corporativo y firmadas.

Reparto de responsabilidades en CI:

| Quién | Qué |
|---|---|
| Plataforma | Runners, soporte para construir imágenes, plantillas de pipeline, escaneo, firma, credenciales de publicación |
| Equipo de producto | Qué se ejecuta en su proyecto (`npm ci`, lint, tests, build) y su `Containerfile` de producción |
| Compartido | El fichero de pipeline vive en el repo de la app, con CODEOWNERS en las partes críticas |

## 8. Riesgos y mitigaciones

| Riesgo | Mitigación |
|---|---|
| WSL2 / virtualización no permitida en los equipos | Confirmar con TI antes de empezar |
| La máquina de Podman actual es *rootful* | **[PENDIENTE]** probar en modo *rootless*, más fácil de defender |
| Entorno sin acceso a internet | **[PENDIENTE]** probar con registry interno y `podman load` |
| Versión de WSL/Windows no compatible con `virtiofs` | Fijar versión mínima; plan B con `--poll` |
| `.wslconfig` afecta a todas las distros WSL2 del usuario | Documentarlo y avisar al cliente |
| Rendimiento con aplicaciones grandes | Medir con una app real; opciones: subir intervalo de polling, clonar en volumen |
| Entornos con varios servicios | Compose con Podman; con `wslc`, script de arranque con `run` |
| Permisos de volúmenes (UID) | Imagen con directorios precreados, o clonar en volumen |
| Equipos macOS | **[PENDIENTE]** verificar el comportamiento de eventos de archivo con el motor elegido |

## 9. Plan de adopción

1. **Prueba de concepto** (hecha en parte): devcontainer con Podman y Angular, con la configuración de la sección 5.
2. **Completar mediciones**: `npm install`, primer build y CPU con una aplicación real; compose; modo rootless.
3. **Imágenes corporativas** por parte de plataforma, a partir de los requisitos de la sección 6.
4. **Piloto** con un proyecto real y un equipo reducido.
5. **Generalización** con documentación y despliegue de la configuración por política.

## 10. Pendientes

- [ ] Versión mínima de WSL y Windows para `virtiofs`; probar en Windows estable.
- [ ] Medir CPU y latencia del polling con una aplicación real (`find src -type f | wc -l`, `top`).
- [ ] Medir `npm install` y primer build con y sin `virtiofs`.
- [ ] Probar `compose.yaml` con devcontainer y `virtiofs` activo.
- [ ] Probar Podman en modo rootless.
- [ ] Probar sin acceso a internet (registry interno / `podman load`).
- [ ] Comportamiento en macOS.
- [ ] Integración de `wslc` con Intune y Defender (si se mantiene como alternativa).
- [ ] Diagnosticar por qué el enlace del aviso de VS Code no recargaba en una prueba.
- [ ] Preguntas del cliente a preparar: ENS, proxy, registry, antivirus.

---

## Anexo A: procedimiento de diagnóstico del hot reload

Para comprobar si el servidor de desarrollo detecta los cambios, sin depender del navegador:

1. En el contenedor, comprobar el tipo de montaje:
   ```
   findmnt -T . -o TARGET,FSTYPE,OPTIONS
   ```
2. Arrancar `ng serve` sin `--poll`, guardar un cambio desde VS Code y mirar solo la terminal. Si aparece *Application bundle generation complete*, el servidor detecta el cambio. Si no aparece, no lo detecta.
3. Si no aparece, observar los eventos directamente:
   ```
   sudo apt-get update && sudo apt-get install -y inotify-tools
   inotifywait -m -e modify,create,move,attrib src/app/app.html
   ```
   En otra terminal: `touch src/app/app.html`, luego guardar desde VS Code, luego desde un editor de Windows. Anotar qué eventos aparecen en cada caso.
4. Si la terminal sí muestra el rebuild pero la página no se actualiza, el problema está en el último tramo (websocket, reenvío de puertos o caché del navegador): probar `http://localhost:4200` en una ventana de incógnito y revisar la consola del navegador.

## Anexo B: notas de la prueba de concepto

- Mismo proyecto, mismo motor (Podman Desktop con máquina rootful sobre WSL), mismo comando `ng serve` sin `--poll`.
- Sin `.wslconfig`: `FSTYPE` = `9p`, el servidor de desarrollo no detectó el cambio.
- Con `virtiofs=true`: `FSTYPE` = `virtiofs`, el servidor detectó el cambio.
- Con `wslc`: el hot reload funcionó sin `--poll` y sin `--host`. Sin compose por ahora.
- Qué no se ha medido: consumo de CPU del polling, tiempos de `npm install` y build, comportamiento en versiones distintas de WSL.