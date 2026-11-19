# 🛠️ Laboratorio 3.2.2 — Pipeline CI/CD DevSecOps con Jenkins



## 📑 Índice

1. [Contexto: NutriBox SpA](#1-contexto-nutribox-spa)
2. [Qué debe lograr tu pipeline](#2-qué-debe-lograr-tu-pipeline)
3. [Cómo leer esta guía](#3-cómo-leer-esta-guía)
4. [Preparación del ambiente](#4-preparación-del-ambiente-antes-de-la-clase)
5. [Ruta del laboratorio (vista general)](#5-ruta-del-laboratorio-vista-general)
6. [Paso a paso](#6-paso-a-paso)
7. [Experimentos de seguridad](#7-experimentos-de-seguridad)
8. [SonarQube (opcional)](#8-sonarqube-opcional--nivel-experto)
9. [Cierre y limpieza](#9-cierre-y-limpieza)
10. [Entregables y rúbrica](#10-entregables-y-rúbrica)
11. [Problemas frecuentes](#11-problemas-frecuentes)
12. [Debriefing y anexos](#12-debriefing-y-anexos)

---

## 1. Contexto: NutriBox SpA

**NutriBox SpA** es una startup chilena que vende cajas de comida saludable con despacho a domicilio. Su backend es la **NutriBox API** (Node.js/Express), que:

- 📦 Lista el catálogo de productos.
- 🧾 Recibe pedidos validando el **RUT** (módulo 11), el stock y los **cupones**.
- 💰 Calcula **IVA (19 %)** y el **despacho**, consultando al courier externo «RapiEnvíos» (con tarifa de respaldo si el courier se cae).

Los despliegues se hacían **a mano**, un viernes en la tarde. El mes pasado ocurrieron dos incidentes:

| # | Incidente | Consecuencia |
|---|---|---|
| 1 | Alguien cambió el IVA a 18 % y nadie ejecutó las pruebas. | **3.200 pedidos** cobrados mal. |
| 2 | Se agregó `lodash 4.17.4`, con vulnerabilidad **crítica** de *Prototype Pollution*. | Detectada recién en una auditoría externa. |

> 🎯 **Tu misión como Ingeniero/a DevSecOps:** implementar en **Jenkins** un pipeline que, en cada commit, compile, ejecute pruebas con cobertura mínima de 80 %, analice la seguridad de dependencias (SCA) y del código (SAST), y **solo si todo está correcto** construya la imagen Docker y la despliegue en *staging*. **Si algo falla, el despliegue queda bloqueado.**

### Objetivos de aprendizaje

1. Ejecutar e interpretar pruebas **unitarias** e **de integración** (Jest + Supertest), con **mocks, stubs y spies**.
2. Medir **cobertura** y usarla como *Quality Gate* (≥ 80 %).
3. Levantar **Jenkins** con Docker y definir un pipeline declarativo (`Jenkinsfile`) versionado en Git.
4. Integrar **SCA** (`npm audit`) y **SAST** (ESLint + `eslint-plugin-security`) que **rompen el build**.
5. Comprobar la política de **Broken Build**: un fallo detiene el pipeline y bloquea el despliegue.
6. *(Opcional)* Configurar **SonarQube** con un Quality Gate que aborta el pipeline.

---

## 2. Qué debe lograr tu pipeline

### Las 8 etapas

```text
 ┌──────────┐   ┌─────────┐   ┌───────────────────┐   ┌──────────────────────┐
 │ 1        │   │ 2       │   │ 3                 │   │ 4a  SCA  (npm audit) │
 │ Checkout ├──►│ Build   ├──►│ Pruebas +         ├──►│                      │
 │          │   │ npm ci  │   │ cobertura ≥ 80 %  │   │ 4b  SAST (ESLint)    │
 └──────────┘   └─────────┘   └───────────────────┘   └──────────┬───────────┘
                                                                 │ (en paralelo)
        ┌────────────────────────────────────────────────────────┘
        ▼
 ┌──────────────┐   ┌──────────────┐   ┌──────────┐   ┌──────────────┐   ┌────────────┐
 │ 5            │   │ 6            │   │ 7        │   │ 8            │   │ Staging    │
 │ SonarQube    ├──►│ Imagen       ├──►│ Deploy   ├──►│ Smoke test   ├──►│ :8081      │
 │ (opcional)   │   │ Docker       │   │ staging  │   │ /health      │   │ actualizado│
 └──────────────┘   └──────────────┘   └──────────┘   └──────────────┘   └────────────┘
```

### Qué pasa si falla cada etapa

| # | Etapa | Comando principal | ¿Qué pasa si falla? |
|---|---|---|---|
| 1 | Checkout | `checkout scm` | Se detiene todo. |
| 2 | Build | `npm ci` | Se detiene todo. |
| 3 | Pruebas + cobertura | `npm run test:ci` | 🔴 **Build roto**, no hay despliegue. |
| 4a | SCA (paralelo) | `npm run audit:gate` | 🔴 Build roto; SAST igual se ejecuta. |
| 4b | SAST (paralelo) | `npm run lint` | 🔴 Build roto; SCA igual se ejecuta. |
| 5 | SonarQube | `sonar-scanner` + `waitForQualityGate` | Solo si `EJECUTAR_SONARQUBE=true`; aborta si el gate falla. |
| 6 | Imagen Docker | `docker build` | — |
| 7 | Deploy a staging | `docker run … -p 8081:3000` | — |
| 8 | Smoke test | `curl /health` y `/api/productos` | Despliegue marcado como fallido. |

### Los cuatro escenarios que verás

Leyenda: ✅ pasó · ❌ falló · ⏭️ omitida

| Escenario | 3 Pruebas | 4a SCA | 4b SAST | 6–8 Imagen, deploy, smoke | Staging |
|---|:---:|:---:|:---:|:---:|---|
| 🟢 Todo correcto | ✅ | ✅ | ✅ | ✅ | **Se actualiza** a la imagen nueva |
| 🔴 IVA incorrecto | ❌ | ⏭️ | ⏭️ | ⏭️ | Conserva la imagen anterior |
| 🔴 Dependencia vulnerable | ✅ | ❌ | ✅ | ⏭️ | Conserva la imagen anterior |
| 🔴 Código con `eval()` | ✅ | ✅ | ❌ | ⏭️ | Conserva la imagen anterior |
| 🔴 Cobertura baja | ❌ | ⏭️ | ⏭️ | ⏭️ | Conserva la imagen anterior |

> 💡 En el escenario de **cobertura baja** ninguna prueba falla: el build se rompe porque la cobertura cae bajo el 80 %.

### Otros elementos clave del `Jenkinsfile`

- `triggers { pollSCM('H/2 * * * *') }` → disparo **automático**: Jenkins revisa GitHub cada ~2 min.
- `post { always { junit …; recordCoverage …; archiveArtifacts … } }` → publica reportes **aunque el build falle**.
- `parallel { … }` **sin** `failFast` → SCA y SAST siempre se ejecutan, para tener ambos reportes.

> 📝 **¿Por qué Poll SCM y no un webhook?** `localhost` no es accesible desde Internet. En un servidor real, o con un túnel como ngrok, se usaría el webhook de GitHub con `githubPush()`.

---

## 3.  Dónde está cada cosa

| Nombre | Ubicación | Contenido |
|---|---|---|
| Carpeta principal | `~/lab-cicd/Laboratorio_Jenkins_NutriBox` | Guía, scripts y todo el laboratorio |
| Aplicación | `…/nutribox-api` | Código, pruebas, Dockerfile y Jenkinsfile (**tu repo Git**) |
| Jenkins | `…/infra-jenkins` | Docker Compose y configuración (**no se sube a GitHub**) |
| Jenkins web | `http://localhost:8080` | Crear y ejecutar el pipeline |
| Staging | `http://localhost:8081` | Versión desplegada |
| SonarQube | `http://localhost:9000` | Análisis opcional |


### Glosario

| Término | Significado simple |
|---|---|
| **Repositorio** | Carpeta administrada por Git con el código y su historial. |
| **Commit / Push** | Registro de cambios / envío de esos registros a GitHub. |
| **CI / CD** | Integración continua (valida cada cambio) / entrega continua (despliega una versión aprobada). |
| **Pipeline** | Secuencia automática de etapas ejecutada por Jenkins. |
| **SCA** | Busca vulnerabilidades conocidas en las **dependencias**. |
| **SAST** | Busca patrones inseguros en el **código**, sin ejecutarlo. |
| **Staging** | Ambiente de prueba parecido a producción. |
| **Smoke test** | Prueba breve que confirma que la app desplegada responde. |
| **Quality Gate** | Condición mínima que debe aprobarse para continuar. |
| **Mock / Stub / Spy** | Objetos de prueba que sustituyen, simulan u observan dependencias. |



---

## 4. Paso a paso

### Paso 1 — Copiar el laboratorio

🖥️ **Terminal** · Crea una copia de trabajo en tu carpeta personal, con rutas simples para Git, npm y Docker.

Reemplaza `<RUTA>` por la ruta donde descargaste el material:

```bash
mkdir -p ~/lab-cicd
cp -a "<RUTA>/Laboratorio_Jenkins_NutriBox" ~/lab-cicd/
cd ~/lab-cicd/Laboratorio_Jenkins_NutriBox
ls
```

> ✅ **Resultado esperado:** ves `nutribox-api`, `infra-jenkins`, `scripts` y los documentos del laboratorio.

Estructura del material:

```text
Laboratorio_Jenkins_NutriBox/
├── scripts/validar-linux.sh   ← revisión automática del ambiente
├── infra-jenkins/             ← infraestructura (NO se sube a GitHub)
│   ├── docker-compose.yml
│   ├── .env.example
│   └── jenkins/ (Dockerfile, plugins.txt, casc/jenkins.yaml)
└── nutribox-api/              ← LA APLICACIÓN (tu repositorio Git)
    ├── src/                   ← código fuente
    ├── tests/unit/            ← pruebas unitarias (mocks)
    ├── tests/integration/     ← pruebas de la API (supertest)
    ├── Jenkinsfile            ← define el pipeline
    ├── Dockerfile             ← imagen de producción
    ├── eslint.config.mjs      ← reglas SAST
    ├── sonar-project.properties
    └── package.json / package-lock.json
```

---

### Paso 2 — Validar el ambiente

🖥️ **Terminal** · Comprueba que Git, Node.js, Docker, las pruebas y las configuraciones estén listos.

```bash
cd ~/lab-cicd/Laboratorio_Jenkins_NutriBox
bash scripts/validar-linux.sh
```

> ✅ **Resultado esperado:** `VALIDACIÓN COMPLETA: OK`.

El script instala dependencias, ejecuta las 70 pruebas, exige la cobertura mínima, aplica SAST/SCA y valida los dos Docker Compose. No levanta Jenkins.

---

### Paso 3 — Crear y publicar el repositorio

🖥️ **Terminal** y 🌐 **Navegador** · Registras el código en Git y lo publicas en GitHub para que Jenkins lo lea.

| Repositorio local | Repositorio remoto |
|---|---|
| Vive en tu equipo. Aquí editas y haces commits. | Vive en GitHub. Jenkins lo consulta para obtener el código. |

**3.1 🖥️ Identidad de Git** (solo la primera vez):

```bash
git config --global user.name "Tu Nombre"
git config --global user.email "tu_correo@duocuc.cl"
```

**3.2 🖥️ Repositorio local:**

```bash
cd ~/lab-cicd/Laboratorio_Jenkins_NutriBox/nutribox-api
git init -b main
git add .
git commit -m "feat: proyecto inicial NutriBox"
git log --oneline
```

**3.3 🌐 Crear el repositorio en GitHub:**

1. **New repository** → nombre `nutribox-api`.
2. Visibilidad **Public** (así Jenkins lo clona sin credenciales).
3. **No** agregues README, `.gitignore` ni licencia.
4. **Create repository** y copia la URL HTTPS.

**3.4 🖥️ Conectar y publicar** (reemplaza `<TU_USUARIO>`):

```bash
git remote add origin https://github.com/<TU_USUARIO>/nutribox-api.git
git remote -v
git push -u origin main
```

> ⚠️ Si GitHub pide autenticación, usa el mecanismo de tu equipo: inicio de sesión web, clave SSH o token personal. La opción `-u` enlaza las ramas; después basta `git push`.

**Alternativa con GitHub CLI:**

```bash
gh repo create nutribox-api --public --source . --remote origin --push
```

> ✅ **Checkpoint 1:** `git status` sin cambios pendientes, rama `main`, `origin` apuntando a GitHub, y GitHub muestra `src/`, `tests/`, `Jenkinsfile` y `Dockerfile`.

---

### Paso 4 — Ejecutar las pruebas en local

🖥️ **Terminal** · Principio **Shift-Left**: corres en tu equipo los mismos controles que hará el pipeline.

```bash
cd ~/lab-cicd/Laboratorio_Jenkins_NutriBox/nutribox-api
npm ci
npm run test:ci
npm run lint
npm run audit:gate
```

> ✅ **Resultado esperado:** `Tests: 70 passed` (5 suites), cobertura sobre 80 %, lint sin errores y 0 vulnerabilidades altas o críticas en producción.
>
> 📸 **Captura 1:** terminal con `Tests: 70 passed` y la tabla de cobertura.

#### Qué hace cada control

| Comando | Control | Detalle |
|---|---|---|
| `npm run test:ci` | Pruebas + cobertura | El `coverageThreshold` de 80 % hace que Jest termine con error si la cobertura baja. |
| `npm run lint` | **SAST** | Las reglas `error` rompen el build; las `warn` no. Revisa `eslint.config.mjs`. |
| `npm run audit:gate` | **SCA** | Ejecuta `npm audit --omit=dev --audit-level=high`: solo falla con vulnerabilidades **altas o críticas** que llegan a **producción**. |

> 🧠 **Pregunta para el informe:** ¿por qué tiene sentido que el gate ignore las `devDependencies`? ¿Qué riesgo se asume?
>
> `npm audit` (sin filtros) puede mostrar vulnerabilidades *moderate* en herramientas de desarrollo como Jest.

#### Qué prueba cada archivo

| Archivo | Tipo | Qué observar |
|---|---|---|
| `tests/unit/rut.test.js` | Unitaria, caja negra | `test.each` con RUT válidos, inválidos y casos borde (DV `K`, DV `0`, tipos no string). |
| `tests/unit/carrito.test.js` | Unitaria | Estructura Setup / Execute / Assert, límites 1 y 50, cupón `__proto__`. |
| `tests/unit/despacho.test.js` | Unitaria con **mocks** | `jest.fn()`, `mockResolvedValue`, `mockRejectedValue`, `toHaveBeenCalledWith`, `beforeEach`/`afterEach`. |
| `tests/integration/api.test.js` | Integración API REST | `supertest`, códigos HTTP, cabeceras de seguridad, **stub** del despacho y **spy** de `console.error`. |

#### Opcional: ver el reporte y probar la API a mano

```bash
npm run test:coverage
xdg-open coverage/lcov-report/index.html
```

```bash
npm start
```

En **otra terminal**:

```bash
curl -s http://localhost:3000/health | jq .
curl -s http://localhost:3000/api/productos | jq .
```

```bash
curl -s -X POST http://localhost:3000/api/pedidos \
  -H 'Content-Type: application/json' \
  -d '{"rut":"12.345.678-5","region":"RM","items":[{"productoId":1,"cantidad":2}],"cupon":"BIENVENIDA10"}' | jq .
```

> ✅ El pedido debe dar `total = 30815` (subtotal 25.980 − 10 % + IVA + despacho RM 2.990).

Un RUT inválido debe responder **400**:

```bash
curl -i -X POST http://localhost:3000/api/pedidos \
  -H 'Content-Type: application/json' \
  -d '{"rut":"12.345.678-0","region":"RM","items":[{"productoId":1,"cantidad":1}]}'
```

> `jq` solo mejora la presentación. Si no lo tienes, quita `| jq .` o instálalo con `sudo apt install -y jq`. Detén la API con `Ctrl + C`.

> ✅ **Checkpoint 2:** pruebas, cobertura, lint y audit pasan en local.

---

### Paso 5 — Levantar Jenkins

🖥️ **Terminal** · Inicia el servidor que automatizará pruebas y despliegue.

```bash
cd ~/lab-cicd/Laboratorio_Jenkins_NutriBox/infra-jenkins
cp .env.example .env
nano .env
```

Cambia `JENKINS_ADMIN_PASSWORD` por una clave propia y guarda.

> 🔐 El archivo `.env` contiene un secreto: **nunca** se sube a Git.

```bash
docker compose up -d --build
docker compose ps
```

Este comando construye la imagen `lab-jenkins:1.0` (Jenkins LTS + Node.js 22 + Docker CLI + plugins), crea la red `cicd-net` y el volumen `jenkins_home`, e inicia el contenedor `jenkins` en el puerto **8080**.

Espera la línea `Jenkins is fully up and running` (sal con `Ctrl + C`; el contenedor sigue corriendo):

```bash
docker logs -f jenkins
```

Comprueba que Jenkins tiene Node y acceso a Docker:

```bash
docker exec jenkins node -v
docker exec jenkins docker ps
```

🌐 **Navegador:** abre <http://localhost:8080> → usuario `admin`, clave la de tu `.env`. No hay asistente inicial porque la configuración viene de `casc/jenkins.yaml` (Configuration as Code).

> ✅ **Checkpoint 3:** inicias sesión y ves el mensaje «Laboratorio CI/CD con Jenkins - NutriBox SpA».
>
> Plugins esperados en **Administrar Jenkins → Plugins → Instalados**: Pipeline, Git, JUnit, Coverage, Pipeline Stage View, Pipeline Graph View, SonarQube Scanner, Configuration as Code.

---

### Paso 6 — Crear el trabajo en Jenkins

🌐 **Navegador** (`http://localhost:8080`) · Conecta Jenkins con tu repositorio y le indica dónde está el pipeline.

1. **Nueva Tarea** → nombre `nutribox-api-pipeline` → tipo **Pipeline** → **OK**.
2. En *Description*: `Pipeline DevSecOps de NutriBox API`.
3. En la sección **Pipeline**:

| Campo | Valor |
|---|---|
| Definition | **Pipeline script from SCM** |
| SCM | **Git** |
| Repository URL | `https://github.com/<TU_USUARIO>/nutribox-api.git` |
| Credentials | **- none -** (repositorio público) |
| Branch Specifier | `*/main` |
| Script Path | `Jenkinsfile` |

4. **Guardar**.

> ✅ **Resultado esperado:** el trabajo aparece sin ejecuciones, listo para leer el `Jenkinsfile` desde GitHub.

> ⚠️ El disparador `pollSCM` y el parámetro `EJECUTAR_SONARQUBE` están **dentro del Jenkinsfile**. Jenkins los registra recién después de la **primera** ejecución.

---

### Paso 6 — Primera ejecución: línea base verde

🌐 **Navegador** y 🖥️ **Terminal** · Establece un punto de partida correcto antes de introducir errores.

1. Selecciona **Construir ahora** (después usarás **Build with Parameters**). La primera vez tarda 2–4 min.
2. Deja `EJECUTAR_SONARQUBE` **desmarcado**.
3. Revisa **Stage View** y **Console Output**.

> ⏳ No canceles si parece colgado en `npm ci` o `docker build`: es normal.

> ✅ **Resultado esperado:** las etapas 1–4 y 6–8 en verde; SonarQube aparece omitida.
>
> 📸 **Captura 2:** Stage View con el pipeline completo en verde.

#### Dónde mirar los resultados

| Dónde | Qué ver |
|---|---|
| Job → **Stage View** | Las etapas en verde. |
| Build → **Pipeline Overview** | Gráfico con SCA y SAST en paralelo. |
| Build → **Console Output** | Log completo: cobertura, `npm audit`, ESLint, `docker build`, `curl /health`. |
| Build → **Test Result** | 70 pruebas, agrupadas por archivo. |
| Build → **Coverage Report** | Cobertura de líneas y ramas por archivo. |
| Build → **Build Artifacts** | `reports/junit.xml`, `reports/npm-audit.json`, `reports/sast-eslint.html`, `coverage/cobertura-coverage.xml`. |

#### 🖥️ Verifica el despliegue y **anota el número de imagen**

```bash
docker ps --filter "name=nutribox-staging"
curl -fsS http://localhost:8081/health
docker images nutribox-api
docker inspect nutribox-staging --format "{{.Config.Image}}"
```

> 💡 Guarda el número, por ejemplo `nutribox-api:5`. Será tu evidencia de que un build fallido **no** reemplaza la última versión correcta. La imagen debe tener el **número de build** (trazabilidad) y `latest`.

#### Cómo se hace el despliegue (etapas 6–8)

1. **Construcción:** Jenkins comprueba acceso a Docker y a `cicd-net`, y crea `nutribox-api:<número-de-build>`.
2. **Despliegue:** elimina solo el contenedor de staging anterior, inicia la imagen nueva y verifica que quede en ejecución.
3. **Smoke test:** espera hasta 30 s y consulta `/health` y `/api/productos` desde la misma red Docker. Si falla, muestra el estado y las últimas 100 líneas del log.

#### 🖥️ Probar el disparo automático

```bash
cd ~/lab-cicd/Laboratorio_Jenkins_NutriBox/nutribox-api
printf '\nPipeline administrado por Jenkins.\n' >> README.md
git add README.md
git commit -m "docs: nota sobre pipeline"
git push
```

> ✅ **Sin hacer clic**, en ≤ 2 minutos debe iniciar un build nuevo, con la causa *«Started by an SCM change»*.
>
> ✅ **Checkpoint 4:** build verde, app en `http://localhost:8081/health` y un build disparado automáticamente.

> ⚠️ Si modificas el `Jenkinsfile`, haz `git add`, `git commit` y `git push`. Jenkins ejecuta la versión **publicada en GitHub**, no la que quedó solo en tu equipo.

#### Si fallan construcción, despliegue o smoke test

```bash
# 1. Jenkins funciona y tiene acceso a Docker
docker ps --filter "name=jenkins"
docker exec jenkins docker info

# 2. Jenkins y staging comparten cicd-net
docker network inspect cicd-net

# 3. Staging existe y sigue en ejecución
docker ps -a --filter "name=nutribox-staging"
docker logs --tail 100 nutribox-staging

# 4. Probar desde Jenkins, igual que el pipeline
docker exec jenkins curl -fsS http://nutribox-staging:3000/health

# 5. Probar desde Linux por el puerto publicado
curl -fsS http://localhost:8081/health
```

| Mensaje | Causa probable | Corrección |
|---|---|---|
| `docker: not found` | Jenkins creado con una imagen anterior o distinta | En `infra-jenkins`: `docker compose down` y `docker compose up -d --build`. |
| `permission denied … docker.sock` | Jenkins sin acceso al motor Docker | Revisa el volumen `/var/run/docker.sock` y `user: root` en Compose; recrea Jenkins. |
| `network cicd-net not found` | La infraestructura no creó la red | Levanta Jenkins con `infra-jenkins/docker-compose.yml`, no con un `docker run` aislado. |
| `port is already allocated` | Otro proceso usa el 8081 | `docker ps --format 'table {{.Names}}\t{{.Ports}}'`; detén ese contenedor o usa otro `STAGING_PORT`. |
| `Could not resolve host: nutribox-staging` | Jenkins y staging no comparten red | Revisa `docker network inspect cicd-net` y recrea Jenkins con Compose. |
| `Connection refused` o timeout | La app cerró o aún no inicia | Lee `docker logs nutribox-staging`. |

---

### Paso 7 — Romper algo a propósito: IVA incorrecto

🖥️ **Terminal** · Demuestra que Jenkins evita desplegar código defectuoso. Esta secuencia es **obligatoria**.

```bash
cd ~/lab-cicd/Laboratorio_Jenkins_NutriBox/nutribox-api
sed -i 's/const IVA = 0.19;/const IVA = 0.18;/' src/services/carrito.js
npm test
```

> Resultado local: **3 pruebas fallan y 67 aprueban**.

Súbelo deliberadamente:

```bash
git add src/services/carrito.js
git commit -m "test: introduce IVA incorrecto para validar el pipeline"
git push
```

> ✅ **Resultado esperado en Jenkins:**
>
> - Etapa **3** en 🔴 rojo; *Test Result* muestra la diferencia (`Expected: 5130 / Received: 4860`).
> - Etapas posteriores ⏭️ **omitidas**: no se construye imagen ni se despliega.
> - `docker inspect` sigue mostrando la imagen anterior.
>
> 📸 **Captura 3:** Stage View con pruebas en rojo y las siguientes omitidas.

Confirma que staging conserva la imagen de la prueba normal:

```bash
docker inspect nutribox-staging --format "{{.Config.Image}}"
curl -fsS http://localhost:8081/health
```

---

### Paso 8 — Corregir y comprobar la recuperación

🖥️ **Terminal** · Demuestra que el proceso vuelve a funcionar cuando el código se corrige.

```bash
sed -i 's/const IVA = 0.18;/const IVA = 0.19;/' src/services/carrito.js
npm run test:ci
npm run lint
npm run audit:gate
```

> ✅ 70 pruebas aprobadas, cobertura sobre 80 %, lint sin errores y 0 vulnerabilidades altas o críticas.

Publica la corrección:

```bash
git add src/services/carrito.js
git commit -m "fix: restaura IVA correcto"
git push
```

Espera el build automático (o usa **Build with Parameters** con SonarQube desmarcado) y verifica:

```bash
docker inspect nutribox-staging --format "{{.Config.Image}}"
curl -fsS http://localhost:8081/health
```

> ✅ La etiqueta de la imagen cambia al número del nuevo build. Esto completa el ciclo **verde → rojo con despliegue bloqueado → verde con despliegue recuperado**.
>
> ✅ **Checkpoint 5:** conserva tres evidencias: pipeline normal en verde, pipeline bloqueado por el IVA y pipeline corregido en verde.


---

## 9. SonarQube (opcional — nivel experto)

> 💾 Requiere ~4 GB de RAM adicionales. Tiempo estimado: 25 min.

### 9.1 🖥️ Levantar SonarQube

```bash
cd ~/lab-cicd/Laboratorio_Jenkins_NutriBox/infra-jenkins
docker compose --profile sonar up -d
docker ps --filter "name=sonarqube"
docker logs -f sonarqube
```

Espera `SonarQube is operational` (2–3 min) y sal con `Ctrl + C`.

### 9.2 🌐 Conectar Jenkins y SonarQube

| # | Dónde | Qué hacer |
|:-:|---|---|
| 1 | SonarQube `localhost:9000` | Ingresa con `admin` / `admin` y cambia la clave. |
| 2 | SonarQube → **My Account → Security** | **Generate Tokens**: Name `jenkins`, Type **Global Analysis Token**. Copia el token. |
| 3 | Jenkins → **Credentials → System → Global → Add Credentials** | Kind **Secret text**, Secret = token, ID `sonar-token`. |
| 4 | Jenkins → **System → SonarQube servers** | Marca **Environment variables**. Agrega: Name `SonarQube`, URL `http://sonarqube:9000`, token `sonar-token`. **Save**. |
| 5 | Jenkins → **Tools → SonarQube Scanner installations** | Name `SonarScanner`, ✔ *Install automatically*. **Save**. |
| 6 | SonarQube → **Administration → Configuration → Webhooks → Create** | Name `Jenkins`, URL `http://jenkins:8080/sonarqube-webhook/`. |
| 7 | SonarQube → **Quality Gates → Create** | Nombre `Lab-DevSecOps`, luego **Set as Default** (condiciones abajo). |

> ⚠️ Los nombres `SonarQube` y `SonarScanner` deben coincidir **exactamente** con los del `Jenkinsfile`. Si aparece `No tool named SonarScanner found`, deja `EJECUTAR_SONARQUBE` desmarcado hasta terminar la configuración.

**Condiciones del Quality Gate** (sobre *Overall Code*):

| Métrica | Operador | Valor |
|---|---|---|
| Coverage | is less than | 80 % |
| Security Rating | is worse than | A |
| Reliability Rating | is worse than | A |

### 9.3 🌐 Prueba normal con SonarQube

En Jenkins: **Construir con parámetros** → marca `EJECUTAR_SONARQUBE` → **Construir**.

- La etapa **5** sube el análisis y espera el veredicto (`waitForQualityGate abortPipeline: true`).
- Dashboard: <http://localhost:9000/dashboard?id=nutribox-api> (Bugs, Vulnerabilities, Security Hotspots, Code Smells, Coverage, Duplications).

```bash
curl -fsS http://localhost:8081/health
```

> ✅ **Quality Gate: Passed** y etapas 6–8 en verde.
>
> 📸 **Captura 6:** dashboard con Quality Gate aprobado.

### 9.4 🌐 Bloquear con el Quality Gate y recuperar

| # | Acción | Resultado |
|:-:|---|---|
| 1 | En **Quality Gates → Lab-DevSecOps**, cambia la cobertura de `80 %` a `100 %`. | — |
| 2 | Ejecuta **Build with Parameters** con `EJECUTAR_SONARQUBE` marcado. | El gate **falla** (el proyecto tiene 97,61 % de cobertura de ramas) y 6–8 quedan ⏭️ omitidas. 📸 **Captura 7** |
| 3 | Verifica con `docker inspect nutribox-staging --format "{{.Config.Image}}"`. | Staging conserva la última imagen aprobada. |
| 4 | Restaura la condición a `80 %` y ejecuta de nuevo. | **Quality Gate: Passed**, deploy y smoke test en 🟢. 📸 **Captura 8** |

---

## 10. Cierre y limpieza

**Detener conservando los datos** (para continuar otro día):

```bash
cd ~/lab-cicd/Laboratorio_Jenkins_NutriBox/infra-jenkins
docker compose --profile sonar stop
docker stop nutribox-staging
```

**Volver a iniciar:**

```bash
docker compose up -d
docker start nutribox-staging
```

**Eliminar todo** (contenedores, volúmenes con la configuración de Jenkins/SonarQube e imágenes del laboratorio):

```bash
docker rm -f nutribox-staging
docker compose --profile sonar down -v
docker rmi lab-jenkins:1.0
docker images nutribox-api -q | xargs -r docker rmi -f
```



## 11. Problemas frecuentes

| Síntoma | Causa probable | Solución |
|---|---|---|
| `Cannot connect to the Docker daemon` | Docker detenido o usuario fuera del grupo `docker` | Inicia Docker, revisa `groups` y reinicia sesión tras `sudo usermod -aG docker "$USER"`. |
| Puerto 8080 / 8081 / 9000 ocupado | Otro servicio lo usa | `sudo ss -ltnp \| grep -E ':8080\|:8081\|:9000'` y cierra el proceso, o cambia el puerto. |
| `localhost:8080` no abre | Jenkins aún no inicia o el puerto está ocupado | Revisa `docker compose ps`, `docker logs jenkins` y los puertos. |
| Etapa 1: `docker: not found` o `node: not found` | El job corre en otra imagen de Jenkins | `docker compose build --no-cache` y `docker compose up -d`. |
| Etapa 1: `permission denied … docker.sock` | Jenkins sin acceso al socket | Verifica `user: root` y el volumen `/var/run/docker.sock`; `docker compose up -d --force-recreate`. |
| `Couldn't find any revision to build` | Rama incorrecta | *Branch Specifier* `*/main` (o `*/master`; compruébalo con `git branch`). |
| `No such DSL method 'recordCoverage'` / `junit` | Falta un plugin | Revisa `plugins.txt` y reconstruye la imagen. |
| El push no dispara el build | `pollSCM` se registra tras la 1.ª ejecución | Ejecuta **Construir ahora** una vez; luego espera ≤ 2 min tras cada push. |
| No aparece *Construir con parámetros* | Igual que el anterior | Ejecuta el pipeline una vez. |
| `npm audit` falla con `ENOTFOUND registry.npmjs.org` | Sin Internet o proxy | Revisa la conexión; con proxy, `npm config set proxy …`. |
| Smoke test: `Could not resolve host: nutribox-staging` | Jenkins no está en `cicd-net` | `docker network inspect cicd-net` y verifica `jenkins` y `nutribox-staging`. |
| Etapa 5: `… does not have an install of "SonarScanner"` | Nombre de herramienta distinto | Usa exactamente `SonarScanner` en *Tools*. |
| Etapa 5 espera hasta el timeout | Falta el webhook | Crea `http://jenkins:8080/sonarqube-webhook/` en SonarQube. |
| SonarQube se reinicia constantemente | Poca memoria para Elasticsearch | Asigna más memoria a la máquina o VM. |
| `exit code 1` en SCA | **Es lo esperado** con vulnerabilidades altas o críticas | Lee el log: es un *fallo exitoso*. |

---

## 12. Debriefing y anexos

### 💬 Preguntas de cierre (5 min)

1. ¿Qué es más barato: detectar el IVA mal calculado en el pipeline o después de 3.200 pedidos?
2. Si fueras jefe de desarrollo, ¿autorizarías un despliegue con `|| true` en el `npm audit`? ¿En qué casos sí?
3. ¿Qué etapa agregarías después del despliegue en staging? (Pista: **DAST** con OWASP ZAP, pruebas E2E.)
4. ¿Qué ventajas tiene un `Jenkinsfile` versionado frente a configurar el pipeline con clics en la UI?

### Anexo A — Jenkins vs. GitHub Actions

| Concepto | GitHub Actions | Jenkins (`Jenkinsfile`) |
|---|---|---|
| Disparador | `on: push` | `triggers { pollSCM(...) }` o `githubPush()` |
| Trabajo / etapa | `jobs:` / `steps:` | `stages { stage { steps { } } }` |
| Dependencia entre trabajos | `needs: unit-tests` | Orden secuencial de `stage` |
| Paralelismo | Jobs sin `needs` | `parallel { }` |
| Continuar ante error | `continue-on-error: true` | `catchError(...)` o `\|\| true` |
| Ejecutor | `runs-on: [self-hosted, linux]` | `agent any` / `agent { docker { ... } }` |
| Reportes | `actions/upload-artifact` | `archiveArtifacts`, `junit`, `recordCoverage` |
| Secretos | `${{ secrets.X }}` | `withCredentials([...])` / `credentials('id')` |

### Anexo B — Comandos útiles

| Para qué | Comando |
|---|---|
| Ver el log de Jenkins en vivo | `docker logs -f jenkins` |
| Entrar al contenedor de Jenkins | `docker exec -it jenkins bash` |
| Ver el log de staging | `docker logs nutribox-staging` |
| Reiniciar Jenkins | `docker compose -f ~/lab-cicd/Laboratorio_Jenkins_NutriBox/infra-jenkins/docker-compose.yml restart jenkins` |
| Historial gráfico de Git | `git log --oneline --graph -n 15` |
