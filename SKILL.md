---
name: auditoriaseguridad
description: |
  Audita la seguridad de una app low-code (Lovable / v0 — React + Vite + Supabase) sobre su repositorio de GitHub: secretos en el código y en el historial de git, RLS y policies de Supabase, Edge Functions, Storage, autenticación y autorización, exposición de datos en el cliente, dependencias y configuración. Escribe AUDITORIA_SEGURIDAD.md y, si el usuario lo pide, aplica las correcciones en una branch y abre el PR.

  Usar cuando: piden auditar o revisar la seguridad de un proyecto, preguntan si una app hecha con Lovable es segura o si está lista para producción, o antes de publicar o compartir una app low-code con usuarios reales.

  NO usar para: revisar solamente el diff o el PR actual (para eso está /security-review), auditar repos que no sean apps web low-code, ni para atacar una URL en producción — esto es revisión de código, no pentesting.
---

# Auditoría de seguridad de apps Lovable

Auditoría estática del repositorio completo de una app generada con Lovable (React + Vite + TypeScript, backend Supabase) que hoy se itera desde Claude Code a través de GitHub.

El entregable es `AUDITORIA_SEGURIDAD.md` en la raíz del proyecto. La corrección es una **fase aparte** que solo se ejecuta si el usuario la pide.

---

## Reglas que gobiernan toda la auditoría

1. **Un hallazgo sin evidencia no es un hallazgo.** Cada uno lleva archivo y línea, nombre de la migración, o el comando que lo demostró. Lo que no pudiste verificar va a la sección *Qué no pude verificar*, no a la lista de hallazgos.
2. **La severidad se define por explotabilidad**, no por lo grave que suena la categoría.
3. **No infles el informe.** Si una categoría salió limpia, decilo explícitamente. Tres hallazgos reales valen más que veinte de relleno.
4. **La seguridad de una app Lovable vive en Supabase**, no en el frontend. Ocultar un botón no es un control de acceso. Si podés leer las migraciones, ahí está el 80% del valor de la auditoría.
5. **Antes de escribir el informe, pasá todos los hallazgos por la lista de falsos positivos.** Está más abajo y es obligatoria.

### Severidad

| Nivel | Criterio |
|---|---|
| 🔴 Crítico | Explotable **sin credenciales** desde internet, o permite leer/escribir datos de otras personas, o escalar a administrador. |
| 🟠 Alto | Explotable por **cualquier usuario registrado** de la app, o expone datos privados a quien conozca un ID o una URL. |
| 🟡 Medio | No explotable directamente, pero facilita un ataque o filtra información útil para atacar. |
| 🔵 Informativo | Inventario y contexto. No es un problema. |

**Un repo público sube la severidad.** Un secreto en el historial de un repo público es crítico siempre: asumí que ya fue scrapeado por un bot.

---

## Fase 0 — Reconocimiento

Nunca audites a ciegas. Primero establecé el terreno, porque define la severidad de todo lo demás.

```bash
# ¿Es una app Lovable? ¿Qué stack?
head -40 package.json
ls -la
ls supabase/ supabase/migrations/ supabase/functions/ 2>/dev/null

# Visibilidad del repo: cambia la severidad de cada secreto que encuentres
gh repo view --json nameWithOwner,visibility,isPrivate,pushedAt 2>/dev/null

# Commit auditado (va en el encabezado del informe)
git rev-parse --short HEAD && git log -1 --date=short --format='%ad'

# ¿Hay una auditoría previa para comparar?
ls AUDITORIA_SEGURIDAD.md 2>/dev/null
```

Anotá para el informe: nombre del repo, visibilidad, commit auditado, si existe `supabase/migrations/` (determina si podés verificar RLS de verdad o solo inferirlo), y si hay auditoría previa.

**Identificá la generación del stack**, porque cambia dónde mirar:

| Señal en `package.json` / `src/` | Generación | Consecuencia |
|---|---|---|
| `vite` + `react-router-dom`, `src/integrations/supabase/client.ts` | Lovable clásico (SPA) | Todo `src/` llega al navegador. Sin código de servidor. |
| `@lovable.dev/cloud-auth-js`, `src/integrations/lovable/`, TanStack Start | Lovable Cloud (SSR) | Los archivos `*.server.ts` y los middlewares **no** llegan al navegador: un secreto ahí puede ser legítimo. Verificalo antes de reportarlo. |

**Ojo con los Supabase de más de un proyecto.** Si aparece más de una URL `*.supabase.co`, el repo habla con una base que no es la suya. Las migraciones locales no dicen nada sobre esa otra base: su RLS va sí o sí a *Qué no pude verificar*, con el `ref` del proyecto anotado para que el usuario lo revise.

**Si existe `AUDITORIA_SEGURIDAD.md`**, leelo antes de empezar. Al final vas a marcar qué se resolvió, qué sigue abierto y qué es nuevo, **reusando los IDs anteriores**: `CRIT-001` sigue siendo `CRIT-001` aunque haya cambiado de línea.

---

## Fase 1 — Secretos en el código y en el historial

El historial de git es lo que el entorno de Claude Code habilita y el de Lovable no. **Un `.env` borrado hace veinte commits sigue expuesto en GitHub.**

