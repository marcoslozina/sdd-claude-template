# sdd-claude-template

Template de desarrollo con **Spec-Driven Development (SDD)** y **Claude Code**.

Cada sesión sigue un flujo guiado: el asistente pregunta antes de codear, presenta opciones de arquitectura con tradeoffs, y documenta cada decisión. No hay código sin entender el problema primero.

---

## ¿Qué incluye?

### Skills por lenguaje
| Lenguaje | Convenciones, patrones, testing |
|----------|---------------------------------|
| Python 3.12+ | uv, dataclasses, ports & adapters |
| Java 21+ | Gradle, records, Spring Boot |
| TypeScript | React, Node.js, Zod, Vitest |
| Go 1.22+ | stdlib, Chi, interfaces pequeñas |

### Skills por rol
| Rol | Qué cubre |
|-----|-----------|
| Architect | Capas, ADRs, regla de dependencia |
| Backend | REST, paginación, N+1, manejo de errores |
| Frontend | React, Container/Presentational, estado, a11y |
| Security | OWASP Top 10, JWT, secrets, threat modeling |
| Testing | Pirámide, fakes vs mocks, Testcontainers |
| CI/CD | GitHub Actions, Blue/Green, Canary, Docker |
| AI Engineer | Claude API, tool use, prompt caching, batch |
| RAG | Chunking, embeddings, reranking — docs oficiales Anthropic |
| ML Engineering | Pipelines, serving, monitoreo de drift |

### Infraestructura
| Skill | Qué cubre |
|-------|-----------|
| AWS | CDK, servicios, IAM least privilege, arquitecturas comunes |
| Docker | Multi-stage builds por lenguaje, Compose, seguridad |

### Archivos listos
- `.github/workflows/ci.yml` — pipeline CI con lint, unit tests, integration tests, security scan
- `.github/pull_request_template.md` — checklist de arquitectura, seguridad y tests
- `.github/dependabot.yml` — actualizaciones automáticas de dependencias
- `docs/adr/ADR-000-template.md` — template para Architecture Decision Records
- `Makefile` — comandos unificados: `make test`, `make lint`, `make build`
- `AGENTS.md` — contexto del proyecto para agentes AI
- `CLAUDE.md` — flujo SDD completo con protocolo de decisiones

---

## Cómo usar este template

### 1. Crear un nuevo proyecto desde el template

```bash
gh repo create nombre-del-proyecto \
  --template marcoslozina/sdd-claude-template \
  --clone

cd nombre-del-proyecto
```

### 2. Abrir con Claude Code

```bash
claude
```

### 3. Iniciar la sesión guiada

```
/start-session
```

El asistente te va a preguntar:
1. ¿Con qué lenguaje/stack vamos a trabajar?
2. ¿Qué tipo de sistema es?
3. ¿Es prototipo o producción?
4. ¿Hay restricciones conocidas?

Con esas respuestas carga los skills relevantes y arranca el flujo SDD.

---

## Flujo de desarrollo

```
Problema
   │
   ▼
/sdd-explore   →  Entender antes de comprometerse
   │
   ▼
Propuesta      →  2-3 opciones con tradeoffs  →  vos elegís
   │
   ▼
Spec + Design  →  Qué hace + cómo lo hace     →  en paralelo
   │
   ▼
Tasks          →  Lista ordenada e implementable
   │
   ▼
/sdd-apply     →  Task por task, decisiones confirmadas
   │
   ▼
/sdd-verify    →  Verificar contra el spec original
```

**Regla:** el asistente no avanza una fase sin tu confirmación. Ante cada decisión de arquitectura presenta opciones y espera.

---

## Comandos disponibles

| Comando | Qué hace |
|---------|----------|
| `/start-session` | Setup guiado: lenguaje, tipo de sistema, restricciones |
| `/status` | Estado actual del flujo SDD y tasks pendientes |
| `/new-adr` | Crear nuevo Architecture Decision Record |

---

## Agregar un nuevo skill

1. Crear `.claude/skills/{nombre}/SKILL.md`
2. Agregar la referencia en `CLAUDE.md` en la tabla de skills disponibles
3. Agregar la referencia en `AGENTS.md`

---

## Stack de los skills de IA

Los skills `role-ai-engineer` y `role-rag` están basados exclusivamente en documentación oficial de Anthropic:

- [Tool Use](https://docs.anthropic.com/en/docs/build-with-claude/tool-use)
- [Prompt Caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching)
- [RAG Guide](https://docs.anthropic.com/en/docs/build-with-claude/rag)
- [Batch API](https://docs.anthropic.com/en/docs/build-with-claude/batch-processing)

---

## Licencia

MIT
