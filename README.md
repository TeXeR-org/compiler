# TeXeR Compiler Service

A dedicated microservice for compiling LaTeX projects into PDFs using a Dockerized TeX Live environment.

## 🚀 Overview

This service provides an HTTP API to compile LaTeX source code. It is built with Node.js (Express) and packaged with a custom Docker image containing TeX Live 2024. It is designed to be stateless, scalable, and secure.

## 🛠 Tech Stack

-   **Runtime:** Node.js 20 (Express)
-   **Container:** Docker (Debian Bookworm Slim base)
-   **LaTeX Distribution:** TeX Live 2024 (Basic scheme + curated packages)
-   **Engines Supported:** `pdflatex`, `xelatex`, `lualatex`, `latex`

## 🐳 Docker Image

The Dockerfile is optimized for size while maintaining compatibility with 90% of common LaTeX use cases. It installs:
-   **Core:** `latex-bin`, `latexmk`, `koma-script`, `memoir`
-   **Math:** `amsmath`, `mathtools`, `unicode-math`
-   **Graphics:** `pgf` (TikZ), `graphicx`, `xcolor`
-   **Fonts:** `collection-fontsrecommended`, `fontspec`
-   **Bib:** `biblatex`, `biber`
-   **Misc:** `markdown`, `minted` (requires shell-escape), `geometry`, `hyperref`

## 🔌 API Endpoints

### 1. Health Check
GET `/health`

Returns the service status and available compilation engines.

**Response:**
```json
{
  "status": "ok",
  "engines": ["pdflatex", "xelatex"],
  "active": 2,
  "queued": 0,
  "capacity": 8,
  "queueLimit": 100
}
```

`active`, `queued`, and `capacity` say how busy this instance is. When `queued` stays above zero, add an instance.

### Metrics
GET `/metrics`

Prometheus metrics: compiles running and waiting, and request durations by outcome. Needs `Authorization: Bearer <METRICS_TOKEN>`; without the setting it answers 404 in production. See `ops/README.md`.

### 2. Compile Project
POST `/compile`

Compiles a set of files into a PDF.

**Request Body:**
```json
{
  "files": [
    { "path": "main.tex", "content": "\\documentclass{article}..." },
    { "path": "image.png", "content": "<base64>", "encoding": "base64" }
  ],
  "mainFile": "main.tex",
  "engine": "pdflatex",
  "compileMode": "normal",
  "stopOnFirstError": false
}
```

-   `engine`: `pdflatex` (default), `xelatex`, `lualatex`, or `latex` (DVI, turned into PDF with dvipdfmx). `latexmk` is accepted and means `pdflatex`.
-   `compileMode`: `normal`, or `draft`, which draws images as empty boxes of the right size.

Each compile runs TeX, then whatever the document needs (BibTeX for `\bibliography`, Biber for biblatex, makeindex for `\makeindex`, glossaries and nomencl), then TeX again until cross-references settle: at most five TeX runs, within 100 seconds in all. Biber never reads a config file from the project or fetches remote bibliographies, and a project containing `dvipdfmx.cfg` is refused.

**Response:**
```json
{
  "success": true,
  "pdf": "JVBERi0xLjQK...", // Base64 encoded PDF
  "log": "...",           // Full compiler log
  "errors": [],           // Extracted error messages, bibliography errors included
  "warnings": [],         // From BibTeX, Biber and makeindex
  "passes": 3,            // How many times TeX ran
  "duration": 1250        // Time in ms
}
```

**Status codes:** `200` whenever the compile ran, whether or not the document had errors (read `success` and `errors`); `400` for a request that cannot be compiled (no files, an unknown engine, a file over 10 MB); `429` when the caller's budget is spent; `503` when the queue is full (100 waiting). Every answer, refusals included, has the same shape: `success`, `pdf` (base64 or `null`), `log`, `errors`, and `duration`. Wait at least 120 seconds before giving up on a request: a compile has 100 seconds of its own, and may queue first.

### 3. Client keys (other projects)

