# Instagram-Fidelity Mobile Grid Planner Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Transform the current planner into a mobile-first, pixel-faithful Instagram clone where each post carries its own *idea*, *script*, and *why* — so the grid doubles as a strategic thinking space, not just a mood board.

**Architecture:** Single-file static site (`index.html`). All state in `localStorage` under key `liren.grid.planner.v2` (bumped — v1 schema is incompatible). Mobile-first CSS, desktop as enhancement. Touch + mouse drag-reorder. Extend post schema with `idea`, `script`, `why`, `hidden` fields. Hide all "tool chrome" (top nav, page meta, pillar legend); surface admin actions through Instagram's native "⋯" menu and a long-press gesture on tiles.

**Tech Stack:** Vanilla HTML / CSS / JS, no build step, deployed from `main` to Vercel via GitHub auto-deploy.

**Reference:** `peter.visuals` Instagram profile (mobile desktop web view) — the target fidelity.

---

## Task 1 — Baseline the current deploy

**Files:**
- Read: `index.html` (current live version)

**Step 1: Confirm current state is clean on main**

Run: `cd "/Volumes/Extreme SSD/liren/grid-planner-deploy" && git status && git log --oneline -3`
Expected: clean working tree, HEAD at `811afba` or later.

**Step 2: Capture a "before" snapshot of the deployed URL**

Open https://grid-planner-deploy.vercel.app in a mobile-sized viewport (390×844) and note what needs to change:
- [ ] Top nav "Grid Planner 立人" + Export/Import/Reset/+New Post → remove
- [ ] Page meta "Liren Labs — Grid Planner v1" → remove
- [ ] Pillar legend block under the grid → remove
- [ ] Profile layout is desktop-grid (168px avatar left, stats right) → needs IG mobile layout
- [ ] Default avatar is a gradient "L" → replace with blue metallic L logo
- [ ] Modal editor shows pillar/type/slides/caption → add idea/script/why
- [ ] No touch drag-reorder → add
- [ ] No hide/delete UX → add

**Step 3: Commit nothing — this is a read-only audit**

No commit in this task.

---

## Task 2 — Bump state version and seed new schema

**Files:**
- Modify: `index.html` (JS section, `STORAGE_KEY` constant + `defaultState()` + seed posts)

**Why first:** Every subsequent task reads/writes state. Nailing the schema first prevents cascading rewrites.

**Step 1: Change storage key to `liren.grid.planner.v2`**

`index.html` — find:
```js
const STORAGE_KEY = 'liren.grid.planner.v1';
```
Replace with:
```js
const STORAGE_KEY = 'liren.grid.planner.v2';
```

**Step 2: Extend each seed post in `SEED_POSTS_META` with `idea`, `script`, `why`, `hidden: false`**

Each entry gains three fields. Example (apply to all 12):
```js
{
  type: 'reel',
  pillar: 'HOT-TAKE',
  caption: '6 meeting Senin, 0 catatan manual...',
  idea: 'Reel 30 detik. Gue duduk di depan laptop, 6 tab Notion Meeting terbuka. Kamera POV. Cut cepat tiap tab nunjukin summary otomatis dari Agent Meeting.',
  script: 'Hook 0-3s: "Gue punya 6 meeting hari ini. Gue gak akan nyatet satu kata pun."\nBeat 3-12s: Zoom in ke Agent Meeting UI, show auto-transcript jalan.\nBeat 12-22s: Jump cut ke 6 hasil ringkasan.\nCTA 22-30s: "Link di bio — namanya Agent Meeting."',
  why: 'Meta hot-take. Hook pertanyaan retoris yang bikin orang nunggu punchline. Ngejual produk tanpa jualin — produk jadi jawaban dari masalah yang lo akuin di depan. lo/gue register karena ini hot-take, bukan positioning.',
}
```

Do this for all 12. Draft content lives in `docs/plans/2026-04-24-seed-content.md` (Task 3 will create it).

**Step 3: Update `defaultState()` to pass through the new fields**

`SEED_POSTS_META.map((m, i) => ({ ... }))` should spread `...m` or explicitly include the three new fields plus `hidden: false`.

**Step 4: Update schema validation in `loadState()`**

Add a migration path: if parsed state has `v1` shape (no idea/script/why on posts), map through and fill with empty strings, then save back under the new key. Do not delete v1 key — leave it for recovery.

```js
function loadState() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (raw) {
      const parsed = JSON.parse(raw);
      if (parsed.profile && Array.isArray(parsed.posts)) return parsed;
    }
    // Migrate from v1
    const v1 = localStorage.getItem('liren.grid.planner.v1');
    if (v1) {
      const parsed = JSON.parse(v1);
      if (parsed.profile && Array.isArray(parsed.posts)) {
        parsed.posts = parsed.posts.map(p => ({
          idea: '', script: '', why: '', hidden: false, ...p
        }));
        return parsed;
      }
    }
  } catch {}
  return defaultState();
}
```

