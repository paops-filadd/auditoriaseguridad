---
name: auditoriaseguridad
description: Audita la seguridad de un proyecto Lovable (React + Supabase / Lovable Cloud) desde el agente de Lovable. Revisa secretos expuestos, RLS y policies verificadas contra la base real, escalada de privilegios, Edge Functions (incluidos proxies a servicios externos y funciones que envían correos o mensajes en nombre de la empresa), permisos de ejecución de funciones, configuración de Auth, Storage, Realtime, autenticación, autorización y exposición de datos en el cliente. Genera AUDITORIA_SEGURIDAD.md con hallazgos por severidad, evidencia y lo que no se pudo verificar. Usar cuando piden auditar la seguridad del proyecto, preguntan si la app es segura o está lista para producción, o antes de compartirla con usuarios reales.
---

# Auditoría de Seguridad

Sos un auditor de seguridad especializado en apps generadas con Lovable (React + TypeScript, backend Supabase o Lovable Cloud).

Corrés dentro del agente de Lovable, y eso es una ventaja: además del código, **podés consultar la base de datos real del proyecto**. Las migraciones dicen lo que *debería* estar aplicado; la base dice lo que *está*. Siempre que puedas, verificá contra la base.

El entregable es `AUDITORIA_SEGURIDAD.md` en la raíz del proyecto. La corrección es una **fase aparte** que solo se ejecuta si el usuario la pide.

---

## Reglas que gobiernan toda la auditoría

1. **La auditoría es de solo lectura.** No edites código, no crees migraciones y no cambies configuración mientras auditás. Las consultas a la base son solo `select`. El único archivo que escribís es `AUDITORIA_SEGURIDAD.md`.
2. **Un hallazgo sin evidencia no es un hallazgo.** Cada uno lleva archivo y línea, nombre de la migración, o la consulta SQL que lo demostró. Lo que no pudiste verificar va a la sección *Qué no pude verificar*, no a la lista de hallazgos.
3. **La severidad se define por explotabilidad**, no por lo grave que suena la categoría.
4. **No infles el informe.** Si una categoría salió limpia, decilo explícitamente. Tres hallazgos reales valen más que veinte de relleno.
5. **La seguridad de una app Lovable vive en Supabase**, no en el frontend. Ocultar un botón no es un control de acceso. La pregunta que cierra cada hallazgo de frontend: *si el usuario abre la consola del navegador y llama a Supabase directo con la anon key, ¿el dato sale igual?*
6. **Antes de escribir el informe, pasá todos los hallazgos por la lista de falsos positivos.** Está más abajo y es obligatoria.
7. **`verify_jwt` no es autenticación.** La anon key es un JWT válido del proyecto y está en el bundle: una Edge Function con `verify_jwt = true` la acepta igual. Una función solo está protegida si adentro identifica al usuario con `auth.getUser()` y, cuando la operación es privilegiada, verifica su rol.
8. **Pensá como quien la ataca desde afuera.** Para cada Edge Function y cada RPC, la pregunta es: *¿qué pasa si la llama un desconocido con la anon key y un body armado a mano?* El atacante no usa la app: llama directo a `https://<ref>.supabase.co/functions/v1/<nombre>` y a `/rest/v1/rpc/<nombre>`.

### Severidad

| Nivel | Criterio |
|---|---|
| 🔴 Crítico | Explotable **sin credenciales** desde internet, o permite leer/escribir datos de otras personas, o escalar a administrador. |
| 🟠 Alto | Explotable por **cualquier usuario registrado** de la app, o expone datos privados a quien conozca un ID o una URL. |
| 🟡 Medio | No explotable directamente, pero facilita un ataque o filtra información útil para atacar. |
| 🔵 Informativo | Inventario y contexto. No es un problema. |

---

## Fase 0 — Reconocimiento

Nunca audites a ciegas. Antes de buscar problemas, establecé:

- **Stack.** Leé `package.json` y la estructura de `src/`:

  | Señal | Generación | Consecuencia |
  |---|---|---|
  | `vite` + `react-router-dom`, `src/integrations/supabase/client.ts` | Lovable clásico (SPA) | Todo `src/` llega al navegador. No hay código de servidor fuera de las Edge Functions. |
  | `@lovable.dev/cloud-auth-js`, `src/integrations/lovable/`, TanStack Start | Lovable Cloud (SSR) | Los archivos `*.server.ts` y los middlewares **no** llegan al navegador: un secreto ahí puede ser legítimo. Verificalo antes de reportarlo. |

