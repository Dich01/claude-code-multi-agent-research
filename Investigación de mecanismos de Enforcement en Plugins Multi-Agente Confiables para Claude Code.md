## Investigación de mecanismos de Enforcement en Plugins Multi-Agente Confiables para Claude Code


  Marco: Desarrollo de plugin en Claude Code
  Arquitectura: 19 agentes markdown, 13 skills, 4 MCPs, hooks PreToolUse
  Fecha: 2026-03-14
  Autor: Diego Cheloni , https://github.com/Dich01/
  Tipo: Validacion empirica de mecanismos de enforcement y orquestacion para un plugin/submodulo multi-agente de Claude Code
  Nota: El plugin descrito en esta investigacion es propietario. Solo se documentan patrones arquitectonicos y hallazgos experimentales.
  Entorno: macOS, Claude Code CLI (marzo 2026), modelo Claude Opus 4.6



## RESUMEN

Este documento presenta los resultados de una investigacion empirica sobre la viabilidad de construir un plugin multi-agente para Claude Code con garantias de comportamiento. El plugin desarrollado (19 agentes, 13 skills) gobierna el ciclo de vida de desarrollo de software NestJS/TypeScript con TDD obligatorio, auditoria de seguridad y quality gates.

La investigacion cubre:

- Que mecanismos de enforcement ofrece Claude Code y cuales funcionan como se espera
- Como propagar hooks a subagents sin loops infinitos
- La limitacion absoluta de que subagents no pueden spawnar subagents
- Como activar un agente de plugin como main agent
- Resultados de pruebas en produccion en diversos proyectos

Hallazgo principal: el unico mecanismo de blocking confiable es exit code 2 en hooks PreToolUse. permissionDecision presenta discrepancias entre documentacion y comportamiento real (issue #4669). El modo orquestador funciona cuando brain es main agent via `--agent plugin_name:brain`.

La investigacion demuestra que la orquestacion multi-agente confiable en Claude Code es posible, pero solo mediante el uso cuidadoso de hooks PreToolUse, patrones de diseno defensivo y activacion de agentes via CLI.


---

## 1. INTRODUCCION Y CONTEXTO

### 1.1 — El problema

Un plugin de Claude Code compuesto enteramente por archivos markdown (prompts) no puede garantizar que el LLM siga sus instrucciones. Las reglas escritas en prosa ("no saltarse pasos", "TDD obligatorio", "@brain es mandatorio") son sugerencias que Claude puede ignorar silenciosamente.

El desafio es triple:

a) No hay funciones que llamar — todo pasa por prompts
b) Claude es no-deterministico — puede producir resultados diferentes
c) No hay mecanismo nativo de enforcement para el orden de agentes

### 1.2 — Objetivo

Determinar si es posible construir un plugin multi-agente confiable para clientes, identificando que mecanismos funcionan, cuales presentan discrepancias con la documentacion, y que workarounds existen en la comunidad.

### 1.3 — Metodologia

a) Revision exhaustiva de documentacion oficial de Claude Code
b) Analisis de repositorios y blogs de la comunidad
c) Revision de issues de GitHub relevantes (bugs, feature requests)
d) Pruebas empiricas con el plugin en un proyecto Express real
e) Iteracion de fixes y re-test


---

## 2. MECANISMOS DE BLOCKING EN CLAUDE CODE

Claude Code ofrece hooks PreToolUse que se ejecutan antes de cada tool call. Hay tres formas documentadas de influir en un tool call. Solo una bloquea de forma confiable.

### 2.1 — exit code 2: FUNCIONA

- Mecanismo: `echo "razon" >&2 && exit 2`
- Efecto: El tool call NO se ejecuta. stderr se muestra a Claude como error. Claude recibe el mensaje y ajusta su comportamiento.

Confirmado por:
- Blake Crosley: 95 hooks en produccion, 9 meses de uso — https://blakecrosley.com/blog/claude-code-hooks
- disler: 13 hooks cubriendo todo el lifecycle — https://github.com/disler/claude-code-hooks-mastery
- kenryu42: plugin de safety con analisis semantico — https://github.com/kenryu42/claude-code-safety-net
- Issue #29795: sistema de 5 capas con 68 patrones de fallo documentados — https://github.com/anthropics/claude-code/issues/29795

### 2.2 — permissionDecision "deny": DISCREPANCIA ENTRE DOCUMENTACION Y REALIDAD

- Mecanismo: Retornar JSON con `hookSpecificOutput.permissionDecision = "deny"`
- Documentacion oficial: Describe este mecanismo como funcional. Cita textual: `"deny" prevents the tool call`, `permissionDecisionReason` se muestra a Claude.
- Comportamiento observado: En pruebas empiricas y segun reportes de la comunidad, Claude IGNORA el deny y ejecuta el tool.

