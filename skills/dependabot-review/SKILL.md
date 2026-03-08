---
name: dependabot-review
license: MIT
description: >
  Workflow completo para revisar, analizar y mergear PRs de dependabot en repositorios GitHub.
  Usá este skill cuando el usuario quiera: listar PRs de dependabot pendientes, revisar updates de dependencias,
  analizar el riesgo de un update según versionado semántico, hacer merge de uno o varios PRs de dependabot,
  o hacer un "dependabot sweep" del repo. También triggerá cuando el usuario mencione "actualizar dependencias",
  "revisar dependabot", "mergear dependabot PRs", o simplemente pregunte qué dependencias tiene pendientes.
---

# Dependabot Review Skill

Workflow para revisar y mergear PRs de dependabot con análisis de riesgo basado en semver.

## Herramientas necesarias

- `gh` CLI autenticado (verificar con `gh auth status`)
- Acceso a internet para buscar changelogs
- Estar parado en el repo correcto o que el usuario indique el repo

---

## Paso 0: Setup inicial

Antes de arrancar, verificar:

```bash
gh auth status
gh repo view --json nameWithOwner  # confirmar en qué repo estamos
```

Si el usuario no indicó repo, preguntar o inferir del directorio actual.

---

## Paso 1: Listar PRs de dependabot pendientes

```bash
gh pr list --author "app/dependabot" --state open --json number,title,headRefName,statusCheckRollup,mergeable,labels \
  | jq '.[] | {number, title, mergeable, ci: (.statusCheckRollup | map(select(.status=="COMPLETED")) | map(.conclusion) | unique)}'
```

Presentar al usuario una tabla con:
- Número de PR
- Dependencia y versión (extraer del título: `Bump X from A to B`)
- Estado del CI (✅ passing / ❌ failing / ⏳ pending)
- Mergeabilidad
- Clasificación semver preliminar (ver Paso 2)

---

## Paso 2: Clasificar riesgo semver

Para cada PR, extraer `from_version` y `to_version` del título y clasificar:

| Cambio | Nivel de riesgo | Acción sugerida |
|--------|----------------|-----------------|
| `x.x.x` → `x.x.y` (patch) | 🟢 Bajo | Merge directo si CI pasa |
| `x.y.z` → `x.z.0` (minor) | 🟡 Medio | Revisar changelog, merge si CI pasa y no hay breaking changes |
| `x.y.z` → `y.0.0` (major) | 🔴 Alto | Revisar en detalle, siempre pedir confirmación |
| Sin semver claro | ⚪ Desconocido | Revisar manualmente |

**Criterio de auto-merge seguro**: CI passing + (patch O minor con bajo riesgo según changelog).
**Siempre pedir confirmación** para major versions.

---

## Paso 3: Analizar changelog de una dependencia

Para una dependencia específica, obtener el changelog en este orden:

### 3a. Release notes del PR de GitHub
```bash
gh pr view <NUMBER> --json body | jq '.body'
```
Dependabot suele incluir el release notes en el body del PR. Extraer y resumir.

### 3b. GitHub Releases del repo de la dependencia
Buscar el repo de la dependencia (inferir de la URL en el body del PR o buscando).
```bash
gh release list --repo <owner/repo> --limit 10
gh release view <tag> --repo <owner/repo> --json body
```

### 3c. CHANGELOG.md del repo
```bash
gh api repos/<owner/repo>/contents/CHANGELOG.md | jq '.content' | base64 -d | head -200
```
O también: `CHANGELOG`, `CHANGES`, `HISTORY.md` como variantes.

### 3d. Web search como fallback
Buscar `"<package-name> <version> changelog"` o `"<package-name> <from-version> to <to-version> migration"`.

### Qué buscar en el changelog

Cuando analizás el changelog, resumir especialmente:
- **Breaking changes o deprecaciones** de APIs usadas
- **Cambios de comportamiento** en funcionalidades core
- **Nuevas dependencias transitivas** con licencias problemáticas
- **Cambios de configuración** requeridos
- **CVEs o fixes de seguridad** (siempre positivo para mergear)

Adaptar el análisis al stack del proyecto si se conoce (ej: Spring Boot, Kotlin, JPA).

---

## Paso 4: Tomar decisión de merge

### Para un PR individual

Después del análisis, presentar:
```
PR #<N>: Bump <dep> from <A> to <B>
Riesgo: 🟡 Minor
CI: ✅ Passing
Changelog: <resumen de 2-3 líneas>
Cambios relevantes: <bullets de lo más importante>
Recomendación: MERGE / REVISAR MANUALMENTE / NO MERGEAR

¿Mergeamos? [s/n]
```

Para ejecutar el merge (usar siempre `gh` CLI o la UI de GitHub — los comandos de comentario `@dependabot merge` fueron deprecados en enero 2026):
```bash
gh pr merge <NUMBER> --merge --delete-branch
# o --squash si el repo lo prefiere
```

### Para batch de PRs

1. Clasificar todos los PRs por riesgo
2. Agrupar: auto-mergeables (patch + CI ok) vs requieren revisión
3. Mostrar resumen al usuario y pedir confirmación antes de mergear el batch
4. Mergear en orden: primero los de menor riesgo
5. Entre merges, esperar que el CI no se rompa en cadena (especialmente si hay lockfile)

```bash
# Merge en batch de los aprobados
for pr in <lista>; do
  echo "Mergeando PR #$pr..."
  gh pr merge $pr --merge --delete-branch
  sleep 5  # dar tiempo a que dependabot rebase los PRs siguientes si comparten lockfile
done
```

> ⚠️ **Lockfile en batch**: Dependabot recrea automáticamente los PRs cuando el branch base cambia, así que el cascade funciona sin intervención manual — pero hay que darle tiempo entre merges. Si dos PRs tocan el mismo lockfile (`package-lock.json`, `pom.xml`, `Gemfile.lock`), el segundo PR quedará desactualizado después del primer merge; dependabot lo rebaseará solo en minutos.

---

## Paso 5: Post-merge

Después de mergear, verificar:
```bash
gh run list --limit 5  # ver que los workflows no se rompieron
```

Si algo falló, indicar cuál PR lo causó y cómo hacer rollback si es necesario.

---

## Modos de uso

### Modo "dame un panorama"
El usuario quiere ver el estado general. Ejecutar Paso 1 y 2, mostrar tabla resumen con riesgo semver y estado de CI.

### Modo "revisá este PR"
El usuario indica un número de PR. Ejecutar Pasos 2, 3 y 4 para ese PR específico.

### Modo "limpiá todo lo que sea seguro"
Ejecutar Pasos 1-4 en batch. Mergear automáticamente los seguros, dejar pendientes los riesgosos con explicación.

---

## Tips y edge cases

- **Monorepos**: puede haber múltiples PRs para el mismo ecosistema. Agruparlos.
- **PRs desactualizados**: si dependabot rebased recientemente, el CI puede estar re-corriendo. Esperar antes de mergear.
- **Conflictos de lockfile**: si dos PRs tocan el mismo lockfile (`pom.xml`, `package-lock.json`), mergear de a uno y esperar que dependabot actualice el segundo.
- **Dependencias de seguridad**: priorizar siempre CVE fixes aunque sean major versions. Mencionar explícitamente.
- **Pre-releases**: `1.0.0-beta` no es semver estable. Tratar con precaución y siempre pedir confirmación.