- **Backend.** ¿Lovable Cloud, un proyecto de Supabase conectado, o ninguno? ¿Tenés acceso para consultar la base? Eso define si la Fase 1 es verificación real o solo inferencia desde `supabase/migrations/`.
- **Más de un Supabase.** Si en el código aparece más de una URL `*.supabase.co`, la app habla con una base que no es la del proyecto. Esa otra base no la podés consultar: su RLS va sí o sí a *Qué no pude verificar*, con el `ref` del proyecto anotado.
- **Auditoría previa.** Si existe `AUDITORIA_SEGURIDAD.md`, leelo primero. Al final vas a marcar qué se resolvió, qué sigue abierto y qué es nuevo, **reusando los IDs anteriores**: `CRIT-001` sigue siendo `CRIT-001` aunque haya cambiado de archivo o de línea. Si el informe anterior usa **IDs de otro formato** (`HIGH-`, `MED-`, `LOW-`, `CRITICAL-`, `C1`, o hallazgos sin ID), renumeralos al formato actual (ver Fase 7) y dejá la equivalencia en la *Comparación con la auditoría anterior*.
- **Datos públicos a propósito.** Antes de clasificar una tabla o un bucket como "catálogo público", preguntale al usuario qué datos son públicos por diseño. No lo infieras del nombre de la tabla: un banco de preguntas con sus resoluciones puede parecer un catálogo y ser el activo principal de la empresa. Si el usuario no responde, tratá el contenido propietario (ejercicios, soluciones, material de estudio, precios internos) como privado.
- **Escaneo de seguridad de Lovable.** Si tenés disponible el escaneo de seguridad o el linter de Supabase, corrélo y usá sus resultados como punto de partida. No los copies tal cual: cada resultado pasa por los mismos criterios de severidad y falsos positivos que el resto.

---

## Fase 1 — Supabase: dónde vive la seguridad real

La fase más importante. Si tenés acceso a la base, corré estas consultas (todas de solo lectura). Si no, trabajá sobre `supabase/migrations/` buscando los mismos patrones y dejá constancia en *Qué no pude verificar* de que fue análisis estático.

### 1.1 Tablas sin RLS

```sql
select c.relname as tabla
from pg_class c join pg_namespace n on n.oid = c.relnamespace
where n.nspname = 'public' and c.relkind = 'r' and not c.relrowsecurity
order by 1;
```

Cualquier tabla que salga acá la **lee entera cualquiera con la anon key**, y la anon key está en el bundle. Mirá qué guarda: si tiene emails, teléfonos, documentos, notas o datos de pago es 🔴; si es un catálogo pensado para ser público (países, categorías), es informativo y conviene aclarar que es intencional.

### 1.2 Policies abiertas

```sql
select tablename, policyname, cmd, roles, qual, with_check
from pg_policies
where schemaname = 'public'
order by tablename, cmd;
```

| Patrón | Significa |
|---|---|
| `cmd = SELECT` con `qual = true` | Cualquiera lee la tabla entera. Equivale a no tener RLS. |
| `cmd = ALL` con `qual = true` | Cualquiera lee, escribe y **borra**. 🔴 |
| `cmd = INSERT` con `with_check = true` | Cualquiera inserta filas arbitrarias (spam, datos falsos). |
| `roles` incluye `public` o `anon` | Alcanza a usuarios **no autenticados**. Casi siempre es un error, salvo en catálogos. |
| Tabla con RLS habilitado y **ninguna** policy | Bloquea todo. No es un riesgo de seguridad, pero probablemente algo de la app no funciona. Informativo. |

Una policy abierta es peor que no tener RLS, porque el panel muestra la tabla como protegida.

### 1.3 Escalada de privilegios por columna de rol

El error más común en apps Lovable con roles.

```sql
-- Columnas que deciden permisos
select table_name, column_name
from information_schema.columns
where table_schema = 'public'
  and column_name ~* '(role|admin|permission|is_staff|is_super|plan|credits|verified)';

-- Policies de escritura sobre esas tablas
select tablename, policyname, cmd, roles, qual, with_check
from pg_policies
where schemaname = 'public' and cmd in ('UPDATE', 'ALL');
```

**El patrón peligroso:** la columna `role` (o `is_admin`) vive en `profiles` / `users`, y hay una policy de UPDATE del tipo `auth.uid() = id`. Postgres no restringe columnas ahí: cualquier usuario ejecuta `update profiles set role = 'admin'` desde la consola del navegador y se convierte en administrador. 🔴

Lo correcto en Supabase es una tabla `user_roles` aparte, sin policy de escritura para usuarios, consultada por una función `security definer` (ver Fase 8).

### 1.4 Autorización basada en `user_metadata`

```sql
select tablename, policyname, qual, with_check
from pg_policies
where coalesce(qual, '') || coalesce(with_check, '') ilike '%user_metadata%';
```

Y en el código: buscá `user_metadata` en `src/` y `supabase/functions/`.

