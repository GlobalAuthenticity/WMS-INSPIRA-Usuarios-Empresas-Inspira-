# Usuarios & Empresas (Inspira)

**Versión:** 1.38.0 · **Autor:** Aprobándome · **Text domain:** `gestionar-empresa` · **Capacidad requerida:** `manage_options`

Panel único de administración para la estructura multiempresa de Inspira. Sustituye el flujo manual de crear usuarios uno por uno en WordPress por un gestor de tres niveles — **empresa → área → usuario** — con control de cupos, fechas de expiración, importación masiva desde Excel y sincronización contra los sistemas externos de la red (WMS Latam, Octágono).

Se instala como menú de primer nivel **Usuarios & Empresas** (icono de edificio, posición 30) con un submenú de **Configuración**.

## Modelo de datos

| Elemento | Dónde vive | Notas |
|---|---|---|
| Empresa | Usuario WP con rol `empresa_admin` | El rol se crea en la activación, clonando las capacidades de `subscriber` |
| Área | Tabla propia `wp_areas` | `id`, `nombre`, `cupo`, `estado`, `tipo`, `grupo_id`, `fecha_expiracion` — InnoDB, utf8mb4 |
| Usuario final | Usuario WP con rol `subscriber` | Vinculado a su área mediante `wp_user_members` |

`grupo_id` apunta al ID del usuario-empresa, que es lo que convierte la jerarquía en consultable. La tabla `wp_areas` se crea en la activación y, además, se autorregenera en `admin_init` si no existe — así una actualización del plugin no exige reactivarlo.

## Funcionalidades

### Pestaña Empresas

CRUD completo de empresas: crear (con envío automático de correo de bienvenida con credenciales en plantilla HTML responsive), editar, cambiar contraseña y eliminar. El borrado muestra **vista previa del impacto** antes de confirmar: cuántas áreas y cuántos usuarios se verán afectados. Dentro de cada empresa se gestionan sus áreas con validación de nombre, cupos (`> 0`), formato de fecha `YYYY-MM-DD` y exigencia de que la expiración sea futura.

### Pestaña Usuarios

Padrón completo de suscriptores con búsqueda por nombre completo, usuario o correo. Cada fila muestra el número de cursos de LearnDash asignados, y el detalle se abre en un modal con el **porcentaje de avance por curso**.

La integración con LearnDash está optimizada deliberadamente: el conteo del listado se resuelve con **una sola consulta** para todos los usuarios (contando las claves `course_{id}_access_from`), porque preguntarle a la API de LearnDash usuario por usuario dispararía cientos de consultas en cada búsqueda. El detalle sí consulta la API, que añade los cursos heredados de un grupo; si encuentra más que el conteo, el front-end corrige el número de la fila. El avance se lee de `learndash_course_progress`, con reserva al metadato `_sfwd-course_progress` y, como último recurso, a la tabla de actividad.

El borrado de un usuario es permanente y en tres pasos: quita sus vínculos en `wp_user_members`, elimina el usuario y hace limpieza defensiva de usermeta huérfano.

### Importación masiva (CSV / XLSX)

Asistente por pasos que lee el archivo en el navegador con SheetJS. Antes de tocar la base de datos, **valida el bloque completo** contra la empresa y el área elegidas y separa los registros en *nuevos* y *omitidos*, indicando en cada omisión el motivo exacto. La creación se ejecuta fila por fila para mostrar progreso en tiempo real y para que un fallo puntual no detenga el resto del lote.

### Actualización masiva de correos

Mismo patrón de validación previa, con cuatro reglas por fila. Regla de negocio destacada: si un usuario tiene **el perfil completo** (todas las meta keys del conjunto diligenciadas), su correo queda bloqueado; solo se permite cambiarlo a quien le falte algún campo. Cada fila se revalida justo antes de ejecutarse, por si el estado cambió durante la sesión.

### Administración de `wp_usermeta`

