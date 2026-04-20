# Building a Browser Automation Tool That Actually Works Against Modern Anti-Bot Portals

Recently my analyst team at work requested for a desktop application that could automate batch downloading from a web portal without any captcha solvers, they also wanted to have control over through out the process — the kind of tool that sounds trivial until you hit every layer of modern browser security at once. The final architecture is clean and works reliably, but the path there involved deliberately ruling out several popular approaches before writing the first meaningful line of code.

This post is about that process: what are the problems that makes simple download a challenge, how I analyzed the problem, why I chose Chrome DevTools Protocol over the obvious alternatives, the specific gotchas I encountered in the Python/WebView layer, and what the working solution looks like.

## What the app needed to do

- Store multiple portal credentials (encrypted) in a local SQLite database
- Launch the portal in the user's real browser and let them log in manually
- Fetch the report list via the portal's own API, display it in a searchable, filterable grid
- Batch-download selected reports with human-like pacing
- Auto-extract zip and gz archives in parallel while downloads continue
- Keep a persistent download history

Standard CRUD and automation requirements. The interesting part was the browser control layer.

## Upfront analysis: understanding what the portal actually does

Before touching any automation code, I spent time with DevTools' Network tab watching exactly what the portal's own JavaScript does when a user downloads a report. This is the step that shapes everything downstream, so I'm going to be specific about what I found.

**The report list**: a single `GET /schedule/api/dashboard/reports/view` call returns all reports as JSON — `reportId`, `reportName`, `status`, `rowCount`, and more. No DOM scraping needed. No pagination in the API response, even though the UI shows paginated tables. The data is all there in one call.

**The download flow**: clicking the download button triggers a `POST /download/api/download/file` with `{"reportId": "..."}`. This endpoint doesn't return the file — it returns a JSON envelope containing a pre-authorized Google Cloud Storage signed URL:

```json
{
  "signedUrl": "https://storage.googleapis.com/bucket/path/file.csv?X-Goog-Credential=...&X-Goog-Expires=60...",
  "templateName": "Report Name",
  "templateId": "..."
}
```

The portal's JS then does a plain `GET` on that signed URL (with `credentials: 'omit'` since GCS doesn't want portal cookies) to retrieve the actual file, then triggers a download via a blob URL. This is a completely standard pattern for cloud-hosted file delivery — temporary pre-signed URL, short expiry, no auth headers required on the file GET. Once I saw this in the Network tab, the download implementation wrote itself:

```javascript
// Step 1: get the signed URL from the portal API
const r = await fetch('/download/api/download/file', {
    method: 'POST',
    credentials: 'include',
    body: JSON.stringify({reportId})
});
const {signedUrl} = await r.json();

// Step 2: fetch the real file directly from GCS
const file = await fetch(signedUrl, {credentials: 'omit'});
const blob = await file.blob();

// Step 3: extract real filename from URL path, trigger download
const filename = decodeURIComponent(new URL(signedUrl).pathname.split('/').pop());
const a = Object.assign(document.createElement('a'), {href: URL.createObjectURL(blob), download: filename});
document.body.appendChild(a); a.click(); document.body.removeChild(a);
```

Knowing this upfront also clarified the file monitoring strategy: since downloads go through the browser's download manager, I'd watch the configured download folder for new files whose size stabilizes (the absence of `.crdownload` partials and two consecutive equal size readings one second apart).

With the API behavior understood, the next question was how to control the browser.

## Why I chose CDP over Selenium and embedded browsers

### Selenium was the wrong tool from the start

Selenium works by injecting a WebDriver object into the browser and communicating through it. Modern anti-bot systems fingerprint this — not just `navigator.webdriver`, but how event dispatch timing differs from human input, subtle DOM mutations Selenium makes, and behavioral patterns that don't match real users. The portal in question used a "Press & Hold" challenge that kept regenerating regardless of what I tried, which is the signature of a system that has already decided it's talking to a bot before the user interaction even starts.

Beyond detection: Selenium needs ChromeDriver version-matched to the installed browser, auto-updates can break your automation overnight, and you're maintaining a parallel browser installation. For a personal tool, this maintenance surface isn't worth it.

### PyWebView with an embedded portal was never going to work

The idea of embedding the portal inside the app via PyWebView is architecturally appealing — one window, everything contained. The problem is that a fresh WebView2 instance is not the user's browser. It has no accumulated cookies, no stored trusted certificates, no installed extensions, no browser history. Portals with any security posture can silently behave differently toward a clean-room browser than toward a browser with a legitimate user profile behind it.

