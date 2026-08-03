---
name: dependabot-fix
description: Resuelve una alerta de Dependabot / GitHub Security de este repo de punta a punta - lee el detalle de la alerta, crea una rama desde master, actualiza la dependencia vulnerable, valida localmente los mismos checks que corre el CI, hace commit, push y abre el PR. Úsala cuando el usuario pida revisar/arreglar una alerta de Dependabot, una vulnerabilidad de seguridad de una dependencia, o pegue un link a github.com/.../security/dependabot/<numero>.
---

Resuelves alertas de Dependabot en el repo `JordanParra96/Salesforce_DevEnv`. Es un proyecto personal (no compartido), así que ejecutas el flujo completo — incluyendo push y creación del PR — sin pedir confirmación intermedia, salvo que algo falle o sea ambiguo.

## Input

El usuario te da un número o link de alerta (`.../security/dependabot/<numero>`). Si no lo da, pregúntaselo — sin el número no puedes continuar.

## 1. Partir de master limpio

```
git status   # si hay cambios sin commitear que no sean tuyos de esta tarea, detente y avisa
git checkout master
git pull origin master
```

## 2. Leer el detalle de la alerta

```
gh api repos/JordanParra96/Salesforce_DevEnv/dependabot/alerts/<numero>
```

Extrae: paquete (`dependency.package.name`), ecosistema, `manifest_path`, versión vulnerable (`security_vulnerability.vulnerable_version_range`), versión parcheada (`security_vulnerability.first_patched_version.identifier`), severidad, y el resumen del advisory (GHSA id / CVE si existe) — los necesitas para el mensaje de commit y el cuerpo del PR.

## 3. Crear la rama desde master

Sigue el patrón de nombres ya usado en el repo (revisa `git log --oneline -20` si tienes dudas):

- Fix de dependencia → prefijo `dep_` (ej. `dep_postCssIssue`, `dep_yamlIssue`, `dep_braceExpansionIssue`). Si ya existe una rama/PR previo para el mismo paquete por otra vulnerabilidad distinta, añade un sufijo descriptivo (ej. `dep_postCssPathTraversalIssue`) para no chocar.
- El nombre en camelCase, terminando en `Issue`.

```
git checkout -b dep_<paqueteEnCamelCase>Issue
```

## 4. Ubicar y actualizar la dependencia

Revisa `package.json`:

- Si el paquete está en la sección `overrides`, súbelo ahí a la `first_patched_version` (o una versión estable posterior si esa ya no existe en el registry).
- Si es dependencia directa/dev, súbela ahí.
- Si es transitiva y no aparece en ningún lado de `package.json`, añade una entrada nueva en `overrides` pineada a la versión parcheada (así es como este repo ya maneja el resto de vulnerabilidades transitivas — sigue el mismo patrón).

Luego regenera el lockfile:

```
npm install
```

Verifica que quedó resuelto a la versión correcta:

```
node -e "console.log(require('./node_modules/<paquete>/package.json').version)"
npm audit 2>&1 | grep -i <paquete>   # no debería aparecer ya
```

No toques otras vulnerabilidades preexistentes que `npm audit` reporte y que no sean la de esta alerta (p. ej. la cadena de minimatch/brace-expansion en eslint/jest) — quedan fuera de alcance salvo que el usuario lo pida explícitamente.

## 5. Validar localmente los mismos steps que corre el CI en PR

Estos son los steps reales de `.github/workflows/ci-pr.yml` job `format-lint-lwc-tests` (revisa el archivo por si cambió):

```
npm run prettier:verify
npm run lint            # solo si hay componentes en ./force-app/main/default/lwc
npm run test:unit:coverage
```

Si algo falla, arréglalo antes de seguir (no es normal que un bump de patch/minor rompa esto, pero si pasa, investiga si es un breaking change real de la dependencia).

El job `developer-org-test` (PMD + despliegue de validación a la org vía `sf`) no se puede correr en local sin los secrets del DevHub — no lo bloquees, pero menciónalo en el resumen final si el cambio tocara Apex (normalmente un bump de dependencia npm no lo afecta).

## 6. Commit

Mensaje siguiendo el estilo ya usado en el historial (`chore(deps): update <paquete> to version <version> in package.json and package-lock.json`), con el detalle de la vulnerabilidad en el cuerpo:

```
git add package.json package-lock.json
git commit -m "$(cat <<'EOF'
chore(deps): update <paquete> to version <version> in package.json and package-lock.json

Fixes <GHSA-id>[ / <CVE-id>]: <resumen corto del advisory>.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
EOF
)"
```

## 7. Push y PR (automático, sin pedir confirmación — proyecto personal)

```
git push -u origin <rama>
gh pr create --title "chore(deps): update <paquete> to version <version> in package.json and package-lock.json" --body "..."
```

El body del PR debe incluir:

- Link a la alerta (`https://github.com/JordanParra96/Salesforce_DevEnv/security/dependabot/<numero>`) y al GHSA/CVE.
- Qué cambió y por qué (versión vieja → nueva, resumen de 1-2 líneas del vector de la vulnerabilidad).
- Checklist de "Test plan" con los checks corridos en el paso 5, marcados.

## 8. Verificar que el CI del PR pasa

Sondea los checks del PR recién creado hasta que todos terminen (no hace falta preguntar, solo repórtalo al final):

```
gh pr checks <numero-pr>
```

Repite cada ~20s hasta que no queden `pending`. Si algo falla, investiga la causa, corrige, haz un nuevo commit en la misma rama y vuelve a pushear — no cierres el PR ni fuerces nada destructivo.

## 9. Resumen final

Reporta en 2-4 líneas: alerta resuelta (número + resumen corto), paquete y versiones, link del PR, y estado de los checks (todos verdes / cuáles fallaron). No hagas merge del PR a menos que el usuario lo pida explícitamente — eso lo decide él.
