# Comandos de detección

Comandos concretos para las fases 1, 4 y 5. Requieren terminal — en modo Lovable no corren.

> Los ejemplos usan `rg` (ripgrep). Si no está instalado, usá la herramienta Grep de Claude Code o `grep -rEn`.

---

## 1. Archivos `.env` versionados

```bash
# Hoy
git ls-files | grep -E '(^|/)\.env'

# Alguna vez en la historia (aunque estén borrados)
git log --all --pretty=format: --name-only --diff-filter=A | sort -u | grep -E '(^|/)\.env'
```

Si aparece algo en el segundo comando y no en el primero: el archivo se borró pero **sigue siendo recuperable** por cualquiera que clone el repo. Ver el contenido:

```bash
git log --all --oneline -- ruta/al/.env
git show <sha>:ruta/al/.env
```

---

## 2. Secretos en el árbol actual

```bash
rg -n --hidden \
  -g '!node_modules' -g '!*.lock' -g '!.git' -g '!dist' \
  -e 'sk-ant-[A-Za-z0-9_-]{20,}' \
  -e 'sk-proj-[A-Za-z0-9_-]{20,}' \
  -e 'sk-[A-Za-z0-9]{32,}' \
  -e 'sk_live_[0-9a-zA-Z]{20,}' \
  -e 'rk_live_[0-9a-zA-Z]{20,}' \
  -e 'AKIA[0-9A-Z]{16}' \
  -e 'AIza[0-9A-Za-z_-]{35}' \
  -e 'ghp_[A-Za-z0-9]{36}' \
  -e 'gho_[A-Za-z0-9]{36}' \
  -e 'github_pat_[A-Za-z0-9_]{50,}' \
  -e 'xox[baprs]-[A-Za-z0-9-]{10,}' \
  -e 'SG\.[A-Za-z0-9_-]{20,}\.' \
  -e '-----BEGIN [A-Z ]*PRIVATE KEY-----' \
  -e '(?i)(password|passwd|secret|token)\s*[:=]\s*["'"'"'][^"'"'"']{8,}'
```

El último patrón genera ruido (matchea nombres de campos de formularios). Filtralo a ojo antes de reportar.

---

## 3. JWT: distinguir `anon` de `service_role`

Ubicarlos:

```bash
rg -n -o -g '!node_modules' -g '!*.lock' 'eyJ[A-Za-z0-9_-]{10,}\.eyJ[A-Za-z0-9_-]{10,}' .
```

Decodificar el payload de cada uno (Node siempre está disponible en un proyecto Vite):

```bash
rg -o -n -g '!node_modules' -g '!*.lock' 'eyJ[A-Za-z0-9_-]{10,}\.eyJ[A-Za-z0-9_-]{10,}' . \
| while IFS= read -r line; do
    jwt=${line##*:}        # el match no tiene ':', así que esto lo aísla
    loc=${line%:*}         # archivo:línea
    payload=$(printf '%s' "$jwt" | cut -d. -f2)
    echo "$loc -> $(node -e "process.stdout.write(Buffer.from(process.argv[1].replace(/-/g,'+').replace(/_/g,'/'),'base64').toString())" "$payload")"
  done
```

Salida esperada:

```
./src/integrations/supabase/client.ts:5 -> {"iss":"supabase","role":"anon","iat":1700}
./src/lib/admin.ts:12          -> {"iss":"supabase","role":"service_role","iat":1700}
```

Cómo leer el resultado:

| `role` en el payload | Veredicto |
|---|---|
| `anon` | Normal. Es pública por diseño. **No es un hallazgo.** |
| `authenticated` | Es un token de sesión de alguien. Si está commiteado, 🔴 — hay que invalidar la sesión. |
| `service_role` | 🔴 **crítico siempre.** Bypasea RLS por completo. Rotar la clave en el panel de Supabase, no alcanza con borrarla. |

Excepción: una `service_role` dentro de `supabase/functions/` leída desde `Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')` es correcta — corre en el servidor. El problema es la clave **literal** escrita en el código, o cualquier `service_role` bajo `src/`.

---

## 4. Secretos en el historial de git

```bash
# Si gitleaks está instalado, es lo mejor que hay
command -v gitleaks >/dev/null && gitleaks detect --no-banner --redact --report-format json --report-path ./gitleaks.json \
  && jq -r '.[] | "\(.RuleID)\t\(.File):\(.StartLine)\t\(.Commit[0:8])"' ./gitleaks.json | sort -u
```