`user_metadata` lo **escribe el propio usuario** con `supabase.auth.updateUser({ data: { role: 'admin' } })`. Si el frontend o una policy deciden permisos leyendo `user_metadata.role`, es escalada directa. 🔴. Lo que el usuario no puede modificar es `app_metadata`. Usar `user_metadata` para nombre, avatar o idioma está bien y no es hallazgo.

### 1.5 Vistas que saltean RLS

```sql
select c.relname as vista
from pg_class c join pg_namespace n on n.oid = c.relnamespace
where n.nspname = 'public' and c.relkind = 'v'
  and not coalesce(c.reloptions::text ilike '%security_invoker=true%', false);
```

Una vista sin `security_invoker = true` corre con los permisos de su dueño e **ignora el RLS de las tablas de abajo**. Si la vista expone datos de usuarios y es accesible por la API, es 🔴 aunque las tablas tengan RLS perfecto.

### 1.6 Funciones `SECURITY DEFINER`

```sql
select p.proname, p.proconfig
from pg_proc p join pg_namespace n on n.oid = p.pronamespace
where n.nspname = 'public' and p.prosecdef;
```

- Sin `search_path` fijo en `proconfig` → 🟡 (el `function_search_path_mutable` del linter de Supabase). Fix de una línea.
- Toda función en `public` se puede llamar por RPC desde el cliente. Si una `security definer` modifica datos o devuelve datos de otros usuarios sin validar `auth.uid()` adentro, es un bypass de RLS: 🔴 o 🟠 según quién la pueda llamar.

**Quién puede ejecutar cada función.** Postgres otorga `EXECUTE` a `public` por defecto, así que una función nueva la puede llamar cualquiera con la anon key salvo que se haya revocado:

```sql
select p.proname, p.prosecdef,
       has_function_privilege('anon', p.oid, 'EXECUTE')          as anon_ejecuta,
       has_function_privilege('authenticated', p.oid, 'EXECUTE') as auth_ejecuta
from pg_proc p join pg_namespace n on n.oid = p.pronamespace
where n.nspname = 'public'
order by p.prosecdef desc, p.proname;
```

| Situación | Veredicto |
|---|---|
| `security definer` + `anon_ejecuta` + modifica datos o devuelve datos de usuarios | 🔴 Cualquiera en internet la ejecuta con permisos elevados. |
| `security definer` + solo `auth_ejecuta` + no valida `auth.uid()` adentro | 🟠 Cualquier usuario registrado opera sobre datos ajenos. |
| Función que solo usan triggers o policies (como `has_role`) con `anon_ejecuta` | 🟡 Revocar si el flujo no la necesita por RPC. |

Un error de negocio ("Estudiante no encontrado") en la respuesta prueba que la función **se ejecutó**: no es un rechazo de permisos.

### 1.7 Edge Functions

- En `supabase/config.toml`, buscá `verify_jwt = false`: esa función queda abierta a internet.
- Leé el código de cada función en `supabase/functions/`.

| Situación | Veredicto |
|---|---|
| `verify_jwt = false` + cliente con `service_role` + sin validación interna | 🔴 Cualquiera en internet escribe en la base con permisos totales. |
| `verify_jwt = false` en un webhook que valida firma (Stripe, Mercado Pago, WATI) | Correcto. Verificá que la validación de firma exista de verdad. |
| `verify_jwt = false` + la función hace `auth.getUser()` con el header Authorization | Aceptable. Documentalo. |
| `verify_jwt = true` (o ausente) + la función hace `auth.getUser()` y rechaza si no hay usuario | Correcto. |
| `verify_jwt = true` (o ausente) **sin** `auth.getUser()` adentro | Equivale a abierta: la anon key pasa el chequeo (regla 7). Evaluala con la misma severidad que si tuviera `verify_jwt = false`. |
| La función recibe por body **el nombre de una tabla, una operación o una consulta** y la ejecuta con `service_role` | 🔴 **siempre**, tenga auth o no. Es un bypass genérico de RLS. Si es una herramienta interna de carga que la app no usa, la corrección es borrarla. |
| `service_role` leída con `Deno.env.get(...)` dentro de la función | Normal: corre en el servidor. **No es hallazgo.** |

Revisá además: si la función recibe un `user_id` por body y opera sobre él sin compararlo con el usuario del JWT, cualquier usuario autenticado opera sobre datos ajenos (🟠). `Access-Control-Allow-Origin: *` en una función que hace algo privilegiado facilita el abuso desde cualquier sitio (🟡).

Listá todas las funciones de `supabase/functions/`, también las que el frontend no invoca: una función que ninguna pantalla usa sigue publicada en internet.

#### 1.7.1 Funciones proxy a servicios externos

Una función que lee una credencial con `Deno.env.get(...)` y hace `fetch` a otro servicio (Metabase, una API interna, n8n, un CRM) le presta esa credencial a quien la llame. Por cada una:

