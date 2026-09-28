# 01 · Inventario funcional heredado de VENTAS_DPP

> Fuente analizada: `FerS00/VENTAS_DPP` (rama `main`, commit `4bac199`) — Angular 20 + Spring Boot 3.4 / Java 17, 38 migraciones Flyway, ~40 entidades JPA, 13 módulos administrativos y ~20 vistas públicas.
>
> Objetivo de este documento: **no perder ninguna capacidad** del sistema anterior al rediseñarlo, y decidir para cada una en qué módulo de VantageEngine vive, bajo qué *feature flag* y en qué fase se implementa. El proyecto anterior es **referencia funcional**, no de código: nada se copia, todo se reimplementa con la nueva arquitectura.

---

## 1. Resumen: qué hacía el sistema anterior

VENTAS_DPP era una tienda de software de diagnóstico (Diesel Power Pro) con **solicitud de compra asistida** (sin pago online), paquetes de programas, ofertas, cuentas de cliente con OTP, un área administrativa con RBAC fina, notificaciones Telegram al vendedor, campañas de correo con consentimiento, sincronización del catálogo desde Google Sheets y un asistente conversacional (guiado + IA opcional).

Sus limitaciones para convertirse en un producto reutilizable (y lo que motiva VantageEngine):

| Limitación en VENTAS_DPP | Consecuencia | Respuesta en VantageEngine |
|---|---|---|
| Marca, textos y navegación *hardcodeados* (`site.config.ts`, `info-pages.config.ts`, títulos de ruta "… \| Diesel Power Pro") | Cambiar de cliente = tocar código y redeploy | **Identidad y contenido dinámicos** desde BD (módulo `platform`) |
| Paleta fija en `styles.css` (`--red`, `--black`…) con nombres semánticos de color, no de rol | No se puede re-tematizar | **Design tokens por rol** + `ThemeService` en tiempo real |
| Funciones activadas por variables de entorno (`CHATBOT_ENABLED`, `OFFERS_ENABLED`, `CAMPAIGNS_ENABLED`…) | Requiere reinicio; no hay UI; no hay dependencias entre funciones | **Feature flags por tenant** en BD, con grafo de dependencias, caché e invalidación |
| Paquetes por capa técnica mezclados con dominio (`admin/*`, `domain/entity` global, un único `SecurityConfig`) | Acoplamiento; imposible extraer un módulo | **Monolito modular** (Spring Modulith) con arquitectura hexagonal por módulo |
| Idioma mezclado (tablas `orden`, `cliente`, `detalle_orden` + `notification_event`, `program_package`) | Esquema difícil de leer para un revisor | Esquema **100 % en inglés**, `snake_case`, convenciones documentadas |
| Dominio específico ("programas", "Telegram del vendedor") | No sirve para otro negocio | Dominio **genérico** (productos, bundles, canales de notificación intercambiables) |
| 40+ fases con documentos de verificación sueltos | Difícil de auditar | ADRs + roadmap corto + CI con gates automáticos |

---

## 2. Mapa de capacidades → módulos y feature flags