Sin gitleaks, buscá por marcador con `git log -S` (pickaxe: encuentra commits donde la cadena apareció o desapareció):

```bash
for m in service_role SUPABASE_SERVICE_ROLE_KEY 'sk-' 'sk_live_' AIza ghp_ github_pat_ 'PRIVATE KEY' AKIA; do
  hits=$(git log --all --oneline -S "$m" | head -5)
  [ -n "$hits" ] && printf '\n== %s ==\n%s\n' "$m" "$hits"
done
```

Para cada commit sospechoso:

```bash
git show <sha> | grep -n -i -C2 '<marcador>'
```

**Importante para el informe:** un secreto solo en el historial sigue siendo un hallazgo activo. La corrección es rotar la clave, no borrar el archivo.

---

## 5. GitHub: visibilidad y secret scanning

```bash
gh repo view --json nameWithOwner,visibility,isPrivate,pushedAt

repo=$(gh repo view --json nameWithOwner -q .nameWithOwner)
gh api "repos/$repo/secret-scanning/alerts" \
  --jq '.[] | "\(.state)\t\(.secret_type)\t\(.html_url)"' 2>/dev/null \
  || echo "Sin acceso a secret scanning (no habilitado o sin permisos)"
```

Si `visibility` es `PUBLIC`, subí un nivel la severidad de todo hallazgo de secreto y decilo en el resumen ejecutivo.

---

## 6. Variables de entorno: qué llega al cliente

```bash
# Variables consumidas desde el frontend
rg -o -I 'import\.meta\.env\.[A-Z0-9_]+' src/ | sort -u

# Variables declaradas
grep -rhE '^[A-Z][A-Z0-9_]*=' .env .env.* 2>/dev/null | cut -d= -f1 | sort -u

# Secretos del lado servidor (Edge Functions) — estos NO son públicos
rg -o -I "Deno\.env\.get\(['\"][A-Z0-9_]+" supabase/functions/ 2>/dev/null | sort -u
```

**Regla:** toda variable `VITE_*` queda embebida en el bundle y la lee cualquiera con el navegador. Si el nombre contiene `SECRET`, `PRIVATE`, `SERVICE`, o es una API key de un servicio que cobra por uso (OpenAI, Anthropic, Resend, Twilio), es un secreto filtrado → 🔴.

---

## 7. Queries sin filtro de pertenencia

```bash
rg -n -A6 "\.from\(" -g '*.ts' -g '*.tsx' src/
```

Revisá cada sitio: ¿hay un `.eq('user_id', ...)`, `.match(...)` u otro filtro de pertenencia, o confía en que RLS lo tape? Si confía en RLS, verificá en Fase 2 que esa tabla realmente lo tenga. Si no lo tiene, el hallazgo es de Fase 2 y este es el sitio donde se explota.

IDOR — parámetros de URL usados directamente como ID de consulta:

```bash
rg -n 'useParams|searchParams\.get' -A8 -g '*.tsx' src/
```

---

## 8. Logs y bundle

```bash
rg -n 'console\.(log|debug|info)' -g '*.ts' -g '*.tsx' src/

# Solo reportá los que imprimen sesión, token, usuario o respuesta completa
rg -n 'console\.(log|debug|info)\(.*(session|token|user|password|data|response)' -g '*.ts' -g '*.tsx' src/
```

```bash
# Si hay build compilado en el repo
[ -d dist ] && rg -o -I 'eyJ[A-Za-z0-9_-]{20,}|sk-[A-Za-z0-9]{20,}|AIza[0-9A-Za-z_-]{35}' dist/ | sort -u | head -20
```

---

## 9. Dependencias

```bash
# Solo dependencias de producción: es lo que llega al usuario
npm audit --omit=dev --audit-level=high

# Detalle en JSON
npm audit --omit=dev --json > audit.json 2>/dev/null
jq -r '.vulnerabilities | to_entries[]
       | select(.value.severity=="critical" or .value.severity=="high")
       | "\(.value.severity)\t\(.key)\t\(.value.via[0].title // .value.via[0])"' audit.json | sort -u
```

Corré primero con `--omit=dev`. Lo que solo aparece **sin** ese flag afecta al entorno de desarrollo y va como 🟡 o informativo, nunca como crítico. Borrá `audit.json` y `gitleaks.json` al terminar: son archivos temporales, no van al commit.

---

## 10. Configuración

```bash
cat .gitignore
rg -n "http://" -g '!node_modules' -g '*.ts' -g '*.tsx' src/
rg -n 'error\.(stack|message)' -A2 -g '*.tsx' src/
```
