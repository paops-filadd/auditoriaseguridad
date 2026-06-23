# 🔐 Auditoria de Seguridad — Skill para Lovable

Skill personalizado para auditar la seguridad de proyectos React/Vite generados con Lovable. Revisa exposición de secretos, autenticación, autorización, configuración de Supabase RLS y buenas prácticas. Genera un archivo `AUDITORIA_SEGURIDAD.md` con todos los hallazgos clasificados por severidad.

---

## 📦 Instalación

### 1. Ingresar a Lovable Settings

Desde cualquier workspace de Lovable, hacé clic en el ícono de configuración (Settings).

### 2. Navegar a Customization → Skills

En el menú lateral izquierdo:

```
Settings → Customization → Skills
```

### 3. Importar desde GitHub

En la sección **Workspace Skills**:

1. Hacé clic en **Add**
2. Seleccioná **Import from GitHub**
3. Pegá la URL del repositorio:

```
https://github.com/paops-filadd/auditoriaseguridad.git
```

4. Hacé clic en **Import**

---

## 🚀 Cómo usar el skill

Una vez instalado, el skill está disponible en **todos los proyectos** del workspace.

Para activarlo, dentro de cualquier proyecto escribí:

```
/
```

Va a aparecer la lista de skills disponibles. Seleccioná **`auditoriaseguridad`** y el agente comenzará la auditoría automáticamente.

---

## 📋 Qué audita

| Severidad | Categoría |
|-----------|-----------|
| 🔴 Crítico | Exposición de API keys y secrets en el código |
| 🔴 Crítico | Autenticación y autorización por rol |
| 🟠 Alto | Datos expuestos en el cliente |
| 🟠 Alto | Configuración de Supabase RLS |
| 🟡 Medio | Información sensible en el bundle |
| 🟡 Medio | Buenas prácticas y configuración |
| 🔵 Informativo | Roles, rutas y servicios externos |

---

## 📄 Resultado

El skill genera un archivo **`AUDITORIA_SEGURIDAD.md`** en la raíz del proyecto con:

- Resumen ejecutivo
- Puntuación de riesgo
- Hallazgos detallados con archivo, línea, descripción y recomendación
- Las 5 acciones prioritarias
- Veredicto final sobre si el proyecto está listo para producción