Los comandos exactos están en `references/deteccion.md`. Qué buscar:

- Archivos `.env*` versionados, **hoy y en cualquier commit pasado**.
- Claves de proveedores en el árbol actual: `sk-`, `sk_live_`, `AIza`, `ghp_`, `github_pat_`, `xox[baprs]-`, `SG.`, bloques `PRIVATE KEY`.
- **JWT de Supabase con rol `service_role`** en cualquier archivo que llegue al cliente: decodificá el payload de todo `eyJ...` que encuentres y mirá el campo `role`. Esto es 🔴 crítico siempre.
- Variables `VITE_*`: **todo lo que empieza con `VITE_` termina en el bundle y es público.** Una `VITE_OPENAI_API_KEY` o cualquier `VITE_*_SECRET` es un secreto filtrado, no una variable de configuración.
- Alertas de secret scanning de GitHub, si el repo las tiene habilitadas.

Cada secreto encontrado necesita dos datos en el informe: **dónde está** y **si además está en el historial**, porque eso cambia la corrección — borrarlo del código no alcanza, hay que rotarlo.

---

## Fase 2 — Supabase: dónde vive la seguridad real

La fase más importante. El detalle, con los patrones SQL y el razonamiento de cada control, está en `references/supabase.md`.

Los seis controles, en orden de impacto:

1. **Tablas sin RLS.** Listá las tablas creadas en `supabase/migrations/` y restale las que tienen `ENABLE ROW LEVEL SECURITY`. La diferencia son tablas que cualquiera con la anon key (que es pública) puede leer enteras. 🔴 si contienen datos de personas.
2. **Policies abiertas.** `USING (true)`, o dirigidas `TO public` / `TO anon`, sobre tablas con datos privados. RLS habilitado con una policy abierta es igual a no tener RLS, y es peor porque parece seguro.
3. **Escalada de privilegios por columna de rol.** Si la columna `role` / `is_admin` vive en una tabla que el propio usuario puede actualizar (policy de UPDATE con `auth.uid() = id` sin restricción de columnas), cualquier usuario se hace admin solo. 🔴. El patrón correcto en Supabase es una tabla `user_roles` aparte más una función `security definer`.
4. **Autorización basada en `user_metadata`.** `raw_user_meta_data` lo edita el propio usuario con `supabase.auth.updateUser()`. Si el frontend o una policy deciden permisos leyendo `user_metadata.role`, es escalada directa. 🔴. Lo que el usuario no puede tocar es `app_metadata`.
5. **Edge Functions sin verificación de JWT.** Buscá `verify_jwt = false` en `supabase/config.toml`: esa función queda abierta a internet. Es válido solo si es un webhook que valida firma, o si la función hace su propia verificación adentro. Si escribe en la base sin validar nada, 🔴.
6. **Storage y Realtime.** Buckets creados con `public = true` que guardan archivos de usuarios; tablas sensibles agregadas a la publicación `supabase_realtime`.

Control secundario: funciones `SECURITY DEFINER` sin `SET search_path` (🟡 — es el `function_search_path_mutable` del linter de Supabase).

**Si no hay `supabase/migrations/`**, no podés verificar nada de esto desde el repo. No inventes: anotalo en *Qué no pude verificar* y pedile al usuario acceso al panel de Supabase.

---

## Fase 3 — Autenticación y autorización en el cliente

- ¿Las rutas protegidas verifican sesión, o solo esconden el componente? Escribir la URL a mano tiene que fallar.
- ¿Hay rutas de admin alcanzables por un usuario común escribiendo la URL?
- **Todo control que exista solo en el frontend es cosmético.** La pregunta que cierra cada hallazgo: *si el usuario abre la consola y llama a Supabase directo, ¿el dato sale igual?* Si sale, el hallazgo real está en Fase 2 y esto es el síntoma.
- Si el proyecto usa auth custom en vez de Supabase Auth, revisá cómo se valida el token y dónde se guarda.

---

## Fase 4 — Exposición de datos en el cliente

- `.from('tabla').select('*')` sin filtro de usuario: si RLS no lo tapa, devuelve la tabla entera.
- IDOR: pantallas que traen un registro por ID tomado de la URL sin validar pertenencia.
- `console.log` que imprimen tokens, sesiones, respuestas completas de API o datos de usuarios.
- Si existe `dist/`, buscá secretos en el bundle compilado.
- Comentarios y TODOs con datos reales, credenciales de prueba o URLs internas.

---

## Fase 5 — Dependencias y configuración

- `npm audit` de verdad, no leer `package.json` a ojo. Reportá **solo lo que tiene ruta de explotación en esta app** — ver falsos positivos.
- `.gitignore`: tiene que cubrir `.env`, `.env.local`, `.env.production`.
- Llamadas externas por HTTP en vez de HTTPS.
- Manejo de errores que devuelve stack traces o mensajes internos al usuario final.

---

## Fase 6 — Inventario (informativo)

Para la sección informativa: roles de usuario que existen en el código, tabla de rutas con su estado de protección (frontend y backend), servicios externos conectados, y variables de entorno en uso separadas en públicas (`VITE_*`) y de servidor.

