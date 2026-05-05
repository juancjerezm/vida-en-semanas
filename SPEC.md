# Vida en Semanas — Spec v3

> **Revision history**: v1 → v2 (prototype, single-file SPA) → **v3 (dashboard MVP, 8 features, May 2026)**

## Concepto

Dashboard SPA con 8 features: el usuario ingresa su fecha de nacimiento y ve, en una cuadrícula de semanas, cuántas ya vivió y cuántas le quedan. Además: tracking semanal de tiempo, racha de actividad, hitos de vida, balance de tiempo, reflexiones, recordatorios, y dark mode. Free/Pro tiers con feature gating client-side. Todo persiste en localStorage. Sin login, sin backend, sin build step.

## Arquitectura

```
index.html          ← Landing page + pricing grid + CTA → app.html
app.html            ← Dashboard SPA con tab navigation + state engine
  ├── Tab: Resumen     (life grid + stats + streak badge)
  ├── Tab: Actividad   (time tracking + milestones + balance)
  └── Tab: Reflexiones (weekly journal)
shared.css          ← Todos los tokens (light/dark), reset, componentes, responsive, print
```

**Dos archivos HTML separados, cero dependencias, cero build step.** `shared.css` vinculado desde ambos HTMLs con `<link>`. Tabs togglean vía `display` block/none (preserva estado de formularios). Los datos persisten en una sola key de localStorage: `vida-en-semanas`.

**v2 original** (`vida-en-semanas-v2.html`) se conserva como referencia, sin modificar.

## Design Tokens

Dirección: **modern-minimal** (Linear / Vercel). Paleta cobalt con tokens oklch. Sin sombras, bordes hairline.

```css
/* Light mode (default) */
--bg:             oklch(99% 0.002 240);
--surface:        oklch(100% 0 0);
--fg:             oklch(18% 0.012 250);
--muted:          oklch(54% 0.012 250);
--border:         oklch(92% 0.005 250);
--accent:         oklch(58% 0.18 255);
--accent-subtle:  oklch(95% 0.025 255);
--accent-milestone: oklch(70% 0.16 65);    /* warm amber para hitos */

/* Time tracking category colors */
--cat-trabajo:     oklch(58% 0.18 255);     /* cobalt */
--cat-familia:     oklch(55% 0.16 145);     /* green */
--cat-ejercicio:   oklch(55% 0.18 25);      /* red */
--cat-aprendizaje: oklch(55% 0.16 300);     /* purple */

/* Dark mode — luminancia invertida, mismo chroma/hue */
[data-theme="dark"] {
  --bg:             oklch(14% 0.002 240);
  --surface:        oklch(18% 0.002 240);
  --fg:             oklch(90% 0.006 250);
  --muted:          oklch(65% 0.008 250);
  --border:         oklch(24% 0.005 250);
  --accent:         oklch(72% 0.16 255);
  --accent-subtle:  oklch(18% 0.03 255);
  --accent-milestone: oklch(75% 0.15 65);
  --cat-trabajo:     oklch(72% 0.16 255);
  --cat-familia:     oklch(68% 0.15 145);
  --cat-ejercicio:   oklch(68% 0.16 25);
  --cat-aprendizaje: oklch(68% 0.14 300);
}

--font-display: -apple-system, BlinkMacSystemFont, 'SF Pro Display', system-ui, sans-serif;
--font-body:    -apple-system, BlinkMacSystemFont, 'SF Pro Text', system-ui, sans-serif;
--font-mono:    'JetBrains Mono', 'IBM Plex Mono', ui-monospace, Menlo, monospace;
```

Reglas de uso:
- Acento cobalt aparece solo en: CTA primario, celdas vividas, highlight del nombre en el heading, celda "semana actual", tab activo. Nada más.
- Acento amber (`--accent-milestone`) exclusivo para marcadores de hitos en la grilla.
- Mono exclusivo para números (stats, slider value, año labels, week labels).
- Sin sombras, sin gradientes, sin bordes redondeados grandes (8–16px max).
- Cero colores hardcodeados — todo referencia `var(--token)`.

## State & Persistencia

### localStorage: key `vida-en-semanas`