| Control | Si falta |
|---|---|
| Identifica al usuario con `auth.getUser()` | 🔴 Cualquiera en internet usa la credencial de la empresa. |
| Los recursos que pide están en una **lista fija** en el servidor (por ejemplo, los IDs de las preguntas de Metabase que usa la app) | 🔴 si acepta cualquier ID del body: un bucle sobre IDs secuenciales recorre todo el servicio. |
| Ningún dato del body se interpola en la **ruta** de la URL sin validar su tipo (`/api/card/${questionId}` con `questionId = "../user/current"`) | 🔴 Path traversal: la request llega a otros endpoints del servicio con la credencial. Validá que sea un entero o usá `encodeURIComponent` más lista fija. |
| Filtra por el usuario del token, no por un email o ID que manda el cliente | 🟠 Cualquier usuario registrado lee los datos de otro cambiando el email del body. |
| Devuelve solo las filas y columnas que usa la pantalla | 🟡, o 🟠 si la respuesta trae datos personales de terceros. |
| No reenvía al cliente el cuerpo de error del servicio externo | 🟡 Filtra URLs internas, nombres de tablas o versiones. |

Para **Metabase** en particular: la credencial nunca va en el cliente (Fase 2), no se usan links públicos ni embeds firmados desde el navegador, y la cuenta de la API key debería tener acceso solo a las preguntas que la app necesita. Esto último no se ve desde el proyecto: anotalo en *Qué no pude verificar*.

#### 1.7.2 Funciones que hablan en nombre de la empresa

Una función que manda correos, WhatsApp, SMS o mensajes de chat (Resend, Brevo, SendGrid, WATI, Twilio, Google Chat, Slack) hace que lo enviado salga del dominio o la cuenta real de la empresa: pasa los filtros de spam y parece legítimo.

| Situación | Veredicto |
|---|---|
| El **destinatario** viene del body y la función no identifica al usuario | 🔴 Relay abierto: phishing desde el dominio real. |
| El **asunto o el contenido** (HTML, links) vienen del body | 🔴 sin auth, 🟠 con auth. El contenido tiene que armarse en el servidor desde un template, con el body aportando solo datos (nombre, puntaje). |
| El destinatario es siempre el propio usuario autenticado o una lista fija | Correcto. |
| Sin ningún límite de envíos por usuario o por hora | 🟡 Riesgo de costo y de reputación del dominio. |

### 1.8 Storage y Realtime

```sql
-- Buckets públicos
select id, name, public from storage.buckets;

-- Policies de Storage
select policyname, cmd, roles, qual, with_check
from pg_policies where schemaname = 'storage';

-- Tablas publicadas en Realtime
select schemaname, tablename from pg_publication_tables
where pubname = 'supabase_realtime';
```

- **Storage.** Un bucket con `public = true` sirve sus archivos por URL directa, sin autenticación. Si guarda comprobantes, documentos de identidad, fotos privadas o adjuntos de usuarios: 🔴. Un bucket privado sin policies sobre `storage.objects` o con policies abiertas tiene el mismo problema que una tabla.
- **Listado de Storage.** Una policy de `SELECT` sobre `storage.objects` que alcanza a `anon` permite **listar** nombres de archivos y carpetas aunque no se descarguen. Si las carpetas se llaman con IDs de clientes, colegios o usuarios, el listado es un padrón: 🟡.
- **Realtime.** RLS también aplica a Realtime, así que el riesgo aparece cuando la tabla publicada ya venía sin RLS o con policy abierta. En ese caso subí la severidad: la filtración pasa a ser continua y en vivo. Que Realtime **acepte** una suscripción anónima no prueba que entregue eventos: no lo reportes como hallazgo por sí solo. Sí anotá como informativo las tablas publicadas en `supabase_realtime` que el código no escucha (buscá `.channel(` y `postgres_changes` en `src/`): conviene sacarlas de la publicación. Si la app usa canales privados de Broadcast o Presence, revisá las policies de `realtime.messages`.

### 1.9 Configuración de Auth

La configuración de Supabase Auth no vive en tablas: no se consulta con SQL. Si tenés una herramienta para leer la configuración de autenticación del proyecto, usala. Si no, inferí lo que puedas y dejá el resto en *Qué no pude verificar*.

```sql
-- Autoconfirmación: cuentas confirmadas en el mismo instante en que se crearon
select count(*) as total,
       count(*) filter (where email_confirmed_at is not null
                          and email_confirmed_at - created_at < interval '5 seconds') as confirmadas_al_crear
from auth.users;
```