Editor directo de la tabla de metadatos por usuario: listado paginado, búsqueda por `meta_key` / `meta_value` y CRUD completo por `umeta_id` — incluido renombrar la clave de un registro concreto. Consulta `$wpdb->usermeta` directamente en lugar de `get_user_meta()`, que cargaría todos los metadatos en memoria.

### Entrar como usuario / Volver al administrador

Suplantación de sesión con vuelta atrás, diseñada con varias capas:

1. El admin pulsa *Entrar como usuario*; el JS envía el `user_id` junto al estado del listado (pestaña, búsqueda, filtros, paginación).
2. El servidor genera un **token de un solo uso válido 60 segundos** y devuelve la URL de cambio.
3. Al consumirlo se abre la sesión del usuario destino y se emiten dos cosas que enlazan esa sesión con el administrador original: una **cookie firmada con HMAC** (`HttpOnly`) y un registro en servidor (transient).
4. Mientras ambos existan y sean válidos aparece el botón *Volver al administrador* (TTL máximo: 8 horas).
5. El retorno revalida todo de nuevo — nonce, cookie, registro en servidor y que el admin siga teniendo `manage_options` — restaura su sesión y lo devuelve al listado de origen **con sus filtros y página intactos**.

El estado se limpia en `wp_logout`.

### Sincronización externa

Consulta los endpoints de verificación de **Octágono** y **WMS Latam** por correo y refleja el resultado en los usermeta `octagono` y `wms`, escribiendo el rol devuelto, actualizándolo o eliminándolo según corresponda (un rol `external_user` en WMS se trata como ausencia de licencia). Cada operación reporta su desenlace como `created` / `updated` / `removed` / `unchanged` / `error`. Al crear cuentas puede además registrarlas contra la API de WMS Latam y notificar los webhooks de Inspira (`creado` / `existente`).

## Configuración

Ocho valores en **Usuarios & Empresas → Configuración**, guardados saneados en la opción `ge_integraciones`:

`wms_api_url` · `wms_api_token` · `webhook_creado` · `webhook_existente` · `octagono_verify_url` · `octagono_verify_token` · `wms_verify_url` · `wms_verify_token`

Ninguna credencial viaja en el código. Si prefieres mantenerlas fuera de la base de datos, cada una admite una constante equivalente en `wp-config.php` (`GE_WMS_API_TOKEN`, etc.) que actúa como reserva cuando el campo está vacío.

## Notas técnicas

**Seguridad.** Los 20+ endpoints AJAX validan sin excepción `check_ajax_referer('ge_nonce')` **y** `current_user_can('manage_options')`. Las consultas a tablas propias usan `$wpdb->prepare()`.

**Convivencia con otros plugins.** Al crear usuarios con `wp_insert_user()`, un hook ajeno de `user_register` (Ultimate Member, WooCommerce, LearnDash) puede hacer auto-login o `redirect + exit` y matar la ejecución a mitad de camino, antes de guardar usermeta o vincular el área. El plugin neutraliza temporalmente esos hooks y los restaura después; si alguno alcanzó a cambiar la cookie de sesión, `ge_restore_admin_session()` la revierte — `wp_set_current_user()` por sí solo no basta, porque solo corrige el contexto de la petición actual.

**Rendimiento.** `ge_get_users_map()` carga todos los usuarios de un listado con un único `get_users( include )`, evitando el patrón N+1 de llamar `get_userdata()` dentro de un bucle.

**Dependencias de front-end** (desde CDN, solo en las pantallas del plugin): SweetAlert2 11, SheetJS 0.18.5, y las tipografías Inter + Plus Jakarta Sans para alinearse con la identidad de wmslatam.com. Los avisos de otros plugins se ocultan dentro de estas pantallas.

**Estructura:** 16 archivos, organizados por responsabilidad en `includes/` (`bootstrap`, `admin-page`, `settings-page`, `switch-user`, y un módulo AJAX por dominio).

