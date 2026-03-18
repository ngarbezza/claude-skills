---
name: mutation-testing
description: >
  Workflow para correr mutation testing en un codebase y analizar la calidad de la suite de tests.
  Usá este skill cuando el usuario quiera: medir qué tan buenos son sus tests, encontrar huecos en la cobertura
  de tests, correr mutation testing, analizar mutantes sobrevivientes, mejorar la efectividad de los tests,
  o saber si sus tests realmente detectan bugs. También triggerá cuando el usuario mencione "mutation testing",
  "mutation score", "mutantes sobrevivientes", "calidad de tests", "correr pitest", "correr stryker",
  "correr mutmut", "correr cargo-mutants", o pregunte si sus tests son buenos o suficientes.
---

# Mutation Testing Skill

Workflow para medir la calidad de los tests usando mutation testing: introducir pequeños cambios en el código y verificar si los tests los detectan.

## Concepto clave

Un **mutante** es una versión del código con un cambio pequeño (ej: `>` → `>=`, `+` → `-`, eliminar una condición).
Si los tests **fallan** con el mutante → el mutante fue **eliminado** (tests buenos ✅).
Si los tests **pasan** con el mutante → el mutante **sobrevivió** (hueco en los tests ⚠️).

El **mutation score** = mutantes eliminados / total mutantes. Un score >80% es generalmente aceptable; >90% es excelente.

---

## Paso 0: Detectar ecosistema y herramienta

Identificar el lenguaje y la herramienta de mutation testing disponible:

```bash
# Detectar lenguaje principal
ls package.json pom.xml build.gradle Cargo.toml pyproject.toml setup.py Gemfile go.mod 2>/dev/null
```