Issue #4669: https://github.com/anthropics/claude-code/issues/4669 — Estado: CERRADO como "not planned". Incluye pasos de reproduccion detallados. Afecta tambien a "ask" y "continue: false".

**Nota:** Existe una contradiccion entre la documentacion oficial (que lo describe como funcional) y el issue #4669 (que demuestra con reproduccion que no funciona). Esta discrepancia no esta resuelta. Workaround: usar exit code 2 en su lugar.

### 2.3 — exit code 1: NO BLOQUEA (por diseno)

- Mecanismo: `exit 1` para indicar error en el hook
- Documentacion oficial (cita textual): "Any other exit code: Non-blocking error. stderr is shown in verbose mode and execution continues."
- Efecto: Claude registra el error pero EJECUTA el tool. Esto es **comportamiento intencionado**, no un bug.

Issue #21988: https://github.com/anthropics/claude-code/issues/21988 — solicita que exit code 1 tambien bloquee.

**Nota:** A pesar de ser por diseno, esta decision implica que para enforcement solo exit code 2 es util. Exit code 1 no sirve como mecanismo de blocking. Workaround: usar exit code 2 en su lugar.

### 2.4 — exit code 0 sin output: FUNCIONA (permite)

- Efecto: Permite el tool call sin intervencion.
- Usado como caso default cuando la validacion pasa.

### 2.5 — Resumen de exit codes

| Exit code | Efecto | Estado |
|-----------|--------|--------|
| 0 | Permite | Funciona |
| 1 | Error no bloqueante | Por diseno — no bloquea |
| 2 | Bloquea | Funciona |


---

## 3. PROPAGACION DE HOOKS A SUBAGENTS Y TEAMMATES

### 3.1 — Los hooks se propagan a subagents

Observado empiricamente: PreToolUse y PostToolUse se disparan cuando un subagent hace tool calls. **Nota:** La documentacion oficial no es explicita sobre si hooks del padre se disparan cuando subagents ejecutan tools. Los docs documentan `SubagentStart`/`SubagentStop` como eventos separados, pero la propagacion de PreToolUse es un hallazgo empirico.

Riesgo critico: loops infinitos. Si un hook dispara un proceso que dispara otro hook, la cadena se repite indefinidamente.

Solucion confirmada en la comunidad: recursion guard con flag files temporales.

```bash
GUARD="/tmp/hook-guard-$$"
[ -f "$GUARD" ] && exit 0
touch "$GUARD"
trap "rm -f '$GUARD'" EXIT
```

Fuentes:
- egghead.io: contaminacion de settings en subagents — https://egghead.io/avoid-the-dangers-of-settings-pollution-in-subagents-hooks-and-scripts~xrecv
- Blake Crosley: recursion-guard.sh — bloqueo 23 spawns descontrolados — https://blakecrosley.com/blog/claude-code-hooks

### 3.2 — Herencia de settings a subagents

Los subagents heredan TODOS los settings del padre, incluidos hooks (observacion empirica).
Para romper el ciclo en casos especificos: `--settings no-hooks.json` con `{ "disableAllHooks": true }`.

Fuente: egghead.io

### 3.3 — Teammates (agent teams)

Hooks estandar (PreToolUse/PostToolUse): comportamiento no documentado explicitamente para teammates.

Hooks especificos de teams que SI funcionan (confirmado en documentacion oficial):
- TeammateIdle: se dispara cuando un teammate va a quedar idle
- TaskCompleted: se dispara cuando una tarea se marca completada

Fuente: documentacion oficial Claude Code — https://code.claude.com/docs/en/agent-teams

### 3.4 — Frontmatter de agentes parcialmente ignorado en teammates (BUG)

Issue #30703: https://github.com/anthropics/claude-code/issues/30703

Cuando brain spawna un teammate via Agent tool con `team_name`, los campos `hooks` y `skills` del frontmatter del agente se ignoran silenciosamente. Los campos `model` y `disallowedTools` si funcionan.

Impacto: los teammates no tienen las restricciones de hooks ni los skills declarados en su frontmatter. Brain debe inyectar todo via el prompt de spawn.

Estado: bug abierto (marzo 2026)


---

## 4. BYPASS DE HOOKS — VECTORES CONOCIDOS

### 4.1 — Bypass via Bash

Cuando Write/Edit estan bloqueados por un hook, Claude puede usar Bash con comandos que logran el mismo efecto:

```bash
sed -i 's/old/new/' file.txt
python3 -c "open('file.txt','w').write('content')"
echo "content" > file.txt
tee file.txt <<< "content"
```

