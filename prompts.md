# Registro de uso de IA

**Proyecto:** Alma Silvestre — Encuestas, Experiencias y Citas
**Autor:** Emilio Fernández Pérez
**Entrega:** 1 · Documentación técnica

---

## Flujo de trabajo con IA

### Herramientas

| Herramienta | Uso | Verificable en |
|---|---|---|
| **Claude Code** (CLI de Anthropic) | Entorno único de desarrollo asistido: generación del código PHP, especificación de cambios, documentación y operaciones git/GitHub. | `.claude/` en el repositorio |
| **OpenSpec** | Marco de gestión de cambios: cada mejora se especifica (propuesta → diseño → spec → tareas) antes de escribir código. | `openspec/` |
| **Git + GitHub CLI (`gh`)** | Control de versiones, creación del repositorio y del fork de entrega. | historial de git |

**Única familia de modelos usada: Claude.** No se usaron ChatGPT, Cursor, Copilot ni otros asistentes.

### Modelos y para qué

| Modelo | Trabajo que hizo | Criterio |
|---|---|---|
| **Claude Opus 4.7** | Núcleo del producto: páginas PHP, lógica de dominio en `lib.php`, correos, esquema de base de datos. Redacción de las 10 propuestas OpenSpec. | Modelo más capaz disponible en ese momento; el coste se asumió a cambio de coherencia arquitectónica entre archivos. |
| **Claude Opus 4.8 (contexto 1M)** | Subida a GitHub protegiendo secretos, primera documentación técnica (`docs/entrega-1.md`) y primer registro de IA (`docs/registro-ia.md`). | El contexto de 1M permite razonar sobre el repositorio completo (≈200 archivos) sin fragmentarlo. |
| **Claude Opus 5 (contexto 1M)** | Esta entrega: especificación OpenSpec del paquete documental, verificación del contrato real de los endpoints contra el código, y redacción de `readme.md` y `prompts.md`. | Igual criterio: el documento exige contrastar simultáneamente `schema.sql`, `lib.php`, las páginas de entrada y las propuestas OpenSpec. |

**No se usaron Sonnet ni Haiku en ninguna fase.** La decisión fue priorizar calidad y coherencia sobre coste, incluso para tareas mecánicas.

### Skills, subagentes, comandos y reglas

**Skills propias del proyecto** (`.claude/skills/`) — codifican las convenciones para que no dependan de que alguien recuerde aplicarlas:

| Skill | Qué hace |
|---|---|
| `marca` | Crea y audita páginas PHP para que sigan el sistema visual exacto (paleta, tipografías, constantes de `config.php`, textos RGPD). Fuente de verdad visual: `index.php`. |
| `db-tabla` | Añade una tabla nueva siguiendo el patrón del proyecto: `CREATE TABLE IF NOT EXISTS` en `db.php` + helpers con sentencias preparadas en `lib.php` + reflejo en `schema.sql`, los tres coherentes. |
| `seguridad-php` | Audita las páginas buscando SQL sin preparar, salida sin escapar, formularios sin CSRF, fallos de indexación y filtraciones RGPD. **Sustituye al linter**, que no se puede ejecutar porque el entorno de desarrollo no tiene PHP CLI. |
| `deploy-ftp` | Checklist guiado de subida a hosting compartido sin sobrescribir `config.php` del servidor ni filtrar secretos. |

**Flujo OpenSpec** — skills `openspec-propose`, `openspec-explore`, `openspec-apply-change`, `openspec-archive-change`, `openspec-sync-specs` y comandos slash equivalentes (`/opsx:propose`, `/opsx:apply`, `/opsx:archive`, …).

**Subagentes** — `Explore` (búsqueda de solo lectura) para mapear en paralelo el stack, las funcionalidades y las propuestas sin cargar el hilo principal. Plugin *cavecrew* disponible con `cavecrew-investigator` (localizar código), `cavecrew-builder` (edición acotada) y `cavecrew-reviewer` (revisión de diff), con salida comprimida.