| Ecosistema | Herramienta recomendada | Cómo verificar si está configurada |
|------------|------------------------|-------------------------------------|
| JavaScript / TypeScript | [Stryker](https://stryker-mutator.io/) | `cat stryker.config.*` o `jq '.stryker' package.json` |
| Java / Kotlin | [PIT (Pitest)](https://pitest.org/) | `grep -r 'pitest\|pit-' pom.xml build.gradle` |
| Python | [mutmut](https://github.com/boxed/mutmut) | `pip show mutmut` o `cat .mutmut-config` |
| Ruby | [mutant](https://github.com/mbj/mutant) | `bundle exec mutant --version` |
| Rust | [cargo-mutants](https://mutants.rs/) | `cargo mutants --version` |
| Go | [go-mutesting](https://github.com/zimmski/go-mutesting) | `go-mutesting --version` |

Si no hay herramienta configurada, ir al **Paso 1b** para instalarla.

---

## Paso 1a: Verificar configuración existente

Si la herramienta ya está instalada, revisar su configuración:

### Stryker (JS/TS)
```bash
cat stryker.config.js stryker.config.mjs stryker.config.json 2>/dev/null
# También puede estar en package.json bajo "stryker"
jq '.stryker // empty' package.json 2>/dev/null
```

### Pitest (Java/Kotlin)
```bash
# Maven
grep -A 30 'pitest' pom.xml
# Gradle
grep -A 20 'pitest\|info.solidsoft' build.gradle build.gradle.kts
```

### mutmut (Python)
```bash
cat setup.cfg .mutmut-config 2>/dev/null | grep -A 20 '\[mutmut\]'
```

### cargo-mutants (Rust)
```bash
cat .cargo/mutants.toml 2>/dev/null
```

---

## Paso 1b: Instalar/configurar la herramienta (si no está)

### Stryker (JS/TS)
```bash
# Instalación interactiva (recomendado — genera config automáticamente)
npx stryker init

# O instalación manual
npm install --save-dev @stryker-mutator/core @stryker-mutator/jest-runner
# para Vitest: @stryker-mutator/vitest-runner
# para Mocha: @stryker-mutator/mocha-runner
```

Configuración mínima en `stryker.config.mjs`:
```js
export default {
  testRunner: 'jest',         // o 'vitest', 'mocha'
  coverageAnalysis: 'perTest',
  reporters: ['html', 'clear-text', 'progress'],
  thresholds: { high: 80, low: 60, break: null },
}
```

### Pitest (Maven)
```xml
<!-- Agregar en pom.xml dentro de <plugins> -->
<plugin>
  <groupId>org.pitest</groupId>
  <artifactId>pitest-maven</artifactId>
  <version>1.16.1</version>
  <configuration>
    <targetClasses>
      <param>com.miapp.*</param>  <!-- ajustar al paquete del proyecto -->
    </targetClasses>
    <targetTests>
      <param>com.miapp.*Test</param>
    </targetTests>
    <outputFormats>
      <outputFormat>HTML</outputFormat>
      <outputFormat>XML</outputFormat>
    </outputFormats>
  </configuration>
</plugin>
```

### Pitest (Gradle/Kotlin)
```kotlin
// build.gradle.kts
plugins {
    id("info.solidsoft.pitest") version "1.15.0"
}
pitest {
    targetClasses.set(setOf("com.miapp.*"))
    threads.set(4)
    outputFormats.set(setOf("XML", "HTML"))
    mutationThreshold.set(80)
}
```

### mutmut (Python)
```bash
pip install mutmut
# Configurar en setup.cfg:
# [mutmut]
# paths_to_mutate=src/
# runner=python -m pytest
```

### cargo-mutants (Rust)
```bash
cargo install cargo-mutants
```

---

## Paso 2: Correr mutation testing

> ⚠️ Mutation testing es **lento** por naturaleza (puede tardar de minutos a horas en proyectos grandes). Para una primera corrida, acotar el scope.

### Stryker (JS/TS)
```bash
# Corrida completa
npx stryker run

# Acotada a un archivo/directorio específico
npx stryker run --mutate "src/utils/**/*.ts"
```

### Pitest (Maven)
```bash
# Corrida completa
mvn test-compile org.pitest:pitest-maven:mutationCoverage

# Acotada a una clase específica
mvn org.pitest:pitest-maven:mutationCoverage \
  -DtargetClasses="com.miapp.MiServicio" \
  -DtargetTests="com.miapp.MiServicioTest"
```

### Pitest (Gradle)
```bash
./gradlew pitest

# Con filtro
./gradlew pitest --tests "com.miapp.MiServicioTest"
```

### mutmut (Python)
```bash
# Corrida completa
mutmut run

# Acotada
mutmut run --paths-to-mutate src/mi_modulo.py
```

### cargo-mutants (Rust)
```bash
# Corrida completa
cargo mutants

# Acotada a un archivo
cargo mutants --file src/lib.rs
```

---

## Paso 3: Leer e interpretar los resultados

### Stryker — resultados en consola y HTML
```bash
# El reporte HTML se genera en reports/mutation/index.html
# Resumen en consola:
# Killed:   X  ← tests buenos
# Survived: Y  ← huecos en tests
# Timeout:  Z  ← posibles loops infinitos
# Score: XX%
```

### Pitest — XML de resultados
```bash
# Reporte HTML en target/pit-reports/<timestamp>/index.html
# Leer XML para análisis programático:
find target/pit-reports -name 'mutations.xml' | sort | tail -1 | xargs cat
```

### mutmut — estado de mutantes
```bash
mutmut results           # resumen
mutmut show <id>         # ver diff de un mutante específico
mutmut html              # generar reporte HTML
```

### cargo-mutants — resultados
```bash
# Resultados en mutants.out/
cat mutants.out/outcomes.json | jq '.outcomes | group_by(.outcome) | map({outcome: .[0].outcome, count: length})'
```

Presentar al usuario un resumen con:
- Mutation score total
- Cantidad de mutantes: total / eliminados / sobrevividos / timeout
- Archivos con peor score (los que más necesitan mejora)

---

## Paso 4: Analizar mutantes sobrevivientes

Los mutantes sobrevivientes son los más valiosos: indican **comportamiento no testeado**.

### Clasificar por tipo de mutación

| Tipo de mutante | Qué significa si sobrevive |
|----------------|---------------------------|
| Cambio de operador (`>` → `>=`) | Los tests no prueban el caso límite (boundary) |
| Eliminación de condición | Hay paths de código no ejercitados |
| Negación de booleano | Falta un test para el caso contrario |
| Cambio de constante | El valor exacto no está siendo verificado |
| Eliminación de línea (void call) | Efectos secundarios no testeados |

### Priorizar mutantes a resolver

Ordenar por impacto:
1. **Mutantes en lógica de negocio crítica** → prioridad máxima
2. **Mutantes en validaciones y condiciones de borde** → alta prioridad
3. **Mutantes en código utilitario** → media prioridad
4. **Mutantes en código de infraestructura/plumbing** → baja prioridad

Para cada mutante sobreviviente importante, mostrar:
```
Archivo: src/services/PaymentService.ts:45
Mutante: ConditionalExpression → amount > 0  →  amount >= 0
Estado: SOBREVIVIÓ ⚠️
Qué test agregarías: test que verifique que amount=0 es rechazado
```

---

## Paso 5: Sugerir mejoras a los tests

Para cada mutante sobreviviente prioritario, proponer un test concreto:

### Ejemplo para mutante de boundary condition
```
Mutante original: if (price > 0)
Mutante inyectado: if (price >= 0)
→ Agregar test: expect(validatePrice(0)).toBeFalsy()
```

### Ejemplo para mutante de void call eliminado
```
Mutante: eliminar llamada a logger.audit(event)
→ Agregar test: expect(logger.audit).toHaveBeenCalledWith(expectedEvent)
```

Si el usuario lo pide, generar el código del test sugerido adaptado al framework de testing del proyecto (Jest, JUnit, pytest, RSpec, etc.).

Para identificar el framework:
```bash
# JS/TS
jq '.devDependencies | keys | map(select(test("jest|vitest|mocha|jasmine")))' package.json

# Java
grep -E 'junit|testng|spock' pom.xml build.gradle

# Python
grep -E 'pytest|unittest|nose' requirements*.txt setup.cfg pyproject.toml

# Ruby
grep -E 'rspec|minitest' Gemfile
```

---

## Paso 6: Seguimiento del score

Si el usuario quiere trackear la evolución del mutation score, sugerir:

```bash
# Guardar score actual en un archivo de referencia
echo "$(date +%Y-%m-%d): score=$(grep -oP 'Score: \K[\d.]+' mutation-report.txt)%" >> .mutation-history

# O en CI: fallar si el score baja del umbral
# Stryker: configurar thresholds.break en stryker.config
# Pitest: <mutationThreshold>80</mutationThreshold> en pom.xml
# mutmut: usar exit code en CI pipeline
```

---

## Modos de uso

### Modo "primera vez"
El usuario nunca corrió mutation testing. Ejecutar Pasos 0 → 1b → 2 (scope acotado) → 3 → dar una primera impresión del score y explicar qué significa.

### Modo "análisis de huecos"
El usuario ya tiene resultados y quiere entender qué mejorar. Ejecutar Pasos 3 → 4 → 5. Foco en los mutantes sobrevivientes más impactantes.

### Modo "mejorar score"
El usuario quiere subir el mutation score. Ejecutar Paso 4 en detalle, generar tests concretos del Paso 5, re-correr para verificar mejora.

### Modo "integrar en CI"
El usuario quiere que mutation testing falle el pipeline si el score baja. Mostrar configuración de umbral de la herramienta correspondiente y el comando de CI.

---

## Tips y edge cases

- **Primera corrida lenta**: en proyectos grandes, acotar con `--mutate` o `targetClasses` a un módulo específico primero. Una vez que funciona, ampliar.
- **Mutantes de timeout**: generalmente indican loops que dependen del valor mutado. No son un problema crítico pero vale investigarlos.
- **Falsos positivos**: algunos mutantes sobreviven porque el comportamiento mutado es equivalente en la práctica. Marcarlos como "equivalentes" en la herramienta si aplica.
- **Cobertura de línea vs mutation score**: 100% de cobertura de líneas NO garantiza buen mutation score. Un test puede ejecutar código sin verificar su resultado.
- **Velocidad**: `coverageAnalysis: 'perTest'` en Stryker y threads en Pitest aceleran la corrida significativamente.
- **Exclude patterns**: excluir código generado, DTOs sin lógica, y adaptadores de infraestructura del scope de mutation testing para reducir ruido.
