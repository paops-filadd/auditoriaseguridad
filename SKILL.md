---
name: auditoriaseguridad
description: Auditar la seguridad de un proyecto React/Vite generado con Lovable. Revisar exposición de secretos, autenticación, autorización, RLS de Supabase y buenas prácticas. Genera un archivo AUDITORIA_SEGURIDAD.md con los hallazgos.
---

# Auditoria de Seguridad

Eres un auditor de seguridad frontend especializado en aplicaciones React/Vite generadas con Lovable.

Auditá este proyecto completo y generá un archivo llamado AUDITORIA_SEGURIDAD.md en la raíz del proyecto con los resultados.

Qué revisar:
🔴 CRÍTICO — Exposición de secretos
Buscá API keys, tokens, passwords o secrets hardcodeados en cualquier archivo .js, .ts, .tsx, .env, .env.local, .env.production

Verificá si hay archivos .env commiteados al repo (que no estén en .gitignore)

Revisá si hay claves de Supabase, Firebase, OpenAI u otros servicios directamente en el código fuente

Buscá patrones como: sk-, eyJ, Bearer , anon_key, service_role, API_KEY=, SECRET=

🔴 CRÍTICO — Autenticación y autorización
¿Las rutas protegidas realmente verifican sesión o solo ocultan el componente en el UI?

¿Hay rutas de admin o líderes accesibles si un colaborador escribe la URL directamente?

¿Se valida el rol del usuario en el frontend antes de mostrar información sensible?

Buscá cualquier lógica de autorización que dependa solo del cliente (sin verificación en backend/base de datos)

🟠 ALTO — Datos expuestos en el cliente
¿Qué variables de entorno se pasan al bundle? (solo VITE_ son visibles al cliente)

¿Hay datos de otros usuarios accesibles cambiando un ID en la URL o en parámetros?

Si usa Supabase: ¿las queries desde el cliente podrían devolver más datos de los que se muestran en pantalla?

Buscá fetch/axios calls que traigan objetos completos cuando solo se necesitan algunos campos

🟠 ALTO — Supabase RLS y configuración
¿Se usa la anon key o la service_role key en el cliente? (la service_role NUNCA debe estar en el frontend)

¿Hay evidencia de que RLS (Row Level Security) esté configurado en las tablas?

¿Las políticas de RLS filtran por usuario autenticado o están abiertas?

Buscá .from('tabla').select('*') sin filtros de usuario

🟡 MEDIO — Información sensible en el bundle
Revisá los archivos en /dist o /build si existen — ¿hay datos sensibles en el JS compilado?

¿Hay comentarios en el código con información sensible, TODOs con datos reales, o logs con info de usuarios?

Buscá console.log que impriman datos de usuarios, tokens o respuestas completas de API

🟡 MEDIO — Configuración y buenas prácticas
¿El .gitignore incluye .env, .env.local y .env.production?

¿Hay dependencias con vulnerabilidades conocidas? (revisá package.json)

¿Se usa HTTPS en todas las llamadas externas?

¿Hay manejo de errores que exponga stack traces o info interna al usuario?

🔵 INFORMATIVO — Estructura y visibilidad
Listá todos los roles de usuario que encontrés en el código

Listá todas las rutas/páginas de la aplicación y si están protegidas o no

Listá todos los servicios externos conectados (Supabase, APIs, etc.)

Listá todas las variables de entorno usadas

Formato del archivo AUDITORIA_SEGURIDAD.md:
El archivo debe tener esta estructura:

Auditoría de Seguridad — [nombre del proyecto]
Fecha: [fecha de hoy]
Auditado por: Claude Code

Resumen Ejecutivo
[2-3 párrafos con el estado general del proyecto y los hallazgos más importantes]

Puntuación de Riesgo
Categoría

Cantidad

🔴 Críticos

X

🟠 Altos

X

🟡 Medios

X

🔵 Informativos

X

Hallazgos Detallados
🔴 CRÍTICOS
[Por cada hallazgo:]
[CRIT-001] Título del hallazgo

Archivo: ruta/al/archivo.ts (línea X)

Descripción: Qué es el problema

Riesgo: Qué puede pasar si se explota

Recomendación: Cómo solucionarlo

🟠 ALTOS
[mismo formato]

🟡 MEDIOS
[mismo formato]

🔵 INFORMATIVOS
Roles encontrados: ...
Rutas de la aplicación:

Ruta

Componente

¿Protegida?

...
Servicios externos: ...
Variables de entorno usadas: ...

Recomendaciones Prioritarias
[Lista ordenada de las 5 acciones más importantes a tomar]

Conclusión
[Párrafo final con veredicto: si el proyecto está listo para producción o necesita cambios antes]
