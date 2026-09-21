# Plantilla de AUDITORIA_SEGURIDAD.md

Escribí el archivo en la raíz del proyecto siguiendo esta estructura. El destinatario suele ser quien armó la app en Lovable, no una persona de seguridad: escribí en criollo y explicá el riesgo en términos de qué le puede pasar a la app, no de categorías abstractas.

**Convención de IDs:** `CRIT-001`, `ALTO-001`, `MEDIO-001`, correlativos dentro de cada severidad. En una re-auditoría, un hallazgo que sigue abierto **conserva su ID original** aunque haya cambiado de archivo o de línea. Los IDs nuevos siguen la numeración desde el último usado. Nunca reciclás un ID de un hallazgo resuelto.

---

```markdown
# Auditoría de Seguridad — <nombre del proyecto>

| | |
|---|---|
| **Repositorio** | `<owner/repo>` · <público / privado> |
| **Commit auditado** | `<sha corto>` (<fecha del commit>) |
| **Fecha de la auditoría** | <YYYY-MM-DD> |
| **Alcance** | Revisión estática del repositorio: código, historial de git, migraciones de Supabase y dependencias. No incluye pruebas contra el entorno productivo. |
| **Auditado por** | Claude Code — skill `auditoriaseguridad` |

## Resumen ejecutivo

<Dos o tres párrafos. Qué es la app, qué maneja, en qué estado está. El hallazgo
más grave primero y en una oración entendible por cualquiera. Si el repo es
público y hay secretos, decilo acá. Cerrá con el veredicto en una línea.>

## Puntuación de riesgo

| Severidad | Cantidad |
|---|---|
| 🔴 Críticos | X |
| 🟠 Altos | X |
| 🟡 Medios | X |
| 🔵 Informativos | X |

## ⚡ Quick wins

Fixes de menos de cinco minutos, con la acción exacta:

- [ ] `<archivo>` — <qué hacer, literal>
- [ ] ...

## Hallazgos

### 🔴 Críticos

**[CRIT-001] <título en una línea>**

- **Dónde:** `<ruta/archivo.ts>:<línea>` — o `supabase/migrations/<archivo>.sql`
- **Qué pasa:** <descripción del problema, dos o tres oraciones>
- **Riesgo concreto:** <qué puede hacer un atacante, en términos de esta app: "cualquier visitante puede descargar la tabla de alumnos con nombre, email y teléfono">
- **Cómo lo verifiqué:** <comando, consulta o archivo que lo demuestra>
- **Cómo se arregla:** <pasos concretos; si es SQL, el bloque de la migración>

### 🟠 Altos
<mismo formato>

### 🟡 Medios
<mismo formato>

## Qué no pude verificar

<Obligatorio, aunque esté vacío — en ese caso poné "Nada: el repositorio
incluía todo lo necesario".>

| Área | Por qué | Qué haría falta |
|---|---|---|
| Policies de RLS aplicadas en la base | El repo no tiene `supabase/migrations/` | Acceso al panel de Supabase o al connector |
| ... | | |

## 🔵 Inventario

**Roles de usuario:** <lista>

**Rutas de la aplicación**

| Ruta | Componente | ¿Protegida en el frontend? | ¿Verificada en el backend / RLS? |
|---|---|---|---|
| | | | |

**Servicios externos:** <lista>

**Variables de entorno**

| Variable | ¿Llega al navegador? | Observación |
|---|---|---|
| `VITE_...` | Sí | Pública por diseño |
| `...` | No (Edge Function) | |

## Acciones prioritarias

1. <la más importante, con el ID del hallazgo>
2. ...
5. ...

## Comparación con la auditoría anterior

<Solo si existía un AUDITORIA_SEGURIDAD.md previo. Si no, borrá la sección.>

| Estado | Hallazgos |
|---|---|
| ✅ Resueltos | CRIT-002, MEDIO-001 |
| ⏳ Siguen abiertos | CRIT-001 (desde <fecha>) |
| 🆕 Nuevos | ALTO-003 |

## Conclusión

<Un párrafo. ¿Se puede publicar o compartir esta app como está? Si no, qué
tiene que pasar primero — nombrando los IDs. Sé directo: "no" es una respuesta
válida y útil.>
```

---

## Después de escribir el informe

Cerrá tu respuesta al usuario con:

1. Los tres hallazgos más graves, en una línea cada uno.
2. Lo que **solo puede hacer él**: rotar claves filtradas, cambiar configuración en el panel de Supabase, hacer privado el repo.
3. La oferta de corrección: *"¿Querés que aplique las correcciones en una branch y te abra el PR?"*

No apliques cambios sin que lo pida.
