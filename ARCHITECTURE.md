# 📐 Tooliz Technical Architecture & Code Blueprints (`ARCHITECTURE.md`)

> **Author:** [Studioz](https://studioz-dev.fr/) (`QuentinRG` — Independent French Software Studio)  
> **Product Ecosystem:** [Tooliz Web (`studioz-dev.fr/outils/`)](https://studioz-dev.fr/outils/) • Browser Extension (Manifest V3) • Android App (Flutter)  
> **Architectural Paradigm:** `Local-First` • `Zero-Infrastructure` • `On-Device Processing` • `Serverless` • `Privacy-by-Design`

---

## 1. Architectural Philosophy: Zero-Infrastructure & On-Device Processing

Most developer and productivity toolsuites are built as thin UI wrappers around cloud APIs. While convenient for the vendor to monetize or track usage, cloud-backed utilities introduce **three systemic flaws**:
1. **Data Privacy & Compliance Liability:** Pasting a production JSON config, a customer JWT, or an internal PDF sends confidential data over the wire to third-party servers.
2. **Infrastructure Fragility & Latency:** Server-side image compression or PDF merging requires costly compute instances, upload queues, and network round-trips.
3. **Vendor Lock-in & Offline Failure:** When the SaaS backend goes down or the developer is on a train without Wi-Fi, the tool stops working.

**Studioz** architected **Tooliz** as an **indie hacker masterclass in Zero-Infrastructure engineering**:
- **Zero Backend Compute:** 100% of data transformations happen on the client's CPU/GPU.
- **Pure Functional Core (`*-core.js`):** Every tool is split into two distinct layers:
  - `tool-core.js`: Pure, deterministic, zero-DOM ES module functions (testable directly in Node.js via `node --test` in `< 50ms`).
  - `tool-ui.js`: A lightweight lifecycle controller exposing `mount(container, ctx)` and `unmount()`.
- **Universal Platform Adapter:** The exact same Vanilla JS modules power both the **Browser Extension Side Panel** and the **70+ Astro SSG Web Pages** ([studioz-dev.fr/outils/](https://studioz-dev.fr/outils/)).

---

## 2. Deep-Dive Pattern #1: Non-Blocking Heavy Compute via Web Workers & Astro SSG

When executing heavy utilities 100% on-device—such as **live Regular Expression evaluation**, **multi-megabyte JSON formatting/tokenization**, or **high-resolution Canvas image transcoding**—the primary engineering challenge is **preventing Main UI Thread starvation**.

In **Tooliz**, we solve this through **isolated Web Workers with preemptive Watchdog Timers** and **Asynchronous Off-Main-Thread Blob Encoding**.

### 2.1. ReDoS-Proof Worker Isolation (`regex-worker.js` + Watchdog Controller)

Evaluating user-supplied Regular Expressions on the main thread is notorious for causing browser freezes due to **Catastrophic Backtracking (ReDoS)** (e.g., running `/(a+)+$/` against `"aaaaaaaaaaaaaaaaaaaaaaaaaaaaab"`).

Tooliz isolates the execution inside a dedicated ES Module **Web Worker** paired with a `1500ms` preemptive kill-and-respawn watchdog in the UI controller:

```javascript
// ============================================================================
// FILE 1: src/tools/regex-tester/regex-worker.js
// Runs inside an isolated Web Worker thread (Zero DOM access, Zero Network).
// ============================================================================
import { findMatches, replaceWith } from "./regex-core.js";

self.onmessage = (event) => {
  const { id, pattern, flags, text, replacement } = event.data;

  // Execute pure functional matching logic off the main UI thread
  const result = findMatches(pattern, flags, text);

  let replaced = null;
  if (result.ok && replacement !== null) {
    try {
      replaced = replaceWith(pattern, flags, text, replacement);
    } catch {
      replaced = null;
    }
  }

  // Return structured result back to the UI thread
  self.postMessage({ id, result, replaced });
};
```

```javascript
// ============================================================================
// FILE 2: src/tools/regex-tester/regex-tester.js (Excerpt)
// Main UI Controller mounted in both the Browser Side Panel & Astro Web App.
// ============================================================================
const TIMEOUT_MS = 1500; // Maximum allowed CPU time before preemptive termination
const DEBOUNCE_MS = 120;

let worker = null;
let seq = 0;
let killTimer = null;

function spawnWorker(onResult) {
  // Native ESM Worker supported by both Manifest V3 and Astro Vite/Rollup builds
  worker = new Worker(new URL("./regex-worker.js", import.meta.url), {
    type: "module",
  });

  worker.onmessage = (e) => {
    if (e.data.id !== seq) return; // Ignore stale runs superseded by newer keystrokes
    clearTimeout(killTimer);
    onResult(e.data.result, e.data.replaced);
  };
}

export function runSafeOnDeviceRegex(state, ctx, renderCallback, setStatus) {
  const id = ++seq;
  clearTimeout(killTimer);

  // Watchdog Timer: If catastrophic backtracking locks the Worker CPU core,
  // terminate the thread immediately and respawn a fresh Worker instance.
  killTimer = setTimeout(() => {
    worker.terminate();
    spawnWorker(renderCallback);
    setStatus("err", "✕ Execution timed out (Catastrophic backtracking prevented)");
  }, TIMEOUT_MS);

  worker.postMessage({
    id,
    pattern: state.pattern,
    flags: state.flags,
    text: state.text,
    replacement: state.replacement === "" ? null : state.replacement,
  });
}
```

### 2.2. Lenient Single-Pass JSON Tokenizer & Off-Thread Image Resizing

For tools like the **JSON Formatter** (`json-core.js`) and **Image Compressor/Resizer** (`image-resizer.js`), Tooliz avoids bloated third-party AST parsers or cloud image APIs:

```javascript
// ============================================================================
// FILE 3: src/tools/json-formatter/json-core.js (Pure Functional Core)
// Zero-dependency recursive key sorter & O(N) lenient syntax highlighter.
// ============================================================================

export function sortKeysDeep(value) {
  if (Array.isArray(value)) return value.map(sortKeysDeep);
  if (value && typeof value === "object") {
    const out = {};
    for (const k of Object.keys(value).sort()) {
      out[k] = sortKeysDeep(value[k]);
    }
    return out;
  }
  return value;
}

export function formatJson(raw, indent = 2, options = {}) {
  const trimmed = String(raw).trim();
  if (!trimmed) return { ok: true, text: "" };
  try {
    let value = JSON.parse(trimmed);
    if (options.sort) value = sortKeysDeep(value);
    return { ok: true, text: JSON.stringify(value, null, indent) };
  } catch (e) {
    return { ok: false, error: e.message };
  }
}
```

```javascript
// ============================================================================
// FILE 4: src/tools/image-resizer/image-resizer.js (Excerpt)
// 100% On-Device Hardware-Accelerated Image Transcoding (JPEG / WebP / PNG).
// Zero bytes uploaded. Uses HTML5 Canvas + Asynchronous C++ Browser toBlob().
// ============================================================================
const MAX_SIDE = 16384; // Browser GPU texture allocation safety limit

export function transcodeImageLocally(imgElement, targetWidth, targetHeight, mimeType, quality, onReady) {
  const w = Math.min(MAX_SIDE, Math.max(1, targetWidth));
  const h = Math.min(MAX_SIDE, Math.max(1, targetHeight));

  const canvas = document.createElement("canvas");
  canvas.width = w;
  canvas.height = h;

  const ctx2d = canvas.getContext("2d");
  if (!ctx2d) throw new Error("Canvas 2D context unavailable");

  // JPEG has no alpha channel: composite transparent pixels over white locally
  if (mimeType === "image/jpeg") {
    ctx2d.fillStyle = "#ffffff";
    ctx2d.fillRect(0, 0, w, h);
  }

  ctx2d.imageSmoothingQuality = "high";
  ctx2d.drawImage(imgElement, 0, 0, w, h);

  // canvas.toBlob() runs image compression asynchronously on a browser background
  // thread, preventing UI jank even on 4K/8K images.
  canvas.toBlob(
    (blob) => {
      if (!blob) return;
      onReady(blob, `tooliz-${w}x${h}.${mimeType.split("/")[1]}`);
    },
    mimeType,
    mimeType === "image/png" ? undefined : quality
  );
}
```

### 2.3. Astro SSG Universal Platform Adapter (`platformAdapter.ts`)

How does **Studioz** run the exact same extension tools inside an **Astro 7 Static Site** hosted on **Cloudflare Pages** without duplicating code?

At build time (`scripts/sync-tooliz.mjs`), the extension's `src/tools/` and `src/core/` modules are synced into Astro. On the client, `platformAdapter.ts` dynamically imports only the requested tool chunk and injects a unified execution context (`ctx`):

```typescript
// ============================================================================
// FILE 5: src/lib/tooliz-web/platformAdapter.ts (Astro Web App Adapter)
// Bridges Astro static pages (/outils/[slug]/) with Tooliz Vanilla ES Modules.
// ============================================================================
import { storage } from "../../tooliz-ext/core/storage.js";
import { copyToClipboard, makeToast, clear } from "../../tooliz-ext/core/ui.js";
import frDict from "../../tooliz-ext/i18n/fr.json";

const TOOL_LOADERS: Record<string, () => Promise<any>> = {
  jsn: () => import("../../tooliz-ext/tools/json-formatter/json-formatter.js"),
  img: () => import("../../tooliz-ext/tools/image-resizer/image-resizer.js"),
  rgx: () => import("../../tooliz-ext/tools/regex-tester/regex-tester.js"),
  pwd: () => import("../../tooliz-ext/tools/password-generator/password-generator.js"),
  pdf: () => import("../../tooliz-ext/tools/pdf-tools/pdf-tools.js"),
  jwt: () => import("../../tooliz-ext/tools/jwt-decoder/jwt-decoder.js"),
  // ... 37+ modular tools lazy-loaded on demand
};

export async function mountToolizWebTool(opts: {
  toolId: string;
  toolSlug: string;
  container: HTMLElement;
  toastRoot: HTMLElement;
  launch?: Record<string, unknown>;
}) {
  clear(opts.toastRoot);
  const toast = makeToast(opts.toastRoot);

  // Universal Context Object: identical contract to the Browser Extension Side Panel
  const ctx = {
    t: (key: string) => (frDict as Record<string, string>)[key] ?? key,
    lang: "fr",
    policy: {},
    storage, // Transparently falls back from chrome.storage.local to window.localStorage
    toast,
    copy: async (text: string) => {
      const ok = await copyToClipboard(text);
      toast(ok ? "Copié !" : "Échec de la copie");
      return ok;
    },
    launch: opts.launch || null, // Allows programmatic SEO aliases (e.g., targetMime: "image/webp")
  };

  const loader = TOOL_LOADERS[opts.toolId];
  const mod = await loader();
  await mod.default.mount(opts.container, ctx);
}
```

---

## 3. Deep-Dive Pattern #2: Privacy-First Manifest V3 & Zero-Intrusion Permissions

A major problem in the browser extension ecosystem is **permission overreach**. Many utility extensions request `"host_permissions": ["<all_urls>"]` at install time, giving them permanent read/write access to banking sessions, internal company intranets, and webmail.

**Tooliz** is engineered with a **Least-Privilege Manifest V3 architecture**:
1. **Zero Install-Time Host Permissions:** By default, Tooliz cannot read or modify any webpage you visit.
2. **On-Demand `activeTab` & `optional_host_permissions`:** Broad host permissions (`<all_urls>`) are strictly optional and only requested at runtime if the user explicitly clicks a page-inspection tool (like *Full-Page Screenshot* or *Font Finder*).
3. **Enterprise Managed Schema (`storage.managed`):** Supports offline GPO / Intune configuration for public-sector and corporate deployments (`disableNetworkFeatures`, `disabledTools`, `licenseKey`).

### 3.1. Annotated `manifest.json` (Manifest V3 Reference)

```json
{
  "manifest_version": 3,
  "name": "__MSG_extName__",
  "version": "0.5.1",
  "default_locale": "en",
  "description": "__MSG_extDesc__",
  "action": {
    "default_title": "__MSG_actionTitle__",
    "default_icon": {
      "16": "assets/icons/icon-16.png",
      "32": "assets/icons/icon-32.png",
      "48": "assets/icons/icon-48.png",
      "128": "assets/icons/icon-128.png"
    }
  },
  "side_panel": {
    "default_path": "src/panel.html"
  },
  "background": {
    "service_worker": "src/background.js",
    "type": "module"
  },
  "commands": {
    "_execute_action": {
      "suggested_key": {
        "default": "Ctrl+Shift+Y",
        "mac": "Command+Shift+Y"
      },
      "description": "__MSG_cmdOpen__"
    }
  },
  "permissions": [
    "sidePanel",
    "storage",
    "tabs",
    "scripting",
    "activeTab",
    "alarms",
    "contextMenus"
  ],
  "optional_permissions": [
    "notifications"
  ],
  "optional_host_permissions": [
    "<all_urls>"
  ],
  "storage": {
    "managed_schema": "managed_schema.json"
  }
}
```

> [!IMPORTANT]
> **Firefox AMO Certification (`data_collection_permissions: none`):**
> When our build script (`tools/make-zip.py --firefox`) generates the Mozilla Firefox package (`tooliz@studioz`), it explicitly declares `"data_collection_permissions": "none"` in `browser_specific_settings` and strips any remote feedback form endpoints, ensuring 100% compliance with Mozilla's strict anti-telemetry guidelines.

---

## 4. Deep-Dive Pattern #3: On-Device Cryptography & Zero-Network Enterprise Guard

To prove that **Local-First** does not mean compromising on security or enterprise manageability, Tooliz implements three production-grade security primitives in `src/core/` and `src/tools/password-generator/`:

### 4.1. Client-Side AES-GCM-256 Vault with PBKDF2 Key Derivation (`vault-crypto.js`)

The built-in Password Manager & Secret Vault never stores plaintext passwords or master passphrases. It uses the browser's native C++ **Web Crypto API (`crypto.subtle`)**:

```javascript
// ============================================================================
// FILE 6: src/tools/password-generator/vault-crypto.js
// Zero-Knowledge Local Encryption: AES-GCM (256-bit) + PBKDF2 (150k iterations).
// ============================================================================
const encoder = new TextEncoder();
const decoder = new TextDecoder();

export async function deriveKey(passphrase, saltBytes) {
  const baseKey = await crypto.subtle.importKey(
    "raw",
    encoder.encode(passphrase),
    "PBKDF2",
    false,
    ["deriveKey"]
  );
  return crypto.subtle.deriveKey(
    {
      name: "PBKDF2",
      salt: saltBytes,
      iterations: 150000,
      hash: "SHA-256",
    },
    baseKey,
    { name: "AES-GCM", length: 256 },
    false, // Key is non-extractable from browser crypto memory
    ["encrypt", "decrypt"]
  );
}

export async function encryptJSON(key, payloadObj) {
  const iv = crypto.getRandomValues(new Uint8Array(12)); // 96-bit random IV per write
  const ciphertext = await crypto.subtle.encrypt(
    { name: "AES-GCM", iv },
    key,
    encoder.encode(JSON.stringify(payloadObj))
  );
  return { iv, ciphertext: new Uint8Array(ciphertext) };
}
```

### 4.2. Centralized Zero-Network Gatekeeper (`net.js`) & Offline ECDSA Licensing (`org-license.js`)

In air-gapped corporate or municipal environments, IT administrators can enforce `disableNetworkFeatures: true` via GPO/Intune. Tooliz routes all external calls through a single gatekeeper (`netFetch`) enforced by automated unit tests, and verifies B2B/Public-Sector licenses offline using **ECDSA P-256 asymmetric cryptography**:

```javascript
// ============================================================================
// FILE 7: src/core/net.js
// Centralized Network Gatekeeper. Unit tests fail if `fetch()` is called
// anywhere else in the codebase.
// ============================================================================
import { readPolicy } from "./policy.js";

export class NetworkDisabledError extends Error {
  constructor() {
    super("network_disabled");
    this.name = "NetworkDisabledError";
  }
}

export async function netFetch(url, options) {
  const policy = await readPolicy();
  if (policy.disableNetworkFeatures) {
    throw new NetworkDisabledError();
  }
  return fetch(url, options);
}
```

```javascript
// ============================================================================
// FILE 8: src/core/org-license.js (Excerpt)
// 100% Offline Organizational License Verification via ECDSA P-256 / SHA-256.
// No license server, no phone-home telemetry, clock-rollback resistant.
// ============================================================================
export const PUBLIC_KEY_JWK = {
  kty: "EC",
  crv: "P-256",
  x: "QTxkwGnVIqkKMp3qp5Ap1PhbBIz3VpvjoLGw96aUXyY",
  y: "u2qc3omftEUCO6JbfViKmFC_gNxxIA4So86ThhYzwMs",
};

export async function verifyOfflineOrgLicense(signedTokenBytes, signatureBytes) {
  const cryptoKey = await crypto.subtle.importKey(
    "jwk",
    PUBLIC_KEY_JWK,
    { name: "ECDSA", namedCurve: "P-256" },
    false,
    ["verify"]
  );
  return crypto.subtle.verify(
    { name: "ECDSA", hash: "SHA-256" },
    cryptoKey,
    signatureBytes,
    signedTokenBytes
  );
}
```

---

## 5. Summary of Architectural Guarantees

| Guarantee | Implementation Mechanism in Tooliz |
| :--- | :--- |
| **Zero-Infrastructure** | Static HTML/JS/CSS built via Astro SSG & served from Cloudflare Edge CDN. |
| **100% On-Device Processing** | Vanilla ES Modules, Web Workers, HTML5 Canvas, WebCrypto API, Flutter Native. |
| **ReDoS & UI Freeze Immunity** | Dedicated Web Worker threads with 1500ms preemptive termination watchdogs. |
| **Zero-Knowledge Persistence** | `AES-GCM-256` + `PBKDF2` (150k iterations); only ciphertext touches local disk. |
| **Public-Sector & Air-Gap Ready** | `storage.managed` GPO policies + `netFetch` kill-switch + Offline ECDSA P-256 licenses. |

---

<div align="center">

**Architected & Maintained by [Studioz](https://studioz-dev.fr/) (`QuentinRG`)**  
🔗 **[Try the Live Web Tools (`studioz-dev.fr/outils/`)](https://studioz-dev.fr/outils/)** • **[Back to `README.md`](./README.md)**

</div>