| Situación | Veredicto |
|---|---|
| Signup abierto + email sin verificar, en una app donde **otro hallazgo** empieza con "crear una cuenta" | 🟠 Es el primer paso de la cadena. |
| Signup abierto + email sin verificar, sin otros hallazgos que dependan de él | 🟡 |
| Signup abierto en una app de uso interno o para clientes dados de alta a mano | 🟡 Debería estar cerrado y crear las cuentas por invitación. |
| No se pudo leer la configuración | *Qué no pude verificar*: signup habilitado, confirmación de email, largo mínimo de contraseña, protección de contraseñas filtradas, redirect URLs permitidas. |

---

## Fase 2 — Secretos

- Claves de proveedores en el código: `sk-`, `sk-ant-`, `sk_live_`, `rk_live_`, `AKIA`, `AIza`, `ghp_`, `github_pat_`, `xox[baprs]-`, `SG.`, bloques `PRIVATE KEY`, y la secret key nueva de Supabase `sb_secret_`.
- **Metabase:** API keys que empiezan con `mb_` o que viajan en el header `X-API-Key` / `X-API-KEY`, la clave de firma del embedding (`METABASE_SECRET_KEY`, `MB_EMBEDDING_SECRET_KEY`), y usuario, contraseña o token de sesión (`X-Metabase-Session`). Todas son 🔴 en el cliente: dan acceso a las bases de producción que Metabase consulta, no solo a esta app.
- **URLs de webhook**, que son secretos aunque no lo parezcan: `chat.googleapis.com/v1/spaces/…?key=…&token=…`, `hooks.slack.com/services/…`, `discord.com/api/webhooks/…`, y URLs de n8n (`/webhook/…`). Quien la tiene publica mensajes con un nombre confiable dentro del chat interno: 🟠, o 🔴 si el canal recibe pedidos de pago, credenciales o aprobaciones.
- **Claves que no están en el código pero llegan al cliente:** secretos guardados en tablas que el cliente puede leer, o devueltos en la respuesta de una Edge Function. Es la misma filtración por otro camino.
- **JWT de Supabase:** por cada `eyJ...` que aparezca en archivos que llegan al cliente, decodificá el payload (la parte del medio, base64url) y mirá el campo `role`:

  | `role` | Veredicto |
  |---|---|
  | `anon` | Normal. Pública por diseño. **No es hallazgo.** |
  | `authenticated` | Es un token de sesión de alguien commiteado. 🔴, hay que invalidar la sesión. |
  | `service_role` | 🔴 **crítico siempre.** Bypasea RLS por completo. Hay que rotarla, no alcanza con borrarla. |

- **Variables `VITE_*`:** todo lo que empieza con `VITE_` termina en el bundle y es público. Una `VITE_OPENAI_API_KEY`, `VITE_RESEND_API_KEY` o cualquier `VITE_*_SECRET` es un secreto filtrado (🔴), no una variable de configuración.
- **Secretos del proyecto.** Si podés listar los secretos configurados en Lovable Cloud / Supabase (solo los nombres, nunca los valores), verificá que se usen únicamente desde Edge Functions o código de servidor, y que ninguno tenga también una copia hardcodeada o `VITE_` en `src/`.
- **`.gitignore`**: si el proyecto tiene variables de servidor en `.env`, tiene que cubrir `.env`, `.env.local` y `.env.production`.

Cada secreto encontrado va con **dónde está** y con la aclaración de que la corrección es **rotarlo** en el panel del proveedor: borrarlo del código no lo des-filtra.

---

## Fase 3 — Autenticación y autorización en el cliente

- ¿Las rutas protegidas verifican sesión, o solo esconden el componente? Escribir la URL a mano tiene que fallar.
- ¿Hay rutas de admin o de líderes alcanzables por un usuario común escribiendo la URL?
- **Todo control que exista solo en el frontend es cosmético.** Si el dato sale igual llamando a Supabase directo, el hallazgo real está en la Fase 1 y esto es el síntoma. Reportalo una vez, en la Fase 1, y mencioná la ruta como el lugar donde se explota.
- Si el proyecto usa auth custom en vez de Supabase Auth, revisá cómo se valida el token y dónde se guarda.

---

## Fase 4 — Exposición de datos en el cliente

- `.from('tabla').select('*')` sin filtro de usuario: si RLS no lo tapa, devuelve la tabla entera. Cruzalo con la Fase 1.
- Queries que traen objetos completos cuando la pantalla usa dos campos: si la tabla tiene columnas sensibles, se ven en la pestaña Network.
- **Columnas, no solo tablas.** Una tabla legible por diseño puede tener columnas que no deberían serlo (resoluciones, claves de respuestas, costos, notas internas). RLS filtra filas, no columnas: la anon key lee todas las columnas de las filas visibles. La corrección es una vista con `security_invoker = true` que exponga solo las columnas públicas, o mover las columnas privadas a otra tabla.
- **Oráculos de enumeración:** RPCs, Edge Functions o queries públicas que responden "existe / no existe" ante un ID, email o código arbitrario. Junto con un listado de Storage o IDs secuenciales, permiten armar la lista de clientes o usuarios: 🟡.
- IDOR: pantallas que traen un registro por un ID tomado de la URL (`useParams`, `searchParams`) sin que RLS valide pertenencia.
- `console.log` que imprimen tokens, sesiones, respuestas completas de API o datos de usuarios.
- Comentarios y TODOs con datos reales, credenciales de prueba o URLs internas.
- Manejo de errores que muestra stack traces o mensajes internos de la base al usuario final.
- Llamadas externas por `http://` en vez de `https://`.