---

## Falsos positivos — NO reportar como hallazgo

Esta lista es obligatoria. Reportar cualquiera de estos como problema hace que el resto del informe pierda credibilidad.

| No es hallazgo | Por qué |
|---|---|
| Un JWT de Supabase con rol **`anon`** hardcodeado en el cliente, **esté en el archivo que esté** | Es pública por diseño. Decide por el campo `role` del payload decodificado, nunca por la ruta del archivo: Lovable la escribe en `src/integrations/supabase/client.ts`, pero es igual de válida en un `src/lib/supabase-*.ts` escrito a mano. Si las tablas están desprotegidas, el hallazgo se llama *"tabla X sin RLS"* y va en Fase 2 — no *"API key expuesta"*. |
| La URL del proyecto (`https://xxxx.supabase.co`) | Es pública por diseño. |
| Secretos en archivos `*.server.ts` o `server/` de un proyecto con SSR (TanStack Start) | No llegan al navegador. Verificá que el archivo sea realmente de servidor antes de descartarlo. |
| Claves publicables declaradas como tales: Stripe `pk_`, Sentry DSN, PostHog project key | Están pensadas para vivir en el cliente. Van en el inventario informativo. |
| Vulnerabilidades de `npm audit` que solo afectan `devDependencies` o el dev server de Vite | No llegan a producción. 🟡 o informativo, nunca 🔴. |
| `console.log` sin datos sensibles | Es ruido, no seguridad. A lo sumo va en Quick Wins. |
| Falta de CSP o headers de seguridad en una SPA servida por Lovable | No se controla desde el repo. Informativo. |
| Ausencia de tests, deuda técnica, código duplicado | No es seguridad. Fuera del alcance. |

Excepción que **sí** es hallazgo: una API key de Google Maps o similar en el cliente **sin restricción de dominio**. No filtra datos, pero cualquiera la usa y la factura llega igual (🟡, riesgo de costo).

---

## Fase 7 — Escribir el informe

Generá `AUDITORIA_SEGURIDAD.md` en la raíz siguiendo la plantilla de `references/informe.md`. La estructura:

1. Encabezado: repo, visibilidad, commit auditado, fecha, alcance.
2. Resumen ejecutivo: 2-3 párrafos, en criollo, sin jerga.
3. Tabla de puntuación por severidad.
4. Quick Wins: fixes de menos de cinco minutos, con archivo y acción exacta.
5. Hallazgos detallados por severidad, cada uno con ID, archivo y línea, riesgo concreto, recomendación y la evidencia con que lo verificaste.
6. **Qué no pude verificar**, y qué haría falta para poder hacerlo.
7. Inventario informativo.
8. Las cinco acciones prioritarias.
9. Comparación con la auditoría previa, si existía.
10. Veredicto: ¿se puede publicar o no?

Cerrá preguntándole al usuario si querés que apliques las correcciones.

---

## Fase 8 — Corrección (solo si el usuario la pide)

No la ejecutes por iniciativa propia. Cuando la pida:

1. **Branch nueva**: `seguridad/fix-YYYY-MM-DD`. Nunca commitees a `main` directo — Lovable sincroniza `main` con el proyecto y un push roto le rompe el editor al usuario.
2. **Un commit por hallazgo**, con el ID adelante: `fix(seguridad): CRIT-001 habilitar RLS en profiles`.
3. **Los fixes de Supabase van en una migración nueva** dentro de `supabase/migrations/`. Nunca edites una migración ya aplicada.
4. **Los secretos hay que rotarlos, y eso no lo podés hacer vos.** Borrar la clave del código no la des-filtra: quien ya la copió la sigue teniendo. Decile al usuario, explícito y destacado, qué clave revocar y en qué panel. Si además está en el historial de un repo público, reescribir el historial tampoco alcanza — rotar es obligatorio.
5. **PR al final** con `gh pr create`, describiendo qué hallazgos cierra y cuáles quedaron pendientes de acción manual del usuario.
6. Avisá que después del merge conviene correr la auditoría de nuevo para actualizar el informe.

Priorizá 🔴 y 🟠. Los 🟡 solo si el usuario lo pide expresamente.

---

## Modo Lovable (sin terminal)

Si la skill corre dentro del agente de Lovable, sin bash ni git, se pierden el historial de git, `npm audit`, la visibilidad del repo y la decodificación de JWT. Lo que **sí** se puede hacer leyendo archivos:

- Fases 2, 3, 4 y 6 completas, y la parte de Fase 1 que mira el árbol actual (`.env`, `VITE_*`, claves hardcodeadas).
- Mismas reglas, misma tabla de severidad, misma lista de falsos positivos, mismo informe.
- En *Qué no pude verificar* declará siempre: historial de git, dependencias vulnerables y visibilidad del repositorio.
- Fase 8 no aplica. En su lugar cerrá el informe con: *"pedile a Lovable: aplicá las recomendaciones prioritarias de AUDITORIA_SEGURIDAD.md"*.

Para auditorías serias, recomendale al usuario conectar el proyecto a GitHub y correr la skill desde Claude Code.