Projects other than TeXeR call the compiler with a named key of their own. Each key has its own budget of compiles a minute, independent of the caller's address, and its name shows in the logs, so one project can neither starve another nor be confused with it.

**What a client sends:**

```
POST https://compile.texer.org/compile
X-Compiler-Key: ck_…
Content-Type: application/json

{ "files": [{ "path": "main.tex", "content": "…" }, { "path": "fig.png", "content": "<base64>", "encoding": "base64" }], "mainFile": "main.tex", "engine": "pdflatex", "compileMode": "normal", "stopOnFirstError": false }
```

The answer is the response above. A recognized key also gets a response header `X-Compiler-Client: <name>`. A key that is not recognized gets `X-Compiler-Client: unrecognized` and is treated as no key at all: the anonymous limit of one compile a second per address. So a client that sees `429` without its own name in that header has a wrong or revoked key, not a spent budget.

**Add a client:**

1. Generate its key: `node scripts/generate-client-key.mjs <name>` (lowercase letters, digits, `-` and `_`, up to 32 characters). It prints `name:key`. Nothing is written anywhere, so copy it before closing the terminal. Use a different key in each environment (development, production).
2. Append the entry to `COMPILER_CLIENT_KEYS` on this service, comma-separated: `COMPILER_CLIENT_KEYS=axya:ck_…,other:ck_…`. To give a client more or fewer than the default 60 compiles a minute, add the limit as a third field: `axya:ck_…:120` (a whole number from 1 to 10000).
3. Restart the compiler (in Railway, redeploy the service) so it loads the keys. Its log lists each client with its limit.
4. Give the key to the project, which sends it as `X-Compiler-Key`. Keep it on their server: it must never reach a browser.

**Revoke a client:** delete its entry from `COMPILER_CLIENT_KEYS` and restart. Its requests become anonymous at once. To rotate a key, generate a new one under the same name, replace the entry, restart, and give the project the new key.

**Limits are per instance.** Budgets are counted in each instance's memory: they start again on a restart, and with two compiler replicas a client effectively gets twice its limit.

**Current clients:** `axya` (Axya's backend). TeXeR's own API does not use a client key: it identifies itself with `COMPILER_INTERNAL_SECRET`.

## 📦 Local Development

### Prerequisites
-   Docker installed
-   Node.js installed (if running outside Docker)

### Running with Docker (Recommended)

1.  **Build the image:**
    ```bash
    docker build -t texer-compiler .
    ```

2.  **Run the container:**
    ```bash
    docker run -p 3500:3500 texer-compiler
    ```

3.  **Test:**
    ```bash
    curl http://localhost:3500/health
    ```

### Running Locally (Without Docker)
*Note: You must have TeX Live installed on your host machine for this to work.*

1.  **Install dependencies:**
    ```bash
    npm install
    ```

2.  **Start the server:**
    ```bash
    node server.js
    ```

## 📝 Configuration

-   **PORT**: Service port (default: `3500`)
-   **ALLOWED_ORIGINS**: Comma-separated list of allowed CORS origins (default: `*`)
-   **MAX_CONCURRENT**: How many TeX runs at once (default: `8`). Set it to the cores this instance has
-   **METRICS_TOKEN**: The bearer token for `GET /metrics`
-   **COMPILER_INTERNAL_SECRET**: Shared with TeXeR's API, which sends it as `X-Internal-Token` to skip the anonymous limit
-   **COMPILER_CLIENT_KEYS**: Other projects' keys, `name:key` or `name:key:limit`, comma-separated (see Client keys)

## 🛡️ Security & Performance

-   **Isolation**: Each compilation job runs in a unique temporary directory (`/tmp/texer-jobs/<uuid>`).
-   **Cleanup**: Job directories are automatically deleted 5 seconds after completion.
-   **Timeouts**: Compilation is hard-capped at 60 seconds to prevent DoS.
-   **Log Limits**: Log output is capped at 10MB to prevent memory exhaustion.

## License

[MIT](LICENSE)
