# <index.html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover" />
<meta name="theme-color" content="#f6f3ee" />
<title>Mide's Collection</title>
<link rel="preconnect" href="https://fonts.googleapis.com" />
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
<link href="https://fonts.googleapis.com/css2?family=Cormorant+Garamond:wght@500;600&family=Outfit:wght@400;500;600&display=swap" rel="stylesheet" />
<script src="https://cdn.tailwindcss.com"></script>
<script>
tailwind.config = {
  theme: {
    extend: {
      colors: {
        bg: "#f6f3ee",
        surface: "#fffefb",
        surface2: "#ece8e1",
        fg: "#1a1b19",
        muted: "#6b6a66",
        subtle: "#9a9892",
        danger: "#8a3d35",
      },
      fontFamily: {
        display: ['"Cormorant Garamond"', "Palatino", "serif"],
        sans: ["Outfit", "Segoe UI", "system-ui", "sans-serif"],
      },
    },
  },
};
</script>
<style>
  html, body { margin: 0; min-height: 100%; background: #f6f3ee; color: #1a1b19; font-family: Outfit, system-ui, sans-serif; }
  button:not(:disabled) { cursor: pointer; }
  .face { background:
    radial-gradient(circle at 50% 42%, #f6f3ee 0 28%, transparent 29%),
    conic-gradient(from 210deg, #1a1b19, #3a3936, #1a1b19);
  }
</style>
</head>
<body>
<div id="root"></div>
<script type="module">
import { createElement, useMemo, useState, useEffect } from "https://esm.sh/react@19";
import { createRoot } from "https://esm.sh/react-dom@19/client";
import htm from "https://esm.sh/htm@3.1.1";
const html = htm.bind(createElement);
const h = createElement;

const CATEGORIES = ["All","Dress","Chronograph","GMT","Diver","Moonphase","Skeleton","Ladies","Field"];
const WATCHES = [
  { id:"noir-40", name:"Noir 40", calibre:"MC-40", category:"Dress", tone:"#1a1b19", price:1850000, was:2150000, sold:214, rating:4.8, blurb:"Charcoal case, milk enamel dial. The quiet house watch." },
  { id:"milk-chrono", name:"Milk Chronograph", calibre:"MC-C72", category:"Chronograph", tone:"#d9d3c7", price:2450000, was:2780000, sold:96, rating:4.7, blurb:"Three-register chronograph on a milk-white ground." },
  { id:"sahara-gmt", name:"Sahara GMT", calibre:"MC-G12", category:"GMT", tone:"#b08968", price:2150000, sold:131, rating:4.6, blurb:"Second time zone for Lagos–London hops." },
  { id:"lagos-diver", name:"Lagos Diver", calibre:"MC-D200", category:"Diver", tone:"#243447", price:1650000, was:1890000, sold:188, rating:4.5, blurb:"200 m, matte charcoal, milk chapter ring." },
  { id:"ivory-moon", name:"Ivory Moonphase", calibre:"MC-M28", category:"Moonphase", tone:"#e8e0d0", price:3200000, sold:41, rating:4.9, blurb:"Slim dress piece with a night-sky window." },
  { id:"charcoal-skel", name:"Charcoal Skeleton", calibre:"MC-S01", category:"Skeleton", tone:"#2c2c2a", price:2850000, sold:57, rating:4.7, blurb:"Open movement, milk chapter, charcoal strap." },
  { id:"pearl-28", name:"Pearl 28", calibre:"MC-P28", category:"Ladies", tone:"#f3ece3", price:1450000, was:1620000, sold:162, rating:4.8, blurb:"Petite case, milk mother-of-pearl, mesh bracelet." },
  { id:"field-38", name:"Field 38", calibre:"MC-F38", category:"Field", tone:"#4a463f", price:980000, sold:276, rating:4.4, blurb:"High-contrast milk dial. Built to be worn every day." },
];
const byId = (id) => WATCHES.find((w) => w.id === id);
const naira = (n) => new Intl.NumberFormat("en-NG", { style:"currency", currency:"NGN", maximumFractionDigits:0 }).format(n);

const enc = new TextEncoder();
function toB64(buf) { return btoa(String.fromCharCode(...new Uint8Array(buf))); }
function fromB64(s) {
  const bin = atob(s);
  const out = new Uint8Array(bin.length);
  for (let i = 0; i < bin.length; i++) out[i] = bin.charCodeAt(i);
  return out;
}
async function hashPassword(password, saltB64) {
  const salt = saltB64 ? fromB64(saltB64) : crypto.getRandomValues(new Uint8Array(16));
  const key = await crypto.subtle.importKey("raw", enc.encode(password), "PBKDF2", false, ["deriveBits"]);
  const bits = await crypto.subtle.deriveBits({ name:"PBKDF2", hash:"SHA-256", salt, iterations:120000 }, key, 256);
  return { hash: toB64(bits), salt: toB64(salt.buffer) };
}
const newId = (p) => p + "_" + crypto.randomUUID().slice(0, 8);

const KEY = "mides.v1";
function load() {
  try { return JSON.parse(localStorage.getItem(KEY) || "null"); } catch { return null; }
}
const empty = {
  users: [], sessionId: null, cart: [], orders: [],
  payout: { bankName:"", accountName:"", accountNumber:"", cryptoNetwork:"USDT TRC-20", cryptoAddress:"" },
};
let db = { ...empty, ...(load() || {}) };
const listeners = new Set();
function save() {
  localStorage.setItem(KEY, JSON.stringify({
    users: db.users, sessionId: db.sessionId, cart: db.cart, orders: db.orders, payout: db.payout,
  }));
  listeners.forEach((fn) => fn());
}
function useShop() {
  const [, tick] = useState(0);
  useEffect(() => {
    const fn = () => tick((n) => n + 1);
    listeners.add(fn);
    return () => listeners.delete(fn);
  }, []);
  return db;
}

async function tryGooglePay(amountNgn) {
  if (typeof PaymentRequest === "undefined") return "unavailable";
  try {
    const request = new PaymentRequest(
      [{
        supportedMethods: "https://google.com/pay",
        data: {
          environment: "TEST", apiVersion: 2, apiVersionMinor: 0,
          merchantInfo: { merchantName: "Mide's Collection" },
          allowedPaymentMethods: [{
            type: "CARD",
            parameters: { allowedAuthMethods: ["PAN_ONLY","CRYPTOGRAM_3DS"], allowedCardNetworks: ["VISA","MASTERCARD"] },
            tokenizationSpecification: { type:"PAYMENT_GATEWAY", parameters: { gateway:"example", gatewayMerchantId:"mides-collection" } },
          }],
        },
      }],
      { total: { label: "Mide's Collection", amount: { currency:"NGN", value: String(amountNgn) } } },
    );
    if (!(await request.canMakePayment())) return "unavailable";
    const result = await request.show();
    await result.complete("success");
    return "paid";
  } catch (err) {
    if (err && err.name === "AbortError") return "cancel";
    return "unavailable";
  }
}

function Face({ tone, className = "" }) {
  return html`<div className=\( {"face relative overflow-hidden " + className} style= \){{ backgroundColor: tone }}>
    <div className="absolute inset-[18%] rounded-full bg-[#f6f3ee] shadow-inner"></div>
    <div className="absolute left-1/2 top-[22%] h-[28%] w-[1.5px] -translate-x-1/2 bg-[#1a1b19]"></div>
    <div className="absolute left-[28%] top-1/2 h-[1.5px] w-[24%] bg-[#1a1b19]"></div>
  </div>`;
}

function Icon({ d, className = "w-5 h-5" }) {
  return html`<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8" className=\( {className}><path d= \){d} /></svg>`;
}
const I = {
  home: "M3 10.5 12 3l9 7.5V21a1 1 0 0 1-1 1h-5v-7H9v7H4a1 1 0 0 1-1-1z",
  bag: "M6 7h12l1 14H5L6 7zm3 0a3 3 0 0 1 6 0",
  user: "M12 12a4 4 0 1 0-4-4 4 4 0 0 0 4 4zm-8 9a8 8 0 0 1 16 0",
  search: "M11 19a8 8 0 1 1 8-8 8 8 0 0 1-8 8zm10 2-4.3-4.3",
  x: "M6 6l12 12M18 6 6 18",
  plus: "M12 5v14M5 12h14",
  minus: "M5 12h14",
  lock: "M7 11V8a5 5 0 0 1 10 0v3M6 11h12v10H6z",
  check: "M5 12l5 5L19 7",
  copy: "M8 8h10v12H8zM6 16H4V4h12v2",
};

function App() {
  const s = useShop();
  const [tab, setTab] = useState("home");
  const [query, setQuery] = useState("");
  const [searchOpen, setSearchOpen] = useState(false);
  const [category, setCategory] = useState("All");
  const [openId, setOpenId] = useState(null);
  const [checkout, setCheckout] = useState(false);
  const me = s.users.find((u) => u.id === s.sessionId) || null;
  const count = s.cart.reduce((n, l) => n + l.qty, 0);
  const total = s.cart.reduce((n, l) => n + (byId(l.watchId)?.price || 0) * l.qty, 0);
  const watches = useMemo(() => {
    const q = query.trim().toLowerCase();
    return WATCHES.filter((w) => {
      if (category !== "All" && w.category !== category) return false;
      if (!q) return true;
      return (w.name + " " + w.calibre + " " + w.category + " " + w.blurb).toLowerCase().includes(q);
    });
  }, [query, category]);
  const open = openId ? byId(openId) : null;

  return html`<div className="mx-auto min-h-dvh max-w-lg bg-bg pb-24 text-fg">
    <header className="sticky top-0 z-20 border-b border-black/10 bg-bg/95 px-4 pb-3 pt-3 backdrop-blur-sm">
      <div className="flex items-center justify-between gap-3">
        <div>
          <p className="font-display text-2xl leading-none">Mide's Collection</p>
          <p className="mt-1 text-xs text-muted">Luxury wrist watches</p>
        </div>
        <button type="button" className="flex h-11 w-11 items-center justify-center rounded-full bg-surface2" onClick=${() => setSearchOpen((v) => !v)} aria-label="Search">
          <\( {Icon} d= \){I.search} />
        </button>
      </div>
      \( {searchOpen && html`<input autoFocus value= \){query} onInput=${(e) => setQuery(e.target.value)} placeholder="Search watches, calibres…" className="mt-3 h-11 w-full rounded-xl bg-surface px-4 text-sm outline-none ring-1 ring-black/10" />`}
    </header>

    ${tab === "home" && html`<div className="px-4 pb-6">
      <section className="mt-4 overflow-hidden rounded-3xl bg-fg px-5 py-6 text-bg">
        <p className="text-xs uppercase tracking-[0.18em] text-bg/70">Atelier drop</p>
        <p className="mt-2 font-display text-3xl leading-none">Milk white. Charcoal. Time kept quietly.</p>
        <p className="mt-3 max-w-[36ch] text-sm text-bg/75">Digital accessories — luxury wrist watches. Sign in to buy. Paid by Google Pay, bank, or crypto.</p>
      </section>
      <div className="-mx-4 mt-4 flex gap-2 overflow-x-auto px-4 pb-1">
        \( {CATEGORIES.map((c) => html`<button key= \){c} type="button" onClick=\( {() => setCategory(c)} className= \){"h-9 shrink-0 rounded-full px-3.5 text-sm " + (category === c ? "bg-fg text-bg" : "bg-surface2 text-fg")}>${c}</button>`)}
      </div>
      <p className="mt-5 text-xs font-medium uppercase tracking-[0.16em] text-muted">Marketplace</p>
      ${watches.length === 0
        ? html`<p className="mt-4 text-sm text-muted">No watches match that search.</p>`
        : html`<ul className="mt-3 grid grid-cols-2 gap-3">
            \( {watches.map((w) => html`<li key= \){w.id}>
              <button type="button" onClick=${() => setOpenId(w.id)} className="w-full overflow-hidden rounded-2xl bg-surface text-left ring-1 ring-black/10">
                <\( {Face} tone= \){w.tone} className="aspect-[3/4] w-full" />
                <div className="px-3 py-2.5">
                  <p className="truncate text-sm font-medium">${w.name}</p>
                  <p className="text-xs text-muted">${w.sold} sold · ${w.rating}</p>
                  <p className="mt-1 text-sm tabular-nums">${naira(w.price)}</p>
                  \( {w.was && html`<p className="text-xs text-subtle line-through tabular-nums"> \){naira(w.was)}</p>`}
                </div>
              </button>
            </li>`)}
          </ul>`}
    </div>`}

    \( {tab === "cart" && html`< \){CartView} cart=\( {s.cart} total= \){total} onShop=\( {() => setTab("home")} onCheckout= \){() => { if (!me) setTab("profile"); else setCheckout(true); }} />`}
    \( {tab === "profile" && html`< \){ProfileView} me=${me} />`}
    \( {open && html`< \){ProductSheet} watch=\( {open} onClose= \){() => setOpenId(null)} />`}
    \( {checkout && html`< \){CheckoutSheet} me=\( {me} payout= \){s.payout} total=\( {total} onClose= \){() => setCheckout(false)} onNeedAccount=${() => { setCheckout(false); setTab("profile"); }} />`}

    <nav className="fixed inset-x-0 bottom-0 z-30 mx-auto max-w-lg border-t border-black/10 bg-surface pb-[env(safe-area-inset-bottom)]">
      <div className="grid grid-cols-3">
        <\( {NavBtn} d= \){I.home} label="Home" active=\( {tab==="home"} onClick= \){() => { setSearchOpen(false); setTab("home"); }} />
        <\( {NavBtn} d= \){I.bag} label="Cart" badge=\( {count} active= \){tab==="cart"} onClick=${() => { setSearchOpen(false); setTab("cart"); }} />
        <\( {NavBtn} d= \){I.user} label="Profile" active=\( {tab==="profile"} onClick= \){() => { setSearchOpen(false); setTab("profile"); }} />
      </div>
    </nav>
  </div>`;
}

