# Supabase: los seis controles

En una app Lovable el frontend es público por definición. **El único control de acceso real está en Postgres.** Esta es la fase que decide si la auditoría vale algo.

Dónde mirar:

```
supabase/migrations/*.sql     ← tablas, RLS, policies, funciones, buckets
supabase/functions/<n>/       ← Edge Functions (código Deno)
supabase/config.toml          ← verify_jwt por función
src/integrations/supabase/    ← cliente generado por Lovable (y types.ts, útil para el inventario)
```

## Verificación en vivo (preferible, si hay acceso)

Si el usuario tiene un connector de Supabase, el MCP, o credenciales de `psql`, esto es la verdad — las migraciones son solo lo que *debería* haberse aplicado:

```sql
-- Tablas públicas SIN RLS
select c.relname
from pg_class c join pg_namespace n on n.oid = c.relnamespace
where n.nspname = 'public' and c.relkind = 'r' and c.relrowsecurity = false
order by 1;

-- Todas las policies, con su condición
select tablename, policyname, cmd, roles, qual, with_check
from pg_policies where schemaname = 'public' order by tablename;

-- Buckets públicos
select id, name, public from storage.buckets;

-- Funciones SECURITY DEFINER sin search_path fijo
select p.proname, p.proconfig
from pg_proc p join pg_namespace n on n.oid = p.pronamespace
where n.nspname = 'public' and p.prosecdef
  and (p.proconfig is null or not (p.proconfig::text like '%search_path%'));
```

Sin acceso a la base, trabajá sobre las migraciones con lo que sigue, y dejá constancia en *Qué no pude verificar* de que es análisis estático.

---

## Control 1 — Tablas sin RLS

```bash
rg -o -I -i -r '$3' 'create table (if not exists )?(public\.)?"?([a-z0-9_]+)"?' supabase/migrations/ | sort -u > /tmp/tablas.txt
rg -o -I -i -r '$3' 'alter table (only )?(public\.)?"?([a-z0-9_]+)"? enable row level security' supabase/migrations/ | sort -u > /tmp/con_rls.txt
comm -23 /tmp/tablas.txt /tmp/con_rls.txt
```

Lo que salga son tablas que **cualquiera con la anon key lee enteras** — y la anon key está en el bundle, o sea que la tiene cualquiera.

Antes de reportar, confirmá que la tabla no fue borrada en una migración posterior (`rg -i 'drop table' supabase/migrations/`) y mirá qué guarda: si tiene emails, teléfonos, documentos, notas o datos de pago es 🔴; si es una tabla de catálogo pensada para ser pública (países, categorías), es informativo y conviene decir que es intencional.

**Fix (migración nueva):**

```sql
alter table public.<tabla> enable row level security;

create policy "<tabla>: cada usuario ve lo suyo"
  on public.<tabla> for select
  to authenticated
  using (auth.uid() = user_id);
```

Habilitar RLS sin crear policies bloquea todo: la app va a empezar a fallar. Siempre van juntos.

---

## Control 2 — Policies abiertas

```bash
rg -n -i -B2 -A8 'create policy' supabase/migrations/
rg -n -i 'using\s*\(\s*true\s*\)|with check\s*\(\s*true\s*\)' supabase/migrations/
rg -n -i '\bto\s+(public|anon)\b' supabase/migrations/
```

> El `\b` inicial no es decorativo: sin él, `to\s+public` matchea `insert into public.user_roles` y el informe se llena de policies abiertas que no existen.

| Patrón | Significa |
|---|---|
| `for select ... using (true)` | Cualquiera lee la tabla entera. Equivale a no tener RLS. |
| `for all ... using (true)` | Cualquiera lee, escribe y **borra**. 🔴 |
| `for insert ... with check (true)` | Cualquiera inserta filas arbitrarias (spam, datos falsos). |
| `to public` / `to anon` | Alcanza a usuarios **no autenticados**. Casi siempre es un error salvo en catálogos. |
| sin cláusula `to` | Aplica a todos los roles, incluido `anon`. Tratalo como `to public`. |

Esto es peor que no tener RLS, porque el panel de Supabase muestra la tabla como protegida.

---

## Control 3 — Escalada de privilegios por columna de rol

El error más común en apps Lovable con roles.

```bash
rg -n -i 'role|is_admin|is_super|permission|nivel' supabase/migrations/*.sql
rg -n -i -A10 'create policy.*for update' supabase/migrations/
```

**El patrón peligroso:** la columna `role` (o `is_admin`) vive en `profiles` / `users`, y existe una policy tipo:

```sql
create policy "users can update own profile" on public.profiles
  for update using (auth.uid() = id);
```

