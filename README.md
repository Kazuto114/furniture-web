# Panelo — panel furniture shop with an AI consultant

An online shop for furniture made only from panels (laminated chipboard, painted MDF, birch plywood), with a Claude-powered consultant that:

- **analyses a photo of the customer's room** (style, palette, free wall space, constraints) and recommends matching catalog pieces;
- **generates custom furniture** sized for the room — a real drawing, cut list, hardware list and exact price from the workshop's build rules;
- **advises** on materials, care, ergonomics, styles and store policies, grounded in an expert knowledge base (RAG);
- **marks liked furniture** (favourites), manages the cart, **places orders** after explicit confirmation, and tracks or cancels them.

The website works fully without the AI (catalog, configurator, favourites, cart, checkout, orders, admin order desk):

- every product and design is shown as an **interactive 3D model** (drag to turn, doors and drawers open) with its length × width × height marked on it, next to the classic front-view drawing;
- customers can order any catalog piece **in their own size**: the 3D model, price, lead time and workshop notes follow the size sliders, and the resized piece goes to the cart as a made-to-order custom design;
- managers **add, edit and hide products** in the order desk with a visual builder (columns → sections, doors, drawers, colours) and a live 3D preview, price, production cost and margin; customers can't reach any of it;
- prices are in **Mongolian tögrög (₮)**.

**Languages:** English, Монгол, Русский, 中文, 한국어 — picked from the visitor's browser (or `DEFAULT_LANGUAGE`) and switchable in the header. The whole storefront is translated, including product names and descriptions, workshop warnings, hardware and cut-list names, and the labels inside the drawings; catalog search works in every language; the consultant answers in the customer's language.

## Quick start

Requires Python 3.10+ (developed on 3.14).

```powershell
# Windows PowerShell
python -m venv .venv
.\.venv\Scripts\python -m pip install -r requirements.txt
copy .env.example .env        # then put your ANTHROPIC_API_KEY (and an ADMIN_TOKEN) in .env
cd backend
..\.venv\Scripts\python -m uvicorn app.main:app --reload --port 8000
```

```bash
# macOS / Linux
python3 -m venv .venv && .venv/bin/pip install -r requirements.txt
cp .env.example .env
cd backend && ../.venv/bin/uvicorn app.main:app --reload --port 8000
```

Open http://127.0.0.1:8000 — the shop. The manager order desk is at http://127.0.0.1:8000/admin.html (enter `ADMIN_TOKEN`): **Orders** to process orders, **Products** to manage the catalog.

Tests: `cd backend && ../.venv/Scripts/python -m pytest` (no API key needed — the agent loop is tested against a scripted fake of the streaming client).

## Architecture

```
frontend/  (no build step: HTML + CSS + ES modules)
  index.html, css/styles.css     design system, light/dark, mobile-first
  js/app.js                      hash router, global likes/cart actions, badges, theme
  js/chat.js                     chat widget: SSE streaming, photo upload (pick/drag/paste), rich cards
  js/views/*.js                  home, catalog, product (3D view + custom size), designs, cart/checkout, orders
  js/viewer3d.js, js/model3d.js  3D model built from the engine layout (three.js, loaded only on pages that show one)
  vendor/three/                  three.js r160 (+ OrbitControls, CSS2DRenderer), served locally — no CDN at runtime
  admin.html, js/admin.js        order desk for managers: orders
  js/admin-products.js           order desk: product list and editor (visual builder, live 3D preview and pricing)

backend/app/
  main.py           FastAPI: REST API, POST /api/chat (SSE), uploads, admin API, static site
  design_engine.py  panel furniture engine: spec -> geometry, cut list, hardware, price, build-rule checks
  svg_render.py     front-view drawings (closed / interior) for every product and design
  rag.py            BM25 retrieval over knowledge/*.md (one chunk per "##" section)
  store.py          shop services shared by the API and the AI tools (catalog, designs, favourites, cart, orders, fees)
  seed.py           the 30-piece starting catalog (priced by the engine, never by hand); products a manager has
                    edited in the order desk are never overwritten by it, only re-priced
  images.py         room-photo intake (validate, EXIF-rotate, resize to 1568 px, JPEG)
  agent/prompts.py  system prompt (stable for caching)
  agent/tools.py    12 tools: JSON schemas, pydantic validation, execution -> result for Claude + UI cards
  agent/chat.py     streaming agent loop, append-only transcript persistence
backend/knowledge/  expert knowledge base (Markdown) — edit freely, re-indexed on restart
```