function NavBtn({ d, label, active, onClick, badge }) {
  return html`<button type="button" onClick=\( {onClick} className= \){"relative flex h-14 flex-col items-center justify-center gap-0.5 text-[11px] " + (active ? "text-fg" : "text-muted")}>
    <span className="relative">
      <\( {Icon} d= \){d} />
      \( {!!badge && html`<span className="absolute -right-2.5 -top-1 min-w-4 rounded-full bg-fg px-1 text-[10px] leading-4 text-bg"> \){badge}</span>`}
    </span>
    ${label}
  </button>`;
}

function ProductSheet({ watch, onClose }) {
  return html`<\( {Sheet} onClose= \){onClose}>
    <\( {Face} tone= \){watch.tone} className="aspect-[3/4] w-full" />
    <div className="px-4 py-4">
      <p className="text-xs uppercase tracking-[0.16em] text-muted">${watch.category} · ${watch.calibre}</p>
      <h2 className="mt-1 font-display text-3xl">${watch.name}</h2>
      <p className="mt-2 text-sm text-muted">${watch.blurb}</p>
      <p className="mt-3 text-lg tabular-nums">${naira(watch.price)}</p>
      <button type="button" className="mt-4 flex h-12 w-full items-center justify-center rounded-xl bg-fg text-bg" onClick=${() => { addToCart(watch.id); onClose(); }}>Add to cart</button>
    </div>
  </${Sheet}>`;
}