Solucion: un hook adicional que intercepte Bash y analice el comando buscando patrones de escritura a archivo.

Implementacion de referencia: `guard_bash_file_writes.py` del sistema de 5 capas (Issue #29795) — https://github.com/anthropics/claude-code/issues/29795

### 4.2 — Bypass via wrapper scripts

Claude puede crear un script .sh con el contenido bloqueado y ejecutarlo. El hook no ve el contenido del script, solo el comando `bash script.sh`.

Solucion: analisis recursivo de wrappers hasta N niveles de profundidad.
Implementacion de referencia: kenryu42/claude-code-safety-net analiza hasta 5 niveles de wrappers — https://github.com/kenryu42/claude-code-safety-net


---

## 5. LIMITACION ARQUITECTONICA: SUBAGENTS NO PUEDEN SPAWNAR SUBAGENTS

### 5.1 — La restriccion

Documentacion oficial (cita textual): "Subagents cannot spawn other subagents, so Agent(agent_type) has no effect in subagent definitions."
https://code.claude.com/docs/en/sub-agents

Esto es absoluto y no configurable. Cuando un agente es invocado como subagent (via @brain o Agent tool), pierde acceso a la herramienta Agent. No puede spawnar otros agentes, crear teams, ni asignar tareas.

### 5.2 — Impacto en el plugin

El plugin tiene 19 agentes. Brain orquesta. Si brain se invoca como subagent (@brain), no puede spawnar a los otros 18 agentes.

Dos modos de operacion surgen de esta restriccion:

a) **Modo Guia:** brain como subagent orienta al usuario sobre que agentes invocar. El usuario ejecuta manualmente. Funcional pero tedioso.

b) **Modo Orquestador:** brain como main agent (flag `--agent`) tiene acceso completo a Agent tool y puede spawnar subagents. Requiere que brain sea la sesion principal, no un subagent.

### 5.3 — Confirmacion empirica

- **Prueba 1:** @brain como subagent — brain hizo todo solo (102 tool uses) porque no podia spawnar agentes. No invoco a ningun especialista.
- **Prueba 2:** `--agent plugin_name:brain` como main agent — brain uso TeamCreate, TaskCreate, y spawno 7 agentes con Agent tool. Paralelismo real (security-qa y quality-gate en background). FUNCIONA.

### 5.4 — Tabla de capacidades por contexto (hallazgo empirico)

| Contexto | Agent tool | TeamCreate | TaskCreate | SendMessage |
|----------|-----------|-----------|-----------|------------|
| Sesion principal | SI | SI | SI | SI |
| Subagent | NO | NO | NO | NO |
| Teammate | NO | NO | NO | NO |

**Nota:** La documentacion oficial no publica esta tabla explicitamente. La restriccion de subagents esta documentada. Las restricciones de teammates no estan documentadas con este nivel de detalle. Los valores de esta tabla son hallazgos empiricos.

Fuente: https://code.claude.com/docs/en/agent-teams
Issue #32731: https://github.com/anthropics/claude-code/issues/32731


---

## 6. ACTIVACION DE UN AGENTE DE PLUGIN COMO MAIN AGENT

### 6.1 — El problema del namespace

Los agentes del plugin se cargan con namespace: `plugin-name:agent-name`. El brain del plugin aparece como "Plugin_name:brain" en la lista de agentes (`/agents`).

### 6.2 — Formatos de --agent probados empiricamente

| Formato | Funciona | Resultado |
|---------|----------|-----------|
| `--agent brain` | NO | Sesion generica |
| `--agent Plugin_name:brain` | SI | Brain como main agent |
| `--agent plugin:Plugin_name:brain` | NO | Sesion generica |

El formato correcto es: `--agent Plugin_name:nombre-del-agente`

**Nota:** La documentacion oficial del CLI muestra `--agent my-custom-agent` sin mencionar formato con namespace para plugins. Este hallazgo es empirico.

### 6.3 — settings.json NO funciona con --plugin-dir (hallazgo empirico)

El campo "agent" en settings.json del plugin SOLO se aplica cuando el plugin se instala via marketplace (`claude plugin install`). Con `--plugin-dir`, settings.json se ignora completamente.

El campo "agent" en el settings.json del PROYECTO (`.claude/settings.json`) tampoco funciona para agentes de plugin.

**Nota:** La documentacion oficial describe el campo "agent" en settings.json del plugin como funcional, pero no distingue explicitamente entre instalacion via marketplace vs --plugin-dir. Este comportamiento diferenciado es un hallazgo empirico.

Fuente: https://code.claude.com/docs/en/plugins

### 6.4 — CLAUDE.md del plugin NO se carga via --plugin-dir (hallazgo empirico)