```js
{
  profile: {
    name: "",              // opcional, max 40 chars
    birthDate: null,       // ISO date string o null
    lifespan: 80,          // 60–100, default 80
    plan: "free",          // "free" | "pro"
    theme: "system"        // "system" | "light" | "dark"
  },
  timeTracking: {
    weeks: {
      "2026-W19": { trabajo: 40, familia: 14, ejercicio: 5, aprendizaje: 3 }
    }
  },
  milestones: [
    { id: "m1", name: "Terminé la facultad", date: "2020-12-15", color: null }
  ],
  reflections: [
    { id: "r1", weekKey: "2026-W19", text: "...", createdAt: "...", updatedAt: "..." }
  ],
  streak: {
    current: 4,
    best: 12,
    lastActiveWeek: "2026-W19"
  },
  reminders: {
    enabled: false,
    day: 6,                // 0=Sun...6=Sat
    time: "10:00",
    permissionGranted: false
  }
}
```

### sessionStorage (no persiste entre sesiones)

- `activeTab`: tab actual ("resumen" | "actividad" | "reflexiones")
- `activeWeek`: ISO week activa en el selector de semanas
- `reflectionDraft`: borrador auto-guardado cada 5s
- `reminderScheduled`: flag para evitar timers duplicados en múltiples tabs

### State Engine (`app.html`)

```js
loadState()    → lee localStorage, deep-merge con defaults
saveState()    → persiste todo menos state.ui
updateState(path, value)  → dot-path setter ("profile.name", "timeTracking.weeks.2026-W19.trabajo")
renderTab()    → dispatcher que llama renderResumen(), renderActividad(), o renderReflexiones()
isPro()        → state.profile.plan === 'pro'
gateFeature(featureName) → si Free, muestra upgrade prompt; si Pro, retorna true
```

## Lógica (core formulas, heredadas de v2)

```js
getWeeksFromBirth(birthDate):
  return floor((today - birthDate) / (7 * 24 * 60 * 60 * 1000))

getTotalWeeks():
  return lifespan * 52

getCurrentWeek(birthDate):
  return getWeeksFromBirth(birthDate) + 1    // 1-indexed

getYearForWeek(weekNum):
  return ceil(weekNum / 52)
```

Stats derivadas:
```
lived       = getWeeksFromBirth(birthDate)
total       = lifespan * 52
remaining   = max(0, total - lived)
pct         = round(lived / total * 100)
pctRemaining = round(remaining / total * 100)
livedYears  = floor(lived / 52)
remainingYears = floor(remaining / 52)
currentWeek = lived + 1
currentYear = ceil(currentWeek / 52)
```

## Features (v3)

### F1: Tracking de Tiempo

4 sliders (range inputs) por categoría: Trabajo, Familia, Ejercicio, Aprendizaje. Rango 0–80h, step 0.5h. Auto-detecta ISO week actual. Datos persisten por semana en `state.timeTracking.weeks`. Week selector ◄► para navegar semanas pasadas (read-only, capped en semana actual). Auto-save con 300ms debounce. Advertencia amber si suma > 168h. Colores por categoría vía `--cat-*` tokens. Visible en tab "Actividad".

### F2: Racha (Streak Counter)

Cuenta semanas ISO consecutivas con al menos una categoría > 0h en `timeTracking.weeks`. Muestra racha actual y mejor racha histórica en el header del dashboard. Visualización de dots (● = activa, ○ = inactiva). Free: últimos 2 dots. Pro: hasta 52 dots. Se recalcula en cada carga de `app.html`.

### F3: Hitos de Vida (Life Milestones)

CRUD de hitos con nombre (max 80 chars) y fecha. Se renderizan como dots amber (`--accent-milestone`) en la life grid en la posición de semana correcta. Vista de lista ordenada por fecha (más reciente primero). Tooltip con nombre al hover. Free: 3 hitos máximo. Pro: ilimitados. Validación: fecha debe ser posterior al nacimiento y ≤ hoy. Hito en semana actual recibe estilo combinado (amber + outline current).

### F4: Balance de Tiempo

Visualización derivada de `timeTracking.weeks`. Stacked horizontal bar chart (CSS puro, sin canvas) — una barra por semana con segmentos coloreados por categoría. Muestra últimas 4–12 semanas. Totales y porcentajes por categoría. PRO-GATED: usuarios Free ven upgrade prompt.

### F5: Reflexiones (Weekly Journal)

