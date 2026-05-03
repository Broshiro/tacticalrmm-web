# Broshiro RMM — Frontend

> Vue/Quasar SPA frontend for [Broshiro/tacticalrmm](https://github.com/Broshiro/tacticalrmm).  
> This is the UI half of the fork — for full installation instructions see the **backend repo README**.

---

## What This Fork Changes

This repo is a fork of [`amidaware/tacticalrmm-web`](https://github.com/amidaware/tacticalrmm-web) with the following modifications to the **Take Control** (remote session) view:

| Change | Description |
|--------|-------------|
| **Clipboard Sync** | Toolbar button that reads the local clipboard and pushes it to the remote machine via MeshCentral postMessage |
| **Send Keys dialog** | Toolbar button that opens a text input; content is pushed to the remote clipboard and auto-pasted |
| **Vaultwarden credential panel** | Slide-in right panel showing client-scoped credentials from Vaultwarden; each field has copy and send-as-keystrokes actions |
| **Credential search** | Real-time filter box in the panel to find credentials by name or username |
| **Password reveal toggle** | Show/hide password inline without copying |

All other views and components are unchanged from upstream.

---

## Files Changed

| File | Change |
|------|--------|
| `src/views/TakeControl.vue` | Complete rewrite of the remote session view with new toolbar and credential panel |
| `src/api/agents.js` | Added `fetchAgentVaultCreds(agent_id)` API call |

---

## Building for Production

### Prerequisites

- Node.js 18 or 20 (`node --version`)
- npm 9+ (`npm --version`)
- Quasar CLI v2 (`npm install -g @quasar/cli`)

### Install and Build

```bash
git clone https://github.com/Broshiro/tacticalrmm-web.git
cd tacticalrmm-web
git checkout develop

npm install
quasar build
```

Build output is placed in `dist/spa/`.

### Deploy to TRMM Server

```bash
# Run these commands on your TRMM server (as root or sudo)
rm -rf /var/www/rmm/dist/*
cp -r dist/spa/. /var/www/rmm/dist/
chown -R www-data:www-data /var/www/rmm/dist/
systemctl reload nginx
```

---

## Local Development

```bash
# Clone and install
git clone https://github.com/Broshiro/tacticalrmm-web.git
cd tacticalrmm-web
npm install

# Create a local .env pointing at your TRMM API
cp .env.example .env
# Edit .env:
#   VITE_APP_API=https://api.yourdomain.com
#   (or http://localhost:8000 for local backend dev)

# Start dev server
quasar dev
# Opens at http://localhost:9000
```

---

## Remote Session Feature Details

### Clipboard Sync

The **Sync Clipboard** button in the Take Control toolbar reads your local browser clipboard (`navigator.clipboard.readText()`) and sends it to the MeshCentral iframe using the `postMessage` API:

```js
iframe.contentWindow.postMessage({ type: 'clipboard', data: text }, '*')
iframe.contentWindow.postMessage({ action: 'setClipboard', data: text }, '*')
```

Both formats are sent to maintain compatibility with different MeshCentral versions.  After clicking the button, press `Ctrl+V` in the remote window to paste.

> **Browser permission required:** Chrome and Edge will request `clipboard-read` permission on first use.  Click Allow when prompted.

### Send as Keystrokes

The **Send Keys** button opens a dialog where you type or paste text.  On submit:

1. Text is written to the local clipboard as a fallback.
2. Text is pushed to the remote clipboard via `postMessage`.
3. A `pasteClipboard` postMessage is sent to trigger auto-paste in the remote session.

If auto-paste does not fire (MeshCentral version-dependent), a notification appears prompting you to press `Ctrl+V` manually in the remote window.

Keyboard shortcut: `Ctrl+Enter` submits the dialog.

### Credential Panel

The **Credentials** button toggles a 320 px right-side panel.  When opened for the first time, it calls:

```
GET /agents/{agent_id}/vaultcreds/
```

The backend (see `Broshiro/tacticalrmm`) authenticates to Vaultwarden, fetches all login items, filters to those whose name contains the agent's client name, and returns:

```json
[
  {
    "id": "uuid",
    "name": "Acme Corp - Domain Admin",
    "username": "administrator",
    "password": "secret",
    "notes": ""
  }
]
```

Each item is shown as a collapsible list entry.  Username and password rows each have:
- **Copy** — copies the value to the local clipboard with a toast notification
- **Keyboard** — sends the value to the remote machine as keystrokes
- **Eye** (password only) — reveals the password inline

The panel narrows the iframe by 320 px; MeshCentral reflows the remote desktop to fit.

---

## Pulling Upstream Changes

```bash
git remote add upstream https://github.com/amidaware/tacticalrmm-web.git
git fetch upstream
git merge upstream/develop
# Resolve conflicts if any, then push:
git push origin develop
```

After merging, rebuild and redeploy the frontend.

---

## Contributing

1. Fork `Broshiro/tacticalrmm-web`.
2. Branch from `develop`: `git checkout -b feature/your-feature`
3. Make changes and run `npm run lint`.
4. Open a PR to `Broshiro/tacticalrmm-web:develop`.

---

## Related

- [Broshiro/tacticalrmm](https://github.com/Broshiro/tacticalrmm) — Backend (Django REST API)
- [amidaware/tacticalrmm](https://github.com/amidaware/tacticalrmm) — Upstream backend
- [amidaware/tacticalrmm-web](https://github.com/amidaware/tacticalrmm-web) — Upstream frontend
- [TacticalRMM Documentation](https://docs.tacticalrmm.com) — Upstream install docs

---

## License

Inherited from the upstream project.  See [LICENSE](./LICENSE).
