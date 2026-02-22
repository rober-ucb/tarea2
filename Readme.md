# 🔐 DevSecOps CI/CD Pipeline — Justificación Técnica

> Pipeline de integración y entrega continua con seguridad integrada para la tarea 2

---

## 📋 Tabla de Contenidos

- [Resumen del Pipeline](#resumen-del-pipeline)
- [Justificación Técnica por Etapa](#justificación-técnica-por-etapa)
  - [Etapa 1 – Checkout del Repositorio](#etapa-1--checkout-del-repositorio)
  - [Etapa 2 – Setup del Entorno Node.js](#etapa-2--setup-del-entorno-nodejs)
  - [Etapa 3 – Configuración de Variables de Entorno](#etapa-3--configuración-de-variables-de-entorno)
  - [Etapa 4 – Instalación y Pruebas del Backend](#etapa-4--instalación-y-pruebas-del-backend)
  - [Etapa 5 – Instalación y Pruebas del Frontend](#etapa-5--instalación-y-pruebas-del-frontend)
  - [Etapa 6 – Análisis Estático de Seguridad (SAST) con Semgrep](#etapa-6--análisis-estático-de-seguridad-sast-con-semgrep)
  - [Etapa 7 – Build de Imágenes Docker](#etapa-7--build-de-imágenes-docker)
  - [Etapa 8 – Análisis de Composición de Software (SCA) con Trivy](#etapa-8--análisis-de-composición-de-software-sca-con-trivy)
  - [Etapa 9 – Smoke Test de Integración](#etapa-9--smoke-test-de-integración)
- [Tabla Resumen](#tabla-resumen)

---

## Resumen del Pipeline

Este pipeline se ejecuta automáticamente ante cada `push` a la rama `main` y en cada `pull_request`. Su propósito es garantizar que ningún cambio de código llegue a producción sin haber pasado por controles de **calidad**, **seguridad** y **funcionamiento mínimo** del sistema.

El pipeline cubre tres microservicios del backend (`users-service`, `academic-service`, `api-gateway`) y el frontend, aplicando las mismas validaciones a cada uno de forma sistemática.

```
push / pull_request
       │
       ▼
  Checkout ──► Setup Entorno ──► Config .env ──► Tests Backend
       │
       ▼
  Tests Frontend ──► SAST (Semgrep) ──► Docker Build ──► SCA (Trivy) ──► Smoke Test
```

---

## Justificación Técnica por Etapa

---

### Etapa 1 – Checkout del Repositorio

**Herramienta:** `actions/checkout@v4`

**Fase DevSecOps:** *Plan / Code*

**Qué hace:** Descarga el estado exacto del código fuente que disparó el pipeline, incluyendo todos los archivos del repositorio en la versión del commit o PR específico.

**Riesgo que mitiga:** Evita que el pipeline trabaje sobre una versión desactualizada o incorrecta del código.

**Por qué es necesaria:** Cada cambio de código, incluso uno aparentemente menor (un comentario, un refactor), puede introducir regresiones o vulnerabilidades. El checkout garantiza que se audita exactamente lo que se quiere desplegar, no una versión anterior cacheada. Es la base de toda trazabilidad en el pipeline.

---

### Etapa 2 – Setup del Entorno Node.js

**Herramienta:** `actions/setup-node@v4` (Node.js 20 LTS)

**Fase DevSecOps:** *Build*

**Qué hace:** Instala una versión fija y conocida de Node.js en el runner, garantizando reproducibilidad del entorno de ejecución.

**Riesgo que mitiga:** Elimina el riesgo de inconsistencias entre entornos. Versiones distintas de Node.js pueden interpretar dependencias o APIs del lenguaje de forma diferente, produciendo comportamientos inesperados en producción o en las pruebas.

**Por qué es necesaria:** Node.js recibe actualizaciones frecuentes. Fijar la versión a la LTS (Long-Term Support) activa protege contra cambios de comportamiento y asegura que el entorno de CI es idéntico al entorno de desarrollo y producción. Sin esto, una actualización silenciosa del runner podría romper el sistema en cualquier momento.

---

### Etapa 3 – Configuración de Variables de Entorno

**Herramienta:** Secrets de GitHub Actions + archivos `.env`

**Fase DevSecOps:** *Seguridad de configuración / Secrets Management*

**Qué hace:** Copia los archivos `.env.example` como base y los completa con credenciales reales almacenadas como Secrets cifrados en GitHub (`secrets.USERS_DB_HOST`, `secrets.USERS_DB_PASSWORD`, etc.). Cada microservicio recibe únicamente las variables que le corresponden.

**Riesgo que mitiga:** Previene dos riesgos críticos:
- **Exposición de credenciales**: las contraseñas y URLs de base de datos nunca se escriben directamente en el código fuente ni en el historial de Git.
- **Configuración incorrecta**: cada servicio se conecta a su propia base de datos (principio de mínimo privilegio), evitando que un microservicio acceda a datos que no le pertenecen.

**Por qué es necesaria:** Las credenciales filtradas son la causa número uno de brechas de seguridad en sistemas en producción. Un sistema funcional con credenciales hardcodeadas en el repositorio es un sistema comprometido esperando ser descubierto. GitHub Secrets garantiza que las credenciales estén cifradas en reposo y nunca aparezcan en los logs del pipeline.

---

### Etapa 4 – Instalación y Pruebas del Backend

**Herramienta:** `npm ci` + framework de testing de cada servicio (Jest / Mocha)

**Fase DevSecOps:** *Test / Calidad*

**Qué hace:** Para cada microservicio (`users-service`, `academic-service`, `api-gateway`), instala las dependencias de forma determinista usando el `package-lock.json` y ejecuta la suite de pruebas automatizadas.

**Diferencia entre `npm ci` y `npm install`:** `npm ci` elimina `node_modules` antes de instalar, respeta exactamente las versiones del lockfile y falla si hay discrepancias. Esto garantiza reproducibilidad total.

**Riesgo que mitiga:** Detecta regresiones funcionales antes de que lleguen a producción. Un cambio en la lógica de predicción de lactancia, por ejemplo, podría romper cálculos de producción sin que nadie lo note si no existe una batería de pruebas automatizada.

**Por qué es necesaria:** Un sistema funcional hoy puede dejar de serlo mañana por un cambio aparentemente inocente. Las pruebas automatizadas son la red de seguridad que permite al equipo desarrollar con confianza. Sin ellas, cualquier cambio es un riesgo no cuantificado.

---

### Etapa 5 – Instalación y Pruebas del Frontend

**Herramienta:** `npm ci` + Vitest / Jest (según configuración de Vite)

**Fase DevSecOps:** *Test / Calidad*

**Qué hace:** Replica el proceso de la etapa anterior para el frontend: instala dependencias con la variable `VITE_API_URL` configurada para apuntar al API Gateway interno, y ejecuta las pruebas unitarias de componentes.

**Riesgo que mitiga:** Asegura que la interfaz de usuario se comporta correctamente con la URL del API definida para el entorno de CI. Evita que el frontend llegue a producción apuntando a una URL de desarrollo o local.

**Por qué es necesaria:** El frontend es la capa más visible para el usuario final (ganaderos, veterinarios, administradores). Un error en la UI puede generar decisiones de manejo animal incorrectas basadas en datos mal presentados, con consecuencias directas en la producción.

---

### Etapa 6 – Análisis Estático de Seguridad (SAST) con Semgrep

**Herramienta:** [Semgrep](https://semgrep.dev/) (`--config=auto --severity=ERROR`)

**Fase DevSecOps:** *Secure Code Review / Shift-Left Security*

**Qué hace:** Analiza el código fuente de cada microservicio sin ejecutarlo, buscando patrones conocidos de vulnerabilidades de seguridad: inyecciones SQL, uso inseguro de funciones criptográficas, manejo incorrecto de tokens, exposición de datos sensibles en logs, entre otros. La bandera `--severity=ERROR` hace que el pipeline falle únicamente ante hallazgos críticos, evitando falsos positivos que bloqueen el desarrollo por problemas menores.

**Riesgo que mitiga:** Detecta vulnerabilidades de seguridad en el código fuente antes de que el sistema se construya o despliegue. Es especialmente relevante en un sistema que maneja datos productivos y sanitarios de animales, donde la integridad de la información es crítica.

**Por qué es necesaria:** Las vulnerabilidades en código fuente no desaparecen porque el sistema funcione correctamente. Una API que no valida correctamente el input puede ser explotada desde el primer día en producción. Semgrep actúa como un revisor de seguridad automático que acompaña a cada commit, a diferencia de las auditorías manuales que son costosas y esporádicas.

---

### Etapa 7 – Build de Imágenes Docker

**Herramienta:** Docker Buildx + `docker compose build`

**Fase DevSecOps:** *Build / Empaquetado*

**Qué hace:** Construye las imágenes de contenedor de todos los microservicios usando Docker Compose como orquestador de la construcción. Docker Buildx habilita capacidades avanzadas como builds multiplataforma y caché optimizada.

**Riesgo que mitiga:** Garantiza que el código puede empaquetarse correctamente en contenedores antes de ser analizado por el escáner de vulnerabilidades. Un build fallido aquí indica problemas en el `Dockerfile` (dependencias del sistema operativo faltantes, permisos incorrectos, etc.) que de otro modo solo se descubrirían al desplegar.

**Por qué es necesaria:** Los contenedores encapsulan el entorno de ejecución. Si el `Dockerfile` tiene instrucciones desactualizadas o instala versiones vulnerables de librerías del sistema operativo, el contenedor construido en producción será diferente al que se probó manualmente. La construcción automatizada garantiza reproducibilidad total.

---

### Etapa 8 – Análisis de Composición de Software (SCA) con Trivy

**Herramienta:** [Trivy](https://trivy.dev/) (`aquasecurity/trivy-action@0.20.0`) — `severity: CRITICAL`, `exit-code: 1`

**Fase DevSecOps:** *SCA / Container Security*

**Qué hace:** Escanea cada imagen Docker construida en la etapa anterior en busca de vulnerabilidades conocidas (CVEs) en: dependencias de Node.js (`node_modules`), librerías del sistema operativo base (Ubuntu/Alpine), y configuraciones inseguras del contenedor. El pipeline falla automáticamente si se detecta alguna vulnerabilidad de severidad CRITICAL.

**Diferencia con Semgrep (SAST):** Mientras Semgrep analiza el código que el equipo escribe, Trivy analiza las dependencias de terceros y el sistema operativo base, zonas sobre las que el equipo no tiene control directo pero que forman parte del sistema desplegado.

**Riesgo que mitiga:** Las dependencias de terceros son el vector de ataque más común en aplicaciones modernas (ataques a la cadena de suministro). Una librería como `express`, `jsonwebtoken` o incluso la imagen base de Node.js pueden contener vulnerabilidades críticas descubiertas después de su publicación. Trivy detecta estas vulnerabilidades antes del despliegue.

**Por qué es necesaria:** Las vulnerabilidades en dependencias no se introducen con cambios de código propios, sino con el paso del tiempo, cuando investigadores de seguridad publican nuevos CVEs sobre librerías que el sistema ya usa. Sin Trivy en el pipeline, el sistema podría estar ejecutando en producción una dependencia comprometida durante semanas sin saberlo.

---

### Etapa 9 – Smoke Test de Integración

**Herramienta:** `docker compose up` + `curl` al endpoint `/health`

**Fase DevSecOps:** *Test de integración / Validación de despliegue*

**Qué hace:** Levanta todos los microservicios juntos usando Docker Compose, espera 15 segundos para que los servicios inicialicen completamente, y realiza una petición HTTP al endpoint `/health` del API Gateway. Si el gateway no responde con HTTP 200, el pipeline falla. Al finalizar (con éxito o error), los contenedores se apagan limpiamente con `if: always()`.

**Riesgo que mitiga:** Verifica que los microservicios pueden arrancar, comunicarse entre sí y responder peticiones en un entorno que replica producción. Un sistema puede pasar todas las pruebas unitarias y aun así fallar al intentar conectar con otros servicios o con la base de datos.

---

## Tabla Resumen

| Etapa | Herramienta | Fase DevSecOps | Riesgo que Mitiga |
|---|---|---|---|
| Checkout | `actions/checkout@v4` | Code | Trabajo sobre versión incorrecta del código |
| Setup Entorno | `actions/setup-node@v4` | Build | Inconsistencias entre entornos de ejecución |
| Config .env | GitHub Secrets | Secrets Mgmt | Exposición de credenciales en el código fuente |
| Test Backend | `npm ci` + Jest/Mocha | Test | Regresiones funcionales en lógica de negocio |
| Test Frontend | `npm ci` + Vitest | Test | Errores en la interfaz de usuario |
| SAST | Semgrep | Secure Code | Vulnerabilidades en código propio (inyecciones, etc.) |
| Docker Build | Docker Buildx | Build | Problemas de empaquetado y Dockerfiles inválidos |
| SCA | Trivy | Container Security | CVEs en dependencias y sistema operativo base |
| Smoke Test | `curl` + Docker Compose | Integration Test | Fallos de integración entre microservicios |

---

*Pipeline diseñado bajo el enfoque DevSecOps: seguridad integrada en cada etapa del ciclo de desarrollo, no como paso final.*