Textarea para escribir reflexión de la semana actual. Una reflexión por ISO week (save = upsert). Lista de reflexiones pasadas ordenadas por semana (más reciente primero). Preview de ~100 chars, expand para ver completo. Auto-save draft cada 5s a sessionStorage. Detección de cruce de semana. Free: últimas 4 entradas visibles. Pro: historial completo.

### F6: Recordatorios (Browser Notifications)

Notification API del navegador. Configuración: día de semana (Mon–Sun) + hora (HH:MM). Toggle enable/disable. setTimeout-based scheduling (re-schedule en cada carga). Flag `sessionStorage` para evitar timers duplicados. Fallback: badge in-app si notificaciones denegadas. PRO-GATED. Sin Service Worker (decisión de scope v3).

### F7: Precios (Free/Pro Tiers)

Dos tiers: Free ($0) y Pro ($4/mes). Client-side feature gating — plan es un flag en localStorage. Sin procesamiento de pagos real (MVP). `index.html` muestra pricing grid con 8 filas comparativas. CTA buttons: "Empezar gratis" (set plan=free → app.html), "Probá Pro" (set plan=pro → app.html). Upgrade/downgrade instantáneo. Pro column destacada ("Más popular").

### F8: Dark Mode

Detección de preferencia del sistema vía `matchMedia('(prefers-color-scheme: dark)')`. Toggle manual con 3 estados: system → dark → light → system. Bloqueante `<script>` en `<head>` de ambas páginas para prevención de FOUC: lee `localStorage.theme` sincrónicamente y setea `data-theme` en `<html>` antes del primer paint. Persiste en `state.profile.theme`. No gateado — disponible en Free y Pro.

## Feature Gating Matrix

| Feature | Free | Pro ($4/mo) |
|---------|------|-------------|
| Grilla de vida + stats | ✅ | ✅ |
| Tracking de tiempo (4 categorías) | ✅ | ✅ |
| Racha (streak dots) | Last 2 weeks | Full 52 |
| Hitos de vida | 3 máx | Ilimitados |
| Balance de tiempo | ❌ (upgrade prompt) | ✅ Chart |
| Reflexiones | Last 4 entries | Historial completo |
| Recordatorios | ❌ (upgrade prompt) | ✅ |
| Dark mode | ✅ | ✅ |
| Exportar PDF | ✅ | ✅ |

## Dependency Graph

```
                    ┌─────────────────────┐
                    │   Precios (F7)      │
                    │   (Free/Pro gate)   │
                    └──────┬──┬──┬──┬─────┘
                           │  │  │  │
              ┌────────────┘  │  │  └────────────┐
              ▼               ▼  ▼               ▼
    ┌─────────────┐  ┌──────────────┐  ┌──────────────────┐
    │ Hitos (F3)  │  │ Balance (F4) │  │ Recordatorios (F6)│
    │ (3 vs ∞)    │  │ (Pro-only)   │  │ (Pro-only)       │
    └──────┬──────┘  └──────┬───────┘  └──────────────────┘
           │                │
           ▼                ▼
    ┌─────────────┐  ┌──────────────┐
    │ Life Grid   │  │Time Tracking │
    │  (v2 grid)  │  │    (F1)      │
    └─────────────┘  └──────┬───────┘
                            │
                            ▼
                    ┌──────────────┐
                    │  Racha (F2)  │
                    └──────────────┘

Dark Mode (F8)  ──── aplica a TODO (solo tokens)
Reflexiones (F5) ── standalone (solo gateada por Precios)
```

## Life Grid

- **Desktop (>700px)**: 52 columnas × lifespan filas. Gap 2px. Year labels visibles (mono 9px, año 1, cada 5, último).
- **Mobile (≤700px)**: 26 columnas × (lifespan × 2) filas. Gap 1px. Year labels ocultos. Cada celda representa 2 semanas.
- Color coding: `lived` = accent cobalt, `current` = cobalt oscuro + outline 2px, `future` = border color.
- Hover: escala 1.8× con z-index 2. `title` tooltip: "Semana N".
- Auto-scroll: al cargar, scrollea celda `current` al centro del viewport.
- Contenedor con scroll vertical (max-height: 60vh), scrollbar fina (4px).
- Marcadores de hitos: dots amber en celdas correspondientes. Hito + current week = estilo combinado.
- Leyenda: Vividas / Esta semana / Por vivir + Hitos (si existen).

## 4 Stat Cards (tab Resumen)