Storage is SQLite (`backend/data/panelo.sqlite3`) plus uploaded photos in `backend/data/uploads/`. Visitors are identified by an anonymous HttpOnly cookie; favourites, cart, designs, chat and orders hang off it.

### Choosing the AI: Claude or Gemini

`AI_PROVIDER` in `.env` selects the model that runs the consultant; everything else (tools, prompt, cards, translations) is shared.

| `AI_PROVIDER` | Key | Model setting | Notes |
| --- | --- | --- | --- |
| `anthropic` (default) | `ANTHROPIC_API_KEY` from platform.claude.com (prepaid credit) | `CLAUDE_MODEL`, `CLAUDE_EFFORT` | Streaming replies, refusal fallbacks, prompt caching — `agent/chat.py` |
| `gemini` | `GEMINI_API_KEY` from aistudio.google.com/apikey (free tier) | `GEMINI_MODEL` (default `gemini-flash-latest`) | Free-tier rate limits apply (check them in AI Studio); 429s are retried with backoff; replies arrive per step rather than token by token — `agent/gemini_chat.py` |

Check the configured AI with one tiny request (it says exactly what to fix — wrong key, no credit, quota, model name):

```powershell
cd backend
..\.venv\Scripts\python -m app.check_ai
```

Each provider keeps its own model history, so switching providers starts a fresh AI context (the visible chat history stays). On Google's free tier, prompts may be used by Google to improve its products — don't test with real customer data.

### How the consultant works

- **Model:** `claude-opus-5-5` with adaptive thinking and `effort` set explicitly (`CLAUDE_EFFORT`, default `medium`). Streaming via `client.beta.messages.stream`, with server-side refusal fallbacks (`fallbacks: "default"`) and automatic prompt caching.
- **RAG:** `search_knowledge_base` retrieves the top passages from `knowledge/`; the system prompt requires grounding every policy, fee, lead time and material fact in tool results. Source chips are shown under answers.
- **Tools:** `search_catalog`, `get_product_details`, `create_custom_design`, `view_cart`, `add_to_cart`, `update_cart_item`, `set_favorite`, `list_favorites`, `place_order`, `list_orders`, `cancel_order`, `search_knowledge_base`. Tool inputs stream eagerly and are validated with pydantic before anything runs; invalid input goes back to Claude as an error result so it can correct itself.
- **Room photos:** sent as image blocks. The prompt has Claude describe what matters, estimate sizes from reference objects (and say they are estimates), recommend catalog pieces and, where there is a clear spot, design a custom piece.
- **Orders:** `place_order` requires `customer_confirmed=true` and the prompt's protocol (quote totals with `view_cart`, collect details, get an explicit "yes"). No payment details are ever collected — a manager calls to confirm.
- **Transcript integrity:** every assistant turn is stored exactly as returned (thinking signatures, tool calls) and replayed verbatim; the system prompt and tools never change mid-conversation; repairs only append. This keeps prompt caching effective and satisfies Opus 5.5's conversation-bound thinking.
- **Cost guard:** 30 chat messages per visitor per 10 minutes (in-memory). Production cost and margin are never exposed to customers or to Claude (`/api/admin/designs/{id}/cost` is staff-only).

### Prices and currency