**Reglas y configuración**

- **Modo Caveman**: hook `SessionStart` que activa comunicación ultracomprimida para reducir el consumo de tokens. El código, los commits y los avisos de seguridad se escriben en prosa normal, no comprimida.
- **Allowlist de permisos** (`.claude/settings.local.json`): comandos preaprobados (`openspec status/new/instructions`, `gh auth/repo`, `git checkout`) para reducir interrupciones por confirmación.
- No hay `CLAUDE.md`: las convenciones viven en las skills, que son ejecutables.

### Ajustes humanos sobre el output de la IA

| Situación | Intervención humana |
|---|---|
| La IA propuso crear `config.example.php` | **Rechazado.** Se mantuvo solo `.gitignore` excluyendo `config.php`. |
| Autenticación de GitHub | La IA no puede autenticarse sola: el humano generó los tokens y ajustó los *scopes* (`repo`, `read:org`) tras varios intentos fallidos. |
| Tokens pegados en el chat | La IA avisó de la exposición; la rotación del token es acción humana. |
| Contenido de `config.php` | Todos los valores reales (base de datos, dominio, correos, localidad SEO) introducidos por el humano. |
| **Contenido de las propuestas OpenSpec** | **El aporte humano decisivo.** La IA generó la estructura; el humano aportó el contexto de negocio que no está en el código y que ningún agente puede deducir: Redsys como pasarela primaria (encaje con banca española y hosting compartido) frente a Stripe; modelo de **señal** en vez de cobro total; capacidad de 1 persona por hueco 1:1; umbral de escasez; ventana de recordatorios; ubicación real para el SEO local; qué páginas van con `noindex`; y las preferencias de la clienta sobre tono y marca. |
| **Publicación de este repositorio** | La IA **bloqueó su propia escritura** en el fork público al detectar que volcaba documentación técnica de un sistema de cliente real sin autorización explícita. El humano revisó el borrador y decidió, punto por punto, qué se publica (detalle técnico y datos de negocio) y qué no (credenciales, host de base de datos, teléfono personal). |

---

## Índice

