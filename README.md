# Wazuh SCA Policy API

## Overview

The Wazuh SCA Policy API is a FastAPI service that acts as a structured gateway to a GitHub repository containing Wazuh SCA (Security Configuration Assessment) YAML policy files. These policies encode CIS (Center for Internet Security) Benchmark checks for applications such as Apache, MySQL, IIS, Nginx, and operating systems across multiple platforms.

Rather than bundling policy files inside the application, the API reads them live from a GitHub repository. This means policies can be updated, versioned, and reviewed independently of the application — a clean separation of data from code.

The API exposes three endpoints to **list**, **filter**, and **download** policies, and uses an in-memory TTL cache to avoid hitting GitHub on every request.

---

## GitHub Repository Structure

The companion GitHub repository (`wazuh-sca-policies`) must follow this layout for the API to function correctly:

```
wazuh-sca-policies/
├── index.json                    ← Machine-readable manifest (required)
├── README.md
└── policies/
    ├── apache/
    │   ├── cis_apache_24_rpm.yml
    │   └── cis_apache_24_deb.yml
    ├── mysql/
    │   ├── cis_mysql_80_rpm.yml
    │   ├── cis_mysql_80_deb.yml
    │   └── cis_mysql_80_win.yml
    ├── iis/
    │   └── cis_iis_10_win.yml
    ├── nginx/
    │   ├── cis_nginx_1_rpm.yml
    │   └── cis_nginx_1_deb.yml
    └── mssql/
        └── cis_mssql_2019_win.yml
```

### File Naming Convention

Every YAML file follows the pattern:

```
{cis}_{application}_{version}_{platform}.yml
```

Examples:
- `cis_apache_24_rpm.yml` — Apache 2.4, RPM-based Linux
- `cis_mysql_80_win.yml` — MySQL 8.0, Windows
- `cis_iis_10_win.yml` — IIS 10, Windows

### The `index.json` Manifest

The `index.json` file at the repo root is the critical piece that powers the API's filtering capability. The API fetches this file once (then caches it) to avoid traversing the repo tree on every request.

Example `index.json`:

```json
{
  "version": "1.0.0",
  "last_updated": "2026-03-06",
  "policies": [
    {
      "id": "cis_apache_24_rpm",
      "name": "CIS Apache HTTP Server 2.4 - RPM",
      "description": "CIS Benchmark for Apache 2.4 on RPM-based Linux",
      "file": "policies/apache/cis_apache_24_rpm.yml",
      "application": "apache",
      "app_version": "2.4",
      "platform": "linux-rpm",
      "cis_version": "1.5.0"
    }
  ]
}
```

> **Note:** The `file` field is a relative path from the repo root. The API constructs the full raw GitHub URL at runtime using `GITHUB_OWNER`, `GITHUB_REPO`, and `GITHUB_BRANCH`.

### Supported Platform Values

| Platform | `platform` value | Example Distros / OS |
|---|---|---|
| Linux RPM-based | `linux-rpm` | RHEL, CentOS, Rocky Linux, AlmaLinux, Fedora |
| Linux Debian-based | `linux-deb` | Ubuntu, Debian, Linux Mint |
| Windows | `windows` | Windows Server 2016/2019/2022, Windows 10/11 |
| macOS | `macos` | macOS 12 Monterey, 13 Ventura, 14 Sonoma |

---

## Project Structure

```
sca-api/
├── app/
│   ├── main.py
│   ├── core/
│   │   └── config.py
│   ├── models/
│   │   └── policy.py
│   ├── services/
│   │   ├── github.py
│   │   ├── cache.py
│   │   └── policy.py
│   └── routers/
│       └── policies.py
├── requirements.txt
└── .env.example
```

### Module Responsibilities

| File | Responsibility |
|---|---|
| `main.py` | FastAPI app factory + router registration |
| `core/config.py` | pydantic-settings — reads `.env`, exposes typed `Settings` object |
| `models/policy.py` | `PolicySummary`, `PolicyDetail`, `PolicyIndex`, `Platform` enum (Pydantic v2) |
| `services/github.py` | Raw HTTP calls to GitHub — builds raw URLs, fetches `index.json` and YAML files |
| `services/cache.py` | Async-safe TTL in-memory cache with lock to prevent thundering herd |
| `services/policy.py` | Business logic — cache coordination, filtering, per-policy fetch |
| `routers/policies.py` | 3 HTTP endpoints wired to the service layer |

---

## Getting Started

### Prerequisites

- Python 3.11 or later
- `pip`
- A GitHub repository structured as described above
- (Optional) A GitHub Personal Access Token

### Installation

1. Clone this repository:

```bash
git clone https://github.com/your-org/sca-api.git
cd sca-api
```

2. Create and activate a virtual environment:

```bash
python -m venv .venv
source .venv/bin/activate        # Linux / macOS
.venv\Scripts\activate           # Windows
```

3. Install dependencies:

```bash
pip install -r requirements.txt
```

4. Copy the example env file and fill in your values:

```bash
cp .env.example .env
```

### Configuration

All configuration is managed through environment variables, loaded from a `.env` file via `pydantic-settings`.