Prices are in Mongolian tögrög (`CURRENCY=MNT`, `CURRENCY_SYMBOL=₮`). The workshop's cost data at the top of `design_engine.py` (board per m², hardware, labour) stays in USD because the board and hardware are imported; the engine converts production cost with `EXCHANGE_RATE` (₮ per 1 USD, default 3600 — **set it to your current rate**) and rounds retail prices up to `PRICE_ROUNDING` (default 1,000 ₮). Changing the rate re-prices the whole catalog on the next restart. Delivery, assembly and measurement fees are fixed ₮ amounts at the top of `store.py` and must match `knowledge/ordering_delivery_payment.md` and `faq.md`.

### 3D view and custom sizes

`GET /api/products/{id}/layout` (and `/api/designs/{id}/layout`) return the engine's panel geometry; `js/viewer3d.js` builds the 3D model from it — carcass, shelves, rails with clothes, books on shelves, doors on hinges, drawers on runners, base, and dimension rods labelled length (side to side), width (front to back) and height. Where WebGL isn't available the page falls back to the SVG drawing.

On a product page, "Custom size" lets the customer set length, width and height within sensible limits (roughly 50–200 % of the standard size, inside the sheet limits). Each change re-runs the engine (`POST /api/designs/customize`), so the model, price, lead time and warnings always match what the workshop would build; sizes the engine can't build can't be ordered. "Add to cart" calls `POST /api/cart/custom`, which saves (or reuses) the customer's design for that size and adds it to the cart — it is then a made-to-order custom piece (custom lead time, free measurement visit).

### Managing products (order desk)

Products → "New product" (or click a product). Start from a copy of a similar product, fill in the name and description, choose category, room and style, set the size, board, base and handles, pick the colours, and describe the inside as columns of sections (shelves, hanging rail, drawers, knee space; open, door, double/glass door or flap). The preview on the right shows the 3D model, the shop price, production cost and margin, and the workshop's checks; "Add to the shop" is only enabled when the piece can be built. One product has one price, so all its colours must cost the same to make (all laminate decors, or all painted MDF colours). Optional Mongolian, Russian, Chinese and Korean names and descriptions are shown to customers in those languages. "Hide from the shop" removes a product from the catalog, search and the assistant without breaking past orders.

### The design engine (carpenter's rules)

Furniture is described as columns (left→right) of compartments (top→bottom). The engine:
- scales relative sizes to the real carcass (18 mm panels, 3 mm HDF back, half-overlay fronts with 3 mm gaps);
- enforces sheet limits (2750 mm LDSP/MDF, 2480 mm plywood), door leaf ≤ 600 mm (auto-splits into double doors), drawer front 100–450 mm and width 250–1000 mm, shelf span (800/700/900 mm), hanging depth ≥ 530 mm (else a pull-out rail), knee space for desks, ergonomic ranges per furniture type;
- counts hinges by door height, runners by depth, handles, legs/plinth/brackets, anti-tip kits for units ≥ 1200 mm;
- prices from the cut list: board area × price/m², edge banding, hardware, labour, × markup × exchange rate → retail rounded up to the price step.

Workshop constants (prices per m² and hardware prices in USD, markup) are at the top of `design_engine.py`; delivery/assembly/measurement fees (₮) are at the top of `store.py` and must match `knowledge/ordering_delivery_payment.md`.

### Languages (i18n)

| What | Where |
| --- | --- |
| Interface texts, rooms/styles/colours/materials, hardware and part names, engine message templates | `frontend/js/locales/{en,mn,ru,zh,ko}.js` (`en.js` is the reference; missing keys fall back to English) |
| Product names and descriptions | `backend/app/catalog_i18n.py` for the starting catalog; managers enter their own in the order desk (served together by `GET /api/catalog/i18n?lang=`) |
| The server's own chat notices (errors, "still answering…") | `backend/app/i18n.py` |
| Engine warnings | the engine sends `{code, params}` (`summary.message_codes`) next to the English text; the browser renders `engine.<code>` in the visitor's language |