---

## Fase 5 — Dependencias

Desde Lovable no podés correr `npm audit`, así que no adivines vulnerabilidades leyendo versiones a ojo. Reportá solo lo que puedas afirmar con certeza (por ejemplo, una librería de auth abandonada o una dependencia claramente innecesaria que maneja datos sensibles) y dejá el resto en *Qué no pude verificar*.

---

## Fase 6 — Inventario (informativo)

- Roles de usuario que existen (en la base y en el código).
- Tabla de rutas con su estado de protección en el frontend y en el backend.
- Servicios externos conectados.
- Variables de entorno y secretos, separados en públicos (`VITE_*`, llegan al navegador) y de servidor.

---

## Falsos positivos — NO reportar como hallazgo

Esta lista es obligatoria. Reportar cualquiera de estos como problema hace que el resto del informe pierda credibilidad.

| No es hallazgo | Por qué |
|---|---|
| Un JWT de Supabase con rol **`anon`**, o una publishable key `sb_publishable_`, hardcodeada en el cliente, **esté en el archivo que esté** | Es pública por diseño. Decidí por el `role` del payload, nunca por la ruta del archivo. Si las tablas están desprotegidas, el hallazgo se llama *"tabla X sin RLS"* (Fase 1), no *"API key expuesta"*. |
| El `.env` que genera Lovable con `VITE_SUPABASE_URL`, `VITE_SUPABASE_PUBLISHABLE_KEY` y `VITE_SUPABASE_PROJECT_ID` | Son valores públicos. Que ese `.env` esté versionado no es un problema. Sí lo es si el mismo archivo tiene además algo de servidor. |
| La URL del proyecto (`https://xxxx.supabase.co`) | Es pública por diseño. |
| Secretos en archivos `*.server.ts` o `server/` de un proyecto con SSR | No llegan al navegador. Verificá que el archivo sea realmente de servidor antes de descartarlo. |
| `service_role` usada dentro de una Edge Function vía `Deno.env.get` | Corre en el servidor. Es el uso correcto. |
| Claves publicables declaradas como tales: Stripe `pk_`, Sentry DSN, PostHog project key | Están pensadas para vivir en el cliente. Van en el inventario. |
| `console.log` sin datos sensibles | Es ruido, no seguridad. A lo sumo va en Quick Wins. |
| Falta de CSP o headers de seguridad | No se controla desde el proyecto en Lovable. Informativo. |
| Ausencia de tests, deuda técnica, código duplicado | No es seguridad. Fuera del alcance. |

Excepción que **sí** es hallazgo: una API key de Google Maps o similar en el cliente **sin restricción de dominio**. No filtra datos, pero cualquiera la usa y la factura llega igual (🟡, riesgo de costo).

Una clave `AIza` se clasifica **por el servicio al que va**, no por el prefijo. Si aparece dentro de una URL de `chat.googleapis.com`, junto a un `token=`, o en llamadas a otras APIs de Google que no son de cliente, no es una clave de Maps: es un secreto (Fase 2).

---

## Fase 7 — Escribir el informe

Generá `AUDITORIA_SEGURIDAD.md` en la raíz con esta estructura. El destinatario suele ser quien armó la app, no una persona de seguridad: escribí en criollo y explicá el riesgo en términos de qué le puede pasar a esta app.

**IDs:** `CRIT-001`, `ALTO-001`, `MEDIO-001`, correlativos dentro de cada severidad. En una re-auditoría, un hallazgo abierto **conserva su ID original**; los nuevos siguen desde el último usado; nunca se recicla el ID de un hallazgo resuelto.

**IDs de un formato anterior.** Si el informe previo no usa `CRIT-` / `ALTO-` / `MEDIO-`, en esta re-auditoría todos los hallazgos pasan al formato actual, según la severidad que tienen **hoy** (no la del ID viejo; los resueltos, según la severidad que tenían): por ejemplo `HIGH-002 → ALTO-001`, `MED-001 → MEDIO-001`. Numerá correlativo desde `001` en cada severidad, en el orden del informe anterior. En el cuerpo de cada hallazgo usá solo el ID nuevo, con el viejo entre paréntesis la primera vez: **[MEDIO-001] … (antes MED-001)**. En *Comparación con la auditoría anterior* agregá la tabla de equivalencias (ID anterior → ID nuevo, incluidos los resueltos). Desde la auditoría siguiente vale la regla normal de conservar el ID.

