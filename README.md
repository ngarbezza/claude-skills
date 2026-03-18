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
| [mutation-testing](skills/mutation-testing/SKILL.md) | Corre mutation testing, analiza mutantes sobrevivientes y sugiere mejoras a los tests |

## Instalación

Para usar un skill de este catálogo, copiá el directorio del skill a `.claude/skills/` en tu proyecto:

```bash
# Ejemplo: instalar dependabot-review
cp -r skills/dependabot-review /tu-proyecto/.claude/skills/
```

O usá [skills.sh](https://skills.sh) si el skill está publicado ahí.

## Convenciones

- **Nombres**: kebab-case en minúsculas (`mi-skill`)
- **Descripción**: en tercera persona, incluye qué hace el skill y cuándo usarlo
- **Instrucciones**: forma imperativa, paso a paso
- **Archivos adicionales**: templates, ejemplos y scripts van en el mismo directorio del skill

## Referencias

- [Documentación oficial de Skills](https://code.claude.com/docs/en/skills)
- [Repo oficial de Anthropic](https://github.com/anthropics/skills)
- [Best practices para autores](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