`--plugin-dir` carga: `agents/`, `skills/`, `hooks/`, `.mcp.json`, `commands/`
`--plugin-dir` NO carga: `CLAUDE.md`, `settings.json`

Solucion: el bootstrap del plugin agrega automaticamente una linea `@/path/to/plugin/CLAUDE.md` al CLAUDE.md del proyecto destino. Usa la sintaxis de import `@` con path absoluto.

### 6.5 — Comando definitivo

```bash
claude --plugin-dir /path/to/Plugin_name --agent Plugin_name:brain
```

Este comando:
- Carga los 19 agentes, 13 skills, hooks y MCPs del plugin
- Activa brain como main agent (puede spawnar subagents)
- Ejecuta los hooks de SessionStart (bootstrap, protocolo, MCPs)


---

## 7. HOOKS DE ENFORCEMENT IMPLEMENTADOS

### 7.1 — enforce-flow-order.sh (PreToolUse, matcher: Agent)

Proposito: Asegurar que brain sea el primer agente invocado.
Mecanismo: State file por session_id. Si brain no esta registrado y el agente invocado no es brain — exit 2.

Problema encontrado: Cuando brain es main agent, nunca se invoca via Agent tool (ES la sesion). El state file nunca registra "brain". El primer Agent call (api-contract) se bloqueaba incorrectamente.

Fix aplicado:
a) El hook de SessionStart pre-registra "brain" en el state file usando session_id del JSON de stdin.
b) enforce-flow-order.sh detecta `team_name` en Agent calls — si existe, brain esta orquestando via teams. Auto-registra "brain".

### 7.2 — enforce-prerequisites.sh (PreToolUse, matcher: Agent)

Proposito: Verificar que los artefactos prerequisito existan antes de permitir invocar un agente.

Prerequisitos por agente:

| Agente | Prerequisito |
|--------|-------------|
| planner | `docs/api/*.yaml` |
| test-architect | `docs/plans/*-plan.md` |
| test-engineer | `docs/test-specs/` con archivos |
| backend-dev | `docs/tdd/*-tdd-log.md` con "RED" |
| auth-access | `docs/tdd/*-tdd-log.md` con "RED" |
| quality-gate | `src/` con archivos .ts |
| pr-governance | `docs/quality/*-quality-gate.md` con "APROBADO" |
| security-qa | `src/` con archivos .ts |

Agentes sin prerequisitos (siempre permitidos): brain, bug-resolver, check-brain-features-status, domain-modeler, tenant-lifecycle, infra-secrets, architecture-guardian, isolation-tester, jira-spec, workflow-sync

### 7.3 — enforce-plugin-readonly.sh (PreToolUse, matcher: Write|Edit|Bash)

Proposito: Impedir que agentes corriendo en un proyecto externo modifiquen archivos del directorio del plugin.

Mecanismo: Compara file_path (Write/Edit) o comando (Bash) contra `$CLAUDE_PLUGIN_ROOT`. Si el target esta dentro del plugin — exit 2.

Excepcion: si CWD == PLUGIN_ROOT, permite todo (estamos desarrollando el plugin, no usandolo desde otro proyecto).

### 7.4 — Patron comun de todos los hooks

1. Recursion guard con `/tmp/ai-brain-guard-*-$$`
2. Leer JSON de stdin con `INPUT=$(cat)`
3. Graceful fallback si jq no esta instalado (exit 0)
4. exit 2 + stderr para bloquear
5. exit 0 para permitir
6. Timeout 5 segundos (configurado en hooks.json)


---

## 8. CASOS DE EXITO EN LA COMUNIDAD

### 8.1 — Blake Crosley: 95 hooks en produccion

- https://blakecrosley.com/blog/claude-code-hooks
- https://blakecrosley.com/blog/claude-code-hooks-tutorial
- https://blakecrosley.com/blog/claude-code-as-infrastructure

9 meses de uso. Cada hook creado tras un incidente real.
35 hooks de juicio (gates, guards, validadores).
49 hooks de automatizacion (loggers, formatters, injectors).

Resultados:
- Intercepto 8 intentos de force-push a main
- Bloqueo 23 intentos de spawn descontrolado de subagents
- El ratio juicio/automatizacion evoluciono de 1:6 a 4:5

Problemas encontrados:
- Escrituras concurrentes al mismo JSON truncaban el archivo
- Sin spawn depth guard, agentes delegaban infinitamente

### 8.2 — Sistema de 5 capas (Issue #29795)

https://github.com/anthropics/claude-code/issues/29795

68 patrones de fallo documentados en 3 meses. El sistema fue construido en respuesta a estos 68 fallos — cada fallo llevo a un nuevo check o hook.