| Card | Label (UPPERCASE, muted) | Value (mono, clamp 22-30px) | Sub (muted) |
|------|--------------------------|----------------------------|-------------|
| Semanas vividas | SEMANAS VIVIDAS | `lived.toLocaleString()` | `livedYears` años |
| Semanas por vivir | SEMANAS POR VIVIR | `remaining.toLocaleString()` | `remainingYears` años |
| Porcentaje vivido | PORCENTAJE VIVIDO | `pct%` | `pctRemaining%` por delante |
| Semanas totales | SEMANAS TOTALES | `total.toLocaleString()` | a `lifespan` años |

Responsive: 4 columnas en desktop (>700px), 2 columnas en mobile.

## Responsive Breakpoints

| Breakpoint | Stats | Grid cols | Tab bar | Year labels | Touch targets |
|---|---|---|---|---|---|
| > 700px | 4 cols | 52 | Horizontal, full labels | Visible | — |
| 700–420px | 2 cols | 26 | Horizontal, compact | Hidden | ≥44px |
| < 420px | 2 cols | 26 | Horizontal, icon-only | Hidden | ≥44px |

Tipografías usan `clamp()` para escalar fluidamente sin media queries adicionales.

## Tab Navigation (app.html)

- Tab bar sticky (`position: sticky; top: 0`), siempre visible.
- 3 tabs: Resumen | Actividad | Reflexiones.
- Toggle vía `display` block/none — no recarga la página.
- Tab activo resaltado con underline accent.
- Estado de tab en `sessionStorage.activeTab`. Default: "Resumen".
- Hash routing: `app.html#actividad` carga el tab Actividad.
- Cambio de tab preserva estado de formularios (textarea, sliders).

## Profile Editor (collapsible, en tab Resumen)

- Colapsado por defecto. Botón "Editar" expande.
- Si `birthDate` no está seteado (primera visita), auto-expandido con prompt.
- Form: name input (opcional, max 40 chars), birth date input (requerido, < hoy), lifespan slider (60–100, default 80).
- "Guardar" persiste a `state.profile` + localStorage, re-renderiza grid + stats.
- Se colapsa después de guardar.

## Landing Page (index.html)

```
Hero (headline + subtext + mini life grid preview)
  ↓
Feature cards (8 features, con descripción)
  ↓
Mini life grid (interactive: 52×5 preview)
  ↓
Pricing grid (Free vs Pro, Pro destacado como "Más popular")
  ↓
CTA: "Empezar gratis" | "Probá Pro" → setea plan en localStorage → app.html
```

- Pricing grid responsive: stacks vertical en mobile (≤700px).
- CTA buttons: si no existe key `vida-en-semanas`, la crea con defaults + plan seleccionado. Si ya existe, solo actualiza `profile.plan`.

## Dark Mode

- Bloqueante `<script>` en `<head>` previene FOUC: lee `theme` de localStorage sincrónicamente, setea `data-theme` en `<html>` antes del primer paint.
- Toggle con 3 íconos geométricos (◑ system / ● dark / ○ light). Ciclo: system → dark → light → system.
- `matchMedia('(prefers-color-scheme: dark)')` listener activo solo cuando `theme === 'system'`.
- Override manual ignora cambios del sistema.
- Consistente entre `index.html` y `app.html` (misma key de localStorage).
- No gateado: disponible para Free y Pro.
- Accessible: `aria-label`, keyboard focusable.

## Print CSS

`@media print` en `shared.css`:
- Oculta: tab bar, todos los botones, modals, upgrade prompts, week selector, sliders, formulario de reflexiones, profile editor.
- Body: blanco y negro.
- Grid: fuerza 52 columnas, sin scroll (todas las filas visibles), year labels visibles.
- Stats cards y leyenda visibles con bordes negros.
- Marcadores de hitos preservados (amber en pantalla, pattern en B&W).
- Botón "Exportar PDF" (en header) llama a `window.print()`.
- Disponible para Free y Pro (no gateado).

## Decisiones & Tradeoffs (v3)

1. **Dos archivos HTML en vez de SPA único**. `index.html` (landing + pricing) y `app.html` (dashboard). Separa concerns: adquisición vs. uso. Tradeoff: código duplicado para dark mode FOUC prevention y toggle — mitigado extrayendo todo el CSS a `shared.css`.

