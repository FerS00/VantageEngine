# 03 · Brief de diseño de interfaz (para la fase de dirección de diseño web)

> Este documento es la entrada para diseñar la interfaz de VantageEngine. Define **qué** hay que diseñar y **qué restricciones técnicas** debe respetar el diseño para que funcione con identidad, tema y módulos dinámicos. **No** define la estética: esa es la decisión de la fase de diseño.
>
> Contexto técnico: [`02-PLAN-TECNICO.md` §3–4](02-PLAN-TECNICO.md#3-theming-y-contenido-dinámico-en-angular).

---

## 1. Qué es el producto (en una frase para el diseñador)

Un motor de sitios comerciales **white-label**: una misma interfaz que cualquier negocio re-marca (logo, colores, tipografías, textos) y a la que activa o desactiva módulos (catálogo, carrito, pagos, reservas, contacto…) desde un panel. **El diseño tiene que verse bien con cualquier marca**, no solo con la de demostración.

## 2. Audiencias

| Audiencia | Superficie | Qué necesita |
|---|---|---|
| Visitante / cliente final | **Storefront** (público) | Encontrar, entender y comprar/reservar/contactar rápido, desde móvil |
| Dueño del negocio / admin | **Studio + consola** | Re-marcar el sitio sin miedo a romperlo; operar pedidos, reservas y clientes |
| Reclutador / revisor técnico | Ambas + repo | Ver en 2 minutos un producto pulido y un cambio de marca en vivo |

## 3. Reglas de diseño impuestas por la arquitectura (no negociables)

1. **Diseñar con roles, no con colores.** Todo color del diseño debe mapearse a un token: `primary`, `on-primary`, `secondary`, `accent`, `surface-0/1/2`, `border`, `text`, `text-muted`, `success`, `warning`, `danger`. Nada de "rojo del botón".
2. **Solo 4 colores semilla los elige el cliente** (`primary`, `secondary`, `accent`, `neutral`) + esquema claro/oscuro. El resto se deriva. El diseño debe probarse con **al menos 3 paletas contrastantes** (p. ej. rojo industrial oscuro, azul corporativo claro, verde/terracota cálido).
3. **Tipografía intercambiable:** una familia *display* y una *body*, elegidas de una lista curada (Google Fonts). Los layouts no pueden depender de las métricas de una fuente concreta (ancho de condensadas, etc.).
4. **Radio y densidad como variables:** `radius-scale` (NONE / SM / MD / LG / FULL) y `density` (COMPACT / COMFORTABLE). Los componentes deben verse bien en los extremos.
5. **Logos de cualquier forma:** horizontal, cuadrado, solo símbolo; claro y oscuro. Header con zona segura para logo de 32–48 px de alto y ancho variable.
6. **Textos de longitud variable** (vienen de BD y en varios idiomas): títulos del hero de 3 a 12 palabras, botones de 1 a 4 palabras, español ~20 % más largo que inglés. Nada de diseño que se rompa con un texto largo.
7. **Todo módulo puede no existir.** Cada pantalla debe tener su versión con y sin: carrito (→ botón "Solicitar cotización"), precios (→ "Consultar precio"), cuentas de cliente (→ sin icono de usuario), buscador, reservas, redes sociales, newsletter.
8. **La home se arma con secciones** reordenables (ver §5). Cada sección debe verse bien en cualquier posición y junto a cualquier otra.
9. **Accesibilidad AA**: contraste validado por el sistema, foco visible diseñado (no el del navegador), objetivos táctiles ≥ 44 px, `prefers-reduced-motion` respetado, sin información solo por color.
10. **Mobile-first**: breakpoints sugeridos 360 · 768 · 1024 · 1280 · 1536. El 70 % del tráfico de la web anterior se espera móvil.

## 4. Inventario de pantallas

### 4.1 Storefront (público)

| # | Pantalla | Módulo | Contenido y estados a diseñar |
|---|---|---|---|
| S1 | **Home** | core | Composición de secciones (§5); variante mínima (solo hero + contacto) y variante completa |
| S2 | **Header / navegación** | core | Logo, menú (1–7 ítems, con submenú), buscador (si `catalog`), cuenta (si `customer-accounts`), carrito con contador (si `cart`), selector de idioma (si `i18n`); menú móvil tipo *drawer*; barra de anuncio opcional |
| S3 | **Footer** | core | Logo, datos de contacto, menú footer, redes, legales, newsletter compacta opcional |
| S4 | **Rail social / botón WhatsApp** | `social-links` | Flotante; no tapar CTA ni el widget del asistente |
| S5 | **Listado de productos** | `catalog` | Grid responsive, filtros (categoría, marca, disponibilidad, rango de precio), orden, búsqueda, chips de filtros activos, paginación o "cargar más", estados vacío / cargando (skeleton) / error |
| S6 | **Tarjeta de producto** | `catalog` | Imagen con cambio al hover (galería de hasta 4), marca, nombre, precio, precio tachado + badge de oferta, estado (agotado / nuevo / actualizado), CTA contextual (añadir / cotizar) |
| S7 | **Detalle de producto** | `catalog` | Galería con miniaturas, video embebido, precio + oferta + vigencia, especificaciones (atributos), descripción rica, CTA, relacionados |
| S8 | **Listado y detalle de paquetes** | `bundles` | Imagen de cabecera, productos incluidos, ahorro frente a comprarlos por separado |
| S9 | **Búsqueda global** | `catalog` | Overlay con resultados instantáneos, teclado, sin resultados |
| S10 | **Carrito** | `cart` | Drawer lateral + página; líneas mixtas (producto/paquete), cantidades, cupón (si `offers`), resumen con desglose lista/descuento/final, vacío |
| S11 | **Checkout** | `checkout` | Modo **solicitud de cotización** (datos de contacto + canal preferido WhatsApp/Telegram/email) y modo **pago online** (si `payments`); invitado vs. cliente autenticado (solo pide campos faltantes); éxito con número de pedido; errores |
| S12 | **Pago** | `payments` | Selección de proveedor, redirección/embebido, éxito, cancelado, pendiente |
| S13 | **Reservas** | `booking` | Elegir servicio → profesional (opcional) → día (calendario) → hora (slots) → datos → confirmación; gestionar/cancelar desde enlace; sin disponibilidad |
| S14 | **Contacto** | `contact-form` | Formulario (nombre, email, canal, destino, mensaje), CAPTCHA, éxito, errores por campo, datos de contacto y mapa opcional |
| S15 | **Auth de cliente** | `customer-accounts` | Login (contraseña → código OTP de 6 dígitos), registro (verificar email → crear contraseña), recuperar acceso; un solo componente por etapas |
| S16 | **Mi cuenta** | `customer-accounts` | Perfil, historial de pedidos (con estados), mis reservas, preferencias de marketing |
| S17 | **Páginas CMS** | core | FAQs (acordeón), legales (texto largo legible), "Nosotros" |
| S18 | **Asistente flotante** | `assistant` | Burbuja, panel de chat, respuestas con tarjetas de producto y fuentes citadas, sugerencias rápidas, handoff a humano, estado "escribiendo", error/desconectado |
| S19 | **Newsletter** | `newsletter` | Sección con consentimiento explícito, éxito, ya suscrito |
| S20 | **404 / módulo no disponible / mantenimiento** | core | Con la marca del tenant |

### 4.2 Studio de marca (admin) — la pantalla "estrella" del portafolio

| # | Pantalla | Estados y detalles |
|---|---|---|
| T1 | **Identidad** | Nombre, eslogan, logo claro/oscuro, favicon, imagen OG, datos de contacto, redes; previsualización de header/footer/pestaña del navegador |
| T2 | **Editor de tema** | Split view: controles a la izquierda (4 colores semilla con *picker* + presets, esquema claro/oscuro, fuentes, radio, densidad) y **preview en vivo** del storefront a la derecha (toggle desktop/móvil); indicador de contraste AA con sugerencia de corrección; guardar borrador / publicar / revertir a versión anterior |
| T3 | **Editor de contenido** | Árbol por vista/namespace, búsqueda por clave o texto, edición en línea por idioma, indicador de traducciones faltantes |
| T4 | **Constructor de páginas** | Lista de secciones arrastrables, añadir sección desde galería de tipos, panel de propiedades de la sección, visibilidad, preview |
| T5 | **SEO** | Título/descr. por página con contador de caracteres y previsualización de resultado de Google y de tarjeta social |
| T6 | **Módulos** | Tarjetas por módulo (icono, descripción, estado on/off, dependencias, "configurar"); diálogo al desactivar con módulos dependientes afectados |
| T7 | **Biblioteca de medios** | Grid, subir (drag&drop), recorte, texto alternativo, uso del recurso |

### 4.3 Consola de operación (admin)

| # | Pantalla | Notas |
|---|---|---|
| C1 | **Login admin + MFA** | Email/contraseña → código TOTP; recuperación |
| C2 | **Shell** | Sidebar colapsable construida por módulos activos y permisos; topbar con buscador, notificaciones (badge), usuario; breadcrumb |
| C3 | **Dashboard** | KPIs con variación, selector de periodo (7/30/90/personalizado), gráfico de tendencia, ranking de productos/clientes, salud de integraciones; tabla accesible equivalente al gráfico |
| C4 | **Patrón de listado** (reutilizable para productos, marcas, categorías, paquetes, ofertas, pedidos, clientes, reservas, mensajes, campañas, usuarios) | Tabla con filtros, búsqueda, orden, paginación, selección múltiple, acciones en lote, exportar, estados vacío/cargando/error; en móvil → tarjetas |
| C5 | **Patrón de formulario** | Secciones, validación inline, guardado con confirmación, cambios sin guardar, campos bloqueados por sincronización (badge de "propiedad de fuente externa") |
| C6 | **Detalle de pedido** | Línea de tiempo de estados, desglose de precios inmutable, datos de contacto, pagos asociados |
| C7 | **Agenda de reservas** | Vista día/semana por recurso, crear/mover/cancelar, bloqueos |
| C8 | **Ofertas** | Tipo, descuento, vigencia (zona local), destinos con selector paginado y búsqueda |
| C9 | **Notificaciones** | Bandeja con filtros (no leído, tipo, severidad), deep link, reintentar entrega fallida |
| C10 | **Campañas** | Compositor por bloques con la marca aplicada, preview desktop/móvil, audiencia, prueba, envío |
| C11 | **Sincronización de catálogo** | Fuente, preview paginado con NEW/UPDATE/UNCHANGED/ERROR, mappings pendientes, doble confirmación |
| C12 | **Asistente** | Conversaciones, conocimiento (versiones), métricas, configuración |
| C13 | **Usuarios, roles y auditoría** | Matriz rol × permiso agrupada por módulo; log filtrable |

## 5. Catálogo de secciones de página (page builder)

Cada sección recibe *props* desde BD. Diseñar cada una con variantes y con contenido mínimo/máximo.

| Tipo | Props principales | Variantes sugeridas |
|---|---|---|
| `HERO` | título, subtítulo, CTA primario/secundario, imagen o video de fondo, alineación | imagen completa, split imagen/texto, slider (2–5 slides) |
| `ANNOUNCEMENT_BAR` | texto, enlace, cerrable | — |
| `FEATURE_GRID` | título, 3–6 ítems (icono, título, texto) | 3 columnas, lista con iconos |
| `PRODUCT_CAROUSEL` | título, fuente (destacados/últimos/categoría/manual) | carrusel, grid 4 |
| `BRAND_MARQUEE` | título, marcas | marquee animado, grid estático (reduced-motion) |
| `CATEGORY_GRID` | categorías con imagen | — |
| `CTA_BANNER` | título, texto, CTA, fondo | color primario, imagen |
| `TESTIMONIALS` | citas con autor | carrusel, grid |
| `FAQ_LIST` | preguntas/respuestas | acordeón |
| `RICH_TEXT` | contenido Markdown sanitizado | ancho de lectura |
| `STATS` | 2–4 cifras con etiqueta | — |
| `NEWSLETTER` | título, texto, consentimiento | inline, tarjeta |
| `CONTACT_CTA` | canales (WhatsApp/Telegram/email/teléfono) | botones, tarjeta |
| `BOOKING_CTA` | servicio destacado | — |

## 6. Sistema de componentes base (a diseñar como librería)

Botón (primario, secundario, fantasma, peligro, solo icono; tamaños; cargando; deshabilitado) · Input, textarea, select, combobox con búsqueda, checkbox, radio, switch, date/time picker, input OTP de 6 dígitos, color picker · Tarjeta · Badge/Chip · Tabs · Acordeón · Diálogo modal · Drawer · Toast · Tooltip · Skeleton · Estado vacío (con ilustración neutral que tome el color primario) · Paginación · Tabla · Breadcrumb · Avatar · Stepper (checkout y reservas) · Calendario/slots · Galería de medios · Precio (normal, tachado, descuento) · Iconografía (set abierto tipo Lucide/Phosphor, trazo consistente).

## 7. Tono y dirección de partida (orientativa, no vinculante)

- **Storefront:** limpio, orientado a producto, con mucho aire y jerarquía tipográfica fuerte; la personalidad la pone la marca del tenant.
- **Studio/consola:** herramienta profesional, neutra, densa pero legible; la marca del tenant **no** tiñe el admin (solo el preview), para que el panel siempre sea usable aunque el cliente elija colores extremos.
- **Tenant de demostración:** reinterpretación de Diesel Power Pro (industrial, oscuro, rojo) **o** una marca ficticia neutra — pendiente de decisión (ver plan §8). Debe existir un **segundo tenant** con estética opuesta para demostrar el white-label.

## 8. Entregables esperados de la fase de diseño

1. Tokens de diseño definitivos (nombres alineados con `styles/tokens.css` del plan) y reglas de derivación.
2. Librería de componentes base con estados.
3. Pantallas clave en móvil y desktop: S1, S2, S5, S6, S7, S10, S11, S13, S15, T2, C2, C3, C4.
4. Las mismas S1/S5/S7 con **3 temas distintos** para validar el white-label.
5. Especificación de animaciones (duraciones, *easing*, alternativa reduced-motion).