Postgres no restringe columnas ahí. Cualquier usuario hace `update profiles set role = 'admin' where id = auth.uid()` desde la consola del navegador y se convierte en administrador. 🔴 **crítico**.

**Fix — el patrón canónico de Supabase:** roles en tabla aparte, sin policy de escritura, consultados por una función `security definer`.

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

Si el proyecto ya guarda el rol en `profiles`, la migración de arreglo tiene que: crear `user_roles`, copiar los valores existentes, apuntar las policies a `has_role()` y recién ahí borrar la columna vieja.

---

## Control 4 — Autorización basada en `user_metadata`

```bash
rg -n 'user_metadata' -g '*.ts' -g '*.tsx' src/ supabase/functions/ 2>/dev/null
rg -n -i "raw_user_meta_data|'user_metadata'" supabase/migrations/
```

`user_metadata` (`raw_user_meta_data`) **lo escribe el propio usuario** con `supabase.auth.updateUser({ data: { role: 'admin' } })`. No es un campo de confianza.

- `if (user.user_metadata.role === 'admin')` en el frontend → 🔴 escalada directa.
- `auth.jwt() -> 'user_metadata' ->> 'role' = 'admin'` dentro de una policy → 🔴, y peor, porque parece una verificación de servidor.

Lo que el usuario **no** puede modificar es `app_metadata`, que solo se escribe con la service_role. Aun así, el patrón recomendado es la tabla `user_roles` del Control 3.

Usarlo para cosas sin consecuencia (nombre para mostrar, avatar, idioma preferido) está bien y no es hallazgo.

---

## Control 5 — Edge Functions

```bash
ls supabase/functions/
rg -n -B3 'verify_jwt' supabase/config.toml
rg -n -i 'SERVICE_ROLE|createClient|Deno\.env\.get' supabase/functions/
rg -n -i 'authorization|auth\.getUser|Access-Control-Allow-Origin' supabase/functions/
```

En `config.toml`:

```toml
[functions.mi-funcion]
verify_jwt = false     # ← la función queda abierta a internet
```

Cómo evaluarlo:

| Situación | Veredicto |
|---|---|
| `verify_jwt = false` + cliente con `service_role` + sin validación interna | 🔴 Crítico. Cualquiera en internet escribe en la base con permisos totales. |
| `verify_jwt = false` en un webhook que valida firma (Stripe, Mercado Pago, WATI) | Correcto. Verificá que la validación de firma exista de verdad. |
| `verify_jwt = false` + la función hace `auth.getUser()` con el header Authorization | Aceptable. Documentalo. |
| `verify_jwt = true` (o ausente, que es el default) | Correcto. |
| `service_role` usada dentro de la función | Normal y correcto: corre en el servidor. **No es hallazgo.** |

Revisá además: si la función recibe un `user_id` por body y opera sobre él sin compararlo con el usuario del JWT, cualquier usuario autenticado opera sobre datos ajenos (🟠). Y `Access-Control-Allow-Origin: *` en una función que hace algo privilegiado facilita el abuso desde cualquier sitio (🟡).

---

## Control 6 — Storage y Realtime

```bash
rg -n -i 'storage\.buckets|storage\.objects' supabase/migrations/
rg -n -i 'supabase_realtime' supabase/migrations/
rg -n -i "from\(['\"]storage|\.storage\.from\(" -g '*.ts*' src/
```

**Storage.** Un bucket creado con `public = true` sirve sus archivos por URL directa, sin autenticación. Si guarda comprobantes, documentos de identidad, fotos de perfil privadas o adjuntos de usuarios: 🔴. Los buckets privados necesitan policies sobre `storage.objects`; si no hay ninguna, o subir falla, o está abierto:

```sql
create policy "cada usuario ve sus archivos"
  on storage.objects for select
  to authenticated
  using (bucket_id = 'documentos' and auth.uid()::text = (storage.foldername(name))[1]);
```

**Realtime.** Una tabla agregada a la publicación `supabase_realtime` empuja cambios a todo cliente suscripto. RLS también aplica a Realtime, así que el riesgo real aparece cuando la tabla ya venía sin RLS o con policy abierta — en ese caso subí la severidad, porque la filtración pasa a ser continua y en vivo, no solo bajo demanda.

---

## Control secundario — `SECURITY DEFINER` sin `search_path`

```bash
rg -n -i -A8 'security definer' supabase/migrations/
```

Una función `security definer` corre con los permisos de quien la creó. Sin `set search_path = public`, un usuario con permiso de crear objetos puede secuestrar la resolución de nombres. Es el `function_search_path_mutable` del linter de Supabase: 🟡, fix de una línea.

```sql
create or replace function public.mi_funcion(...)
returns ...
language plpgsql
security definer
set search_path = public   -- ← esta línea
as $$ ... $$;
```