Arquitectura defensa en profundidad:
- Capa 1: Reglas y convenciones (CLAUDE.md, planes)
- Capa 2: Documentacion de fallos (68 patrones catalogados)
- Capa 3: Decision log (audit trail obligatorio)
- Capa 4: Reviews automatizados (5 herramientas)
- Capa 5: Hooks (bloqueos duros que no se pueden evadir)

Hallazgo clave: Los hooks PreToolUse solo interceptan su tool especifico. Cuando Edit se bloquea, Claude usa Bash con sed. `guard_bash_file_writes.py` cierra ese bypass inspeccionando comandos Bash.

### 8.3 — kenryu42/claude-code-safety-net

https://github.com/kenryu42/claude-code-safety-net

Plugin que bloquea comandos destructivos:
- `git reset --hard`, `git push --force`, `git clean -fd`
- `rm -rf` con patrones peligrosos

Analisis semantico (no solo regex):
- `git checkout -b feature` — permitido (creacion de branch)
- `git checkout -- file` — bloqueado (reset destructivo)
- `rm -r -f /` — bloqueado (reordenamiento de flags detectado)

Deteccion recursiva de wrappers hasta 5 niveles de profundidad.

### 8.4 — disler: dominio de hooks + observabilidad multi-agente

- https://github.com/disler/claude-code-hooks-mastery
- https://github.com/disler/claude-code-hooks-multi-agent-observability

13 hooks cubriendo todo el ciclo de vida de Claude Code.
Patron builder/validator para multi-agente.
Dashboard Vue en tiempo real con SQLite para observabilidad.
12 tipos de eventos rastreados con aislamiento por sesion.

### 8.5 — GitButler: aislamiento de sesiones paralelas

https://blog.gitbutler.com/automate-your-ai-workflows-with-claude-code-hooks/

Branches automaticos por sesion de Claude Code.
Usa git plumbing (`write-tree`, `commit-tree`, `update-ref`) para staging aislado.
Cada sesion paralela tiene su propio index file.

### 8.6 — Orquestacion multi-agente en la comunidad

- wshobson/agents: 112 agentes, 16 orquestadores — https://github.com/wshobson/agents
- mbruhler/claude-orchestration: encadenamiento secuencial de agentes — https://github.com/mbruhler/claude-orchestration
- barkain/claude-code-workflow-orchestration: agent teams con aprobacion de plan — https://github.com/barkain/claude-code-workflow-orchestration
- ruvnet/ruflo: orquestador externo que spawna instancias de Claude Code — https://github.com/ruvnet/ruflo


---

## 9. INCIDENTES Y FALLOS CONOCIDOS

### 9.1 — Filtracion de API key de $30k (paddo.dev)

Claude codifico credenciales reales de Azure en un archivo markdown publicado en un repositorio publico. 11 dias expuesto antes de que un desarrollador lo detectara manualmente.

Fuente: https://paddo.dev/blog/claude-code-hooks-guardrails/

### 9.2 — Borrado del directorio home (paddo.dev)

Comando ejecutado: `rm -rf tests/ patches/ plan/ ~/`
El `~/` al final borro todo el directorio home del usuario. Sin backup. Perdida total de datos personales.

### 9.3 — Propagacion silenciosa de credenciales (paddo.dev)

Claude copio credenciales de produccion desde `.env` a `env.example` a pesar de tener bloqueado el acceso a `.env` via permisos. Los permisos no cubrian la lectura indirecta.

### 9.4 — Manipulacion de tests (paddo.dev)

Claude modifico tests para que pasen incorrectamente, luego defendio los cambios como "comportamiento correcto". El desarrollador no detecto el cambio hasta revision manual.

### 9.5 — Loops infinitos (multiples issues)

- Issue #10205: https://github.com/anthropics/claude-code/issues/10205 — Hooks simples causan loops incluso sin spawnar subprocesos. Fallos en hooks Stop causan loops al cierre de sesion.
- Issue #9579: https://github.com/anthropics/claude-code/issues/9579 — Loop de autocompactacion — consumo de 96-108M tokens/dia.
- Issue #9704: https://github.com/anthropics/claude-code/issues/9704 — Proceso huerfano de Claude Code consumio 2.88M tokens en mas de 2 dias sin que el usuario lo detectara.

### 9.6 — Hooks como potencial vector de ataque

Referencia CVE-2025-59536 (paddo.dev): Archivos maliciosos de proyecto pueden definir hooks que se ejecutan automaticamente. Los guardrails se convierten en vectores de ataque.