The site language is sent with every chat message and added to the page-context line of the user turn ("reply in it unless the customer writes in another language"). It is deliberately **not** put in the system prompt, which must stay byte-identical for caching and for Claude's conversation-bound thinking. The knowledge base stays in English; Claude translates what it retrieves.

To add a language: copy `en.js` to `locales/<code>.js` and translate it, import it in `frontend/js/i18n.js`, add the code to `LANGUAGES` and `SERVER_TEXT` in `backend/app/i18n.py` and to `DEPTH_LABEL` in `svg_render.py`, and optionally add product texts to `catalog_i18n.py`. `tests/test_i18n.py` checks that server texts and product translations are complete.

## Team ownership

| Area | Owner | Where |
| --- | --- | --- |
| Build rules, materials, hardware, prices per m², care & assembly content | Furniture carpenter | `design_engine.py`, `knowledge/materials.md`, `hardware.md`, `care_maintenance.md`, `assembly_safety.md`, `custom_design_service.md` |
| Design system, page flows, chat UX, drawings | UI/UX | `frontend/`, `svg_render.py` |
| Agent, tools, prompt, retrieval, evaluation | AI engineer | `backend/app/agent/`, `rag.py`, `tests/test_chat.py`, `tests/test_rag.py` |
| Range, policies, fees, trends, room guides | Heads of market | `seed.py`, `store.py` (fees), `knowledge/ordering_delivery_payment.md`, `warranty_returns.md`, `styles_and_trends.md`, `room_guides.md`, `faq.md` |

The knowledge files and catalog are realistic starting content — have each owner review them against your actual business before launch (fees in the knowledge base are in ₮ and must be updated together with `store.py`).

## API overview

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/api/products`, `/api/products/{id}`, `/api/products/{id}/image.svg?color=&open=` | catalog, details, drawings |
| GET | `/api/products/{id}/layout?color=`, `/api/designs/{id}/layout` | panel geometry for the 3D view |
| GET | `/api/catalog/facets` | filter values |
| POST | `/api/designs/customize` | resize/recolour a product (preview, or `save: true`) |
| GET | `/api/designs`, `/api/designs/{id}`, `/api/designs/{id}/image.svg` | my designs |
| GET/POST | `/api/favorites` | liked items |
| GET/POST/PATCH | `/api/cart`, `/api/cart/{id}` | cart with fee quote (`?delivery_method=&assembly=`) |
| POST | `/api/cart/custom` | add a catalog product in the customer's own size |
| GET/POST | `/api/orders`, `/api/orders/{id}/cancel` | orders |
| POST | `/api/uploads` | room photo |
| POST | `/api/chat` | chat turn (Server-Sent Events) |
| GET/POST | `/api/chat/history`, `/api/chat/new` | restore / reset conversation |
| GET/PATCH | `/api/admin/orders[/{id}]` | order desk (header `X-Admin-Token`) |
| GET | `/api/admin/catalog-options`, `/api/admin/products[/{id}]` | product editor data, all products incl. hidden (admin) |
| POST | `/api/admin/products/preview` | live price, cost, checks and layout while editing (admin) |
| POST/PUT/PATCH | `/api/admin/products`, `/api/admin/products/{id}` | create, update, hide/show a product (admin) |

## Before going to production

- Real customer accounts (the anonymous cookie is per browser) and staff authentication beyond a shared token.
- HTTPS, a production server (e.g. `uvicorn --workers` behind a reverse proxy) and Postgres if you run several workers; move the chat rate limiter and per-conversation locks to Redis.
- Notifications to managers on new orders (email/Telegram), and an SMS/call workflow.
- An evaluation set of real customer questions and room photos to tune the prompt and `CLAUDE_EFFORT` (the API bill scales with effort and photo count).
- Real product photography alongside the generated drawings.