**Step 5: Visual QA**

Open in browser. DevTools → Application → localStorage. Confirm:
- New posts have `idea`, `script`, `why`, `hidden` fields.
- Existing v1 users get migrated on first load.

**Step 6: Commit**

```bash
git add index.html
git commit -m "feat: extend post schema with idea/script/why/hidden + v2 migration"
```

---

## Task 3 — Seed strategic content (idea/script/why) for the 12 default posts

**Files:**
- Create: `docs/plans/2026-04-24-seed-content.md` (content doc — lives in repo as the source of truth for default copy)
- Modify: `index.html` (paste drafted copy into `SEED_POSTS_META`)

**Step 1: Draft the 12 idea/script/why triples**

Write them in `docs/plans/2026-04-24-seed-content.md` first so they're reviewable in isolation. Structure per post:

```markdown
## Post 01 — 6 meeting Senin (Reel · HOT-TAKE)

**Idea:** [30-60 words — the visual concept]
**Script:** [beat-by-beat, timestamps for reels, slide 1/2/3 for carousels]
**Why:** [30-60 words — what reaction it should trigger, which pillar objective it hits, voice register decision]
```

Draft all 12. Keep voice registers consistent with the pillar legend rules (lo/gue for hot-takes/teardowns/trend-riffs, kamu for tool-drop/philosophy/founder).

**Step 2: Paste the drafted content into `SEED_POSTS_META` in `index.html`**

**Step 3: Hard-reload the planner in browser. Click through all 12 posts in the modal. Confirm idea/script/why render.**

Note: the Strategy tab doesn't exist yet — verify the data is in state (DevTools → Application → localStorage → parse JSON and grep for "idea":).

**Step 4: Commit**

```bash
git add docs/plans/2026-04-24-seed-content.md index.html
git commit -m "content: draft idea/script/why for 12 default posts"
```

---

## Task 4 — Embed the blue "L" logo as default avatar (SVG)

**Files:**
- Modify: `index.html` — replace the gradient-mark avatar with an SVG Liren logo

**Note on the asset:** The reference is a metallic blue circular "L" mark. The plan embeds an SVG recreation (good enough, no HTTP request). **If the user prefers the exact PNG they showed**, they can save it to `/Volumes/Extreme SSD/liren/grid-planner-deploy/logo.png` and replace `DEFAULT_AVATAR` with `'./logo.png'` in Task 4.5.

**Step 1: Define `DEFAULT_AVATAR` constant near the top of the script block**

```js
const DEFAULT_AVATAR = `data:image/svg+xml;utf8,` + encodeURIComponent(`
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 200 200">
  <defs>
    <radialGradient id="bgGrad" cx="50%" cy="50%" r="55%">
      <stop offset="0%" stop-color="#ffffff"/>
      <stop offset="100%" stop-color="#eef1f7"/>
    </radialGradient>
    <linearGradient id="metallic" x1="0%" y1="0%" x2="100%" y2="100%">
      <stop offset="0%" stop-color="#6ea8ff"/>
      <stop offset="35%" stop-color="#1b44ff"/>
      <stop offset="70%" stop-color="#0b2bb8"/>
      <stop offset="100%" stop-color="#4b7dff"/>
    </linearGradient>
    <filter id="shine"><feGaussianBlur stdDeviation="0.6"/></filter>
  </defs>
  <circle cx="100" cy="100" r="94" fill="url(#bgGrad)" stroke="url(#metallic)" stroke-width="6"/>
  <path d="M70 60 L70 140 L130 140" fill="none" stroke="url(#metallic)" stroke-width="10" stroke-linecap="round" stroke-linejoin="round" filter="url(#shine)"/>
  <path d="M112 122 L130 122 L130 140" fill="none" stroke="url(#metallic)" stroke-width="10" stroke-linecap="round" stroke-linejoin="round" filter="url(#shine)"/>
</svg>
`);
```

**Step 2: Update `renderProfile()` to use `DEFAULT_AVATAR` when `state.profile.avatar` is null**

Change the block that currently mounts the `.mark` gradient bubble. Always mount an `<img>`, defaulting to `DEFAULT_AVATAR`:

```js
function renderProfile() {
  const avatar = document.getElementById('avatar');
  avatar.innerHTML = ''; // clear
  const img = document.createElement('img');
  img.src = state.profile.avatar || DEFAULT_AVATAR;
  img.alt = state.profile.username;
  avatar.appendChild(img);
  const editBadge = document.createElement('div');
  editBadge.className = 'avatar-edit';
  editBadge.innerHTML = `<svg viewBox="0 0 24 24" ...>...</svg>`;
  avatar.appendChild(editBadge);
  // mini avatar in modal
  const mini = document.getElementById('miniAvatar');
  mini.innerHTML = '';
  const miniImg = document.createElement('img');
  miniImg.src = state.profile.avatar || DEFAULT_AVATAR;
  mini.appendChild(miniImg);
  // ... rest
}
```