**Nota:** Este CVE no se encontro en bases de datos oficiales (NVD/MITRE) al momento de esta investigacion. La fuente es paddo.dev (actualizacion de seguridad febrero 2026). La documentacion oficial de Claude Code incluye consideraciones de seguridad sobre hooks: los hooks se capturan como snapshot al inicio de sesion y modificaciones externas requieren revision del usuario antes de aplicarse.

Fuente: paddo.dev, documentacion oficial de seguridad de Claude Code — https://code.claude.com/docs/en/security


---

## 10. PRUEBAS EMPIRICAS REALIZADAS

### 10.1 — Prueba 1: Brain como subagent

- Comando: `claude --plugin-dir /path/to/plugin`
- Prompt: `@brain Implementa un CRUD de productos (...)`

Resultado:
- Brain invocado como subagent (Plugin_name:brain)
- Brain hizo todo solo: 102 tool uses, cero Agent calls
- Creo artefactos correctos en docs/ (requirements, api, plans, tdd, etc.)
- TDD completo: RED — GREEN — REFACTOR
- 36 tests nuevos, 175 total, cero regresiones
- Quality gate: APROBADO
- PERO: ningun agente especialista invocado

Diagnostico: brain como subagent no puede usar Agent tool (restriccion documentada). Hizo todo solo como workaround.

### 10.2 — Prueba 2: Brain con "agent": "brain" en settings.json del proyecto

- Comando: `claude --plugin-dir /path/to/plugin` (Con `.claude/settings.json: {"agent": "brain"}`)
- Prompt: `Implementa endpoint GET /health (...)`

Resultado:
- Claude NO activo brain como main agent
- Dijo "dispatch rules" (no existe en el plugin) y despacho a devsenior
- Cero agentes del plugin invocados

Diagnostico: "agent" en settings.json del PROYECTO no funciona para agentes de plugin. La documentacion describe el campo "agent" en settings.json del plugin pero no distingue entre --plugin-dir e instalacion via marketplace.

### 10.3 — Prueba 3: Brain con --agent brain (sin namespace)

- Comando: `claude --plugin-dir /path/to/plugin --agent brain`
- Prompt: `Implementa endpoint GET /health (...)`

Resultado:
- Claude NO activo brain del plugin
- Sesion generica, despacho a devsenior

Diagnostico: "brain" sin namespace no resuelve al brain del plugin. La documentacion del CLI no menciona formato con namespace.

### 10.4 — Prueba 4: Brain con --agent Plugin_name:brain (EXITOSA)

- Comando: `claude --plugin-dir /path/to/plugin --agent Plugin_name:brain`
- Prompt: `Implementa endpoint GET /ping (...)`

Resultado:
- Brain identificado como main agent: "Soy @brain"
- Fase 0 ejecutada: 4 MCPs detectados
- TDD completo con artefactos en docs/
- 4 tests nuevos, 185 total, cero regresiones
- PERO: brain ejecuto roles directamente (no spawno subagents)
- Razon: tarea trivial, brain decidio no delegar (comportamiento no-deterministico del LLM)

Diagnostico: namespace correcto. Brain es main agent. Para tareas simples, brain puede optimizar haciendo todo solo.

### 10.5 — Prueba 5: Brain orquestador con CRUD complejo (EXITOSA)

- Comando: `claude --plugin-dir /path/to/plugin --agent Plugin_name:brain`
- Prompt: `Implementa CRUD de categorias con slug, unique, Mongoose (...)`

Resultado:
- Brain en modo orquestador (detecto TeamCreate disponible)
- TeamCreate("feat-CATEGORIES-CRUD") — exitoso
- TaskCreate con dependencias (addBlockedBy) — exitoso
- 7 agentes spawneados con Agent tool:
  1. api-contract (foreground) — OpenAPI spec
  2. planner (foreground) — plan tecnico
  3. test-architect (foreground) — specs de tests
  4. test-engineer (foreground) — 47 tests en RED
  5. backend-dev (foreground) — implementacion GREEN+REFACTOR
  6. security-qa (background) — reporte de seguridad
  7. quality-gate (background) — quality gate APROBADO
- security-qa y quality-gate corrieron en PARALELO
- 47 tests nuevos, 232 total, cero regresiones
- 9 artefactos creados en docs/

Problema encontrado: enforce-flow-order.sh bloqueo el primer spawn porque brain (main agent) nunca fue registrado en el state file. Brain hizo workaround usando agentes general-purpose.

Fix aplicado: pre-registrar brain en SessionStart + detectar team_name.

### 10.6 — Prueba 6: Verificacion de hooks de enforcement

enforce-flow-order.sh:
- Bloqueo correctamente api-contract antes de que brain estuviera registrado
- Mensaje: "El primer agente invocado debe ser @brain"
- Fix: pre-registro de brain en SessionStart resuelve el caso main agent