Additionally, PyWebView and Tkinter cannot cleanly coexist. Both frameworks want to run their event loop on the main thread, which produces either threading exceptions or deeply unpleasant workarounds. The `html=` parameter to `webview.create_window()` also triggers a known EdgeChromium accessibility API bug (the accessibility object walker hits infinite recursion on a page with no real URL, printing hundreds of `Empty.Empty.Empty...` lines to the console). The correct approach is to always use `url="file:///path/to/ui.html"`.

### Cookie extraction is blocked by ABE

Reading session cookies directly from Edge's SQLite cookie database is a technique that appears in many tutorials. As of mid-2024 it no longer works on browsers that have enabled Application-Bound Encryption (ABE). Chromium's ABE encrypts cookie values with a key that only the browser process can unseal via a privileged system call. Attempting external decryption produces `Failed to decrypt cookie: invalid tag`. This is intentional — ABE was designed specifically to block the class of attack that reads cookies from outside the browser process.

### Why stealth frameworks didn't fit the problem

There's a well-developed ecosystem of libraries specifically built to make automated browsers harder to detect: `puppeteer-extra-plugin-stealth`, `nodriver`, `Patchright`, `Selenium Driverless`, and others. These patch canvas fingerprinting, WebGL signatures, audio context, TLS/JA3 handshakes, and `navigator` properties to make a headless browser look more like a real user's browser. They're genuinely effective at what they do.

The reason none of them were the right choice here comes down to what problem they actually solve. Stealth frameworks fix *fingerprinting* — they help you look like a real browser to a site that's trying to distinguish automated traffic from human traffic. That's the right tool when you're scraping a public site that's trying to block bots.

This portal has a different posture. The barrier isn't a site trying to decide if I'm a bot or a human making the same request. The barrier is a private authenticated portal that requires a legitimate user session — one tied to a real orgnaizational managed account, established through a login flow that may involve MFA, managed browser policies, or environment-specific trust signals. A stealth-patched Chromium instance still starts with a blank profile, no stored session, no trusted organizational context. It might make it past a fingerprint check only to fail authentication for entirely separate reasons.

The question to ask before reaching for a stealth library is: *what specifically is being protected?* If it's "this site blocks headless Chromium," stealth patches help. If it's "this site requires you to actually log in," no amount of fingerprint spoofing substitutes for a genuine session.

### Why cloud browser services (Bright Data, Browserbase, ScraperAPI, ZenRows) were irrelevant

These services solve a real problem — scraping public content at scale, with managed proxy rotation, residential IP pools, CAPTCHA solving, and anti-bot bypass. They're well built for that use case.

The use case here is the opposite in almost every dimension. The content isn't public — it's private data behind an authenticated session. Scale isn't the concern — it's one user's own reports, not millions of pages across a domain. IP reputation isn't the barrier — the portal doesn't care what IP address a legitimate user connects from. And these are cloud services, meaning the session and its credentials would have to be transmitted externally, which introduces security concerns that aren't worth taking on for personal tooling.

When I mapped the actual requirements — authenticated private portal, single user, desktop app, credentials stay local — cloud browser services dropped off the list immediately. They're the right tool for a completely different problem shape.

### CDP: control the real browser without fingerprinting it

The Chrome DevTools Protocol is the side-channel Chromium exposes for tooling — DevTools itself uses it, as do Puppeteer and Playwright. The critical difference from Selenium is that CDP doesn't inject anything into the page. From the page's JavaScript perspective, nothing unusual is happening. There's no `navigator.webdriver` flag, no synthetic DOM mutations, no timing anomalies.

The approach: launch the user's real Edge with their actual profile directory and `--remote-debugging-port=<random>`. This produces an Edge window that is completely indistinguishable from Edge opened normally — same profile, same cookies, same extensions, same trusted certificates. The user logs in manually. Then SZilla connects to the debug port's WebSocket interface and issues `Runtime.evaluate` commands that run JavaScript inside the authenticated tab.

```python
# Find a free port, launch Edge with the user's real profile
port = find_free_port()
subprocess.Popen([
    edge_path,
    f"--remote-debugging-port={port}",
    f"--user-data-dir={user_data_dir}",
    f"--profile-directory=Default",
    "--remote-allow-origins=*",
    landing_url
])

# Wait for debug port to be ready, then connect via WebSocket
tab = get_active_tab(port)
ws = create_connection(tab['webSocketDebuggerUrl'])
```

Once connected, any JavaScript that runs in the page has full access to the tab's session — cookies, headers, CORS policies are all handled natively because the fetch runs in the correct origin context. There's no authentication to replicate, no headers to forge, no cookies to extract and re-inject.

