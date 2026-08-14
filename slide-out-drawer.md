# Slide-Out Drawer / Inquiry Panel — Build Guide

How to build a right-side slide-out drawer like the OnlySport "Grails & Retros"
inquiry panel: a card's button opens a full-height panel that slides in from the
edge over a dimmed backdrop, carries context about *which* item was clicked,
holds a form, and shows an inline success state.

The **mechanics are framework-agnostic** — vanilla HTML/CSS/JS below works on any
site. §8 maps the pieces back to how the original did it on Shopify (Liquid +
`{% form %}`), which you can skip if you're not on Shopify.

---

## 1. Anatomy

```
[ trigger button ]  (data-drawer-item="Air Jordan 1 …")   ← lives on each card
     │ click
     ▼
[ backdrop ]   position:fixed, full-screen, dim + blur, fades in   (z 9998)
[ panel ]      position:fixed, top/right, slides in via transform  (z 9999)
   └─ close button (×)
   └─ context strip ("Selected: <item name>")
   └─ form  ──submit──►  success state (swap form → checkmark panel)
```

Two always-present fixed elements (backdrop + panel) that start hidden and get
an `.is-open` class toggled on them. Triggers are any number of buttons that
carry the item's identity in a `data-` attribute.

---

## 2. The three moving parts

1. **Backdrop** — a fixed full-screen overlay. Hidden via `opacity:0` +
   `pointer-events:none`; shown by adding a class. Clicking it closes the drawer.
2. **Panel** — a fixed, full-height box pinned to the right edge, pushed off
   screen with `transform: translateX(100%)`; slid in by setting
   `translateX(0)`. `transform` (not `right`/`left`) so it's GPU-composited and
   smooth.
3. **Triggers** — buttons anywhere on the page carrying `data-drawer-item="…"`.
   One delegated listener opens the panel and copies that value into the panel.

---

## 3. Minimal HTML

Put the backdrop + panel **once** near the end of `<body>` (not inside a card,
or `position:fixed` can be trapped by an ancestor `transform`/`filter`).

```html
<!-- Triggers: one per card -->
<button class="drawer-open" data-drawer-item="Air Jordan 1 'Chicago' 1985">
  Request a quote →
</button>

<!-- The drawer (once per page) -->
<div class="drawer-backdrop" id="drawerBackdrop"></div>

<aside class="drawer" id="drawer" role="dialog" aria-modal="false"
       aria-hidden="true" aria-label="Product inquiry">
  <div class="drawer-inner">
    <button class="drawer-close" id="drawerClose" aria-label="Close">✕</button>

    <p class="drawer-eyebrow">Inquiry</p>
    <h2 class="drawer-title">Interested in this piece?</h2>
    <p class="drawer-subtitle">Leave your details and we'll reach out personally.</p>

    <!-- Context strip: which item this inquiry is about -->
    <div class="drawer-item">
      <span class="drawer-item-label">Selected item</span>
      <span class="drawer-item-name" id="drawerItemName">—</span>
    </div>

    <!-- Form -->
    <div id="drawerFormWrap">
      <form class="drawer-form" id="drawerForm">
        <input type="hidden" name="item" id="drawerItemField" value="">
        <input name="name"  placeholder="Full name" required>
        <input name="phone" placeholder="Phone"     required type="tel">
        <input name="email" placeholder="Email"     required type="email">
        <textarea name="message" placeholder="Anything you'd like us to know?"></textarea>
        <button type="submit">Send inquiry →</button>
      </form>
    </div>

    <!-- Success state (hidden until submit) -->
    <div class="drawer-success" id="drawerSuccess" hidden>
      <div class="drawer-success-check">✓</div>
      <p class="drawer-success-item" id="drawerSuccessItem"></p>
      <h3>Inquiry sent.</h3>
      <p>We'll get back to you personally, shortly.</p>
      <button id="drawerSuccessClose">Back to the collection</button>
    </div>
  </div>
</aside>
```

---

## 4. CSS — the slide + the backdrop + scroll lock

