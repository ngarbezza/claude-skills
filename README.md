# claude-skills

Catálogo de [Agent Skills](https://code.claude.com/docs/en/skills) para Claude Code.

## Estructura

```
skills/
└── <nombre-del-skill>/
    └── SKILL.md        # Definición del skill (frontmatter + instrucciones)
```

Cada skill es un directorio dentro de `skills/` con un `SKILL.md` como entrypoint. El frontmatter YAML define el nombre y cuándo se activa; el cuerpo markdown contiene las instrucciones que Claude sigue.

## Skills disponibles

| Skill | Descripción |
|-------|-------------|
| [dependabot-review](skills/dependabot-review/SKILL.md) | Revisa, analiza y mergea PRs de Dependabot con clasificación de riesgo semver |

## Instalación

Usá [skills.sh](https://skills.sh) para instalar cualquier skill de este catálogo en tu proyecto:

```bash
npx skills add ngarbezza/claude-skills/skills/dependabot-review
```

El comando copia el skill a `.claude/skills/` en tu proyecto actual y queda disponible como `/dependabot-review` en Claude Code.

## Convenciones

- **Nombres**: kebab-case en minúsculas (`mi-skill`)
- **Descripción**: en tercera persona, incluye qué hace el skill y cuándo usarlo
- **Instrucciones**: forma imperativa, paso a paso
- **Archivos adicionales**: templates, ejemplos y scripts van en el mismo directorio del skill

## Referencias

- [Documentación oficial de Skills](https://code.claude.com/docs/en/skills)
- [Repo oficial de Anthropic](https://github.com/anthropics/skills)
- [Best practices para autores](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