Leyenda de fase: ver [`02-PLAN-TECNICO.md` §5](02-PLAN-TECNICO.md#5-hoja-de-ruta-de-implementación). **Core** = siempre activo (no se puede desactivar).

### 2.1 Vistas públicas (storefront)

| # | Capacidad en VENTAS_DPP | Ruta anterior | Módulo nuevo | Feature flag | Fase |
|---|---|---|---|---|---|
| P1 | Home: hero/slider, destacados, últimos, marquee de marcas, bloques promocionales, anuncio superior | `/` | `content` (page builder por secciones) + `catalog` | Core (`home`) | F4 / F5 |
| P2 | Búsqueda global con resultados que abren detalle | Home | `catalog` | `catalog` | F5 |
| P3 | Colección con filtros (categoría, disponibilidad, rango de precio) y ordenamiento | `/products` | `catalog` | `catalog` | F5 |
| P4 | Detalle de producto: galería 4 imágenes con hover-rotate, video YouTube, especificaciones, relacionados | `/products/:id` | `catalog` + `media` | `catalog` | F5 |
| P5 | Paquetes (bundles) y su detalle, precio y programas incluidos | `/packages`, `/packages/:id` | `catalog.bundles` | `bundles` (requiere `catalog`) | F6 |
| P6 | Carrito local mixto (productos + paquetes) | panel en Home, `/checkout` | `cart` | `cart` (requiere `catalog`) | F7 |
| P7 | Checkout como **solicitud de compra** (anónimo con formulario completo / autenticado reutilizando perfil y pidiendo solo campos faltantes) | `/checkout` | `ordering` | `checkout` (requiere `cart`) — estrategia `QUOTE_REQUEST` | F7 |
| P8 | Código de oferta en carrito + preview del precio | `/checkout` | `pricing` | `offers` | F6 |
| P9 | Login cliente: email+contraseña → Turnstile → OTP por email | `/login` | `iam` | `customer-accounts` | F7 |
| P10 | Registro con verificación de email previa | `/register` | `iam` | `customer-accounts` | F7 |
| P11 | Recuperación de contraseña con desafío de un solo uso | `/forgot-password` | `iam` | `customer-accounts` | F7 |
| P12 | Mi cuenta: perfil, historial de solicitudes, consentimiento de marketing | `/account` | `iam` + `ordering` + `marketing` | `customer-accounts` | F7 |
| P13 | Contacto: nombre, email, canal preferido (WhatsApp/Telegram) + destino, Turnstile, idempotencia, rate limit | `/contact` | `engagement.contact` | `contact-form` | F8 |
| P14 | Newsletter "Workshop notes" con consentimiento afirmativo y lead | sección Home | `marketing.newsletter` | `newsletter` | F8 |
| P15 | FAQs y Política de privacidad | `/faqs`, `/privacy-policy` | `content` (páginas CMS) | Core (`legal-pages`) / `faq` | F4 |
| P16 | Rail social fijo (Facebook, Telegram, Instagram, TikTok, Teams, WhatsApp) | Home | `platform.identity` (redes sociales) | `social-links` | F4 |
| P17 | Asistente flotante (guiado determinista + IA opcional grounded, fuentes verificables, handoff humano) | global | `assistant` | `assistant` (+ `assistant-ai`) | F14 |
| P18 | Página de fuente verificada del conocimiento | `/knowledge/:id` | `assistant.knowledge` | `assistant` | F14 |
| P19 | Baja one-click de marketing | `/api/public/marketing/unsubscribe` | `marketing` | `newsletter` / `campaigns` | F12 |
| P20 | Redirecciones legacy (`/pages/main-faqs`, `/blogs/news`) | — | `content.redirects` (tabla de redirecciones) | Core | F4 |

### 2.2 Consola administrativa

| # | Capacidad en VENTAS_DPP | Permisos anteriores | Módulo nuevo | Feature flag | Fase |
|---|---|---|---|---|---|
| A1 | Login administrativo separado, cookie HttpOnly | — | `iam` | Core | F1 |
| A2 | Dashboard: KPIs por periodo 7/30/90/custom, pipeline (no ventas), clientes únicos/recurrentes, ranking de demanda, impacto de ofertas, actividad estimada; Chart.js + tabla accesible | `DASHBOARD_READ` | `analytics` | Core (`dashboard`) | F9 |
| A3 | Exportar dashboard a PDF vectorial multipágina | `DASHBOARD_EXPORT` | `reporting` | `exports` | F9 |
| A4 | Clientes: listado paginado, filtros, CRUD, baja lógica, export XLSX | `CLIENT_READ/WRITE/EXPORT` | `customers` | Core | F7 / F9 |
| A5 | Productos: CRUD, baja lógica, 4 imágenes por slot, video, ownership de campos sincronizados, export XLSX | `PRODUCT_*` | `catalog` + `media` | `catalog` | F5 |
| A6 | Marcas: CRUD + imagen única + export XLSX | `BRAND_WRITE` | `catalog` | `catalog` | F5 |
| A7 | Categorías: CRUD + export XLSX | `CATEGORY_WRITE` | `catalog` | `catalog` | F5 |
| A8 | Paquetes: CRUD, imagen de encabezado, multimedia | `PACKAGE_*` | `catalog.bundles` | `bundles` | F6 |
| A9 | Ofertas: automáticas o por código, % o monto fijo, prioridad, vigencia UTC, destinos (productos/paquetes) con selectores paginados; una oferta ganadora sin apilar | `OFFER_READ/WRITE` | `pricing` | `offers` | F6 |
| A10 | Solicitudes/pedidos: desglose inmutable (lista, oferta, descuento, final), estados | `ORDER_READ/WRITE` | `ordering` | `checkout` | F7 |
| A11 | Bandeja de notificaciones: no leídas, tipo, severidad, deep link, reintento de entregas | `NOTIFICATION_READ/MANAGE` | `notifications` | Core | F8 |
| A12 | Outbox durable + Telegram al vendedor (backoff, jitter, claim atómico) y **comandos Telegram** para cambiar estado de pedidos | — | `notifications` (canal `telegram`) | `notify-telegram` | F8 |
| A13 | Campañas de correo: bloques seguros v2, tema, assets PNG/JPEG/GIF, audiencia congelada, outbox por destinatario, allowlist de prueba, Resend + webhooks firmados (Svix), supresiones | `CAMPAIGN_READ/WRITE/SEND` | `marketing.campaigns` | `campaigns` (requiere `newsletter`) | F12 |
| A14 | Sincronización de catálogo desde Google Sheets: preview paginado, mappings, bindings, APPLY atómico con doble confirmación, scheduler opt-in con lock distribuido | `PRODUCT_SYNC_*` | `integrations.catalogsync` | `catalog-sync` (requiere `catalog`) | F13 |
| A15 | Usuarios administrativos y matriz rol-permiso | `USER_MANAGE` | `iam` | Core | F1 / F4 |
| A16 | Chatbot admin: configuración, conocimiento versionado con rollback, prompts, feedback, analítica, registro de modelos | `CHATBOT_*` | `assistant.admin` | `assistant` | F14 |
| A17 | Registro de actividad autenticada | — | `platform.audit` | Core | F1 |

### 2.3 Capacidades transversales (no visibles pero críticas)

| Capacidad | Nuevo lugar |
|---|---|
| CSRF + JWT en cookie `HttpOnly`, `SameSite`, `Secure` | `iam` + `SecurityConfig` modular |
| Turnstile (CAPTCHA) por acción | `platform.antiabuse` → `CaptchaVerifier` (Strategy: Turnstile / hCaptcha / NoOp) |
| Rate limiting (login, OTP, contacto, newsletter) | `platform.antiabuse` → Bucket4j |
| Idempotency-Key en pedidos, contacto y chat | `shared.idempotency` (filtro + tabla `idempotency_key`) |
| Sanitización de entrada (jsoup) y neutralización de fórmulas en XLSX | `shared.sanitization`, `reporting` |
| Almacenamiento de medios en disco con URL pública `/media` | `media` → `StorageProvider` (Strategy: Local / S3-compatible) |
| Snapshots de precio en pedidos | `ordering` (value object `PriceSnapshot`) |
| Borrado/anonimización de datos de chat y clientes | `platform.privacy` |

---

## 3. Funciones **nuevas** propuestas (no existían en VENTAS_DPP)

Marcadas como *opcionales*; cada una es un feature flag independiente. Las dos primeras las pediste explícitamente.

| Nueva capacidad | Módulo | Flag | Prioridad | Fase |
|---|---|---|---|---|
| **Pasarela de pago online** (Stripe, Mercado Pago, PayPal) vía `PaymentGateway` Strategy + webhooks idempotentes; convive con "solicitud de compra" | `payments` | `payments` (requiere `checkout`) | Alta | F10 |
| **Citas / reservas**: servicios, recursos/personal, horarios, excepciones, slots, confirmación y recordatorios | `booking` | `booking` | Alta | F11 |
| **Studio de marca**: editor de identidad, tema con *preview* en vivo, contenido por vista, toggles de módulos | `platform` (admin) | Core | Alta | F4 |
| **Multi-idioma** del contenido (es/en/…) con *fallback* | `content` | `i18n` | Media | F4 |
| **SEO dinámico**: metadatos por ruta, Open Graph, `sitemap.xml` y `robots.txt` generados | `content.seo` | Core | Media | F4 |
| **Inventario/stock** opcional por producto (para bienes físicos) | `catalog.inventory` | `inventory` | Media | F15+ |
| **Reseñas y valoraciones** con moderación | `engagement.reviews` | `reviews` | Baja | backlog |
| **Lista de deseos** | `engagement.wishlist` | `wishlist` | Baja | backlog |
| **Blog / noticias** (reutiliza páginas CMS) | `content.blog` | `blog` | Baja | backlog |
| **Botón WhatsApp** flotante y enlaces de chat | `platform.identity` | `whatsapp-button` | Baja | F8 |
| **Multimoneda** con tasa fija administrable | `pricing` | `multi-currency` | Baja | backlog |
| **Log de auditoría** visible en admin (quién cambió tema/flags/precios) | `platform.audit` | Core | Media | F4 |

---

## 4. Decisiones de migración funcional

1. **"Solicitud de compra" no desaparece**: se convierte en una estrategia de checkout (`QUOTE_REQUEST`). Con `payments` activo, el tenant elige `QUOTE_REQUEST`, `ONLINE_PAYMENT` o ambos.
2. **Telegram deja de ser "el" canal**: es un `NotificationChannel` más (Telegram, email, WhatsApp Cloud API, webhook genérico).
3. **Google Sheets deja de ser "la" fuente**: `CatalogSource` Strategy (Google Sheets público, CSV/XLSX subido). Se conserva la semántica de *preview → mappings → apply* y el *ownership* de campos.
4. **La IA del asistente es un adaptador**: `AiChatProvider` (OpenAI, Ollama, Anthropic, Fake para tests). El modo guiado determinista siempre existe como *fallback*.
5. **Nada de datos del dominio "diesel"** en el código: los datos de Diesel Power Pro pasan a ser un **tenant de demostración** (`seed` opcional) para probar que el motor reproduce la web anterior.