```css
/* Backdrop */
.drawer-backdrop {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.65);
  -webkit-backdrop-filter: blur(3px);
  backdrop-filter: blur(3px);
  z-index: 9998;
  opacity: 0;
  pointer-events: none;                 /* un-clickable while hidden */
  transition: opacity 0.35s ease;
}
.drawer-backdrop.is-open { opacity: 1; pointer-events: all; }

/* Panel */
.drawer {
  position: fixed;
  top: 0;
  right: 0;
  width: min(520px, 100vw);             /* capped on desktop, full-width on phones */
  height: 100dvh;                        /* dvh = correct height under mobile URL bars */
  background: #fff;
  z-index: 9999;
  transform: translateX(100%);          /* parked off-screen to the right */
  transition: transform 0.42s cubic-bezier(0.4, 0, 0.2, 1);
  overflow-y: auto;                      /* the PANEL scrolls, not the page */
  overflow-x: hidden;
  -webkit-overflow-scrolling: touch;
}
.drawer.is-open {
  transform: translateX(0);
  box-shadow: -30px 0 80px rgba(0,0,0,0.25);
}

.drawer-inner { padding: 52px 44px 44px; min-height: 100%; display: flex; flex-direction: column; }

/* Success state toggles from the form */
.drawer-success { display: none; flex-direction: column; align-items: center; text-align: center; flex: 1; }
.drawer-success.is-shown { display: flex; }

/* Mobile */
@media (max-width: 640px) {
  .drawer-inner { padding: 32px 16px 24px; }
}
```

Slide from the **left** instead: pin `left:0`, park with `translateX(-100%)`.
Slide up from the **bottom** (mobile sheet): pin `left:0; right:0; bottom:0;
height:auto; max-height:90dvh; border-radius:16px 16px 0 0;` and park with
`translateY(100%)`.

---

## 5. JS — open, close, dismiss, context

```js
(function () {
  const drawer   = document.getElementById('drawer');
  const backdrop = document.getElementById('drawerBackdrop');
  const itemName = document.getElementById('drawerItemName');   // display
  const itemFld  = document.getElementById('drawerItemField');  // hidden form field

  function openDrawer(itemLabel) {
    itemName.textContent = itemLabel || '—';   // textContent, never innerHTML (untrusted)
    if (itemFld) itemFld.value = itemLabel || '';

    drawer.classList.add('is-open');
    backdrop.classList.add('is-open');
    document.body.style.overflow = 'hidden';   // lock page scroll behind the drawer
    drawer.setAttribute('aria-modal', 'true');
    drawer.removeAttribute('aria-hidden');

    // Focus the first field AFTER the slide finishes (focusing mid-transition
    // can make some browsers jump-scroll the panel).
    setTimeout(() => {
      const first = drawer.querySelector('input, textarea, button');
      if (first) first.focus();
    }, 450);   // ≈ the transform transition duration
  }

  function closeDrawer() {
    drawer.classList.remove('is-open');
    backdrop.classList.remove('is-open');
    document.body.style.overflow = '';         // restore page scroll
    drawer.setAttribute('aria-modal', 'false');
    drawer.setAttribute('aria-hidden', 'true');
  }

  // Triggers — delegated so it works for any number of cards, incl. ones
  // added to the DOM later (filtering, pagination, infinite scroll).
  document.addEventListener('click', (e) => {
    const btn = e.target.closest('.drawer-open');
    if (btn) openDrawer(btn.dataset.drawerItem);
  });

  // Dismissals: close button, backdrop click, Escape.
  document.getElementById('drawerClose').addEventListener('click', closeDrawer);
  backdrop.addEventListener('click', closeDrawer);
  document.addEventListener('keydown', (e) => {
    if (e.key === 'Escape' && drawer.classList.contains('is-open')) closeDrawer();
  });
})();
```

That's the entire drawer. Everything below is refinements.

---

## 6. Passing context into the drawer

The whole point of "which card did I click": the trigger carries the identity,
`openDrawer()` distributes it to (a) the visible "Selected item" strip and (b) a
hidden form field so it's submitted with the inquiry. Keep the identifier on the
trigger as a `data-` attribute — don't try to infer it from DOM position.

If you need more than a label (id, price, image), stash them as multiple
`data-*` attributes and read `btn.dataset.*`, or a single
`data-item='{"id":…,"name":…}'` JSON blob parsed with `JSON.parse`.

**Security:** item labels/any user-or-catalog text go in via `textContent` /
`.value`, never `innerHTML`.

---

## 7. The success state — two variants

### 7a. AJAX submit (recommended off-Shopify)
Intercept submit, POST via `fetch`, then swap form → success **without a page
reload**:

```js
document.getElementById('drawerForm').addEventListener('submit', async (e) => {
  e.preventDefault();
  const data = new FormData(e.target);
  await fetch('/your-endpoint', { method: 'POST', body: data });   // + error handling
  document.getElementById('drawerSuccessItem').textContent = itemFld.value;
  document.getElementById('drawerFormWrap').style.display = 'none';
  document.getElementById('drawerSuccess').classList.add('is-shown');
});
document.getElementById('drawerSuccessClose').addEventListener('click', closeDrawer);
```