function CartView({ cart, total, onShop, onCheckout }) {
  if (cart.length === 0) {
    return html`<div className="px-6 py-16 text-center">
      <p className="font-display text-3xl">Cart is empty</p>
      <p className="mt-2 text-sm text-muted">Pick a watch from the floor.</p>
      <button type="button" className="mt-5 h-11 rounded-xl bg-fg px-5 text-bg" onClick=${onShop}>Browse</button>
    </div>`;
  }
  return html`<div className="px-4 py-4">
    <ul className="space-y-3">
      ${cart.map((line) => {
        const w = byId(line.watchId);
        if (!w) return null;
        return html`<li key=${line.watchId} className="flex gap-3 rounded-2xl bg-surface p-2 ring-1 ring-black/5">
          <\( {Face} tone= \){w.tone} className="h-20 w-20 rounded-xl" />
          <div className="min-w-0 flex-1">
            <p className="truncate text-sm font-medium">${w.name}</p>
            <p className="text-sm tabular-nums">${naira(w.price)}</p>
            <div className="mt-2 flex items-center gap-2">
              <button type="button" className="flex h-8 w-8 items-center justify-center rounded-lg bg-surface2" onClick=\( {() => setQty(w.id, line.qty - 1)}>< \){Icon} d=${I.minus} className="w-3.5 h-3.5" /></button>
              <span className="w-6 text-center text-sm tabular-nums">${line.qty}</span>
              <button type="button" className="flex h-8 w-8 items-center justify-center rounded-lg bg-surface2" onClick=\( {() => setQty(w.id, line.qty + 1)}>< \){Icon} d=${I.plus} className="w-3.5 h-3.5" /></button>
            </div>
          </div>
        </li>`;
      })}
    </ul>
    <div className="mt-5 flex items-center justify-between">
      <p className="text-sm text-muted">Total</p>
      <p className="text-lg tabular-nums">${naira(total)}</p>
    </div>
    <button type="button" className="mt-4 h-12 w-full rounded-xl bg-fg text-bg" onClick=${onCheckout}>Checkout</button>
    <p className="mt-2 text-center text-xs text-muted">Sign in is required. We never take your card number.</p>
  </div>`;
}