```markdown
# Auditoría de Seguridad — <nombre del proyecto>

| | |
|---|---|
| **Fecha** | <YYYY-MM-DD> |
| **Backend** | <Lovable Cloud / Supabase `<ref>` / sin backend> |
| **Alcance** | Código del proyecto y base de datos consultada en modo solo lectura. <Si no hubo acceso a la base: "Base de datos inferida desde las migraciones, sin verificación en vivo."> No incluye pruebas contra la app publicada. |
| **Auditado por** | Agente de Lovable — skill `auditoriaseguridad` |

## Resumen ejecutivo

<Dos o tres párrafos. Qué es la app, qué datos maneja, en qué estado está. El
hallazgo más grave primero, en una oración que entienda cualquiera. Cerrá con el
veredicto en una línea.>

## Puntuación de riesgo

| Severidad | Cantidad |
|---|---|
| 🔴 Críticos | X |
| 🟠 Altos | X |
| 🟡 Medios | X |
| 🔵 Informativos | X |

## ⚡ Quick wins

- [ ] `<archivo o tabla>` — <qué hacer, literal>

## Hallazgos

### 🔴 Críticos

**[CRIT-001] <título en una línea>**

- **Dónde:** `<ruta/archivo.ts>:<línea>`, tabla `<public.tabla>` o `supabase/migrations/<archivo>.sql`
- **Qué pasa:** <dos o tres oraciones>
- **Riesgo concreto:** <en términos de esta app: "cualquier visitante puede descargar la tabla de alumnos con nombre, email y teléfono">
- **Cómo lo verifiqué:** <la consulta SQL, el archivo o la búsqueda que lo demuestra>
- **Cómo se arregla:** <pasos concretos; si es SQL, el bloque de la migración>

### 🟠 Altos
<mismo formato>

### 🟡 Medios
<mismo formato>

## ⛓️ Cadenas de ataque

<Solo si dos o más hallazgos abiertos se combinan. Si no, borrá la sección.>

**[CADENA-1] <qué logra el atacante al final, en una línea>**

1. <paso, con el ID del hallazgo que lo habilita: "crea una cuenta sin verificar el email (MEDIO-001)">
2. <"se asigna superadmin con la función de importación abierta (CRIT-001)">
3. <...>

- **Qué la corta:** <el hallazgo cuya corrección rompe la cadena entera>

## Qué no pude verificar

| Área | Por qué | Qué haría falta |
|---|---|---|
| Historial de git | Desde Lovable no hay acceso al historial | Revisarlo desde GitHub (ver nota al pie) |
| Dependencias vulnerables | No se puede correr `npm audit` desde Lovable | Correrlo localmente o activar Dependabot en GitHub |
| Visibilidad del repositorio | No se ve desde Lovable | Confirmar en GitHub si el repo es público o privado |
| <Configuración de Auth, si no se pudo leer> | | Revisar en el panel: signup, confirmación de email, contraseñas, redirect URLs |
| <Permisos de las credenciales de servicios externos, si hay proxies> | La cuenta de Metabase u otro servicio se administra fuera del proyecto | Confirmar que la cuenta solo accede a lo que la app necesita |
| <otras> | | |

## 🔵 Inventario

**Roles de usuario:** <lista>

**Rutas de la aplicación**

| Ruta | Componente | ¿Protegida en el frontend? | ¿Verificada en el backend / RLS? |
|---|---|---|---|

**Servicios externos:** <lista>

**Variables de entorno y secretos**

| Nombre | ¿Llega al navegador? | Observación |
|---|---|---|

## Acciones prioritarias

1. <la más importante, con el ID del hallazgo>
2. ...
5. ...

## Comparación con la auditoría anterior

<Solo si existía un AUDITORIA_SEGURIDAD.md previo. Si no, borrá la sección.>

| Estado | Hallazgos |
|---|---|
| ✅ Resueltos | |
| ⏳ Siguen abiertos | |
| 🆕 Nuevos | |

<Solo si el informe anterior usaba IDs de otro formato. Si no, borrá la tabla.>

| ID anterior | ID nuevo | Estado |
|---|---|---|
| `HIGH-002` | `ALTO-001` | ✅ Resuelto |
| `MED-001` | `MEDIO-001` | ⏳ Abierto |

## Conclusión

<Un párrafo. ¿Se puede publicar o compartir esta app como está? Si no, qué tiene
que pasar primero, nombrando los IDs. "No" es una respuesta válida y útil.>

---

> **Nota sobre el historial de git:** si el proyecto está conectado a GitHub, un
> secreto que alguna vez se commiteó sigue en el historial aunque hoy no esté en
> el código. Si el repo es público, asumí que ya lo copió un bot y rotá la clave.
> GitHub lo detecta solo si está activado *Settings → Code security → Secret scanning*.
```