### 7b. Full-page-reload submit (what Shopify's `{% form %}` forces)
A classic form POST reloads the page, which **loses the open drawer and the
clicked item**. Bridge the reload with `sessionStorage`:

1. Before submit, save the item name: `sessionStorage.setItem('drawer_item', name)`.
2. On the server-rendered "posted successfully" branch, print a tiny script that
   sets `sessionStorage.setItem('drawer_success','true')`.
3. On page load, if `drawer_success` is set: read back the item, show the success
   panel, and re-open the drawer immediately (no animation needed) — then clear
   the flag.

**Always wrap storage access** — `sessionStorage`/`localStorage` throw in Safari
Private Mode and some in-app webviews:

```js
const safeGet = k => { try { return sessionStorage.getItem(k); } catch { return null; } };
const safeSet = (k,v) => { try { sessionStorage.setItem(k,v); } catch {} };
```

---

## 8. Shopify / Liquid mapping (skip if not on Shopify)

How the OnlySport version implemented the above:

- **One panel per section instance**, id-scoped with `{{ section.id }}` so two
  copies on a page don't collide (`os-inquiry-panel-{{ section.id }}`, etc.).
- **Rendered once at section root**, outside the card grid:
  `{%- render 'os-grails-inquiry-panel', section: section -%}` after the
  `</section>`. Keeps it clear of any card ancestor with a `transform` that would
  trap `position:fixed`.
- **The form is `{% form 'contact' %}`** → posts to Shopify's contact endpoint
  and **reloads**, so it uses the §7b sessionStorage bridge (`os_grails_success_…`
  + `os_grails_shoe_…`). Hidden fields (`contact[subject]`, a hidden
  `contact[body]`) are assembled on submit so the merchant's notification email
  reads "Produs solicitat: <item>\n\nMesaj: …".
- **Trigger contract:** each card renders a button `class="os-open-inquiry"` with
  `data-shoe="<product name>"`; the panel JS does
  `document.querySelectorAll('.os-open-inquiry').forEach(... openPanel(btn.dataset.shoe))`.
- **Item data source:** cards come from a metaobject list
  (`shop.metafields.custom.grails_list.value`), but the drawer doesn't care where
  cards come from — it only reads `data-shoe`.
- **All UI strings via locale keys** (`{{ 'os.grails.panel_title' | t }}` …) and
  passed to JS through a hidden `data-*` element
  (`#os-grails-i18n-{{ section.id }}`), not string-interpolated into the script.
- **GDPR consent**: a required checkbox plus a hidden field stamped with an
  ISO timestamp on submit, for a defensible consent record.

---

## 9. Gotchas checklist

- **Scroll lock:** set `document.body.style.overflow = 'hidden'` on open, restore
  `''` on close — otherwise the page scrolls behind the drawer. (For iron-clad
  iOS lock you also need `position:fixed` on `body` + restore scroll position;
  the simple overflow lock is usually enough.)
- **`position:fixed` trap:** a fixed element is positioned relative to the
  nearest ancestor that has a `transform`, `filter`, or `will-change` — not the
  viewport. If your drawer renders inside a card that animates on hover
  (`transform: translateY`), it'll be trapped. Render the drawer at body/section
  root, not inside a repeating card.
- **Use `100dvh`, not `100vh`**, for the panel height — `vh` includes the space
  under a mobile browser's URL bar, so the bottom of a `100vh` drawer gets cut
  off on phones.
- **Focus timing:** autofocus the first field *after* the slide transition
  (`setTimeout ≈ duration`); focusing mid-transition can trigger a jump-scroll.
- **Transition with `transform`, not `left`/`right`** — compositor-friendly and
  smooth; animating layout properties jank.
- **`pointer-events:none` on the hidden backdrop** so it doesn't eat clicks on
  the page while closed (opacity:0 alone still intercepts clicks).
- **Delegate the trigger listener** (`document.addEventListener('click', … closest('.drawer-open'))`)
  so cards added later (filter/paginate) still open the drawer.
- **Escape + backdrop-click + explicit close button** — offer all three; users
  reach for different ones.
- **`aria-modal`/`aria-hidden` toggling** and a `role="dialog"` + `aria-label`
  give screen readers the modal semantics; flip them in open/close.
```