function CheckoutSheet({ me, payout, total, onClose, onNeedAccount }) {
  const [method, setMethod] = useState("card");
  const [note, setNote] = useState("");
  const [busy, setBusy] = useState(false);
  const [done, setDone] = useState(null);
  const [hint, setHint] = useState(null);

  if (!me) {
    return html`<\( {Sheet} onClose= \){onClose}>
      <div className="px-4 py-6">
        <h2 className="font-display text-3xl">Sign in to buy</h2>
        <p className="mt-2 text-sm text-muted">Purchases sit on your account so the atelier can match payment.</p>
        <button type="button" className="mt-4 h-12 w-full rounded-xl bg-fg text-bg" onClick=${onNeedAccount}>Go to profile</button>
      </div>
    </${Sheet}>`;
  }
  if (done) {
    return html`<\( {Sheet} onClose= \){onClose}>
      <div className="px-4 py-6">
        <\( {Icon} d= \){I.check} className="w-8 h-8" />
        <h2 className="mt-3 font-display text-3xl">Order ${done}</h2>
        <p className="mt-2 text-sm text-muted">Pending until the atelier marks it received. Keep your transfer note or transaction hash.</p>
        <button type="button" className="mt-4 h-12 w-full rounded-xl bg-fg text-bg" onClick=${onClose}>Done</button>
      </div>
    </${Sheet}>`;
  }

  async function submit() {
    setHint(null);
    if (method === "google") {
      setBusy(true);
      const result = await tryGooglePay(total);
      setBusy(false);
      if (result === "paid") { const o = placeOrder("google", "Google Pay (device)"); if (o) setDone(o.id); return; }
      if (result === "cancel") return;
      setHint("Google Pay is not available on this device. Use card transfer or crypto.");
      return;
    }
    if (method === "card" && (!payout.bankName || !payout.accountNumber)) { setHint("The atelier has not posted a bank account yet."); return; }
    if (method === "crypto" && !payout.cryptoAddress) { setHint("The atelier has not posted a wallet yet."); return; }
    if (method === "crypto" && note.trim().length < 6) { setHint("Paste the transaction hash after you send."); return; }
    const order = placeOrder(method, note);
    if (order) setDone(order.id);
  }

  return html`<\( {Sheet} onClose= \){onClose}>
    <div className="px-4 py-4">
      <p className="flex items-center gap-1.5 text-xs uppercase tracking-[0.16em] text-muted"><\( {Icon} d= \){I.lock} className="w-3.5 h-3.5" /> Secure checkout</p>
      <h2 className="mt-1 font-display text-3xl">${naira(total)}</h2>
      <div className="mt-4 grid grid-cols-3 gap-2">
        ${[["google","Google Pay"],["card","Card"],["crypto","Crypto"]].map(([id, label]) => html`
          <button key=\( {id} type="button" onClick= \){() => setMethod(id)} className=\( {"flex h-20 items-center justify-center rounded-xl text-xs " + (method===id ? "bg-fg text-bg" : "bg-surface2")}> \){label}</button>
        `)}
      </div>
      ${method === "google" && html`<p className="mt-4 text-sm text-muted">Opens Google Pay on this device. Live merchant settlement is not wired — if it fails, pay by bank or crypto.</p>`}
      \( {method === "card" && html`< \){PayoutBlock} title="Bank transfer" lines=${[["Bank", payout.bankName],["Account name", payout.accountName],["Account number", payout.accountNumber]]} hint="We never ask for your card number. Transfer from your bank app, then confirm." />`}
      \( {method === "crypto" && html`< \){PayoutBlock} title="Crypto" lines=${[["Network", payout.cryptoNetwork],["Wallet", payout.cryptoAddress]]} hint="Send the exact total, then paste the transaction hash." />`}
      \( {method !== "google" && html`<input value= \){note} onInput=\( {(e) => setNote(e.target.value)} placeholder= \){method==="crypto" ? "Transaction hash" : "Transfer reference (optional)"} className="mt-3 h-11 w-full rounded-xl bg-surface2 px-3 font-mono text-xs outline-none" />`}
      \( {hint && html`<p className="mt-3 text-sm text-danger"> \){hint}</p>`}
      <button type="button" disabled=\( {busy} className="mt-4 h-12 w-full rounded-xl bg-fg text-bg disabled:opacity-40" onClick= \){() => submit()}>${busy ? "Talking to Google Pay…" : "Confirm payment"}</button>
    </div>
  </${Sheet}>`;
}

function PayoutBlock({ title, lines, hint }) {
  return html`<div className="mt-4 rounded-2xl bg-surface2 px-3 py-3">
    <p className="text-xs uppercase tracking-[0.16em] text-muted">${title}</p>
    \( {lines.map(([k, v]) => ht