## PyWebView as a UI frame — the right use of the tool

Having settled on CDP for browser control, PyWebView still serves a genuine purpose: it gives the app a native-feeling window that renders the HTML/CSS/JS UI. The UI is a single `ui.html` file loaded via `file://` URL. Python exposes a class (`Api`) as `js_api` to pywebview, making every method callable from JavaScript as `await window.pywebview.api.method_name(args)`.

This separation is deliberate and valuable. The frontend holds all UI state; the Python backend is stateless and functional. When the browser control layer needed to be rewritten (and it did, twice), the HTML/CSS/JS frontend was untouched across every iteration. The UI stack — vanilla HTML, CSS custom properties, no frameworks — is genuinely more durable than any framework would have been for a tool of this scope.

The full tab layout: Run (credential select, browser launch, report grid with split completed/in-progress sections, batch download controls, live log), Configure (processes and credentials with full CRUD), History (searchable download log), Settings (database management).

## Human-like pacing as a design requirement

Even with CDP producing no detectable automation fingerprint, there's still the matter of request rate. Automated batch operations that fire 30 requests in 10 seconds look unusual in server logs regardless of how they're made. The download loop uses randomized delays drawn from configurable ranges:

```python
DELAY_BEFORE_FIRST = (1.5, 3.0)       # before starting the batch
DELAY_BETWEEN_DOWNLOADS = (3.0, 8.0)  # between each item
DELAY_AFTER_DOWNLOAD_CLICK = (1.0, 2.5)  # after triggering each download
```

The randomization matters — uniform delays are easier to detect algorithmically than human-variable ones. These are implemented in `downloader.py` via `random.uniform()` and applied at every automation step, not just downloads.

## Parallel unzip without blocking downloads

Archive extraction runs in a background `threading.Queue` worker, completely decoupled from the download loop. When the download monitor detects a new stable file with a `.zip` or `.gz` extension, it's pushed onto the unzip queue and the download loop immediately moves to the next item. The unzip worker handles the queue independently. For large archives this saves significant time in a batch of 20+ files.

```python
class UnzipWorker:
    def enqueue(self, filepath, history_id, status_callback):
        self.queue.put((filepath, history_id, status_callback))

    def _run(self):  # runs in a daemon thread
        while self.running:
            filepath, hid, scb = self.queue.get(timeout=2)
            # extract, then call scb(hid, "Completed")
```

## Things worth knowing if you're doing something similar

**Classify the problem before picking a tool.** The automation ecosystem roughly splits into two problem shapes: scraping public content at scale (where stealth libraries, residential proxies, and cloud browsers shine) and automating a private authenticated session (where the right move is to use the real browser and real session, not to impersonate them). Picking a tool from the wrong category wastes a lot of time. Ask first: is the barrier a bot detector, or is it a login?

**Analyze the network before writing code.** The download's two-step signed URL pattern was visible in DevTools before any code existed. Understanding it upfront meant implementing it correctly the first time. The same goes for the report list API — one GET with all data, no pagination to implement.

**PyWebView's `html=` parameter has a known bug with EdgeChromium.** Pass `url="file:///absolute/path/to/ui.html"` instead. This bug manifests as either source code rendering as plain text or an AccessibilityObject recursion error filling the console.

**ABE makes external cookie reading unreliable on modern browsers.** If a tutorial from before 2024 describes reading Chromium cookies with DPAPI, test it before building around it. It may not work on current browsers.

**CDP doesn't fingerprint, but rate does.** Random delays aren't defensive magic — they're just good behavior toward any API you don't own. They cost almost nothing and prevent accidental rate-limiting on long batches.

**Decouple UI and backend as early as possible.** Having all UI state in JavaScript with a clean Python API boundary meant the backend could be replaced without touching a line of HTML. For a tool where the backend approach changed substantially during development, this saved a significant amount of rework.

## The stack

- **PyWebView** — window frame, renders `ui.html` locally
- **HTML/CSS/vanilla JS** — complete single-file UI, no frameworks
- **Python `websocket-client`** — CDP communication with Edge
- **SQLite** — local database, auto-schema creation
- **`cryptography.fernet`** — password encryption with machine-bound key
- **`threading.Queue`** — unzip worker decoupled from download loop
- **stdlib only** for extraction: `zipfile`, `gzip`, `tarfile`

The full project is around 2,000 lines across nine files. The patterns here — CDP control, two-step signed URL handling, flexible API response parsing, single-file HTML UI with a Python backend bridge — are all reasonably portable to other automation problems in the same space.
