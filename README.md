# sdd-claude-template

Template de desarrollo con **Spec-Driven Development (SDD)** y **Claude Code**.

El asistente guía cada sesión: pregunta antes de codear, presenta opciones de arquitectura con tradeoffs, documenta cada decisión y nunca avanza una fase sin confirmación. No hay código sin entender el problema primero.

---

## Inicio rápido

```bash
# 1. Crear proyecto desde el template
gh repo create nombre-del-proyecto \
  --template marcoslozina/sdd-claude-template \
  --clone

# 2. Abrir con Claude Code
cd nombre-del-proyecto
claude

# 3. Iniciar sesión guiada
/start-session
```

El asistente va a preguntar lenguaje, tipo de sistema y restricciones antes de arrancar.

---

## Flujo de desarrollo

```
Problema
   │
   ▼
Exploración  →  entender antes de comprometerse
   │
   ▼
Propuesta    →  2-3 opciones con tradeoffs  →  vos elegís
   │
   ▼
Spec+Design  →  qué hace + cómo lo hace     →  en paralelo
   │
   ▼
Tasks        →  lista ordenada e implementable
   │
   ▼
Apply        →  task por task, decisiones confirmadas
   │
   ▼
Verify       →  contra el spec original
```

El asistente no avanza una fase sin tu confirmación. Ante cada decisión de arquitectura presenta opciones y espera.

---

## Skills disponibles

### Lenguajes

| Skill | Cubre |
|-------|-------|
| `lang-python` | Python 3.12+, uv, dataclasses, ports & adapters, mypy |
| `lang-java` | Java 21+, Gradle, records, Spring Boot, JUnit 5 |
| `lang-typescript` | TypeScript estricto, React, Node.js, Zod, Vitest |
| `lang-go` | Go 1.22+, stdlib, Chi, interfaces pequeñas, testcontainers |

### Arquitectura y diseño

| Skill | Cubre |
|-------|-------|
| `role-architect` | Capas, ADRs, regla de dependencia, señales de alerta |
| `role-backend` | REST, paginación cursor/offset, N+1, manejo de errores, HTTP semántico |
| `role-frontend` | React, Container/Presentational, estado, a11y, performance |
| `role-code-review` | Checklist por capa, OWASP, privacidad, señales de mal diseño |

### Calidad y operaciones

| Skill | Cubre |
|-------|-------|
| `role-testing` | Pirámide, fakes vs mocks, Testcontainers, naming conventions |
| `role-performance` | Load/stress/soak testing con k6 y Locust, SLOs, profiling |
| `role-observability` | Logging estructurado, métricas OpenTelemetry, tracing distribuido, alertas |
| `role-cicd` | GitHub Actions, Blue/Green, Canary, Docker, secrets en pipeline |
| `role-environments` | Multi-ambiente, feature flags, config tipada, zero-downtime migrations |

### Seguridad y privacidad

| Skill | Cubre |
|-------|-------|
| `role-security` | OWASP Top 10, JWT, authn/authz, input validation, headers HTTP |
| `role-privacy` | Clasificación de datos, PII en logs/responses, cifrado, detección de secrets, retención |

### IA y Machine Learning

| Skill | Cubre |
|-------|-------|
| `role-ai-engineer` | Claude API oficial: tool use, prompt caching, streaming, batch API |
| `role-rag` | Chunking, embeddings, reranking — basado en docs oficiales Anthropic |
| `role-ml` | Pipelines, embeddings, serving con FastAPI, monitoreo de drift |

### Infraestructura

| Skill | Cubre |
|-------|-------|
| `infra-aws` | CDK, servicios AWS, IAM least privilege, arquitecturas comunes |
| `infra-docker` | Multi-stage builds por lenguaje, Compose local, seguridad de imágenes |

---

## Comandos disponibles

| Comando | Qué hace |
|---------|----------|
| `/start-session` | Setup guiado: lenguaje, tipo de sistema, restricciones |
| `/status` | Estado actual del flujo SDD y tasks pendientes |
| `/new-adr` | Crear nuevo Architecture Decision Record |

---

## Archivos incluidos

| Archivo | Propósito |
|---------|-----------|
| `CLAUDE.md` | Instrucciones completas del asistente + reglas de seguridad |
| `AGENTS.md` | Contexto del proyecto para cualquier agente AI |
| `Makefile` | Comandos unificados: `make test`, `make lint`, `make build` |
| `docs/adr/ADR-000-template.md` | Template para Architecture Decision Records |
| `.github/workflows/ci.yml` | Pipeline CI: lint → unit → integration → security scan → build |
| `.github/pull_request_template.md` | Checklist de arquitectura, seguridad y tests |
| `.github/dependabot.yml` | Actualizaciones automáticas de dependencias (Python, Node, Go, Docker, Actions) |

---

## Reglas de seguridad

El template tiene reglas que se aplican siempre, sin excepciones:

- **`git push` requiere confirmación explícita** — el asistente muestra los commits y espera tu OK
- **Escaneo de secrets antes de cada commit** — busca API keys, tokens, passwords hardcodeados
- **Operaciones destructivas requieren confirmación** — `reset --hard`, `rm -rf`, `DROP TABLE`
- **PII en código** — el asistente señala y pide reemplazar por datos ficticios

---

## Agregar un nuevo skill

```bash
# 1. Crear el archivo
mkdir -p .claude/skills/{nombre}
touch .claude/skills/{nombre}/SKILL.md

# 2. Registrar en CLAUDE.md (tabla de skills disponibles)
# 3. Registrar en AGENTS.md
```

---

## Skills de IA — fuentes oficiales

Los skills `role-ai-engineer` y `role-rag` están basados exclusivamente en documentación oficial de Anthropic:

- [Tool Use](https://docs.anthropic.com/en/docs/build-with-claude/tool-use)
- [Prompt Caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [RAG Guide](https://docs.anthropic.com/en/docs/build-with-claude/rag)
- [Batch API](https://docs.anthropic.com/en/docs/build-with-claude/batch-processing)
- [Streaming](https://docs.anthropic.com/en/docs/build-with-claude/streaming)

---

## Licencia

MIT