2. **Grilla de 52 columnas fijas** (heredado de v2). 52 semanas/año es estándar ISO 8601. No usamos semanas "reales" con días sobrantes (52.1429). Tradeoff: el último año "sobra" o "falta" una fracción.

3. **Grilla mobile: 26 columnas**. Cada celda representa 2 semanas en vez de 1. Mantiene legibilidad en pantallas chicas sin perder la metáfora visual. Tradeoff: los pares de semanas no se distinguen individualmente — mitigado con alternate shading.

4. **Persistencia completa en localStorage**. Single key `vida-en-semanas` con todo el estado. Sin backend. El usuario no pierde datos al cerrar el navegador. Tradeoff: sin sync entre dispositivos, sin backup.

5. **Feature gating client-side**. Plan es un string en localStorage. Sin procesamiento de pagos real. El usuario técnico puede modificarlo desde devtools. Aceptable para MVP. Tradeoff: no es seguro para producción con pagos reales.

6. **Sin Service Worker**. Notificaciones vía `setTimeout` re-scheduleado en cada carga. No persisten si el navegador está cerrado. Tradeoff: simplicidad de implementación vs. confiabilidad de notificaciones.

7. **Cero dependencias, cero build step**. Todo vanilla HTML/CSS/JS. Se abre directo en el navegador. Tradeoff: sin componentización, sin type-checking, sin hot reload. Para v4+ considerar framework si el producto escala.

8. **Sin autenticación ni backend**. La app funciona completamente client-side. No hay datos que proteger, no hay servidor que mantener.

## Decisiones Revertidas de v2

Estas decisiones explícitas de la spec v2 fueron revertidas en v3:

| # | Decisión v2 | Reversión v3 | Razón |
|---|------------|-------------|-------|
| R1 | Sin milestones/hitos (§Decisión #4) | **Hitos de vida con nombre y fecha**, dots amber en la grilla. v2 rechazaba dots sin labels; v3 implementa eventos nombrados configurables. | Feature #3 del MVP. |
| R2 | Sin persistencia (§Decisión #5) | **Full localStorage** bajo key `vida-en-semanas`. Datos sobreviven cierre/refresh del navegador. | Requisito de MVP: retener datos entre sesiones. |
| R3 | Sin precios/tiers (§"Lo que NO tiene") | **Free/Pro tiers** con feature gating client-side. Pro = $4/mo. | Feature #7, pricing grid en landing page. |
| R4 | Single-file SPA (§Decisión #1) | **Dos archivos**: `index.html` (landing) + `app.html` (dashboard). | 8 features no entran en un solo archivo mantenible. |
| R5 | Sin dark mode (§"Lo que NO tiene") | **Dark mode** con system detection + toggle manual + preferencia persistida. | Feature #8. |
| R6 | Birth date se pierde al cerrar (sin persistencia) | Birth date persiste en `localStorage.profile.birthDate`. | Consecuencia natural de R2. |

## Lo que NO tiene (y por qué)

- **Autenticación / backend**: la app es 100% client-side. No hay datos sensibles que proteger.
- **Procesamiento de pagos real**: el plan es un flag en localStorage. MVP tradeoff.
- **Service Worker**: notificaciones vía setTimeout. Complejidad innecesaria para v3.
- **Emojis / iconos decorativos**: rompen la dirección modern-minimal. Se usan formas geométricas (◑●○) y CSS puro.
- **"Metas semanales" con checkboxes**: estaba en el mockup de diseño pero no en los 8 features. Diferido a v3.1.
- **Sync multi-dispositivo**: requiere backend. Fuera de scope.
- **Animaciones de entrada**: distraen del dato. La grilla ya es suficientemente impactante.

## Implementación

| Archivo | Líneas | Rol |
|---------|--------|-----|
| `shared.css` | 1742 | Design tokens (50+ vars oklch), reset, tipografía, todos los componentes, responsive, dark mode, print |
| `index.html` | 353 | Landing page: hero, feature cards, pricing grid, CTA, dark mode FOUC script + toggle |
| `app.html` | 1647 | Dashboard SPA: 3 tabs, 8 features, state engine (`updateState`/`loadState`/`saveState`), feature gating (`isPro`/`gateFeature`), hash routing |

**Total**: 3742 líneas. 16 tasks en 9 batches. 8 features end-to-end. Cero dependencias externas.
