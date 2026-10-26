# 🔐 Auditoría de Seguridad — Skill para Lovable

Skill para auditar la seguridad de proyectos hechos con Lovable (React + Supabase / Lovable Cloud). Corre dentro del agente de Lovable, que además del código **puede consultar la base de datos real del proyecto**: el RLS, las policies, los buckets y las funciones se verifican contra lo que está aplicado, no solo contra lo que dicen las migraciones.

Genera un archivo `AUDITORIA_SEGURIDAD.md` en la raíz del proyecto con los hallazgos clasificados por severidad, la evidencia de cada uno y lo que no se pudo verificar.

---

## 📦 Instalación

1. En Lovable: **Settings → Customization → Skills**
2. En **Workspace Skills**, clic en **Add → Import from GitHub**
3. Pegá la URL del repositorio:

```
https://github.com/paops-filadd/auditoriaseguridad.git
```

4. Clic en **Import**

Queda disponible en todos los proyectos del workspace.

---

## 🚀 Cómo usarla

Dentro de cualquier proyecto escribí `/` en el chat, elegí **`auditoriaseguridad`** y el agente arranca la auditoría.

1. El agente audita en **modo solo lectura**: no toca código ni base, solo escribe `AUDITORIA_SEGURIDAD.md`.
2. Leé el informe, sobre todo **Qué no pude verificar**.
3. Si querés, pedile que aplique las correcciones. Va de a un hallazgo por vez, empezando por los críticos, y cada cambio de base es una migración que tenés que aprobar.
4. **Rotá a mano las claves que hayan quedado expuestas.** Borrarlas del código no las des-filtra.

Si ya existía un informe anterior, la skill lo compara: marca qué se resolvió, qué sigue abierto y qué es nuevo, conservando los IDs.

---

## 📋 Qué audita

| Severidad | Qué revisa |
|---|---|
| 🔴 Crítico | `service_role` y secretos en el código · credenciales de Metabase en el cliente · variables `VITE_*` con claves privadas · tablas sin RLS · policies abiertas · escalada de privilegios por columna de rol o por `user_metadata` · vistas que saltean RLS · funciones `security definer` ejecutables por `anon` · Edge Functions abiertas a internet (también con `verify_jwt = true` sin `auth.getUser()`) · proxies a servicios externos sin auth, sin lista fija de recursos o con path traversal · funciones que envían correos o mensajes con destinatario o contenido del body (relay abierto) · buckets públicos con archivos privados |
| 🟠 Alto | IDOR y datos de otros usuarios accesibles · funciones RPC que no validan al usuario · Edge Functions que operan sobre un `user_id` o email recibido por body · URLs de webhook (Google Chat, Slack, n8n) en el código · signup abierto sin verificar email cuando habilita otro hallazgo |
| 🟡 Medio | Logs con datos sensibles · `security definer` sin `search_path` · errores que exponen información interna · API keys sin restricción de dominio · listado de Storage y oráculos de enumeración · envíos sin límite |
| 🔵 Informativo | Inventario de roles, rutas, servicios externos, variables y secretos · tablas en Realtime que el código no usa |

La severidad se asigna por **explotabilidad real**, no por categoría. Antes de clasificar datos como públicos, la skill **pregunta qué es público por diseño**, y verifica la base real (permisos `EXECUTE`, policies, configuración de Auth) en lugar de inferir desde las migraciones. Cuando dos o más hallazgos se combinan, el informe los muestra como **cadenas de ataque**.

### Lo que deliberadamente NO reporta

La skill trae una lista obligatoria de falsos positivos para que el informe no pierda credibilidad. Entre otros:

- La **anon key** / publishable key de Supabase y el `.env` que genera Lovable: son públicos por diseño. Si las tablas están desprotegidas, el hallazgo es *"tabla sin RLS"*, no *"API key expuesta"*.
- La `service_role` usada dentro de una Edge Function.
- Claves publicables (Stripe `pk_`, Sentry DSN, PostHog).
- Falta de tests o deuda técnica: no es seguridad.

---

## ⚠️ Limitaciones

Desde Lovable no hay terminal, así que estas tres cosas siempre quedan en *Qué no pude verificar*:

- **Historial de git.** Un secreto que se commiteó alguna vez sigue en GitHub aunque hoy no esté en el código. Activá *Secret scanning* en el repo.
- **Dependencias vulnerables** (`npm audit`). Activá Dependabot en el repo.
- **Visibilidad del repositorio** (público o privado).

Además: es revisión de código y configuración, no un pentest contra la app publicada; y no rota claves, eso lo tiene que hacer una persona en el panel del proveedor.

---

Mantiene: Process Automation (PA-OPS) · Filadd