| Variable | Required? | Description | Default / Example |
|---|---|---|---|
| `GITHUB_OWNER` | **Required** | GitHub user or organisation name | `your-org` |
| `GITHUB_REPO` | **Required** | Repository name | `wazuh-sca-policies` |
| `GITHUB_BRANCH` | Optional | Branch to read from | `main` |
| `GITHUB_TOKEN` | Optional | Personal access token — avoids rate limiting | `ghp_xxxx` |
| `CACHE_TTL_SECONDS` | Optional | How long to cache `index.json` (seconds) | `300` |

> **Tip:** For public GitHub repositories, `GITHUB_TOKEN` is not strictly required but is strongly recommended. Unauthenticated GitHub API requests are rate-limited to 60 requests per hour per IP. With a token, that limit rises to 5,000 per hour.

### Running the Server

Development (with auto-reload):

```bash
uvicorn app.main:app --reload
```

Production (with multiple workers):

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 4
```

The interactive API docs will be available at:
- **Swagger UI:** http://localhost:8000/docs
- **ReDoc:** http://localhost:8000/redoc

---

## API Reference

| Method | Path | Description |
|---|---|---|
| `GET` | `/health` | Liveness check — returns `{ "status": "ok" }` |
| `GET` | `/policies` | List all policies. Supports `?application=` and `?platform=` filters |
| `GET` | `/policies/{id}` | Full policy metadata + raw YAML content as a JSON response |
| `GET` | `/policies/{id}/raw` | Raw plain-text YAML — pipe directly into Wazuh agents |

### `GET /policies` — Query Parameters

- **`application`** *(string, optional)* — Filter by application name. Case-insensitive. Examples: `apache`, `mysql`, `iis`
- **`platform`** *(enum, optional)* — Filter by platform. Must be one of: `linux-rpm`, `linux-deb`, `windows`, `macos`

### Example Requests

List all policies:
```
GET /policies
```

All Apache policies across all platforms:
```
GET /policies?application=apache
```

Apache policies for RPM-based Linux only:
```
GET /policies?application=apache&platform=linux-rpm
```

All Windows policies:
```
GET /policies?platform=windows
```

Get full metadata and YAML content for a specific policy:
```
GET /policies/cis_apache_24_rpm
```

Download the raw YAML — pipe directly to a file:
```bash
curl http://localhost:8000/policies/cis_apache_24_rpm/raw > policy.yml
```

### Response Shapes

`GET /policies` returns an array of `PolicySummary` objects:

```json
[
  {
    "id": "cis_apache_24_rpm",
    "name": "CIS Apache HTTP Server 2.4 - RPM",
    "description": "CIS Benchmark for Apache 2.4 on RPM-based Linux",
    "file": "policies/apache/cis_apache_24_rpm.yml",
    "application": "apache",
    "app_version": "2.4",
    "platform": "linux-rpm",
    "cis_version": "1.5.0"
  }
]
```

`GET /policies/{id}` returns a `PolicyDetail` object (`PolicySummary` + `raw_yaml` field):

```json
{
  "id": "cis_apache_24_rpm",
  "name": "CIS Apache HTTP Server 2.4 - RPM",
  "...",
  "raw_yaml": "policy:\n  id: cis_apache\n  ..."
}
```

---

## Caching Strategy

The API uses a simple async-safe in-memory TTL cache to avoid hitting GitHub on every request.

### How it works

- When the first request arrives, the API fetches `index.json` from GitHub and stores it in memory with an expiry timestamp.
- Subsequent requests within the TTL window (default: 5 minutes) are served from memory — no GitHub call is made.
- Once the TTL expires, the next request re-fetches from GitHub and refreshes the cache.
- An `asyncio.Lock` prevents a thundering herd: if many requests arrive simultaneously during a cache miss, only one fetches from GitHub. The rest wait and are served from the freshly populated cache.

### Individual YAML files

Individual policy YAML files (fetched via `GET /policies/{id}` and `GET /policies/{id}/raw`) are **not cached** — they are fetched live from GitHub on each request. This is intentional since they are only requested one at a time, and caching all YAMLs in memory would be wasteful.

> **Note:** If you expect heavy traffic to individual policy endpoints, consider adding a second TTL cache keyed by policy ID, or integrating a shared cache like Redis.

### Adjusting the TTL

Set `CACHE_TTL_SECONDS` in your `.env` file. A value of `0` effectively disables caching (every request hits GitHub). A value of `3600` caches for one hour.

---

## Adding New Policies to the Repo

Because the API reads everything from the GitHub repository, adding a new policy requires **no code changes** — only changes to the repo.

1. Add your YAML file under the appropriate application folder: `policies/{application}/{filename}.yml`
2. Add a corresponding entry to `index.json` at the repo root.
3. Update the `last_updated` field in `index.json`.
4. Commit and push to the branch configured in `GITHUB_BRANCH`.

The API will pick up the change within one TTL window (default: 5 minutes) without any restart required.

> **Tip:** Consider automating `index.json` generation with a small CI script that walks the `policies/` directory and rebuilds the manifest on every push. This eliminates the risk of the index drifting out of sync with the actual files.

---

## Error Handling

| Status | When it occurs |
|---|---|
| `404 Not Found` | Policy ID does not exist in `index.json` |
| `422 Unprocessable Entity` | Invalid query parameter value (e.g. unrecognised `platform`) |
| `502 / 5xx` | GitHub raw content request failed (network error, repo unavailable, bad token) |

All error responses follow FastAPI's standard error schema:

```json
{ "detail": "Policy 'cis_apache_24_rpm' not found." }
```

---

*Wazuh SCA Policy API — Based on [CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks/)*