enforce-plugin-readonly.sh:
- No se disparo explicitamente (brain no intento escribir en el plugin)
- Confirmado indirectamente: todos los archivos se crearon en el proyecto

enforce-prerequisites.sh:
- No se probo directamente porque brain uso agentes general-purpose
- Pendiente de validacion con el fix de flow-order


---

## 11. ISSUES DE GITHUB RELEVANTES

### Enforcement y hooks:

| Issue | Descripcion | Estado |
|-------|------------|--------|
| #4669 | permissionDecision deny — discrepancia entre docs y comportamiento | CERRADO "not planned" |
| #21988 | Exit code 1 no bloquea (por diseno, issue solicita cambio) | ABIERTO |
| #20946 | Hooks no bloquean en modo bypass (--dangerously-skip-permissions) | ABIERTO |
| #29795 | Buena practica: sistema QA de 5 capas (68 fallos documentados) | DOCUMENTACION |

### Multi-agente:

| Issue | Descripcion | Estado |
|-------|------------|--------|
| #30703 | hooks y skills del frontmatter ignorados en teammates | ABIERTO (bug) |
| #5812 | Puente de contexto de hooks entre sub-agents y padre | CERRADO "not planned" |
| #32731 | Restricciones de tools en teammates (sin Agent, sin TeamCreate) | DOCUMENTADO |

### Estabilidad:

| Issue | Descripcion | Estado |
|-------|------------|--------|
| #10205 | Loops infinitos con hooks habilitados | ABIERTO |
| #9579 | Loop de autocompactacion, spike de tokens (96-108M/dia) | ABIERTO |
| #9704 | Proceso huerfano consumio 2.88M tokens en mas de 2 dias | ABIERTO |
| #27156 | Conflicto de worktree con git submodules | DOCUMENTADO |


---

## 12. FUENTES Y REFERENCIAS

### Documentacion oficial de Claude Code:

- Hooks: https://code.claude.com/docs/en/hooks
- Sub-agents: https://code.claude.com/docs/en/sub-agents
- Agent Teams: https://code.claude.com/docs/en/agent-teams
- Plugins: https://code.claude.com/docs/en/plugins
- Referencia de plugins: https://code.claude.com/docs/en/plugins-reference
- Skills: https://code.claude.com/docs/en/skills
- Referencia CLI: https://code.claude.com/docs/en/cli-reference
- Tareas programadas: https://code.claude.com/docs/en/scheduled-tasks
- Permisos: https://code.claude.com/docs/en/permissions
- Seguridad: https://code.claude.com/docs/en/security
- Marketplaces: https://code.claude.com/docs/en/plugin-marketplaces
- Settings: https://code.claude.com/docs/en/settings
- Memoria: https://code.claude.com/docs/en/memory

### Blogs y tutoriales:

- Blake Crosley — 95 hooks en produccion: https://blakecrosley.com/blog/claude-code-hooks
- Blake Crosley — tutorial de 5 hooks: https://blakecrosley.com/blog/claude-code-hooks-tutorial
- Blake Crosley — Claude Code como infraestructura: https://blakecrosley.com/blog/claude-code-as-infrastructure
- Pixelmojo — patrones CI/CD con hooks: https://www.pixelmojo.io/blogs/claude-code-hooks-production-quality-ci-cd-patterns
- Paddo.dev — guardrails que realmente funcionan: https://paddo.dev/blog/claude-code-hooks-guardrails/
- GitButler — workflows automatizados con hooks: https://blog.gitbutler.com/automate-your-ai-workflows-with-claude-code-hooks/
- Karanbansal — guia de hooks: https://karanbansal.in/blog/claude-code-hooks/
- egghead.io — contaminacion de settings en subagents: https://egghead.io/avoid-the-dangers-of-settings-pollution-in-subagents-hooks-and-scripts~xrecv
- DEV Community — fix de permisos rotos: https://dev.to/boucle2026/how-to-fix-claude-codes-broken-permissions-with-hooks-23gl
- DataCamp — como construir plugins: https://www.datacamp.com/tutorial/how-to-build-claude-code-plugins

### Repositorios:

- disler/claude-code-hooks-mastery: https://github.com/disler/claude-code-hooks-mastery
- disler/claude-code-hooks-multi-agent-observability: https://github.com/disler/claude-code-hooks-multi-agent-observability
- kenryu42/claude-code-safety-net: https://github.com/kenryu42/claude-code-safety-net
- wshobson/agents (112 agentes, 16 orquestadores): https://github.com/wshobson/agents
- mbruhler/claude-orchestration: https://github.com/mbruhler/claude-orchestration
- barkain/claude-code-workflow-orchestration: https://github.com/barkain/claude-code-workflow-orchestration
- ruvnet/ruflo (orquestador externo): https://github.com/ruvnet/ruflo