**Step 3: Remove the `.mark` gradient pseudo-element CSS** (it's no longer used)

Remove `.avatar::before` and `.avatar .mark` rules.

**Step 4: Visual QA**

Reload. Confirm profile picture is now the blue L logo. Upload a replacement through the avatar click → confirm it overrides. Clear localStorage → reload → logo returns.

**Step 5: Commit**

```bash
git add index.html
git commit -m "feat: Liren L logo as default avatar (SVG inline)"
```

---

## Task 5 — Strip tool chrome (top nav, page meta, legend)

**Files:**
- Modify: `index.html` — delete three HTML blocks and their CSS

**Step 1: Remove the `<nav class="topnav">…</nav>` block entirely**

Also remove its CSS rules: `.topnav`, `.topnav-inner`, `.topnav-brand`, `.topnav-actions`, `.tnbtn*`.

**Step 2: Remove the `.page-meta` block**

```html
<div class="page-meta">
  <h1>Liren Labs — Grid Planner <span style="opacity:0.4; font-size:20px;">v0</span></h1>
  <div class="caption" id="metaCount">…</div>
</div>
```
Also remove `.page-meta` CSS and the JS line that writes to `#metaCount`.

**Step 3: Remove the entire `<div class="legend">…</div>` block**

Also remove `.legend*` CSS.

**Step 4: Wire admin actions into the profile `⋯` menu (done in Task 6 — placeholder for now)**

For now, leave Export/Import/Reset/+New Post inaccessible. Task 6 re-adds them via the IG ⋯ button.

**Step 5: Visual QA**

Reload. Page is: profile header → highlights → tabs → grid. Nothing else. Confirm no JS errors from missing elements (the `#metaCount` write needs to be either removed or guarded with `el && …`).

**Step 6: Commit**

```bash
git add index.html
git commit -m "refactor: strip top nav, page meta, and pillar legend (pure IG)"
```

---

## Task 6 — Action sheet (replaces top nav)

**Files:**
- Modify: `index.html` — wire the existing `.dots` span in the profile top row into an action sheet

**Step 1: Make `.dots` interactive**

Add `id="profileDots"` and `role="button"` to the span.

**Step 2: Build a bottom-sheet component**

New HTML at the end of `<body>`:

```html
<div class="sheet-backdrop" id="sheetBackdrop">
  <div class="sheet" id="sheet">
    <div class="sheet-handle"></div>
    <button class="sheet-item" data-action="new">+ New post</button>
    <button class="sheet-item" data-action="hidden">Hidden posts</button>
    <button class="sheet-item" data-action="export">Export JSON</button>
    <button class="sheet-item" data-action="import">Import JSON</button>
    <button class="sheet-item danger" data-action="reset">Reset to default</button>
    <button class="sheet-item cancel" data-action="cancel">Cancel</button>
  </div>
</div>
```

**Step 3: Style it mobile-first (bottom-sheet pattern)**

```css
.sheet-backdrop { position: fixed; inset: 0; background: rgba(0,0,0,0.6); z-index: 90; display: none; }
.sheet-backdrop.open { display: block; }
.sheet {
  position: fixed; left: 0; right: 0; bottom: 0;
  background: #1c1c1c; border-radius: 16px 16px 0 0;
  padding: 12px 0 max(12px, env(safe-area-inset-bottom));
  animation: sheetUp 0.18s ease-out;
  max-width: 540px; margin: 0 auto;
}
@keyframes sheetUp { from { transform: translateY(100%); } to { transform: translateY(0); } }
.sheet-handle { width: 36px; height: 4px; background: #444; border-radius: 2px; margin: 0 auto 10px; }
.sheet-item {
  display: block; width: 100%;
  padding: 16px 20px;
  background: transparent; border: none; border-top: 1px solid #2a2a2a;
  color: #fff; font-size: 15px; text-align: left; cursor: pointer;
}
.sheet-item:first-of-type { border-top: none; }
.sheet-item.danger { color: var(--ig-red); }
.sheet-item.cancel { color: var(--ig-muted); font-weight: 600; text-align: center; border-top: 8px solid #000; }
```

**Step 4: Wire the sheet — open on `.dots` click, close on backdrop / cancel / action-complete**

```js
const sheetBackdrop = document.getElementById('sheetBackdrop');
document.getElementById('profileDots').addEventListener('click', () => sheetBackdrop.classList.add('open'));
sheetBackdrop.addEventListener('click', (e) => {
  if (e.target === sheetBackdrop) sheetBackdrop.classList.remove('open');
});
document.querySelectorAll('.sheet-item').forEach(b => b.addEventListener('click', () => {
  const a = b.dataset.action;
  sheetBackdrop.classList.remove('open');
  if (a === 'new') addNewPost();
  else if (a === 'hidden') openHiddenView(); // Task 10
  else if (a === 'export') doExport();
  else if (a === 'import') jsonPicker.click();
  else if (a === 'reset') doReset();
}));
```

Hoist `doExport` and `doReset` into named functions (move code out of the now-deleted top nav button handlers).

**Step 5: Visual QA**

Tap `⋯` on profile → sheet slides up. Tap outside → closes. Tap "+ New post" → creates post and opens modal.

**Step 6: Commit**

```bash
git add index.html
git commit -m "feat: action sheet from profile ⋯ menu (new/hidden/export/import/reset)"
```

---

## Task 7 — Mobile-first profile header

**Files:**
- Modify: `index.html` — rewrite `.profile` HTML and CSS to match IG mobile

**Reference:** peter.visuals screenshot shows: avatar top-left, username + verified + ⋯ in a row, stats in a row below, name + bio block, then Follow/Message/+ button row, then highlights.

**Step 1: Restructure the HTML**

```html
<section class="profile">
  <div class="profile-head">
    <div class="avatar" id="avatar"></div>
    <div class="profile-username-row">
      <input class="username" id="username" value="liren.labs" />
      <span class="verified" title="verified"></span>
      <span class="dots" id="profileDots" role="button">⋯</span>
    </div>
  </div>

  <div class="profile-stats">
    <div><b id="statPosts">12</b><span>posts</span></div>
    <div><b>247</b><span>followers</span></div>
    <div><b>18</b><span>following</span></div>
  </div>

  <div class="profile-bio-block">
    <input class="profile-name" id="profileName" value="Liren Labs · 立人" />
    <textarea class="profile-bio" id="profileBio">…</textarea>
  </div>

  <div class="profile-cta">
    <button class="btn btn-primary">Follow</button>
    <button class="btn btn-ghost">Message</button>
    <button class="btn btn-icon">+</button>
  </div>
</section>
```

Note: username row no longer contains the CTA buttons. Stats are centered-flex (IG mobile style). CTAs drop into their own row.

**Step 2: CSS**

```css
.profile { padding: 12px 16px 16px; border-bottom: 1px solid var(--ig-border); }
.profile-head { display: flex; align-items: center; gap: 20px; margin-bottom: 16px; }
.avatar { width: 88px; height: 88px; flex-shrink: 0; margin: 0; }
.profile-username-row { display: flex; align-items: center; gap: 10px; flex: 1; min-width: 0; }
.username { font-size: 17px; font-weight: 500; min-width: 0; flex: 1; }
.profile-stats {
  display: flex; justify-content: space-around;
  padding: 8px 0 14px; border-bottom: none;
  margin: 0 -4px;
}
.profile-stats > div { display: flex; flex-direction: column; align-items: center; font-size: 13px; color: var(--ig-muted); }
.profile-stats b { color: #fff; font-size: 15px; font-weight: 600; margin-bottom: 2px; }
.profile-bio-block { margin-bottom: 14px; }
.profile-name { font-size: 14px; margin-bottom: 4px; display: block; }
.profile-bio { font-size: 14px; min-height: 0; height: auto; }
.profile-cta { display: flex; gap: 6px; }
.profile-cta .btn { flex: 1; padding: 8px; font-size: 14px; }
.profile-cta .btn-icon { flex: 0 0 40px; }

@media (min-width: 735px) {
  /* Desktop enhancement — closer to IG desktop but keep mobile-first DNA */
  .profile { padding: 24px 20px; }
  .profile-head { gap: 40px; }
  .avatar { width: 150px; height: 150px; }
  .profile-stats { justify-content: flex-start; gap: 40px; padding: 14px 0 18px; margin: 0 0 6px; }
  .profile-stats > div { flex-direction: row; gap: 6px; font-size: 15px; }
  .profile-stats b { font-size: 15px; margin-bottom: 0; }
}
```

**Step 3: Visual QA**

Resize browser to 390px width. Compare to peter.visuals reference. Avatar smaller, stats centered, CTAs full-width — should read as IG mobile.

**Step 4: Commit**

```bash
git add index.html
git commit -m "refactor: mobile-first profile header matching IG layout"
```

---

## Task 8 — Mobile-friendly grid tile sizing + hover-free affordances

**Files:**
- Modify: `index.html` — `.grid`, `.tile`, `.tile-hover`, `.pillar`

**Step 1: Tighten grid on mobile**

```css
.grid { gap: 2px; margin-top: 2px; }
.tile .pillar {
  font-size: 8px; padding: 2px 5px;
  top: 6px; left: 6px;
}
.tile .corner-icon svg { width: 18px; height: 18px; }
```

**Step 2: Remove the fake-metrics hover overlay on mobile**

The "likes/comments" hover overlay is a nice desktop affordance but doesn't work on touch. Guard it:

```css
@media (hover: none) { .tile-hover { display: none; } }
```

**Step 3: Hide the drag handle on mobile (we'll use long-press in Task 9)**

```css
@media (hover: none) { .tile-drag { display: none !important; } }
```

**Step 4: Mark hidden posts visually** (relevant after Task 11)

```css
.tile.is-hidden {
  opacity: 0.35;
  filter: grayscale(0.7);
}
.tile.is-hidden::before {
  content: 'HIDDEN';
  position: absolute; inset: 0;
  display: grid; place-items: center;
  font-family: 'JetBrains Mono', monospace;
  font-size: 10px; letter-spacing: 0.2em;
  color: #fff; background: rgba(0,0,0,0.5);
  z-index: 5; pointer-events: none;
}
```

**Step 5: Visual QA**

On mobile viewport, tiles are edge-to-edge with 2px gutter. No hover overlay appears.

**Step 6: Commit**

```bash
git add index.html
git commit -m "refactor: mobile-friendly grid (tighter gaps, no hover affordances on touch)"
```

---

## Task 9 — Touch drag-reorder with long-press activation

**Files:**
- Modify: `index.html` — replace HTML5 drag-drop with a pointer-event + long-press system that works on both desktop and touch

**Why:** HTML5 drag-and-drop is broken on iOS Safari. Long-press activation is the IG/iOS convention and prevents accidental drags while scrolling.

**Step 1: Remove existing `draggable="true"` and `dragstart/dragend/dragover/dragleave/drop` wiring from `renderGrid()`**

**Step 2: Implement pointer-based long-press drag**

Add at module scope:

```js
const LONGPRESS_MS = 380;
let lpTimer = null;
let lpStart = null;
let dragTile = null;
let dragPostId = null;
let dragGhost = null;

function onTilePointerDown(e) {
  const tile = e.currentTarget;
  if (!tile.dataset.postId) return;
  lpStart = { x: e.clientX, y: e.clientY, tile };
  lpTimer = setTimeout(() => startDrag(e, tile), LONGPRESS_MS);
}
function onTilePointerMove(e) {
  if (!lpStart || dragTile) return;
  const dx = Math.abs(e.clientX - lpStart.x);
  const dy = Math.abs(e.clientY - lpStart.y);
  if (dx > 6 || dy > 6) { // user started scrolling — cancel long-press
    clearTimeout(lpTimer); lpTimer = null; lpStart = null;
  }
}
function onTilePointerUp() {
  clearTimeout(lpTimer); lpTimer = null; lpStart = null;
  if (dragTile) endDrag();
}

function startDrag(e, tile) {
  dragTile = tile;
  dragPostId = tile.dataset.postId;
  tile.classList.add('dragging');
  navigator.vibrate?.(10);
  // ghost
  const rect = tile.getBoundingClientRect();
  dragGhost = tile.cloneNode(true);
  dragGhost.classList.add('drag-ghost');
  dragGhost.style.width = rect.width + 'px';
  dragGhost.style.height = rect.height + 'px';
  dragGhost.style.left = rect.left + 'px';
  dragGhost.style.top = rect.top + 'px';
  document.body.appendChild(dragGhost);
  document.addEventListener('pointermove', onDragMove);
  document.addEventListener('pointerup', onDragEndDoc);
}
function onDragMove(e) {
  if (!dragGhost) return;
  const rect = dragTile.getBoundingClientRect();
  dragGhost.style.left = (e.clientX - rect.width/2) + 'px';
  dragGhost.style.top = (e.clientY - rect.height/2) + 'px';
  // find target tile under cursor
  dragGhost.style.pointerEvents = 'none';
  const el = document.elementFromPoint(e.clientX, e.clientY);
  const target = el?.closest('.tile');
  document.querySelectorAll('.drop-target').forEach(t => t.classList.remove('drop-target'));
  if (target && target !== dragTile && target.dataset.postId) {
    target.classList.add('drop-target');
  }
}
function onDragEndDoc(e) {
  document.removeEventListener('pointermove', onDragMove);
  document.removeEventListener('pointerup', onDragEndDoc);
  const target = document.querySelector('.drop-target');
  if (target && target.dataset.postId && target.dataset.postId !== dragPostId) {
    const from = state.posts.findIndex(p => p.id === dragPostId);
    const to = state.posts.findIndex(p => p.id === target.dataset.postId);
    if (from >= 0 && to >= 0) {
      const [moved] = state.posts.splice(from, 1);
      state.posts.splice(to, 0, moved);
      saveState();
      renderGrid();
    }
  }
  endDrag();
}
function endDrag() {
  dragTile?.classList.remove('dragging');
  dragGhost?.remove();
  document.querySelectorAll('.drop-target').forEach(t => t.classList.remove('drop-target'));
  dragTile = null; dragPostId = null; dragGhost = null;
}
```

Wire in `renderGrid()`:
```js
tile.addEventListener('pointerdown', onTilePointerDown);
tile.addEventListener('pointermove', onTilePointerMove);
tile.addEventListener('pointerup', onTilePointerUp);
tile.addEventListener('pointercancel', onTilePointerUp);
tile.style.touchAction = 'pan-y'; // allow vertical scrolling; drag activates via long-press
```

Also: the existing tile `click` handler must not fire if a drag happened. Track `dragHappened` boolean that's set in `startDrag` and checked (then cleared) in the click handler.

**Step 3: CSS for `.drag-ghost`**

```css
.drag-ghost {
  position: fixed !important;
  z-index: 200;
  pointer-events: none;
  transform: rotate(2deg) scale(1.05);
  box-shadow: 0 20px 40px rgba(0,0,0,0.6);
  opacity: 0.92;
}
.tile.dragging { opacity: 0.25; }
```

**Step 4: Visual QA**

Desktop: mouse-down on tile, hold ~400ms, drag → ghost follows cursor, drop target highlights, release reorders. Short click → modal opens.

Mobile (Chrome DevTools device mode with touch simulation): tap-hold tile → haptic feedback (vibrate), ghost appears, drag → reorders.

**Step 5: Commit**

```bash
git add index.html
git commit -m "feat: long-press drag-reorder for touch + mouse"
```

---

## Task 10 — Full-screen mobile post modal

**Files:**
- Modify: `index.html` — `.modal`, `.mp`, `.me` + responsive rules

**Current state:** Modal is desktop-first (two-column grid, 1200px max). Mobile gets a stacked version. We want mobile to feel native — full-screen, header + preview + scrollable editor.

**Step 1: Restructure modal layout so preview is on top and editor scrolls below (single column on mobile)**

```css
.modal {
  border-radius: 0;
  max-height: 100vh;
  height: 100vh;
  max-width: 100%;
  grid-template-columns: 1fr;
  grid-template-rows: auto auto 1fr auto;
}
.modal-backdrop { padding: 0; }
.mp { min-height: 0; aspect-ratio: 1 / 1; width: 100%; }
.me { border-left: none; border-top: 1px solid var(--ig-border); min-height: 0; }

@media (min-width: 900px) {
  .modal {
    border-radius: 12px;
    max-width: 1100px;
    max-height: 92vh;
    height: auto;
    grid-template-columns: minmax(400px, 1fr) 420px;
    grid-template-rows: 1fr;
  }
  .mp { aspect-ratio: auto; }
  .modal-backdrop { padding: 24px; }
}
```

**Step 2: Make the modal open animation slide up on mobile**

```css
@media (max-width: 899px) {
  .modal { animation: mobileUp 0.2s ease-out; }
  @keyframes mobileUp { from { transform: translateY(100%); } to { transform: translateY(0); } }
}
```

**Step 3: Visual QA**

On 390px viewport: modal covers full screen, square preview on top, editor scrolls below, close × is reachable with thumb.

**Step 4: Commit**

```bash
git add index.html
git commit -m "refactor: full-screen mobile modal (IG-native feel)"
```

---

## Task 11 — Strategy tab in modal (idea / script / why)

**Files:**
- Modify: `index.html` — add tabbed section inside `.me-body`

**Step 1: Add tab switcher in the editor**

Inside `.me-body`, above the fields:

```html
<div class="me-tabs">
  <button class="me-tab active" data-tab="post">Post</button>
  <button class="me-tab" data-tab="strategy">Strategy</button>
</div>
<div class="me-pane" data-pane="post">
  <!-- existing fields: type, pillar, slides, caption -->
</div>
<div class="me-pane" data-pane="strategy" hidden>
  <div class="field">
    <label>Idea</label>
    <textarea id="ideaInput" placeholder="The visual concept. What does this post LOOK like?"></textarea>
  </div>
  <div class="field" id="scriptField">
    <label>Script <span class="hint">(only shown for Reels)</span></label>
    <textarea id="scriptInput" placeholder="Hook 0-3s: …&#10;Beat 3-12s: …&#10;CTA: …"></textarea>
  </div>
  <div class="field">
    <label>Why it matters</label>
    <textarea id="whyInput" placeholder="What reaction does this trigger? Which pillar objective? Voice register?"></textarea>
  </div>
</div>
```

**Step 2: Styles**

```css
.me-tabs { display: flex; border-bottom: 1px solid var(--ig-border); margin: 0 -18px 16px; padding: 0 18px; }
.me-tab { flex: 1; background: transparent; border: none; color: var(--ig-muted); padding: 10px 0; font-size: 12px; letter-spacing: 0.12em; text-transform: uppercase; font-weight: 600; cursor: pointer; border-bottom: 1px solid transparent; margin-bottom: -1px; }
.me-tab.active { color: #fff; border-bottom-color: #fff; }
.me-pane[hidden] { display: none; }
.field .hint { color: var(--ig-muted); font-size: 10px; text-transform: none; letter-spacing: 0; }
```

**Step 3: Wire tab switching**

```js
document.querySelectorAll('.me-tab').forEach(t => t.addEventListener('click', () => {
  document.querySelectorAll('.me-tab').forEach(x => x.classList.toggle('active', x === t));
  document.querySelectorAll('.me-pane').forEach(p => p.hidden = p.dataset.pane !== t.dataset.tab);
}));
```

**Step 4: Bind strategy inputs to state**

In `renderModal()`:
```js
document.getElementById('ideaInput').value = post.idea || '';
document.getElementById('scriptInput').value = post.script || '';
document.getElementById('whyInput').value = post.why || '';
document.getElementById('scriptField').hidden = post.type !== 'reel';
```

And listeners:
```js
['ideaInput','scriptInput','whyInput'].forEach(id => {
  document.getElementById(id).addEventListener('input', (e) => {
    const p = currentPost(); if (!p) return;
    const key = id.replace('Input','');
    p[key] = e.target.value;
    saveState();
  });
});
```

**Step 5: When type changes to/from reel, toggle script field visibility**

In the type-button click handler, after `renderModal()` runs, the visibility will auto-adjust because we re-read `post.type` on render.

**Step 6: Visual QA**

Open a post → modal → tap "Strategy" tab → idea/why fields visible, script only shows for reels. Type updates, edit script, close modal, reopen → script persists.

**Step 7: Commit**

```bash
git add index.html
git commit -m "feat: Strategy tab in post modal (idea / script / why)"
```

---

## Task 12 — Hide / unhide posts + hidden posts view

**Files:**
- Modify: `index.html` — add hide toggle in modal footer, filter grid, implement hidden view

**Step 1: Add "Hide from grid" toggle in the modal footer**

Replace the existing footer:

```html
<div class="me-footer">
  <button class="footer-btn hide" id="hideBtn">Hide</button>
  <button class="footer-btn del" id="deletePostBtn">Delete</button>
  <button class="footer-btn done" id="doneBtn">Done</button>
</div>
```

Wire:

```js
document.getElementById('hideBtn').addEventListener('click', () => {
  const p = currentPost(); if (!p) return;
  p.hidden = !p.hidden;
  saveState();
  renderHideBtn();
  renderGrid();
});
function renderHideBtn() {
  const p = currentPost(); if (!p) return;
  document.getElementById('hideBtn').textContent = p.hidden ? 'Unhide' : 'Hide';
}
```

Call `renderHideBtn()` at the end of `renderModal()`.

**Step 2: Filter hidden posts from the main grid by default**

In `renderGrid()`:
```js
const visible = state.posts.filter(p => !p.hidden);
visible.forEach((post, idx) => { /* … */ });
```

But keep the **full** state.posts array intact in storage so unhide works.

Note: drag-reorder must now operate on `state.posts` indexes, not `visible` indexes. When computing `from`/`to`, look up by `postId` — we're already doing that, so it's fine.

**Step 3: Hidden posts view**

Add a toggle at the top of the grid area: "Show hidden (N)". Tapping it flips a view state flag:

```js
let showHidden = false;
function renderGrid() {
  const list = showHidden ? state.posts : state.posts.filter(p => !p.hidden);
  // …
  list.forEach((post, idx) => {
    const tile = /* … */;
    if (post.hidden) tile.classList.add('is-hidden');
    // …
  });
}
```

Add a toggle UI — a pill button between the tabs and the grid:

```html
<div class="grid-toolbar">
  <button class="toggle-hidden" id="toggleHiddenBtn">Show hidden · 0</button>
</div>
```

Update the button label to reflect `state.posts.filter(p => p.hidden).length`.

**Step 4: Wire the sheet "Hidden posts" action**

```js
function openHiddenView() {
  showHidden = true;
  renderGrid();
  document.getElementById('toggleHiddenBtn').textContent = `Hide hidden · ${hiddenCount()}`;
}
```

**Step 5: Visual QA**

Hide a post → it disappears from the grid → stats count decreases visually but state.posts.length untouched → `Show hidden` button shows count → tap it → hidden post appears grayscale with "HIDDEN" overlay → tap into it → "Unhide" in footer → back to visible.

**Step 6: Commit**

```bash
git add index.html
git commit -m "feat: hide/unhide posts from feed (grid-only soft delete)"
```

---

## Task 13 — Update Liren brand defaults (bio, highlights, name)

**Files:**
- Modify: `index.html` — `defaultState()` and the `<section class="highlights">` copy

**Step 1: Refresh the default profile copy to match Liren brand**

Current: `Liren Labs · 立人`, `AI agents yang bantu kamu berdiri lebih tinggi. Jakarta · Hangzhou · Bahasa-first. 🔗 lirenlabs.ai`. This is fine but audit for the reference IG pattern (emoji-led bullets, link at bottom).

Suggested new bio (keep or edit):
```
AI Agents · 立人
⚡ Bantu kamu berdiri lebih tinggi
✉️ hi@lirenlabs.ai
💎 Jakarta · Hangzhou · Bahasa-first
🔗 lirenlabs.ai
```

**Step 2: Update stat defaults to reflect aspirational / current numbers**

`posts` is computed from visible count. Leave followers/following defaults where user wants them (keep editable? — for now leave hard-coded at 247/18).

**Step 3: Highlights — optional polish**

Peter.visuals has About / 立人 / Meeting / Lamaran / LinkedIn / Founder. These are already in place. Leave as-is unless user asks for changes.

**Step 4: Visual QA** — Reset to default → confirm new bio renders.

**Step 5: Commit**

```bash
git add index.html
git commit -m "content: refresh Liren default bio"
```

---

## Task 14 — Mobile viewport + safe-area polish

**Files:**
- Modify: `index.html` — `<meta>` tags, safe-area env(), tap highlight removal

**Step 1: Add iOS web app meta tags in `<head>`**

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover, user-scalable=no">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
<meta name="theme-color" content="#000000">
<link rel="apple-touch-icon" href="data:image/svg+xml;utf8,<...logo SVG...>">
```

**Step 2: Safe-area padding**

```css
body {
  padding-top: env(safe-area-inset-top);
  padding-bottom: env(safe-area-inset-bottom);
}
.sheet { padding-bottom: max(12px, env(safe-area-inset-bottom)); }
```

**Step 3: Remove tap highlight and text-size-adjust**

```css
* { -webkit-tap-highlight-color: transparent; }
html { -webkit-text-size-adjust: 100%; }
```

**Step 4: Visual QA on iPhone (or DevTools device mode)**

Confirm status bar looks integrated (black), content doesn't hide behind notch, tap doesn't flash gray.

**Step 5: Commit**

```bash
git add index.html
git commit -m "chore: mobile viewport + safe-area polish"
```

---

## Task 15 — Sync source file, push, verify deploy

**Files:**
- Modify: `/Volumes/Extreme SSD/liren/liren-grid-planner.html` (source file outside the repo — keep in sync)

**Step 1: Copy deployed version back to source**

```bash
cp "/Volumes/Extreme SSD/liren/grid-planner-deploy/index.html" "/Volumes/Extreme SSD/liren/liren-grid-planner.html"
```

**Step 2: Push**

```bash
cd "/Volumes/Extreme SSD/liren/grid-planner-deploy"
git push origin main
```

**Step 3: Watch Vercel deploy**

```bash
vercel ls | head -5
```
Wait ~15s, then open https://grid-planner-deploy.vercel.app on phone.

**Step 4: Full mobile QA checklist**

- [ ] Profile picture is the blue L logo
- [ ] No top nav, no page meta, no pillar legend
- [ ] Profile layout matches IG mobile (avatar left, stats centered, full-width CTAs)
- [ ] ⋯ menu opens action sheet
- [ ] Action sheet has: New, Hidden, Export, Import, Reset
- [ ] Tap a tile → full-screen modal slides up
- [ ] Modal has Post / Strategy tabs
- [ ] Strategy tab shows idea + why always, script only for Reels
- [ ] Long-press a tile → haptic buzz → drag-reorder works
- [ ] Short tap on tile → opens modal (doesn't trigger drag)
- [ ] Hide button in modal footer removes from grid
- [ ] Hidden posts accessible via Show hidden pill or action sheet
- [ ] localStorage v2 key populated, v1 migrated

**Step 5: Final commit if any fixes**

---

## Out of scope (explicitly)

- Image compression for uploaded photos (IG doesn't — browsers handle it; we accept the localStorage quota hit)
- Multi-grid support (one brand profile for now)
- Sharing/exporting the grid as PNG
- Reels video playback (thumbnails only — script field compensates)
- Authentication (single-user, single-device; export JSON for backup)
- Server-side persistence (localStorage is the source of truth)

## Rollback path

If any task breaks the deployed site:

```bash
cd "/Volumes/Extreme SSD/liren/grid-planner-deploy"
git revert HEAD
git push
```

Vercel auto-redeploys the revert in ~15s.

---

**Plan complete.** 15 tasks, each scoped to 5-20 minutes. Every change is a separate commit so rollback is granular.
