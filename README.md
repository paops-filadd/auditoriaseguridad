# 🔐 Auditoría de Seguridad — apps Lovable

Skill que audita la seguridad de una app hecha con Lovable (React + Vite + Supabase). Revisa secretos en el código **y en el historial de git**, RLS y policies de Supabase, Edge Functions, Storage, autenticación y autorización, exposición de datos en el cliente y dependencias.

Produce un `AUDITORIA_SEGURIDAD.md` en la raíz del proyecto y, si se lo pedís, aplica las correcciones en una branch y abre el PR.

---

## Los dos modos

La skill corre en dos entornos y no hace lo mismo en cada uno.

| | **Claude Code** (recomendado) | **Lovable** |
|---|---|---|
| Secretos en el código actual | ✅ | ✅ |
| Secretos en el **historial de git** | ✅ | ❌ |
| RLS, policies, Edge Functions, Storage | ✅ | ✅ |
| Visibilidad del repo y secret scanning | ✅ | ❌ |
| `npm audit` sobre dependencias reales | ✅ | ❌ |
| Decodificar JWT (`anon` vs `service_role`) | ✅ | ❌ |
| Aplicar las correcciones en una branch + PR | ✅ | ❌ (se le pide a Lovable) |

El historial de git es la diferencia importante: **un `.env` borrado hace veinte commits sigue expuesto para cualquiera que clone el repo.** Eso solo se detecta desde Claude Code.

---

## Instalación

### Opción A — Claude Code (recomendado)

Cloná la skill en tu carpeta de skills personales:

```bash
git clone https://github.com/paops-filadd/auditoriaseguridad.git \
  ~/.claude/skills/auditoriaseguridad
```

En Windows (PowerShell):

```powershell
git clone https://github.com/paops-filadd/auditoriaseguridad.git `
  "$env:USERPROFILE\.claude\skills\auditoriaseguridad"
```

Reiniciá Claude Code. Para actualizarla más adelante: `git -C ~/.claude/skills/auditoriaseguridad pull`.

> Cuando la skill se publique en el marketplace de Filadd se instala con `/plugin` y esta copia manual deja de hacer falta.

### Opción B — Lovable (modo degradado)

1. En Lovable: **Settings → Customization → Skills**
2. En **Workspace Skills**, clic en **Add → Import from GitHub**
3. Pegá `https://github.com/paops-filadd/auditoriaseguridad.git` y clic en **Import**

Queda disponible en todos los proyectos del workspace.

---

## Cómo usarla

**En Claude Code**, desde la carpeta del proyecto — con decirlo alcanza:

```
auditá la seguridad de este proyecto
```

o invocándola directo: `/auditoriaseguridad`

**En Lovable**, escribí `/` en el chat del proyecto y elegí `auditoriaseguridad`.

### Flujo típico en Claude Code

1. Conectá el proyecto de Lovable a GitHub y clonalo localmente.
2. Corré la auditoría. Escribe `AUDITORIA_SEGURIDAD.md`.
3. Leé el informe, sobre todo la sección **Qué no pude verificar**.
4. Si querés, pedile que aplique las correcciones: crea la branch `seguridad/fix-<fecha>`, un commit por hallazgo y abre el PR.
5. **Rotá a mano las claves que hayan quedado expuestas.** Borrarlas del código no las des-filtra.

---

## Qué audita

| Severidad | Qué revisa |
|---|---|
| 🔴 Crítico | Secretos y `service_role` keys en el código o en el historial · tablas sin RLS · policies abiertas · escalada de privilegios por columna de rol o por `user_metadata` · Edge Functions abiertas a internet |
| 🟠 Alto | IDOR y datos de otros usuarios accesibles · buckets de Storage públicos con archivos privados · controles de acceso que existen solo en el frontend |
| 🟡 Medio | Logs con datos sensibles · dependencias vulnerables con ruta de explotación · `security definer` sin `search_path` · errores que exponen información interna |
| 🔵 Informativo | Inventario de roles, rutas, servicios externos y variables de entorno |

La severidad se asigna por **explotabilidad real**, no por categoría. Un repositorio público sube la severidad de todo hallazgo de secreto.

### Lo que deliberadamente NO reporta

La skill trae una lista de falsos positivos obligatoria, para que el informe no pierda credibilidad:

- La **anon key** de Supabase en `src/integrations/supabase/client.ts` — es pública por diseño. Si las tablas están desprotegidas, el hallazgo es *"tabla sin RLS"*, no *"API key expuesta"*.
- La URL del proyecto Supabase y las claves publicables (Stripe `pk_`, Sentry DSN, PostHog).
- Vulnerabilidades de `npm audit` que solo afectan al entorno de desarrollo.
- Falta de tests, deuda técnica o código duplicado: no es seguridad.

---

## Estructura del repo

```
SKILL.md                      # El procedimiento: fases, reglas, severidad, falsos positivos
references/
  ├── deteccion.md            # Comandos exactos: git history, secretos, JWT, npm audit
  ├── supabase.md             # Los seis controles de RLS/policies/functions/storage, con los fixes SQL
  └── informe.md              # Plantilla de AUDITORIA_SEGURIDAD.md y convención de IDs
```

`SKILL.md` es autosuficiente para el modo Lovable. Los archivos de `references/` se cargan bajo demanda en Claude Code.

---

## Limitaciones

- Es **revisión estática de código**, no un pentest. No prueba la app corriendo.
- Sin `supabase/migrations/` en el repo no se puede verificar RLS: la skill lo declara en *Qué no pude verificar* en vez de inventar. Si tenés el connector de Supabase o acceso `psql`, la skill consulta `pg_policies` y `pg_class` y ahí el resultado es definitivo.
- No rota claves. Eso lo tiene que hacer una persona en el panel del proveedor.

---

Mantiene: Process Automation (PA-OPS) · Filadd