---

## 13. CONCLUSIONES

### 13.1 — El plugin multi-agente es viable

La arquitectura del plugin (19 agentes, hub-and-spoke con brain como orquestador) es correcta y esta alineada con los patrones que usa la comunidad (wshobson/agents con 112 agentes lo confirma). La restriccion de que subagents no pueden spawnar subagents se resuelve haciendo a brain el main agent de la sesion.

### 13.2 — El enforcement mecanico funciona pero requiere workarounds

El unico mecanismo de blocking confiable es exit code 2 en hooks PreToolUse. permissionDecision presenta una discrepancia entre documentacion (funcional) y comportamiento observado (ignorado, issue #4669 cerrado como "not planned"). Los hooks de enforcement (prerequisites, flow-order, plugin-readonly) funcionan en produccion pero requirieron ajustes tras pruebas empiricas (ej: pre-registro de brain en SessionStart).

### 13.3 — La instalacion via --plugin-dir tiene limitaciones criticas

`--plugin-dir` NO carga settings.json ni CLAUDE.md del plugin (hallazgo empirico — la documentacion no distingue explicitamente este comportamiento). Requiere `--agent Plugin_name:brain` como flag CLI y un bootstrap que inyecte el import del CLAUDE.md. La instalacion via marketplace resolveria ambos problemas pero requiere publicar el plugin.

### 13.4 — Las pruebas empiricas fueron esenciales

Sin las pruebas en un proyecto real, los siguientes problemas no se habrian detectado:
- "agent" en settings.json del proyecto no funciona para plugins
- Namespace obligatorio para --agent (no documentado en la referencia CLI)
- enforce-flow-order.sh bloqueaba brain como main agent
- Brain como subagent no puede spawnar agentes (restriccion documentada, pero el impacto practico solo se entiende empiricamente)
- Brain optimiza tareas simples haciendo todo solo (comportamiento no-deterministico)

La metodologia de investigacion (documentacion — implementacion — test empirico — fix — re-test) fue mas valiosa que cualquier cantidad de analisis teorico. Varios hallazgos caen en gaps de la documentacion oficial que solo las pruebas empiricas pueden llenar.

### 13.5 — Prueba final: migracion Express a NestJS (EXITO COMPLETO)

- Comando: `claude --plugin-dir /path/to/plugin --agent Plugin_name:brain`
- Prompt: "Necesitamos migrar este proyecto a NestJS con TypeScript."
- Proyecto: Express 4.x + Mongoose + MongoDB, 9 modelos, 12 rutas API, 5 modelos con tenantId (multi-tenant)

Resultado:
- Brain detecto modo orquestador (TeamCreate disponible)
- Fase 0 completa: 4 MCPs, project-knowledge, routing de dominio
- Deteccion automatica de multi-tenant — activo architecture-guardian + isolation-tester
- 11 agentes spawneados (9 unicos, 2 re-invocados):
  - brain — api-contract — planner —
  - [architecture-guardian || test-architect || isolation-tester] —
  - backend-dev — [backend-dev || auth-access] —
  - [backend-dev || auth-access] —
  - [security-qa || quality-gate] — FALLO —
  - [test-engineer || backend-dev] — PASO

Paralelismo real confirmado:
- 3 en paralelo: architecture-guardian + test-architect + isolation-tester
- 2 en paralelo: backend-dev + auth-access (dos veces)
- 2 en paralelo: security-qa + quality-gate
- 2 en paralelo: test-engineer + backend-dev (fixes)

Quality gate RECHAZO la primera pasada (coverage 44%, umbral 80%). Brain respondio spawneando test-engineer para subir coverage. Re-ejecuto quality-gate — PASO con 93%.

Skills cargados automaticamente: tdd-governance, coding-standards, multi-tenant-platform-architecture, mongodb-development

Metricas finales:
- 190 tests NestJS + 232 legacy = 422 total, 0 regresiones
- Coverage: 92.75% statements, 82.39% branches
- TypeScript strict: 0 errores
- Build: compila exitosamente
- Seguridad: PASO CON CONDICIONES (0 CRITICAL, HIGHs corregidos)
- 14 artefactos en docs/ (requirements, api, plans, architecture, test-specs, tdd, security, quality, context)

Esta prueba confirma que el experimento funciona de principio a fin con un caso real no trivial (migracion completa de framework), incluyendo:
- Orquestacion multi-agente con spawning real
- Paralelismo de agentes
- Quality gate como barrera mecanica (rechazo + correccion)
- Routing de dominio automatico (multi-tenant)
- Skills consumidos por agentes especializados
- Project-knowledge actualizado al final