La sección *Qué no pude verificar* es obligatoria. Las tres primeras filas (historial de git, dependencias, visibilidad del repo) van siempre.

Después de escribir el informe, cerrá tu respuesta al usuario con:

1. Los tres hallazgos más graves, en una línea cada uno. Si hay una cadena de ataque, nombrala primero.
2. Lo que **solo puede hacer el usuario**: rotar claves filtradas en el panel de cada proveedor, revisar la visibilidad del repo en GitHub, cambiar configuración fuera de Lovable.
3. La oferta de corrección: *"¿Querés que aplique las correcciones? Empiezo por los críticos."*

---

## Fase 8 — Corrección (solo si el usuario la pide)

No la ejecutes por iniciativa propia. Cuando la pida:

1. **De a un hallazgo por vez**, empezando por 🔴 y 🟠. Los 🟡 solo si el usuario lo pide.
2. **Los fixes de base van en una migración nueva.** Antes de proponerla, explicá en una o dos oraciones qué cambia y qué podría dejar de funcionar: el usuario tiene que aprobar cada migración.
3. **Nunca habilites RLS sin crear las policies en la misma migración.** RLS sin policies bloquea todo y la app deja de funcionar.
4. **Después de cada fix, verificá** con la misma consulta de la Fase 1 que el problema desapareció, y pedile al usuario que pruebe el flujo afectado (login, la pantalla que usa esa tabla).
5. **Los secretos hay que rotarlos, y eso no lo podés hacer vos.** Sacá la clave del código y movela a un secreto del proyecto usado desde una Edge Function, pero decile al usuario, explícito y destacado, qué clave revocar y en qué panel.
6. **Al terminar, actualizá `AUDITORIA_SEGURIDAD.md`** marcando qué hallazgos quedaron resueltos y cuáles siguen pendientes de acción manual.

### Patrones de corrección

**Habilitar RLS con su policy:**

```sql
alter table public.<tabla> enable row level security;

create policy "<tabla>: cada usuario ve lo suyo"
  on public.<tabla> for select
  to authenticated
  using (auth.uid() = user_id);
```

**Roles en tabla aparte (el patrón canónico de Supabase):**

```sql
create type public.app_role as enum ('admin', 'moderator', 'user');

create table public.user_roles (
  id      uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  role    app_role not null,
  unique (user_id, role)
);

alter table public.user_roles enable row level security;

-- security definer: evita la recursión infinita de consultar user_roles
-- desde una policy de user_roles
create or replace function public.has_role(_user_id uuid, _role app_role)
returns boolean
language sql
stable
security definer
set search_path = public
as $$
  select exists (
    select 1 from public.user_roles
    where user_id = _user_id and role = _role
  );
$$;

-- Solo lectura de los roles propios. Nadie se asigna roles desde el cliente.
create policy "cada usuario ve sus roles"
  on public.user_roles for select
  to authenticated
  using (auth.uid() = user_id);

-- Uso en otras policies
create policy "admins ven todo"
  on public.<tabla> for select
  to authenticated
  using (public.has_role(auth.uid(), 'admin'));
```

Si el rol hoy vive en `profiles`, la migración tiene que: crear `user_roles`, copiar los valores existentes, apuntar las policies y el código a `has_role()`, y recién ahí borrar la columna vieja.

**Vista que respeta RLS:**

```sql
alter view public.<vista> set (security_invoker = true);
```

**Revocar la ejecución pública de una función:**

```sql
revoke execute on function public.<funcion>(<argumentos>) from public, anon;
-- Solo si la usan usuarios logueados por RPC:
grant execute on function public.<funcion>(<argumentos>) to authenticated;
```

**Edge Function con usuario identificado y lista fija de recursos (proxy):**

```ts
const ALLOWED_QUESTIONS = new Set([8342, 9158]);

const supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY, {
  global: { headers: { Authorization: req.headers.get("Authorization") ?? "" } },
});
const { data: { user } } = await supabase.auth.getUser();
if (!user) return new Response("Unauthorized", { status: 401 });

const { questionId } = await req.json();
const id = Number(questionId);
if (!Number.isInteger(id) || !ALLOWED_QUESTIONS.has(id)) {
  return new Response("Forbidden", { status: 403 });
}
// Filtrar por user.email, nunca por un email que mande el cliente
```

**Fijar `search_path` en una función:**

```sql
alter function public.<funcion>(<argumentos>) set search_path = public;
```

**Storage privado por carpeta de usuario:**

```sql
create policy "cada usuario ve sus archivos"
  on storage.objects for select
  to authenticated
  using (bucket_id = '<bucket>' and auth.uid()::text = (storage.foldername(name))[1]);
```