1. [Descripción general del producto](#1-descripción-general-del-producto)
2. [Arquitectura del sistema](#2-arquitectura-del-sistema)
3. [Modelo de datos](#3-modelo-de-datos)
4. [Especificación de la API](#4-especificación-de-la-api)
5. [Historias de usuario](#5-historias-de-usuario)
6. [Tickets de trabajo](#6-tickets-de-trabajo)
7. [Pull requests](#7-pull-requests)

---

## 1. Descripción general del producto

**Prompt 1** *(Opus 4.7 — origen del producto)*

> "Necesito una web para un negocio de experiencias en la naturaleza: una encuesta de satisfacción de 4 preguntas que recoja nombre, email y teléfono de WhatsApp, lo guarde en base de datos y permita exportar los contactos para newsletter y WhatsApp cumpliendo RGPD, exportando solo a quien dio consentimiento. PHP plano, sin framework, hosting compartido barato, subida por FTP."

*Resultado:* núcleo inicial (`encuesta.php`, `guardar.php`, `admin.php`, `exportar.php`, `db.php`, `lib.php`, `config.php`). Este prompt fijó las restricciones que gobiernan todo el proyecto: sin framework, sin build, sin dependencias.

**Prompt 2** *(Opus 4.7 — ampliación a reservas)*

> "Propón un cambio para vender plazas de experiencias grupales con señal por Bizum, teniendo en cuenta que la clienta cobra una señal y no el total, que no puede haber sobreventa de la última plaza, y que si el evento está completo la gente se tiene que poder apuntar a una lista de espera."

*Resultado:* módulo de eventos e inscripciones. El detalle de negocio del prompt (señal, no sobreventa, lista de espera) es lo que produjo la decisión técnica de resolver el aforo dentro de una transacción con `SELECT … FOR UPDATE`.

**Prompt 3** *(Opus 5 — esta entrega)*

> "Lo primero que vamos a hacer es crear un spec de entrega de documentación siguiendo estas recomendaciones. Es la primera parte, solo documentación, aunque nosotros ya tenemos más cosas. […] https://github.com/LIDR-academy/AI4Devs-finalproject esa sería la url final"

*Resultado:* change OpenSpec `add-entrega-1-documentacion` (propuesta, diseño, spec con 11 requisitos y 36 tareas) antes de escribir una línea del `readme.md`. Se aplicó al propio entregable el método que el proyecto usa para el código: especificar antes de producir.

---

## 2. Arquitectura del Sistema

### **2.1. Diagrama de arquitectura:**

**Prompt 1**

> "Convierte el diagrama de arquitectura en un bloque Mermaid que GitHub renderice nativamente, sin imágenes externas."

**Prompt 2**

> "Justifica por qué esta arquitectura y no un framework, y di también qué sacrificas con ella. No lo vendas como si fuera gratis."

*Ajuste humano:* la respuesta inicial listaba solo beneficios. Se pidió explícitamente la tabla de déficits asumidos (sin ORM, sin router, sin plantillas, sin tests), cada uno con su contrapartida real.

### **2.2. Descripción de componentes principales:**

**Prompt 1**

> "Mapea el stack, las funcionalidades y las propuestas OpenSpec pendientes."

*Ejecución:* delegado al subagente **Explore** en paralelo, para no cargar el hilo principal con volcados de archivos.

### **2.3. Descripción de alto nivel del proyecto y estructura de ficheros**

**Prompt 1**

> "Explica la estructura de ficheros diciendo qué patrón sigue, no solo listando carpetas."

*Resultado:* la explicación de "un archivo por caso de uso HTTP" y el motivo de que la raíz sea plana (el nombre del archivo *es* la URL).

### **2.4. Infraestructura y despliegue**

**Prompt 1**

> "Documenta la infraestructura real y el proceso de despliegue, incluido por qué no hay CI/CD."

### **2.5. Seguridad**

**Prompt 1** *(Opus 4.7 — durante el desarrollo)*

> "Revisa la seguridad antes de subir."

*Ejecución:* dispara la skill `seguridad-php`. Es el prompt más repetido del proyecto: se ejecuta antes de cada despliegue y sustituye al linter ausente.

**Prompt 2** *(Opus 5 — esta entrega)*

> "Extrae del código las medidas de seguridad reales, con el archivo donde está cada una. Nada que no puedas señalar en el código."

*Resultado:* la tabla de 12 prácticas con su ubicación. La restricción "nada que no puedas señalar" evitó que el documento reclamara defensas inexistentes.

### **2.6. Tests**

**Prompt 1**

> "Declara el estado real de los tests. Si no hay suite automatizada, dilo, y explica qué se verifica a mano y qué se planifica para la entrega final."

*Ajuste humano:* decisión explícita de **no** maquillar la ausencia de tests. Es la sección donde la tentación de inventar era mayor.

---

## 3. Modelo de Datos

**Prompt 1**

> "Verifica las tablas y sus relaciones contra `schema.sql` y `db.php` antes de escribir nada del modelo de datos."

*Resultado:* las 7 tablas documentadas con sus campos, tipos, índices y restricciones reales. La verificación previa destapó que **no existen claves foráneas declaradas**: las relaciones son lógicas y las mantiene la aplicación.

**Prompt 2**

> "Escribe el diagrama ER en Mermaid con el máximo detalle: claves primarias, foráneas, tipos y restricciones."

**Prompt 3**

> "Explica por qué `respuestas` no tiene relación con `eventos`, si eso fue una decisión o un descuido."

*Ajuste humano:* la respuesta correcta era de negocio, no técnica — se envían encuestas de actividades que nunca se publicaron como evento. La IA no podía saberlo; lo aportó el humano y quedó documentado como decisión deliberada.

---

## 4. Especificación de la API

**Prompt 1**

> "Este proyecto no tiene API REST. No inventes una. Documenta el contrato HTTP real de los tres POST que crean recursos, leyéndolo del código."

*Resultado:* especificación OpenAPI 3.0 con `application/x-www-form-urlencoded` de `guardar.php`, `inscribir.php` y `reservar-cita.php`.

**Prompt 2**

> "Los nombres de los campos tienen que coincidir exactamente con los que espera el código."

*Ejecución:* lectura directa de los tres archivos y de las funciones `crear_inscripcion` y `crear_cita` en `lib.php` antes de escribir el YAML. De ahí salieron detalles que no estaban en ninguna documentación previa: el consentimiento viaja como la cadena literal `"Sí"`, y `fecha_hora` se normaliza a `'Y-m-d H:i:00'`.

---

## 5. Historias de Usuario

**Prompt 1**

> "Desarrolla en formato completo las tres historias principales, con criterios de aceptación verificables, no intenciones."

**Prompt 2**

> "Elige las tres que mejor demuestren un flujo end-to-end y que cubran roles distintos, y di por qué descartas las demás."

*Resultado:* reserva 1:1 (flujo principal con estado, correos y concurrencia), inscripción grupal con lista de espera, y panel de administración. Se descartó la encuesta pese a ser la más antigua: es la más simple y no demuestra manejo de concurrencia ni de estado.

---

## 6. Tickets de Trabajo

**Prompt 1** *(Opus 4.7 — origen de las propuestas)*

> "Propón un cambio para que el cliente pueda cancelar o reprogramar su cita desde un enlace del email, sin cuenta ni contraseña, y que el hueco se libere solo."

*Resultado:* propuesta `add-autogestion-citas` (26 tareas), una de las 10 documentadas en `openspec/changes/`.

**Prompt 2** *(Opus 5 — esta entrega)*

> "Los tres tickets tienen que ser backend, frontend y base de datos de una misma funcionalidad real, no tres cosas sueltas."

*Resultado:* la funcionalidad de autogestión cortada por capas. Al descomponerla apareció un problema que la propuesta original no había detectado: la clave única `(servicio_id, fecha_hora)` **impide** reservar un hueco cuya cita anterior figure como cancelada. Por eso el ticket de base de datos precede al de backend.

**Prompt 3**

> "Cada ticket con criterios de aceptación, tareas técnicas, estimación y dependencias, de forma que otra persona pueda ejecutarlo sin preguntarme nada."

*Ajuste humano:* las estimaciones (8 h / 6 h / 4 h) las propuso la IA y las validó el humano; no proceden de medición real.

---

## 7. Pull Requests

**Prompt 1** *(Opus 4.8 — sesión previa)*

> "Vamos a subir esto a GitHub para generar el proyecto."

*Ejecución:* `git init`, `.gitignore` protegiendo `config.php`, `stats/` y `uploads/`, **verificación de que los secretos quedaban excluidos antes del primer commit**, y `gh repo create --private --push`. La propuesta de proteger los secretos antes de subir la hizo el agente, no el humano.

**Prompt 2** *(Opus 5 — esta entrega)*

> "¿Podemos generar una carpeta fuera del proyecto y eso sería para el fork? ¿Podemos volcar allí la entrega? ¿Y podemos hacer las secciones faltantes?"

*Resultado:* fork de `LIDR-academy/AI4Devs-finalproject` clonado **fuera** del repositorio de trabajo, con rama `feature/entrega-1-emiliofdz1`. La separación física resuelve el problema de fondo: el fork público recibe solo documentación, el código de la clienta permanece en el repositorio privado.

**Prompt 3** *(Opus 5 — control humano)*

> "Antes de entregar nada vamos a repasar todo bien."

*Resultado:* los documentos se redactaron como borradores dentro del repositorio privado (`docs/*-BORRADOR.md`) para revisión humana, y no se copió nada al fork público hasta la aprobación explícita, dato por dato, de qué se publica y qué no.
