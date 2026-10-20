# FastAPI – Die ausführliche Dokumentation

> Eine praxisnahe Einführung und Referenz zu **FastAPI**, durchgängig illustriert am Projekt **subnetplan** (`web/api.py`), das FastAPI sehr sauber einsetzt: zustandslose JSON-API, vollständig dokumentiertes OpenAPI-Schema, einheitliche Fehlerantworten, dünne Endpunkte und eine strikte Trennung von Web- und Fachlogik, siehe https://github.com/naphets86/sbntgg

Diese Dokumentation hat zwei Ziele:

1. **FastAPI von Grund auf erklären** – von „Hello World“ bis Deployment.
2. **Zeigen, wie ein reales Projekt FastAPI nutzt** – jedes Kapitel enthält Abschnitte mit dem Titel *„So macht es subnetplan“*, die auf echten Code aus dem Projekt verweisen, und Hinweise, was man davon übernehmen sollte.

Alle Beispiele sind für **Python ≥ 3.11**, **FastAPI ≥ 0.110** und **Pydantic v2** geschrieben (die im Projekt verwendeten Versionen).

---

## Inhaltsverzeichnis

1. [Was ist FastAPI?](#1-was-ist-fastapi)
2. [Das Beispielprojekt subnetplan im Überblick](#2-das-beispielprojekt-subnetplan-im-überblick)
3. [Installation und Projektsetup](#3-installation-und-projektsetup)
4. [Die erste Anwendung](#4-die-erste-anwendung)
5. [Pfadoperationen und Routing](#5-pfadoperationen-und-routing)
6. [Parameter: Pfad, Query, Header, Cookie, Formular, Dateien](#6-parameter-pfad-query-header-cookie-formular-dateien)
7. [Request Body und Pydantic-Modelle](#7-request-body-und-pydantic-modelle)
8. [Response-Modelle und Antworten](#8-response-modelle-und-antworten)
9. [Fehlerbehandlung](#9-fehlerbehandlung)
10. [OpenAPI, Swagger UI und ReDoc](#10-openapi-swagger-ui-und-redoc)
11. [Dependency Injection](#11-dependency-injection)
12. [async oder def? Nebenläufigkeit verstehen](#12-async-oder-def-nebenläufigkeit-verstehen)
13. [Anwendung strukturieren mit APIRouter](#13-anwendung-strukturieren-mit-apirouter)
14. [Middleware, CORS und weitere Querschnittsthemen](#14-middleware-cors-und-weitere-querschnittsthemen)
15. [Lifespan, Konfiguration und Anwendungszustand](#15-lifespan-konfiguration-und-anwendungszustand)
16. [Background Tasks](#16-background-tasks)
17. [Sicherheit: API-Key, OAuth2 und JWT](#17-sicherheit-api-key-oauth2-und-jwt)
18. [Datenbanken anbinden](#18-datenbanken-anbinden)
19. [WebSockets, Streaming, Dateien und statische Inhalte](#19-websockets-streaming-dateien-und-statische-inhalte)
20. [Testen](#20-testen)
21. [Betrieb und Deployment](#21-betrieb-und-deployment)
22. [Performance und Skalierung](#22-performance-und-skalierung)
23. [Best Practices und Anti-Patterns](#23-best-practices-und-anti-patterns)
24. [Verbesserungsvorschläge für subnetplan](#24-verbesserungsvorschläge-für-subnetplan)
25. [Fehlersuche und FAQ](#25-fehlersuche-und-faq)
26. [Spickzettel](#26-spickzettel)
27. [Glossar](#27-glossar)
28. [Weiterführende Quellen](#28-weiterführende-quellen)

---

## 1. Was ist FastAPI?

**FastAPI** ist ein modernes Python-Webframework zum Bau von HTTP-APIs. Es steht auf zwei Säulen:

| Baustein | Aufgabe |
| --- | --- |
| **Starlette** | ASGI-Framework: Routing, Requests/Responses, Middleware, WebSockets, Testclient |
| **Pydantic** | Datenvalidierung, Serialisierung und JSON-Schema anhand von Python-Typannotationen |

FastAPI selbst ergänzt darauf vor allem:

- **Typgetriebene Validierung:** Aus Funktionssignaturen werden Parameter, Request-Bodies und Responses abgeleitet und geprüft.
- **Automatische OpenAPI-Dokumentation:** Aus dem Code entsteht ein OpenAPI-3.1-Schema und daraus interaktive Dokumentation (Swagger UI unter `/docs`, ReDoc unter `/redoc`).
- **Dependency Injection:** Wiederverwendbare Abhängigkeiten (Datenbank-Sessions, Authentifizierung, Konfiguration) werden deklarativ eingebunden.
- **Editor-Unterstützung:** Weil alles Standard-Python-Typing ist, funktionieren Autovervollständigung und statische Prüfung (mypy, pyright) hervorragend.

### 1.1 Wann FastAPI, wann nicht?

**Gut geeignet für**

- JSON-/REST-APIs, auch mit vielen Endpunkten und komplexen Datenmodellen
- Microservices und interne Dienste
- Backends für Single-Page-Apps und mobile Apps
- ML-/Daten-Services, die ein Modell oder eine Bibliothek per HTTP verfügbar machen (wie **subnetplan**, das eine reine Rechenbibliothek als API ausliefert)
- Anwendungen mit WebSockets oder Streaming

**Weniger geeignet für**

- Klassische serverseitig gerenderte Websites mit Templates, Formularen und Admin-Oberfläche (hier ist Django „batteries included“)
- Sehr kleine Skripte, bei denen Flask-Minimalismus reicht und keine Validierung nötig ist

### 1.2 Das mentale Modell

```text
HTTP-Request
   │
   ▼
ASGI-Server (Uvicorn)
   │
   ▼
Middleware-Kette (CORS, GZip, eigene …)
   │
   ▼
Routing  ──►  passende Pfadoperation (Funktion)
   │
   ├─ Parameter, Body, Header … aus dem Request lesen und validieren (Pydantic)
   ├─ Abhängigkeiten (Depends) auflösen
   ▼
Endpunktfunktion läuft  ──►  Rückgabewert
   │
   ▼
response_model: filtern, validieren, serialisieren (Pydantic)
   │
   ▼
HTTP-Response (JSON)
```

Fehler (`HTTPException`, `RequestValidationError`, eigene Exceptions) werden von **Exception-Handlern** in Antworten übersetzt. Diese Kette ist der rote Faden durch dieses Dokument.

---

## 2. Das Beispielprojekt subnetplan im Überblick

**subnetplan** ist ein Python-Projekt (Version 4.0.0) zum automatischen Subnetting für Mitarbeitergruppen inklusive Firewall-Regeln (iptables, nftables, Cisco-ACL). Es besteht aus einer **Fachbibliothek** (`src/`, als Paket `subnetplan` installiert), einer **Kommandozeile** (`src/cli.py`) und einer **JSON-API** auf FastAPI-Basis (`web/api.py`).

### 2.1 Projektstruktur

```text
sbntgg-main/
├── src/                  # Fachlogik; wird als Paket "subnetplan" installiert
│   ├── planner.py        # plan_subnets(cfg)
│   ├── config.py         # build_config(), build_extension()
│   ├── firewall.py       # render_firewall(), FirewallOptions
│   ├── verify.py, stats.py, checks.py, roundtrip.py, semantics.py, policy.py …
│   └── models.py         # Konstanten (STRATEGIES, FIREWALLS …) und PlanError
├── web/
│   ├── __init__.py
│   └── api.py            # <- die FastAPI-Anwendung (567 Zeilen)
├── tests/
│   └── test_web_api.py   # API-Tests mit TestClient
├── examples/config.example.yaml
├── pyproject.toml        # Abhängigkeiten, Entry-Points, pytest-Konfiguration
└── README.md
```

### 2.2 Die Architektur in einem Satz

> **Die API ist eine dünne, streng typisierte Hülle um eine gut getestete Fachbibliothek.**

Das bedeutet konkret:

- Die Fachlogik (`src/`) kennt **kein** FastAPI und **kein** HTTP. Sie wirft `PlanError`, wenn etwas fachlich nicht stimmt.
- `web/api.py` enthält **nur** Web-Belange: Request-/Response-Modelle, Routen, Dokumentation, Fehlerabbildung auf HTTP.
- Jeder Endpunkt ist **zwei bis zehn Zeilen** lang: Konfiguration bauen → Fachfunktion aufrufen → Ergebnis zurückgeben.

### 2.3 Die zwölf Endpunkte

| Methode und Pfad | Tag | Zweck |
| --- | --- | --- |
| `GET /api/v1/health` | meta | Prozessstatus |
| `GET /api/v1/options` | meta | zulässige Auswahlwerte für Clients |
| `POST /api/v1/plans` | plans | Subnetzplan berechnen |
| `POST /api/v1/plans/extend` | plans | bestehenden Plan erweitern |
| `POST /api/v1/plans/verify` | analysis | Plan formal prüfen |
| `POST /api/v1/plans/stats` | analysis | Auslastungsstatistiken |
| `POST /api/v1/plans/warnings` | analysis | Planungshinweise |
| `POST /api/v1/plans/firewall` | firewall | Firewall-Regeltext erzeugen |
| `POST /api/v1/plans/roundtrip` | analysis, firewall | Regeln zurücklesen und prüfen |
| `POST /api/v1/plans/compare` | analysis | zwei Zugriffspolitiken vergleichen |
| `POST /api/v1/plans/compare-services` | analysis | Freigaben auf Dienstebene vergleichen |
| `POST /api/v1/plans/verdict` | analysis | `accept`/`drop`-Urteil für einen Paketfluss |

### 2.4 Welche FastAPI-Features das Projekt nutzt

| Feature | Wo im Projekt | Kapitel |
| --- | --- | --- |
| `FastAPI(...)` mit `title`, `summary`, `description`, `version`, `openapi_tags` | Kopf von `api.py` | [10](#10-openapi-swagger-ui-und-redoc) |
| Pydantic-`BaseModel` für Requests **und** Responses | ca. 20 Klassen | [7](#7-request-body-und-pydantic-modelle), [8](#8-response-modelle-und-antworten) |
| `Field(description=…, examples=…, default_factory=…, ge=…, le=…)` | in allen Modellen | [7](#7-request-body-und-pydantic-modelle) |
| `ConfigDict(json_schema_extra=…)` für Beispiel-Requests | `ConfigRequest` u. a. | [10](#10-openapi-swagger-ui-und-redoc) |
| Modellvererbung (`FirewallRequest(ConfigRequest)`) | alle Request-Klassen | [7](#7-request-body-und-pydantic-modelle) |
| `Literal[...]` als Enum-Ersatz | `firewall`, `state`, `relation` … | [7](#7-request-body-und-pydantic-modelle) |
| Union-Typen (`ErrorResponse \| ValidationErrorResponse`) | `COMMON_RESPONSES` | [8](#8-response-modelle-und-antworten) |
| `response_model`, `response_model_exclude_none` | alle Endpunkte | [8](#8-response-modelle-und-antworten) |
| `responses={…}` für 422/503 inkl. Beispielen | `COMMON_RESPONSES` | [8](#8-response-modelle-und-antworten), [10](#10-openapi-swagger-ui-und-redoc) |
| `@app.exception_handler(PlanError)` | zentrales Fehlerformat | [9](#9-fehlerbehandlung) |
| `HTTPException(503)` für optionale Abhängigkeit | `compare` | [9](#9-fehlerbehandlung) |
| `tags`, `summary`, `description`, Docstrings | jeder Endpunkt | [10](#10-openapi-swagger-ui-und-redoc) |
| Synchrone `def`-Endpunkte für CPU-Arbeit | alle | [12](#12-async-oder-def-nebenläufigkeit-verstehen) |
| Konsolen-Entry-Point mit `uvicorn.run` und Umgebungsvariablen | `run()` | [21](#21-betrieb-und-deployment) |
| `TestClient`, OpenAPI-Schema-Tests | `tests/test_web_api.py` | [20](#20-testen) |

Was das Projekt **bewusst nicht** nutzt (weil es nicht gebraucht wird): `APIRouter`, `Depends`, Authentifizierung, Datenbank, Middleware, Background Tasks, async. Zu all dem gibt es in diesem Dokument eigene Kapitel, damit du beim Wachsen des Projekts weißt, wie es weitergeht.

---

## 3. Installation und Projektsetup

### 3.1 Virtuelle Umgebung

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
```

### 3.2 FastAPI installieren

Es gibt zwei gängige Varianten:

```bash
# Empfohlen für den Einstieg: FastAPI + Uvicorn + CLI + häufige Extras
pip install "fastapi[standard]"

# Minimal: nur das Framework, Server separat
pip install fastapi "uvicorn[standard]"
```

`fastapi[standard]` bringt unter anderem Uvicorn, die `fastapi`-CLI, `httpx` (für den TestClient), `python-multipart` (für Formulare) und `jinja2` mit.

**So macht es subnetplan** – Abhängigkeiten stehen in der `pyproject.toml`:

```toml
[project]
requires-python = ">=3.11"
dependencies = [
    "pyyaml>=6.0",
    "numpy>=1.24",
    "fastapi>=0.110",
    "uvicorn[standard]>=0.27",
    "subgraph @ git+https://github.com/naphz86/subgraph.git",
]

[project.optional-dependencies]
dev = ["pytest", "pytest-cov", "coverage", "httpx2"]

[project.scripts]
subnetplan = "subnetplan.cli:main"
subnetplan-api = "web.api:run"
```

Hinweise dazu:

- Die **Untergrenze** `fastapi>=0.110` genügt, weil das Projekt nur Basisfunktionen nutzt. Für Produktion sollte zusätzlich eine **Lock-Datei** (`pip-tools`, `uv`, `poetry`) exakte Versionen festschreiben.
- `uvicorn[standard]` installiert optionale Beschleuniger (`uvloop`, `httptools`, `websockets`).
- `httpx` ist **Voraussetzung des `TestClient`**. Die README nennt zu Recht `httpx`; in der `pyproject.toml` steht allerdings `httpx2`. Das ist vermutlich ein Tippfehler, das Paket heißt `httpx` (siehe [Kapitel 24](#24-verbesserungsvorschläge-für-subnetplan)).
- `[project.scripts]` erzeugt beim Installieren das Kommando `subnetplan-api`, das die Funktion `web.api:run` aufruft.

### 3.3 Das Projekt als Paket installieren

```bash
pip install -e ".[dev]"
```

`-e` (editable) verlinkt den Quellcode, Änderungen wirken sofort. Das Projekt bildet in der `pyproject.toml` zwei Pakete ab:

```toml
[tool.setuptools]
packages = ["subnetplan", "web"]
package-dir = { subnetplan = "src", web = "web" }
```

Das Verzeichnis `src` wird also als Paket **`subnetplan`** installiert, `web` als Paket **`web`**. Deshalb lauten die Importe in `api.py` `from subnetplan.planner import plan_subnets` und der Uvicorn-Pfad `web.api:app`.

### 3.4 Empfohlene Verzeichnisstruktur für größere Projekte

```text
meine_api/
├── app/
│   ├── __init__.py
│   ├── main.py            # FastAPI-Instanz, Router einbinden, Lifespan
│   ├── config.py          # Einstellungen (pydantic-settings)
│   ├── dependencies.py    # gemeinsame Depends-Funktionen
│   ├── errors.py          # eigene Exceptions und Handler
│   ├── routers/
│   │   ├── __init__.py
│   │   ├── users.py
│   │   └── items.py
│   ├── schemas/           # Pydantic-Modelle (Request/Response)
│   ├── models/            # ORM-Modelle (falls Datenbank)
│   └── services/          # Fachlogik, frei von FastAPI
├── tests/
├── pyproject.toml
└── README.md
```

Die wichtigste Regel dahinter ist die von subnetplan: **Fachlogik in `services/` (oder `src/`) darf FastAPI nicht importieren.**

---

## 4. Die erste Anwendung

### 4.1 Hello World

```python
# main.py
from fastapi import FastAPI

app = FastAPI()


@app.get("/")
def root() -> dict[str, str]:
    return {"message": "Hallo Welt"}
```

Starten:

```bash
fastapi dev main.py              # Entwicklungsmodus mit Auto-Reload (fastapi[standard])
# oder klassisch:
uvicorn main:app --reload
```

Danach:

| URL | Inhalt |
| --- | --- |
| `http://127.0.0.1:8000/` | die JSON-Antwort |
| `http://127.0.0.1:8000/docs` | Swagger UI (interaktiv ausprobieren) |
| `http://127.0.0.1:8000/redoc` | ReDoc (gut lesbare Referenz) |
| `http://127.0.0.1:8000/openapi.json` | das maschinenlesbare OpenAPI-Schema |

### 4.2 Ein Endpunkt mit Pfadparameter, Query und Body

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()


class Item(BaseModel):
    name: str
    price: float
    in_stock: bool = True


@app.put("/items/{item_id}")
def update_item(item_id: int, item: Item, verbose: bool = False) -> dict:
    return {"item_id": item_id, "item": item, "verbose": verbose}
```

FastAPI erkennt die Herkunft jedes Parameters **automatisch**:

| Parameter | Erkennung |
| --- | --- |
| `item_id: int` | steht im Pfad (`{item_id}`) → **Pfadparameter** |
| `item: Item` | Pydantic-Modell → **JSON-Body** |
| `verbose: bool = False` | einfacher Typ, nicht im Pfad → **Query-Parameter** (`?verbose=true`) |

### 4.3 So macht es subnetplan – die Anwendungsinstanz

Das Projekt beginnt mit einer **ausführlich konfigurierten** Instanz:

```python
from fastapi import FastAPI

API_PREFIX = "/api/v1"

app = FastAPI(
    title="subnetplan API",
    summary="IPv4- und IPv6-Subnetzplanung als JSON API",
    description="""
    Die **subnetplan API** berechnet Subnetze ... (Markdown ist erlaubt)
    """,
    version="4.0.0",
    openapi_tags=[
        {"name": "meta", "description": "Status und zulässige Konfigurationsoptionen."},
        {"name": "plans", "description": "Subnetzpläne berechnen oder ohne Verschieben bestehender Netze erweitern."},
        {"name": "analysis", "description": "Pläne verifizieren, auswerten, vergleichen oder Paketflüsse beurteilen."},
        {"name": "firewall", "description": "Firewall-Regeltexte erzeugen und deren Rückübersetzung prüfen."},
    ],
)
```

Was man daraus lernt:

- `description` unterstützt **Markdown** – das Projekt erklärt dort Ablauf, Eingaben, Fehler und Sicherheitshinweise. Das ist die Startseite der interaktiven Doku.
- `openapi_tags` legt **Reihenfolge und Beschreibung** der Gruppen in Swagger UI fest.
- `version` ist die **API-Version** (hier identisch mit der Projektversion; die README betont das ausdrücklich).
- `API_PREFIX = "/api/v1"` ist eine Konstante, die in allen Routen per f-String eingesetzt wird. Eine Alternative mit `APIRouter(prefix=...)` zeigt [Kapitel 13](#13-anwendung-strukturieren-mit-apirouter).

### 4.4 Entwicklungsserver und Produktionsstart

```bash
# Entwicklung (README des Projekts)
uvicorn web.api:app --reload

# Über den Entry-Point (ohne Reload, Host/Port per Umgebungsvariablen)
SUBNETPLAN_API_HOST=0.0.0.0 SUBNETPLAN_API_PORT=9000 subnetplan-api
```

Der Aufbau `modul.pfad:variable` (`web.api:app`) bedeutet: Importiere das Modul `web.api` und nimm daraus das Objekt `app`.

---

## 5. Pfadoperationen und Routing

Eine **Pfadoperation** ist eine Funktion, die per Dekorator an einen Pfad und eine HTTP-Methode gebunden wird.

### 5.1 HTTP-Methoden

```python
@app.get("/items")          # lesen
@app.post("/items")         # anlegen / Aktion auslösen
@app.put("/items/{id}")     # ersetzen
@app.patch("/items/{id}")   # teilweise ändern
@app.delete("/items/{id}")  # löschen
@app.head("/items")         # nur Header
@app.options("/items")      # erlaubte Methoden
```

Mehrere Methoden für einen Pfad: `@app.api_route("/x", methods=["GET", "POST"])`.

### 5.2 Wichtige Dekorator-Argumente

```python
@app.post(
    "/api/v1/plans",
    tags=["plans"],                       # Gruppe in der Doku
    summary="Subnetzplan berechnen",      # kurze Überschrift
    description="Ausführliche Erklärung (Markdown) …",
    response_model=PlanResponse,          # Form der Antwort
    response_model_exclude_none=True,     # null-Felder weglassen
    responses={422: {...}, 503: {...}},   # zusätzliche Antworten in der Doku
    status_code=200,                      # Standard-Statuscode
    deprecated=False,                     # als veraltet markieren
    operation_id="create_plan",           # stabile ID für Client-Generatoren
    include_in_schema=True,               # False versteckt in der Doku
)
```

### 5.3 Reihenfolge ist wichtig

FastAPI prüft Routen **in der Reihenfolge der Deklaration**. Der erste passende Pfad gewinnt:

```python
@app.get("/users/me")            # muss VOR der Pfadvariante stehen
def me(): ...

@app.get("/users/{user_id}")
def user(user_id: int): ...
```

Stünde `/users/{user_id}` zuerst, würde `me` als `user_id` interpretiert und mit 422 abgelehnt.

### 5.4 Statuscodes

```python
from fastapi import status

@app.post("/items", status_code=status.HTTP_201_CREATED)
def create_item(item: Item) -> Item: ...

@app.delete("/items/{id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_item(id: int) -> None: ...
```

Übliche Zuordnung: `200` Erfolg, `201` angelegt, `204` ohne Inhalt, `400` fehlerhafte Anfrage, `401` nicht authentifiziert, `403` verboten, `404` nicht gefunden, `409` Konflikt, `422` Validierungsfehler, `429` zu viele Anfragen, `500` Serverfehler, `503` Dienst nicht verfügbar.

### 5.5 So macht es subnetplan – Routen

Alle Routen folgen demselben Muster, hier `verify`:

```python
@app.post(
    f"{API_PREFIX}/plans/verify",
    tags=["analysis"],
    summary="Plan formal verifizieren",
    description="Berechnet den Plan und prüft anschließend die implementierten Planeigenschaften. …",
    response_model=VerificationResponse,
    response_model_exclude_none=True,
    responses=COMMON_RESPONSES,
)
def verify(request: ConfigRequest) -> dict[str, Any]:
    """Gibt Plan, Gesamturteil und gegebenenfalls die Liste verletzter Eigenschaften zurück."""
    cfg, plan = _make_plan(request.config, request.overrides)
    problems = verify_plan(plan, cfg)
    return {"valid": not problems, "problems": problems, "plan": _plan_payload(plan)}
```

Beobachtungen:

- **Alle Aktionen sind `POST`** (auch reine Berechnungen). Das ist hier richtig: Die Konfiguration ist groß und strukturiert und gehört in den Body, nicht in die URL. `GET` ist nur für `health` und `options` verwendet, die keine Eingaben brauchen.
- **Verben stehen nicht im Pfad der Ressource**, sondern als Unterpfad: `/plans/verify`, `/plans/stats`. Das gruppiert die Operationen sauber unter `/plans`.
- **Versionierung im Pfad** (`/api/v1`) erlaubt später ein `/api/v2` ohne Bruch für bestehende Clients.
- Der Endpunkt `roundtrip` hat **zwei Tags** (`["analysis", "firewall"]`) und erscheint deshalb in beiden Gruppen der Doku.
- Der **Docstring** wird von FastAPI als Beschreibung verwendet, falls kein `description=` angegeben ist. Das Projekt setzt beides: `description` (ausführlich, für Nutzer der API) und Docstring (kurz, für Entwickler im Code).

---

## 6. Parameter: Pfad, Query, Header, Cookie, Formular, Dateien

Auch wenn subnetplan fast ausschließlich JSON-Bodies verwendet, braucht jede reale API die weiteren Parameterarten.

### 6.1 Pfadparameter

```python
from typing import Annotated
from fastapi import FastAPI, Path

@app.get("/groups/{group_id}")
def get_group(
    group_id: Annotated[int, Path(ge=1, le=4094, description="VLAN-ID", examples=[10])],
) -> dict:
    return {"group_id": group_id}
```

- Der Typ wird erzwungen: `/groups/abc` → HTTP 422.
- `Path(...)` ergänzt Einschränkungen und Dokumentation. Pfadparameter sind immer **Pflicht**.

Pfade mit Schrägstrichen im Wert (z. B. Dateipfade): `@app.get("/files/{file_path:path}")`.

**Enum als Pfadparameter** – Alternative zu `Literal`:

```python
from enum import Enum

class Firewall(str, Enum):
    iptables = "iptables"
    nftables = "nftables"
    cisco = "cisco"

@app.get("/rules/{kind}")
def rules(kind: Firewall) -> dict:
    return {"kind": kind.value}
```

### 6.2 Query-Parameter

```python
from typing import Annotated
from fastapi import Query

@app.get("/search")
def search(
    q: Annotated[str | None, Query(min_length=3, max_length=50)] = None,
    limit: Annotated[int, Query(ge=1, le=100)] = 10,
    offset: int = 0,
    tags: Annotated[list[str] | None, Query()] = None,   # ?tags=a&tags=b
) -> dict:
    return {"q": q, "limit": limit, "offset": offset, "tags": tags}
```

Regel: Hat der Parameter einen **Default**, ist er optional; ohne Default ist er **Pflicht**. Für Listen im Query-String muss `Query()` explizit stehen, sonst würde FastAPI einen Body erwarten.

**Gruppe von Query-Parametern als Modell** (ab FastAPI 0.115):

```python
class Paging(BaseModel):
    model_config = {"extra": "forbid"}
    limit: int = Field(10, ge=1, le=100)
    offset: int = Field(0, ge=0)

@app.get("/items")
def items(paging: Annotated[Paging, Query()]) -> dict:
    return paging.model_dump()
```

### 6.3 Header und Cookies

```python
from fastapi import Header, Cookie

@app.get("/whoami")
def whoami(
    user_agent: Annotated[str | None, Header()] = None,   # Header "User-Agent"
    x_request_id: Annotated[str | None, Header()] = None, # Header "X-Request-ID"
    session: Annotated[str | None, Cookie()] = None,
) -> dict:
    return {"ua": user_agent, "request_id": x_request_id, "session": session}
```

FastAPI wandelt Unterstriche in Header-Namen automatisch in Bindestriche um (`x_request_id` ↔ `X-Request-Id`).

### 6.4 Formulardaten und Datei-Upload

Benötigt `python-multipart` (in `fastapi[standard]` enthalten).

```python
from fastapi import File, Form, UploadFile

@app.post("/upload")
async def upload(
    file: UploadFile,
    description: Annotated[str, Form()] = "",
) -> dict:
    content = await file.read()
    return {"filename": file.filename, "size": len(content), "description": description}
```

- `UploadFile` ist für große Dateien besser als `bytes`, weil es auf Platte auslagert statt alles im Speicher zu halten.
- Mehrere Dateien: `files: list[UploadFile]`.

### 6.5 Validierung im Überblick

| Validator | Gilt für | Bedeutung |
| --- | --- | --- |
| `gt`, `ge`, `lt`, `le` | Zahlen | größer, größer-gleich, kleiner, kleiner-gleich |
| `min_length`, `max_length` | Strings, Listen | Länge |
| `pattern` | Strings | regulärer Ausdruck |
| `multiple_of` | Zahlen | Vielfaches |
| `alias` | alle | anderer Name in JSON als in Python |
| `deprecated` | alle | als veraltet markieren |

**So macht es subnetplan** – Validierung direkt im Feld:

```python
class WarningsRequest(ConfigRequest):
    threshold: float = Field(default=80.0, ge=0, le=100,
        description="Auslastungsschwelle in Prozent, ab der ein Hinweis ausgegeben wird.",
        examples=[80])
```

Ein `threshold` von `150` führt automatisch zu HTTP 422 mit dem Pydantic-Fehler `less_than_equal` – ohne eine Zeile eigenen Prüfcode.

---

## 7. Request Body und Pydantic-Modelle

### 7.1 Grundlagen

Ein Parameter, dessen Typ ein Pydantic-`BaseModel` ist, wird als **JSON-Body** gelesen, validiert und in ein Objekt umgewandelt:

```python
from pydantic import BaseModel, Field

class Group(BaseModel):
    name: str = Field(min_length=1, max_length=64)
    hosts: int = Field(gt=0)
    vlan_id: int | None = Field(default=None, ge=1, le=4094)
    access: list[str] = Field(default_factory=list)

@app.post("/groups")
def create_group(group: Group) -> Group:
    return group
```

Was FastAPI dabei für dich tut:

1. JSON lesen (bei kaputtem JSON: 422).
2. Gegen das Modell validieren und Typen umwandeln (`"5"` → `5` im Standardmodus).
3. Bei Fehlern **eine Liste aller Fehler** mit Feldpfad zurückgeben (HTTP 422).
4. Das Schema in OpenAPI veröffentlichen.

Das Fehlerformat:

```json
{
  "detail": [
    {"type": "int_parsing", "loc": ["body", "hosts"], "msg": "Input should be a valid integer, unable to parse string as an integer", "input": "viele"}
  ]
}
```

`loc` beginnt mit der Quelle (`body`, `query`, `path`, `header`), danach folgt der Feldpfad.

### 7.2 Mehrere Body-Parameter und eingebettete Bodies

```python
from fastapi import Body

@app.put("/x")
def f(item: Item, user: User) -> dict: ...
# erwartet {"item": {...}, "user": {...}}

@app.put("/y")
def g(item: Annotated[Item, Body(embed=True)]) -> dict: ...
# erwartet {"item": {...}}  statt  {...}
```

### 7.3 `Field` – Metadaten und Einschränkungen

`Field` ist das Werkzeug, mit dem man Felder gleichzeitig **validiert und dokumentiert**:

```python
class SubnetResponse(BaseModel):
    group: str = Field(description="Name der Gruppe aus der Konfiguration.", examples=["Server"])
    vlan_id: int = Field(description="Zugewiesene VLAN-ID.", examples=[10])
    gateway: str | None = Field(description="Gateway-Adresse oder `null`.", examples=["10.20.0.1"])
    site: str | None = Field(default=None, description="Standortname bei hierarchischer Planung.")
    services: dict[str, list[str]] | None = Field(default=None, examples=[{"Server": ["tcp/22", "tcp/443"]}])
```

Merke:

- **`X | None` ohne Default** heißt in Pydantic v2: *Pflichtfeld, das `null` sein darf*. Mit `= None` ist es zusätzlich **optional**. Das Projekt nutzt beide Varianten bewusst: `gateway` ist immer vorhanden (evtl. `null`), `site` kann fehlen.
- **Veränderliche Defaults** (Listen, Dicts) immer mit `default_factory` anlegen:

```python
overrides: dict[str, Any] = Field(default_factory=dict, description="Optionale Überschreibungen …")
icmpv6_types: list[str] = Field(default_factory=list)
```

(Pydantic kopiert zwar Defaults, `default_factory` ist aber die eindeutige, idiomatische Form.)

### 7.3.1 `Literal` statt Freitext

```python
class FirewallRequest(ConfigRequest):
    firewall: Literal["iptables", "nftables", "cisco"] = Field(description="Zielsystem …")
```

- Ungültige Werte werden automatisch mit 422 abgewiesen.
- Im OpenAPI-Schema erscheint ein `enum`; Swagger UI zeigt ein Dropdown.
- `Literal` in **Response**-Modellen (`status: Literal["ok"]`, `relation: Literal["equal", "a_in_b", "b_in_a", "incomparable"]`) macht die Antwort zu einem **dokumentierten Vertrag**: Clients wissen, welche Werte möglich sind.

### 7.4 Modellvererbung für gemeinsame Felder

Das Herzstück der Request-Modellierung in subnetplan: **eine Basisklasse, viele Varianten.**

```python
class ConfigRequest(BaseModel):
    config: dict[str, Any] = Field(description="Subnetzplan-Konfiguration. …")
    overrides: dict[str, Any] = Field(default_factory=dict, description="Optionale Überschreibungen …")


class FirewallRequest(ConfigRequest):
    firewall: Literal["iptables", "nftables", "cisco"]
    options: FirewallSettings = Field(default_factory=FirewallSettings)


class WarningsRequest(ConfigRequest):
    threshold: float = Field(default=80.0, ge=0, le=100)
    vpn_networks: list[str] = Field(default_factory=list)
    firewall: Literal["none", "iptables", "nftables", "cisco"] = "none"
    cisco_return: Literal["none", "established"] = "none"


class RoundtripRequest(ConfigRequest):
    firewalls: list[Literal["iptables", "nftables", "cisco"]] = Field(
        default_factory=lambda: ["iptables", "nftables", "cisco"])
    options: FirewallSettings = Field(default_factory=FirewallSettings)


class VerdictRequest(ConfigRequest):
    source: str
    target: str
    state: Literal["new", "established"] = "new"
    ingress_group: str | None = None
    service: str | None = None
```

Vorteile: `config`/`overrides` werden **einmal** definiert und dokumentiert; `verify`, `stats` und `plans` verwenden `ConfigRequest` direkt, ohne eigene Klasse.

Die Endpunkte `compare` und `compare-services` haben ein **anderes** Schema (`CompareRequest` mit `config_a`/`config_b`) und erben deshalb nicht – Vererbung nur dort, wo sie fachlich stimmt.

### 7.5 Verschachtelte Modelle

```python
class FirewallSettings(BaseModel):
    idempotent: bool = True
    group_rules: bool = False
    cisco_return: Literal["none", "established"] = "none"
    ascii_names: bool = False
    log_drops: bool = False
    icmpv6_types: list[str] = Field(default_factory=list)

    def to_options(self) -> FirewallOptions:      # Übersetzung ins Fachmodell
        return FirewallOptions(
            idempotent=self.idempotent,
            group_rules=self.group_rules,
            cisco_return=self.cisco_return,
            ascii_names=self.ascii_names,
            log_drops=self.log_drops,
            icmpv6_types=tuple(self.icmpv6_types),
        )
```

Ein Request kann dann so aussehen:

```json
{
  "config": {"base_network": "10.0.0.0/24", "groups": [{"name": "A", "hosts": 50, "access": ["B"]}, {"name": "B", "hosts": 10}]},
  "firewall": "nftables",
  "options": {"group_rules": true, "log_drops": true}
}
```

Das Muster **„API-Modell → `to_options()` → Fachmodell“** ist besonders lehrreich: Das Web-Modell (Pydantic) und das Fachmodell (`FirewallOptions`, eine Dataclass in `src/`) bleiben getrennt. Die Fachlogik braucht Pydantic nicht zu kennen.

### 7.6 Eigene Validierung

Wenn `Field`-Einschränkungen nicht reichen:

```python
import ipaddress
from pydantic import BaseModel, field_validator, model_validator

class VerdictRequest(BaseModel):
    source: str
    target: str

    @field_validator("source", "target")
    @classmethod
    def must_be_ip(cls, value: str) -> str:
        ipaddress.ip_address(value)      # ValueError -> wird zu 422
        return value

    @model_validator(mode="after")
    def different_hosts(self) -> "VerdictRequest":
        if self.source == self.target:
            raise ValueError("source und target dürfen nicht identisch sein")
        return self
```

**So macht es subnetplan:** Das Projekt validiert fachliche Regeln **nicht** in Pydantic, sondern in der Fachbibliothek (`build_config`, `PlanError`), und bildet sie über einen Exception-Handler auf 422 ab (siehe [Kapitel 9](#9-fehlerbehandlung)). Das ist eine bewusste Entscheidung: Die Regeln gelten identisch für CLI, YAML-Dateien und API und müssen nur einmal existieren. Der Preis: `config` ist in der API nur als `dict[str, Any]` typisiert, die Feldstruktur erscheint daher nicht im OpenAPI-Schema (siehe [Kapitel 24](#24-verbesserungsvorschläge-für-subnetplan)).

### 7.7 `extra`-Felder und Strenge

Standardmäßig **ignoriert** Pydantic unbekannte Felder. Strenger:

```python
from pydantic import ConfigDict

class StrictRequest(BaseModel):
    model_config = ConfigDict(extra="forbid")   # unbekannte Felder → 422
```

`strict=True` (pro Modell oder Feld) verhindert Typumwandlungen wie `"5"` → `5`.

### 7.8 Beispiele für die Dokumentation

Pydantic v2 kennt zwei Wege, die beide im Projekt vorkommen:

```python
# 1) pro Feld
config: dict[str, Any] = Field(examples=[{"base_network": "10.20.0.0/16", "groups": [...]}])

# 2) pro Modell – ein kompletter Beispiel-Request, der in Swagger UI vorbelegt wird
class ConfigRequest(BaseModel):
    model_config = ConfigDict(json_schema_extra={"examples": [{
        "config": {"base_network": "10.20.0.0/16", "growth_factor": 1.25,
                   "allocation_strategy": "vlsm", "gateway_position": "first",
                   "groups": [{"name": "Server", "hosts": 40, "vlan_id": 100, "access": ["Drucker"]},
                              {"name": "Drucker", "hosts": 12}]},
        "overrides": {},
    }]})
```

In Swagger UI kann man mit **„Try it out“** das Beispiel direkt abschicken. Das ist für die Akzeptanz einer API Gold wert.

### 7.9 Nützliche Pydantic-Methoden

| Methode | Zweck |
| --- | --- |
| `model.model_dump()` | Modell → `dict` |
| `model.model_dump(exclude_none=True)` | ohne `None`-Felder |
| `model.model_dump_json()` | Modell → JSON-String |
| `Model.model_validate(data)` | `dict` → Modell (mit Validierung) |
| `Model.model_validate_json(text)` | JSON-String → Modell |
| `Model.model_json_schema()` | JSON-Schema des Modells |
| `model.model_copy(update={...})` | Kopie mit Änderungen |

---

## 8. Response-Modelle und Antworten

### 8.1 `response_model`

```python
@app.post("/items", response_model=ItemPublic)
def create_item(item: ItemCreate) -> Any:
    return db_save(item)        # z. B. mit Passwort-Hash – wird herausgefiltert
```

`response_model` bewirkt:

1. **Filtern:** nur die im Modell deklarierten Felder verlassen die API (Schutz vor versehentlichem Datenleck).
2. **Validieren:** der Rückgabewert wird gegen das Modell geprüft – Programmierfehler fallen früh auf (HTTP 500).
3. **Serialisieren:** `datetime`, `Decimal`, `UUID` usw. werden JSON-tauglich.
4. **Dokumentieren:** das Schema erscheint in OpenAPI unter dem Statuscode 200.

Seit FastAPI 0.89 reicht oft die **Rückgabe-Typannotation** (`-> ItemPublic`). Gibt die Funktion – wie in subnetplan – ein `dict` zurück, nutzt man `response_model=` und annotiert den Rückgabetyp als `dict[str, Any]`:

```python
@app.post(f"{API_PREFIX}/plans", response_model=PlanResponse, response_model_exclude_none=True, responses=COMMON_RESPONSES)
def create_plan(request: ConfigRequest) -> dict[str, Any]:
    _, plan = _make_plan(request.config, request.overrides)
    return _plan_payload(plan)
```

Hier gibt die Fachbibliothek JSON-Text zurück (`render_json(plan)`), der Helfer wandelt ihn in ein `dict` (`json.loads`), und FastAPI validiert dieses `dict` anschließend gegen `PlanResponse`. So stimmt die **veröffentlichte Doku garantiert mit der tatsächlichen Antwort überein**: Weicht die Fachbibliothek vom Schema ab, schlägt es sofort auf (Tests und HTTP 500) statt still.

### 8.2 Steuerung der Serialisierung

| Parameter | Wirkung |
| --- | --- |
| `response_model_exclude_none=True` | Felder mit Wert `None` entfallen |
| `response_model_exclude_unset=True` | nur explizit gesetzte Felder |
| `response_model_exclude_defaults=True` | Felder mit Standardwert entfallen |
| `response_model_include={"a","b"}` / `response_model_exclude={"x"}` | Feldauswahl |
| `response_model_by_alias=False` | Feldnamen statt Aliasse |

**So macht es subnetplan:** `response_model_exclude_none=True` steht bei `plans`, `extend`, `verify` und `verdict`. Damit erscheinen optionale Felder wie `site`, `services`, `pools`, `reserved`, `site_blocks` **nur, wenn sie wirklich Inhalt haben**:

```python
def test_plan_response_contains_services_only_when_restricted():
    plain = client.post("/api/v1/plans", json={"config": CONFIG}).json()
    assert all(not item.get("services") for item in plain["subnets"])
```

Achtung, Nebenwirkung: `exclude_none` entfernt **alle** `None`-Werte, auch solche, die fachlich „leer, aber vorhanden“ bedeuten (z. B. `gateway: null`). Das Projekt akzeptiert das; für Clients heißt es: Felder immer mit „Standardwert, falls fehlend“ lesen.

### 8.3 Eigene Antworttypen

Normalerweise antwortet FastAPI mit `JSONResponse`. Alternativen:

```python
from fastapi.responses import (
    JSONResponse, PlainTextResponse, HTMLResponse,
    RedirectResponse, FileResponse, StreamingResponse,
)

@app.get("/plain", response_class=PlainTextResponse)
def plain() -> str:
    return "nur Text"

@app.get("/legacy")
def legacy() -> RedirectResponse:
    return RedirectResponse("/api/v1/health", status_code=308)
```

Gibt man eine `Response` direkt zurück, **umgeht** man `response_model`, Filterung und Validierung.

Ein Hinweis zum Projekt: Der Firewall-Endpunkt gibt die Regeln **als JSON-Feld** `rules` zurück (`{"firewall": "...", "rules": "table inet …"}`), nicht als `text/plain`. Das ist komfortabel für Clients, die weitere Metadaten erwarten, und bleibt im einheitlichen JSON-Vertrag.

### 8.4 Zusätzliche Antworten dokumentieren (`responses`)

`responses` beschreibt Statuscodes **zusätzlich** zum Standardfall. Das beeinflusst nur die Doku, nicht das Laufzeitverhalten.

```python
COMMON_RESPONSES = {
    422: {
        "model": ErrorResponse | ValidationErrorResponse,
        "description": "Fachliche Konfigurations- oder Planungsfehler (Schema `error`) oder ungültige Request-Felder (Schema `detail`).",
        "content": {"application/json": {"examples": {
            "plan_error": {
                "summary": "Fachlicher Planungsfehler",
                "value": {"error": {"type": "plan_error", "message": "Pflichtfeld 'base_network' fehlt"}},
            },
            "request_validation": {
                "summary": "Fehlerhafte Request-Validierung",
                "value": {"detail": [{"loc": ["body", "firewall"], "msg": "Field required", "type": "missing"}]},
            },
        }}},
    }
}
```

Das Muster:

- Eine **wiederverwendbare Konstante** (`COMMON_RESPONSES`) wird an jeden Endpunkt gehängt (`responses=COMMON_RESPONSES`).
- Ein **Union-Modell** (`ErrorResponse | ValidationErrorResponse`) dokumentiert, dass 422 **zwei verschiedene** Formen haben kann – genau das passiert im Projekt (fachlicher Fehler vs. Pydantic-Fehler).
- **Benannte Beispiele** (`examples`) zeigen beide Formen in Swagger UI.
- Zusammenführen mit Zusatz-Codes über Dict-Entpacken:

```python
responses={
    **COMMON_RESPONSES,
    503: {"model": ServiceUnavailableResponse, "description": "Der Subgraph Algorithmus ist nicht verfügbar."},
},
```

### 8.5 Response-Modelle als Schnittstellenvertrag

Die Antwortmodelle in subnetplan sind durchgehend **dokumentiert**, z. B.:

```python
class VerificationResponse(BaseModel):
    valid: bool = Field(description="`true`, wenn alle geprüften Planeigenschaften erfüllt sind.", examples=[True])
    problems: list[str] = Field(description="Liste der gefundenen Verletzungen; bei gültigem Plan leer.", examples=[[]])
    plan: PlanResponse = Field(description="Der geprüfte Plan im JSON-Format.")
```

Das ist die Referenz, wie man Antwortmodelle anlegt: **Jedes Feld hat eine Beschreibung und ein Beispiel.** Verschachtelte Modelle (`PlanResponse` in `VerificationResponse`, `NetworkPair` in `RoundtripCheck`) halten die Definitionen klein und wiederverwendbar.

### 8.6 Header, Cookies und Statuscode zur Laufzeit setzen

```python
from fastapi import Response

@app.get("/x")
def x(response: Response) -> dict:
    response.headers["X-Cache"] = "miss"
    response.set_cookie("seen", "1", httponly=True, samesite="lax")
    response.status_code = 202
    return {"ok": True}
```

---

## 9. Fehlerbehandlung

### 9.1 Die drei Fehlerquellen

| Quelle | Auslöser | Standardantwort |
| --- | --- | --- |
| `RequestValidationError` | Request passt nicht zum Schema | 422, `{"detail": [ … ]}` |
| `HTTPException` | du wirfst sie bewusst | gewählter Code, `{"detail": …}` |
| Unbehandelte Exception | Programmierfehler | 500, „Internal Server Error“ |

### 9.2 `HTTPException`

```python
from fastapi import HTTPException

@app.get("/items/{item_id}")
def read_item(item_id: int) -> dict:
    if item_id not in DB:
        raise HTTPException(
            status_code=404,
            detail="Item nicht gefunden",
            headers={"X-Error": "not-found"},
        )
    return DB[item_id]
```

`detail` darf ein beliebiges JSON-taugliches Objekt sein.

**So macht es subnetplan** – genau einmal, für eine **optionale Abhängigkeit**:

```python
def compare(request: CompareRequest) -> dict[str, Any]:
    _, plan_a = _make_plan(request.config_a, request.overrides_a)
    _, plan_b = _make_plan(request.config_b, request.overrides_b)
    try:
        from subnetplan.policy import compare_policies
    except ImportError as exc:
        raise HTTPException(status_code=503, detail="Richtlinienvergleich ist nicht verfügbar") from exc
    return asdict(compare_policies(plan_a, plan_b))
```

Das Muster ist lehrreich:

- Der Import erfolgt **lazy innerhalb der Funktion**. Fehlt das Paket `subgraph`, startet die API trotzdem; nur dieser Endpunkt antwortet mit **503 Service Unavailable**.
- `raise … from exc` bewahrt die Ursache im Traceback.
- Der Fall ist im OpenAPI-Schema als `503` dokumentiert (`ServiceUnavailableResponse`).

### 9.3 Eigene Exceptions und `exception_handler` – das Herzstück

Das wichtigste Fehlerkonzept von subnetplan: **Eine Fachexception, ein zentraler Handler, ein einheitliches Fehlerformat.**

```python
# src/models.py (Fachbibliothek, ohne FastAPI)
class PlanError(ValueError):
    """Fehler in Konfiguration, Planung oder Ausgabe."""
```

```python
# web/api.py
@app.exception_handler(PlanError)
async def plan_error_handler(_: Request, exc: PlanError) -> JSONResponse:
    return JSONResponse(status_code=422, content={"error": {"type": "plan_error", "message": str(exc)}})
```

Wirkung: **Jede** Stelle in der Fachlogik darf `raise PlanError("…")` verwenden, auch tief in einer Hilfsfunktion. Die Endpunkte brauchen **kein einziges `try/except`**. Sie bleiben dadurch kurz und lesbar.

Der Test dazu:

```python
def test_domain_errors_have_consistent_json_response():
    response = client.post("/api/v1/plans", json={"config": {"groups": []}})
    assert response.status_code == 422
    assert response.json()["error"]["type"] == "plan_error"
    assert "base_network" in response.json()["error"]["message"]
```

Auch Hilfsfunktionen im API-Modul nutzen das Muster – `_network_list` übersetzt Python-Fehler in `PlanError`:

```python
def _network_list(values: list[str]) -> list[Any]:
    networks = []
    for value in values:
        try:
            networks.append(ipaddress.ip_network(value, strict=False))
        except ValueError as exc:
            raise PlanError(f"Ungültiges VPN-Netz {value!r}: {exc}") from exc
    return networks
```

Merke: Der Handler ist `async def`, weil Handler immer vom Event-Loop ausgeführt werden. Er hat keinen blockierenden Code.

### 9.4 Zwei Fehlerformate – und wie man das bewusst entscheidet

subnetplan liefert absichtlich **zwei** Formen von 422:

```json
// fachlicher Fehler (eigener Handler)
{"error": {"type": "plan_error", "message": "Pflichtfeld 'base_network' fehlt"}}

// Strukturfehler (FastAPI-Standard)
{"detail": [{"loc": ["body", "firewall"], "msg": "Field required", "type": "missing"}]}
```

Beides wird in der OpenAPI-Doku erklärt (siehe 8.4). Wer **ein einziges** Format möchte, überschreibt den Standard-Handler:

```python
from fastapi.exceptions import RequestValidationError

@app.exception_handler(RequestValidationError)
async def validation_handler(_: Request, exc: RequestValidationError) -> JSONResponse:
    return JSONResponse(
        status_code=422,
        content={"error": {"type": "validation_error", "message": "Ungültige Eingabe", "details": exc.errors()}},
    )
```

Hinweis: `exc.errors()` kann nicht-serialisierbare Objekte enthalten (z. B. die Original-Exception in `ctx`); in dem Fall hilft `fastapi.encoders.jsonable_encoder`.

### 9.5 Globale Absicherung gegen 500er

```python
import logging
logger = logging.getLogger("api")

@app.exception_handler(Exception)
async def unhandled(_: Request, exc: Exception) -> JSONResponse:
    logger.exception("Unbehandelter Fehler")
    return JSONResponse(status_code=500, content={"error": {"type": "internal_error", "message": "Interner Fehler"}})
```

Interne Details (Stacktrace, SQL) gehören **ins Log**, nie in die Antwort.

### 9.6 Mehrere eigene Fehlerklassen

Wächst die API, lohnt sich eine kleine Hierarchie, die auf Statuscodes abgebildet wird:

```python
class AppError(Exception):
    status_code = 400
    error_type = "app_error"

class NotFoundError(AppError):
    status_code = 404
    error_type = "not_found"

class ConflictError(AppError):
    status_code = 409
    error_type = "conflict"

@app.exception_handler(AppError)
async def app_error_handler(_: Request, exc: AppError) -> JSONResponse:
    return JSONResponse(status_code=exc.status_code,
                        content={"error": {"type": exc.error_type, "message": str(exc)}})
```

### 9.7 Welcher Statuscode für welchen Fehler?

subnetplan wählt für „der Request ist syntaktisch gültiges JSON, aber fachlich unmöglich“ den Code **422 Unprocessable Content**. Das ist eine gute Faustregel:

| Situation | Code |
| --- | --- |
| JSON/Typen falsch | 422 (automatisch) |
| JSON gültig, fachlich unmöglich (z. B. Netz zu klein) | 422 |
| Ressource existiert nicht | 404 |
| Zustandskonflikt (Duplikat) | 409 |
| Optionale Funktion fehlt/Abhängigkeit down | 503 |
| Programmierfehler | 500 |

---

## 10. OpenAPI, Swagger UI und ReDoc

FastAPI erzeugt aus deinem Code ein **OpenAPI-Schema**. Es ist der Vertrag deiner API: Aus ihm entstehen interaktive Doku, Client-SDKs, Mock-Server und Tests.

### 10.1 Wo es herkommt

| Im Code | Im Schema |
| --- | --- |
| `FastAPI(title=, summary=, description=, version=)` | `info` |
| `openapi_tags=[…]` | `tags` mit Beschreibung |
| Dekorator `summary=`, `description=`, `tags=` | Operation |
| Docstring der Funktion | Beschreibung (falls kein `description=`) |
| Pydantic-Modelle | `components.schemas` |
| `Field(description=, examples=)` | Feldbeschreibung/-beispiele |
| `responses=` | weitere Antwortcodes |
| Parameter-Typen und `Query`/`Path`/… | `parameters` |

### 10.2 Dokumentation für Menschen schreiben

subnetplan ist ein Vorbild für **vollständig beschriebene Operationen**. Jede Operation hat `summary` **und** `description`, jedes Feld eine `description`, viele Felder ein `examples`. Das stellt der Test sogar sicher:

```python
def test_openapi_has_descriptive_operations_and_examples():
    schema = app.openapi()
    operations = [
        operation
        for path in schema["paths"].values()
        for operation in path.values()
        if isinstance(operation, dict) and "summary" in operation
    ]

    assert len(operations) == 12
    assert all(operation.get("description") for operation in operations)
    assert "application/json" in schema["paths"]["/api/v1/plans"]["post"]["requestBody"]["content"]
    assert schema["paths"]["/api/v1/plans"]["post"]["responses"]["422"]["description"]
    assert schema["paths"]["/api/v1/plans/compare"]["post"]["responses"]["503"]["description"]
```

> **Dokumentation als Testgegenstand**: Wird ein neuer Endpunkt ohne Beschreibung hinzugefügt, schlägt der Test fehl. Dieses Muster lässt sich 1:1 übernehmen.

Das zweite Beispiel prüft, dass ein neues Feld im Schema auftaucht:

```python
assert "service_mismatch" in schema["components"]["schemas"]["RoundtripCheck"]["properties"]
```

### 10.3 Markdown in Beschreibungen

`description` (App und Operation) und `Field(description=…)` unterstützen Markdown. subnetplan nutzt Überschriften (`## Ablauf`), nummerierte Listen, Code-Format (`` `base_network` ``) und Fettdruck (`**422**`). Für mehrzeilige Texte ist ein dreifach gequoteter String üblich. Vorsicht bei der Einrückung: Markdown behandelt Zeilen mit vier oder mehr führenden Leerzeichen als Codeblock; bei eingerückten Texten (wie in subnetplan) sollte man das im gerenderten Ergebnis in Swagger UI kontrollieren oder den Text mit `inspect.cleandoc` aufbereiten. Bei Docstrings gilt: Alles ab einem Form-Feed-Zeichen (`\f`) wird nicht in die Doku übernommen.

### 10.4 Beispiele

```python
class Item(BaseModel):
    model_config = ConfigDict(json_schema_extra={
        "examples": [{"name": "Kabel", "price": 3.5}]
    })
```

Oder pro Parameter:

```python
q: Annotated[str, Query(examples=["router"])]
item: Annotated[Item, Body(examples=[{"name": "Kabel", "price": 3.5}])]
```

Mehrere benannte Beispiele für den Body: `Body(openapi_examples={"normal": {"summary": "…", "value": {…}}})`.

### 10.5 Die Doku-Oberflächen konfigurieren

```python
app = FastAPI(
    docs_url="/docs",             # None schaltet Swagger UI ab
    redoc_url="/redoc",           # None schaltet ReDoc ab
    openapi_url="/openapi.json",  # None schaltet die ganze Doku ab
    swagger_ui_parameters={"defaultModelsExpandDepth": -1, "persistAuthorization": True},
    contact={"name": "Team", "email": "team@example.org"},
    license_info={"name": "MIT"},
    terms_of_service="https://example.org/tos",
    servers=[{"url": "https://api.example.org", "description": "Produktion"}],
)
```

Für interne APIs ohne öffentliche Doku: `openapi_url=None`.

### 10.6 Schema exportieren

```python
import json
from web.api import app

with open("openapi.json", "w", encoding="utf-8") as f:
    json.dump(app.openapi(), f, indent=2, ensure_ascii=False)
```

Daraus lassen sich Clients generieren (z. B. mit `openapi-generator`, `openapi-python-client`, `openapi-typescript`). Damit generierte Methodennamen schön werden, vergibt man `operation_id`s – oder lässt sie zentral erzeugen:

```python
from fastapi.routing import APIRoute

def use_route_names_as_operation_ids(app: FastAPI) -> None:
    for route in app.routes:
        if isinstance(route, APIRoute):
            route.operation_id = route.name     # Funktionsname

use_route_names_as_operation_ids(app)           # NACH allen Routen aufrufen
```

### 10.7 Veraltete Endpunkte kennzeichnen

```python
@app.get("/v1/old", deprecated=True)
def old(): ...
```

Swagger UI zeigt ihn durchgestrichen.

---

## 11. Dependency Injection

Dependency Injection (DI) ist FastAPIs Mechanismus, **wiederverwendbare Bausteine** in Endpunkte einzuspeisen. subnetplan nutzt es nicht, weil jede Anfrage selbsterklärend ist – aber in fast jedem realen Projekt ist es unverzichtbar.

### 11.1 Das Grundprinzip

```python
from typing import Annotated
from fastapi import Depends

def pagination(skip: int = 0, limit: int = 100) -> dict[str, int]:
    return {"skip": skip, "limit": limit}

Paging = Annotated[dict[str, int], Depends(pagination)]

@app.get("/items")
def items(paging: Paging) -> dict:
    return paging

@app.get("/users")
def users(paging: Paging) -> dict:
    return paging
```

FastAPI ruft `pagination` pro Request auf, liest dessen Parameter (`skip`, `limit` als Query) und übergibt das Ergebnis. Die Parameter erscheinen **automatisch in der OpenAPI-Doku** beider Endpunkte. Der Alias `Paging = Annotated[…]` hält die Signaturen kurz.

### 11.2 Klassen als Abhängigkeiten

```python
class Pagination:
    def __init__(self, skip: int = 0, limit: int = 100):
        self.skip, self.limit = skip, limit

@app.get("/items")
def items(p: Annotated[Pagination, Depends()]) -> dict:
    return {"skip": p.skip, "limit": p.limit}
```

### 11.3 Abhängigkeiten mit Aufräumen (`yield`)

Typisch für Datenbank-Sessions oder Dateien:

```python
from collections.abc import Iterator
from sqlalchemy.orm import Session

def get_session() -> Iterator[Session]:
    session = SessionLocal()
    try:
        yield session          # wird dem Endpunkt übergeben
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()        # Aufräumen

SessionDep = Annotated[Session, Depends(get_session)]

@app.get("/users/{user_id}")
def get_user(user_id: int, session: SessionDep) -> UserPublic:
    ...
```

### 11.4 Unterabhängigkeiten

Abhängigkeiten können selbst Abhängigkeiten haben. FastAPI löst den Baum auf und **cacht** pro Request: Wird dieselbe Abhängigkeit mehrfach benötigt, läuft sie nur einmal (`use_cache=False` schaltet das ab).

```python
def get_token(authorization: Annotated[str, Header()]) -> str:
    return authorization.removeprefix("Bearer ")

def get_current_user(token: Annotated[str, Depends(get_token)], session: SessionDep) -> User:
    user = lookup(session, token)
    if user is None:
        raise HTTPException(status_code=401, detail="Ungültiges Token")
    return user

CurrentUser = Annotated[User, Depends(get_current_user)]

@app.get("/me")
def me(user: CurrentUser) -> UserPublic:
    return user
```

### 11.5 Abhängigkeiten ohne Rückgabewert

Wenn nur eine **Prüfung** nötig ist:

```python
def require_admin(user: CurrentUser) -> None:
    if not user.is_admin:
        raise HTTPException(status_code=403, detail="Nur für Administratoren")

@app.delete("/users/{id}", dependencies=[Depends(require_admin)])
def delete_user(id: int): ...
```

Auf Router- oder App-Ebene wirkt `dependencies=[…]` für **alle** enthaltenen Endpunkte (siehe Kapitel 13).

### 11.6 Muster aus subnetplan als Abhängigkeit

Der Helfer `_make_plan` wird in zehn Endpunkten aufgerufen:

```python
def _make_plan(config: dict[str, Any], overrides: dict[str, Any] | None = None) -> tuple[Any, Plan]:
    cfg = build_config(config, overrides)
    return cfg, plan_subnets(cfg)
```

Das Projekt löst die Wiederverwendung mit einer **normalen Funktion** statt mit `Depends`. Das ist völlig richtig: Hier gibt es keine Ressource mit Lebenszyklus und keine Request-Parameter, die `Depends` zusätzlich auflösen müsste. Als Faustregel:

| Nimm … | wenn … |
| --- | --- |
| normale Hilfsfunktion | reine Berechnung auf bereits vorhandenen Daten |
| `Depends` | Ressource mit Auf-/Abbau (DB, HTTP-Client), Authentifizierung, Header-/Query-Parameter, Überschreibbarkeit im Test |

Eine äquivalente DI-Variante (nur zur Anschauung):

```python
def planned(request: ConfigRequest) -> tuple[Any, Plan]:
    return _make_plan(request.config, request.overrides)

PlannedDep = Annotated[tuple[Any, Plan], Depends(planned)]

@app.post(f"{API_PREFIX}/plans/verify")
def verify(planned: PlannedDep) -> dict[str, Any]:
    cfg, plan = planned
    ...
```

Reizvoll, aber hier kein Gewinn; der Body-Parameter müsste ohnehin im Schema auftauchen.

### 11.6a Abhängigkeiten im Test überschreiben

```python
def fake_user() -> User:
    return User(id=1, name="test", is_admin=True)

app.dependency_overrides[get_current_user] = fake_user
# … Tests …
app.dependency_overrides.clear()
```

Das ist der Hauptgrund für DI in größeren Projekten: Datenbank, Authentifizierung und externe Dienste lassen sich im Test **ersetzen**, ohne Code zu ändern.

### 11.7 Globale Abhängigkeiten

```python
app = FastAPI(dependencies=[Depends(verify_api_key)])   # schützt ALLE Endpunkte
```

---

## 12. async oder def? Nebenläufigkeit verstehen

### 12.1 Die Regel

| Endpunkt | Wie FastAPI ihn ausführt | Wann verwenden |
| --- | --- | --- |
| `async def` | direkt im Event-Loop | wenn der Code **`await`-bare** Bibliotheken nutzt (httpx.AsyncClient, asyncpg, aiofiles …) |
| `def` | in einem **Thread-Pool** (Starlette `run_in_threadpool`) | wenn der Code **blockiert** oder CPU-lastig ist, oder einfach synchrone Bibliotheken nutzt |

Ein `async def`-Endpunkt, der **blockierenden** Code ausführt (`time.sleep`, `requests.get`, schwere Berechnung, synchrone DB-Treiber), **friert den gesamten Server ein**, weil der Event-Loop blockiert ist. Ein `def`-Endpunkt hingegen blockiert nur seinen Thread.

### 12.2 So macht es subnetplan

**Alle zwölf Endpunkte sind synchrone `def`-Funktionen.** Das ist die richtige Wahl: Planung, Rückübersetzung von Firewall-Regeln und Richtlinienvergleich sind **CPU-gebundene** Berechnungen (NumPy, Graphalgorithmen) ohne I/O. Als `async def` würden sie den Event-Loop blockieren; als `def` laufen sie im Thread-Pool, und andere Requests (etwa `/health`) bleiben erreichbar.

```python
@app.get(f"{API_PREFIX}/health", ...)
def health() -> dict[str, str]:        # def, nicht async def
    return {"status": "ok"}
```

Einzige `async def`: der **Exception-Handler** `plan_error_handler` – er enthält keinen blockierenden Code und läuft im Loop.

### 12.3 Grenzen des Thread-Pools

- Der Pool hat standardmäßig **40 Threads**. Mehr gleichzeitige blockierende Requests warten in einer Schlange.
- Wegen des **GIL** parallelisieren Threads reine Python-Berechnung nicht. Für echte Parallelität bei CPU-Last: mehrere **Worker-Prozesse** (siehe Kapitel 21/22) oder `ProcessPoolExecutor`.
- NumPy gibt den GIL bei vielen Operationen frei, deshalb profitieren auch Thread-Pools davon teilweise.

### 12.4 Typische `async`-Beispiele

```python
import httpx

@app.get("/weather")
async def weather(city: str) -> dict:
    async with httpx.AsyncClient(timeout=5) as client:
        r = await client.get("https://api.example.org/weather", params={"q": city})
        r.raise_for_status()
        return r.json()
```

Blockierende Arbeit aus `async def` auslagern:

```python
from fastapi.concurrency import run_in_threadpool

@app.post("/heavy")
async def heavy(data: Data) -> dict:
    result = await run_in_threadpool(compute, data)     # compute ist eine normale Funktion
    return {"result": result}
```

---

## 13. Anwendung strukturieren mit APIRouter

### 13.1 Warum Router?

subnetplan hat 12 Endpunkte in einer Datei – das ist noch überschaubar (567 Zeilen inklusive aller Modelle). Ab etwa 20 Endpunkten oder mehreren Fachbereichen trennt man mit `APIRouter`.

### 13.2 Ein Router

```python
# app/routers/plans.py
from fastapi import APIRouter

router = APIRouter(
    prefix="/plans",
    tags=["plans"],
    responses={422: {"description": "Validierungsfehler"}},
)

@router.post("", summary="Subnetzplan berechnen")
def create_plan(request: ConfigRequest) -> PlanResponse: ...

@router.post("/verify")
def verify(request: ConfigRequest) -> VerificationResponse: ...
```

```python
# app/main.py
from fastapi import FastAPI
from app.routers import plans, analysis, meta

app = FastAPI(title="subnetplan API", version="4.0.0")

for r in (meta.router, plans.router, analysis.router):
    app.include_router(r, prefix="/api/v1")
```

### 13.3 subnetplan auf Router umgestellt (Vorschlag)

Der heutige Stil `f"{API_PREFIX}/plans/verify"` lässt sich ohne Verhaltensänderung so umbauen:

```python
# web/routers/plans.py
router = APIRouter(prefix="/plans")

@router.post("/verify", tags=["analysis"], response_model=VerificationResponse,
             response_model_exclude_none=True, responses=COMMON_RESPONSES)
def verify(request: ConfigRequest) -> dict[str, Any]: ...

# web/api.py
app.include_router(meta_router, prefix=API_PREFIX)
app.include_router(plans_router, prefix=API_PREFIX)
```

Die Pfade bleiben `/api/v1/plans/verify`; die Tests laufen unverändert, weil sie nur HTTP sehen. Genau diese Eigenschaft – **Tests gegen die HTTP-Schnittstelle** statt gegen interne Funktionen – macht solche Umbauten risikoarm.

### 13.4 Router-Optionen

| Option bei `APIRouter(...)` oder `include_router(...)` | Wirkung |
| --- | --- |
| `prefix` | Pfadpräfix |
| `tags` | Standard-Tags |
| `dependencies` | Abhängigkeiten für alle Routen (z. B. Authentifizierung) |
| `responses` | gemeinsame zusätzliche Antworten |
| `deprecated` | alle Routen als veraltet markieren |
| `include_in_schema` | aus der Doku ausblenden |

### 13.5 Versionierung

```python
v1 = APIRouter(prefix="/api/v1")
v2 = APIRouter(prefix="/api/v2")
v1.include_router(plans_v1.router)
v2.include_router(plans_v2.router)
app.include_router(v1)
app.include_router(v2)
```

Alternative: komplett getrennte Sub-Apps mit `app.mount("/v1", v1_app)`; jede hat dann eigene Doku unter `/v1/docs`.

Sinnvolle Versionierungsregeln: abwärtskompatible Erweiterungen (neue optionale Felder, neue Endpunkte) brauchen **keine** neue Version; Umbenennen, Entfernen oder geänderte Bedeutung schon. Der Wert `version="4.0.0"` in `FastAPI(...)` beschreibt die Release-Version, `/v1` die Vertragsversion – beides darf unterschiedlich laufen.

---

## 14. Middleware, CORS und weitere Querschnittsthemen

### 14.1 Eigene Middleware

```python
import time
from fastapi import Request

@app.middleware("http")
async def timing(request: Request, call_next):
    start = time.perf_counter()
    response = await call_next(request)
    response.headers["X-Process-Time"] = f"{time.perf_counter() - start:.4f}"
    return response
```

Die zuletzt hinzugefügte Middleware ist die **äußerste** (sie sieht den Request zuerst und die Response zuletzt).

### 14.2 CORS

Wird die API aus einem Browser auf einer **anderen Origin** aufgerufen (z. B. Frontend auf `https://app.example.org`, API auf `https://api.example.org`), braucht sie CORS:

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.example.org"],
    allow_credentials=True,
    allow_methods=["GET", "POST"],
    allow_headers=["Authorization", "Content-Type"],
)
```

Fallen:

- `allow_origins=["*"]` zusammen mit `allow_credentials=True` ist nach Browserstandard **nicht zulässig**; konkrete Origins aufführen.
- Der Browser sendet bei JSON-`POST` einen **Preflight** (`OPTIONS`); die Middleware beantwortet ihn automatisch.

**So macht es subnetplan:** keine CORS-Konfiguration. Die API bindet standardmäßig nur an `127.0.0.1` und wird offenbar serverseitig oder über einen Reverse-Proxy genutzt. Soll ein Browser-Frontend direkt zugreifen, muss `CORSMiddleware` ergänzt werden.

### 14.3 Weitere eingebaute Middleware

```python
from fastapi.middleware.gzip import GZipMiddleware
from fastapi.middleware.trustedhost import TrustedHostMiddleware
from fastapi.middleware.httpsredirect import HTTPSRedirectMiddleware

app.add_middleware(GZipMiddleware, minimum_size=1000)     # große JSON-Antworten komprimieren
app.add_middleware(TrustedHostMiddleware, allowed_hosts=["api.example.org"])
```

Gerade bei umfangreichen Plänen (viele Subnetze, lange Regeltexte) spart GZip spürbar Bandbreite.

### 14.4 Request-ID und Logging

```python
import uuid, logging
logger = logging.getLogger("api")

@app.middleware("http")
async def request_id(request: Request, call_next):
    rid = request.headers.get("X-Request-ID", str(uuid.uuid4()))
    response = await call_next(request)
    response.headers["X-Request-ID"] = rid
    logger.info("%s %s -> %s [%s]", request.method, request.url.path, response.status_code, rid)
    return response
```

### 14.5 Größenbegrenzung des Bodys

FastAPI begrenzt die Body-Größe **nicht selbst**. Bei öffentlich erreichbaren APIs sollte der Reverse-Proxy (z. B. nginx `client_max_body_size`) oder eine Middleware das tun, sonst kann ein Client riesige JSON-Dokumente schicken. Bei subnetplan relevant, da `config` ein beliebig großes `dict` ist.

---

## 15. Lifespan, Konfiguration und Anwendungszustand

### 15.1 Lifespan

Code, der beim Start und Ende **einmal** laufen soll (DB-Pools öffnen, Modelle laden, HTTP-Clients anlegen):

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Start
    app.state.http = httpx.AsyncClient(timeout=5)
    yield
    # Ende
    await app.state.http.aclose()

app = FastAPI(lifespan=lifespan)
```

Die älteren Dekoratoren `@app.on_event("startup")`/`("shutdown")` sind veraltet; `lifespan` ersetzt sie.

Wichtig: Der `TestClient` führt den Lifespan nur aus, wenn man ihn als Kontextmanager nutzt (`with TestClient(app) as client:`).

### 15.2 Zustand

Auf Instanzen leben, was mit der Anwendung geteilt wird:

```python
@app.get("/ping")
async def ping(request: Request):
    return {"client": type(request.app.state.http).__name__}
```

**So macht es subnetplan:** Es gibt **keinen Zustand**. Die Dokumentation betont: *„Jede Planoperation erhält ihre Konfiguration im Request; Pläne werden nicht serverseitig gespeichert.“* Das ist ein Architekturmerkmal mit großen Vorteilen:

- beliebig **horizontal skalierbar** (jeder Worker/Container ist austauschbar),
- **keine Race Conditions**, keine Cache-Invalidierung, kein Datenbank-Setup,
- Tests brauchen **kein Setup/Teardown**,
- Das Modell `extend` funktioniert trotzdem: Der Client schickt den früheren `plan` zurück, den er von `POST /plans` erhalten hat.

Der Preis: Der Client trägt den Zustand (und muss ihn bei jedem Request mitsenden).

### 15.3 Einstellungen aus der Umgebung

Das Projekt liest zwei Umgebungsvariablen direkt:

```python
def run() -> None:
    """Console entry point for ``subnetplan-api``."""
    import uvicorn

    uvicorn.run(
        "web.api:app",
        host=os.getenv("SUBNETPLAN_API_HOST", "127.0.0.1"),
        port=int(os.getenv("SUBNETPLAN_API_PORT", "8000")),
    )
```

Sichere Standardwerte (`127.0.0.1` statt `0.0.0.0`) verhindern, dass die API versehentlich im Netz erreichbar ist. Für mehr Einstellungen empfiehlt sich **pydantic-settings**:

```bash
pip install pydantic-settings
```

```python
from functools import lru_cache
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="SUBNETPLAN_API_", env_file=".env")

    host: str = "127.0.0.1"
    port: int = 8000
    log_level: str = "info"
    cors_origins: list[str] = []
    max_groups: int = 500

@lru_cache
def get_settings() -> Settings:
    return Settings()

# Als Abhängigkeit (im Test überschreibbar):
SettingsDep = Annotated[Settings, Depends(get_settings)]
```

Dann gilt `SUBNETPLAN_API_PORT=9000` automatisch, inklusive Typprüfung (`port` muss Zahl sein).

---

## 16. Background Tasks

Für kurze Nacharbeiten **nach** der Antwort (E-Mail versenden, Log schreiben):

```python
from fastapi import BackgroundTasks

def write_audit(user: str, action: str) -> None:
    with open("audit.log", "a") as f:
        f.write(f"{user}: {action}\n")

@app.post("/plans")
def create_plan(request: ConfigRequest, background: BackgroundTasks) -> dict:
    result = compute(request)
    background.add_task(write_audit, "anna", "plan created")
    return result
```

Grenzen: Die Aufgabe läuft im selben Prozess; stürzt er ab, ist sie weg. Für lange, kritische oder verteilte Jobs gehören **Task-Queues** her (Celery, RQ, Arq, Dramatiq). subnetplan braucht das nicht, da jede Berechnung synchron in Millisekunden bis Sekunden endet.

---

## 17. Sicherheit: API-Key, OAuth2 und JWT

**So macht es subnetplan:** Die API besitzt **keine Authentifizierung** und läuft standardmäßig nur lokal (`127.0.0.1`). Die Firewall-Ausgaben sind ausdrücklich nur Textvorlagen, die API ändert kein System. Das ist ein gültiges Sicherheitsmodell für ein lokales Werkzeug. Sobald die API im Netzwerk erreichbar sein soll, **muss** man Zugriffsschutz ergänzen.

### 17.1 API-Key im Header

```python
import secrets
from fastapi import Security
from fastapi.security import APIKeyHeader

api_key_header = APIKeyHeader(name="X-API-Key", auto_error=False)

def verify_api_key(key: Annotated[str | None, Security(api_key_header)]) -> None:
    expected = get_settings().api_key
    if key is None or not secrets.compare_digest(key, expected):
        raise HTTPException(status_code=401, detail="Ungültiger API-Schlüssel",
                            headers={"WWW-Authenticate": "ApiKey"})

app = FastAPI(dependencies=[Depends(verify_api_key)])
```

`secrets.compare_digest` verhindert Timing-Angriffe. Swagger UI zeigt automatisch einen „Authorize“-Knopf.

### 17.2 OAuth2 mit Passwort-Flow und JWT

```bash
pip install pyjwt "pwdlib[argon2]"
```

```python
from datetime import datetime, timedelta, timezone
import jwt
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from pwdlib import PasswordHash

SECRET = "aus-der-umgebung-laden"
ALGORITHM = "HS256"
password_hash = PasswordHash.recommended()
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/token")

def create_token(sub: str, minutes: int = 30) -> str:
    exp = datetime.now(timezone.utc) + timedelta(minutes=minutes)
    return jwt.encode({"sub": sub, "exp": exp}, SECRET, algorithm=ALGORITHM)

@app.post("/api/v1/token")
def login(form: Annotated[OAuth2PasswordRequestForm, Depends()]) -> dict:
    user = USERS.get(form.username)
    if not user or not password_hash.verify(form.password, user["hash"]):
        raise HTTPException(status_code=401, detail="Falsche Zugangsdaten")
    return {"access_token": create_token(form.username), "token_type": "bearer"}

def current_user(token: Annotated[str, Depends(oauth2_scheme)]) -> str:
    try:
        payload = jwt.decode(token, SECRET, algorithms=[ALGORITHM])
    except jwt.PyJWTError:
        raise HTTPException(status_code=401, detail="Token ungültig oder abgelaufen",
                            headers={"WWW-Authenticate": "Bearer"})
    return payload["sub"]
```

### 17.3 Rollen und Berechtigungen (Scopes)

```python
from fastapi import Security
from fastapi.security import SecurityScopes

def require(scope: str):
    def check(user: Annotated[User, Depends(get_current_user)]) -> User:
        if scope not in user.scopes:
            raise HTTPException(status_code=403, detail=f"Berechtigung '{scope}' fehlt")
        return user
    return check

@app.post("/plans/firewall", dependencies=[Depends(require("firewall:render"))])
def firewall(...): ...
```

### 17.4 Sicherheits-Checkliste

- [ ] HTTPS erzwingen (am Reverse-Proxy)
- [ ] Authentifizierung für alles außer `health`
- [ ] Geheimnisse nur aus Umgebung/Secret-Store, nie im Code
- [ ] Passwörter nur mit modernen Hash-Verfahren (Argon2, bcrypt) speichern
- [ ] CORS auf konkrete Origins beschränken
- [ ] Body-Größe und Rate-Limit am Proxy begrenzen
- [ ] Fehlermeldungen ohne interne Details
- [ ] `/docs` in Produktion schützen oder abschalten
- [ ] Eingaben, die in Dateien, Kommandos oder Regeltexte fließen, **validieren**

Der letzte Punkt ist bei subnetplan besonders relevant: Gruppennamen landen in Firewall-Kommentaren und ACL-Namen. Deshalb begrenzt die Fachbibliothek sie per Regex (`NAME_RE = ^[\w\- ]+$` in `models.py`) auf harmlose Zeichen. Das ist **Eingabevalidierung als Injection-Schutz** – ein gutes Beispiel dafür, dass Sicherheit auch in der Fachlogik beginnt.

---

## 18. Datenbanken anbinden

subnetplan hat keine Datenbank. Wer FastAPI mit einer Datenbank nutzt, kombiniert typischerweise **SQLModel** oder **SQLAlchemy 2** mit Abhängigkeiten.

### 18.1 SQLModel-Beispiel

```bash
pip install sqlmodel
```

```python
from sqlmodel import SQLModel, Field, Session, create_engine, select

class PlanRecord(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    name: str = Field(index=True)
    base_network: str
    payload: str                      # JSON des Plans

engine = create_engine("sqlite:///plans.db", connect_args={"check_same_thread": False})

@asynccontextmanager
async def lifespan(app: FastAPI):
    SQLModel.metadata.create_all(engine)
    yield

app = FastAPI(lifespan=lifespan)

def get_session():
    with Session(engine) as session:
        yield session

SessionDep = Annotated[Session, Depends(get_session)]

@app.post("/stored-plans", status_code=201)
def store(name: str, plan: PlanResponse, session: SessionDep) -> PlanRecord:
    record = PlanRecord(name=name, base_network=plan.base_network, payload=plan.model_dump_json())
    session.add(record)
    session.commit()
    session.refresh(record)
    return record

@app.get("/stored-plans")
def list_plans(session: SessionDep, limit: int = 20) -> list[PlanRecord]:
    return list(session.exec(select(PlanRecord).limit(limit)))
```

### 18.2 Trennung der Modelle

In echten Projekten bewährt sich die Aufteilung:

| Modell | Zweck |
| --- | --- |
| `…Create` | Eingabe beim Anlegen (ohne `id`) |
| `…Update` | Teilaktualisierung (alle Felder optional) |
| `…Public` | Antwort (ohne interne/geheime Felder) |
| `…` (Tabellenmodell) | Datenbankzeile |

subnetplan zeigt die Idee bereits mit seiner Trennung in **Request-** und **Response-Modelle**.

### 18.3 Asynchrone Datenbank

Mit `async def` Endpunkten braucht man einen **asynchronen Treiber** (`asyncpg`, `aiosqlite`) und `AsyncSession` von SQLAlchemy. Mischt man `async def` mit einem synchronen Treiber, blockiert man den Event-Loop (siehe Kapitel 12).

### 18.4 Migrationen

Schemaänderungen mit **Alembic** versionieren; `create_all` ist nur für Prototypen gedacht.

---

## 19. WebSockets, Streaming, Dateien und statische Inhalte

### 19.1 WebSocket

```python
from fastapi import WebSocket, WebSocketDisconnect

@app.websocket("/ws")
async def ws_endpoint(ws: WebSocket):
    await ws.accept()
    try:
        while True:
            text = await ws.receive_text()
            await ws.send_text(f"Echo: {text}")
    except WebSocketDisconnect:
        pass
```

### 19.2 Streaming

```python
from fastapi.responses import StreamingResponse

def generate_lines():
    for i in range(1000):
        yield f"{i}\n"

@app.get("/stream")
def stream():
    return StreamingResponse(generate_lines(), media_type="text/plain")
```

Für **Server-Sent Events** (`text/event-stream`) liefert man Zeilen im Format `data: …\n\n`.

### 19.3 Dateien ausliefern

```python
from fastapi.responses import FileResponse

@app.get("/download/rules")
def download() -> FileResponse:
    return FileResponse("rules.txt", media_type="text/plain", filename="firewall-rules.txt")
```

Der Firewall-Endpunkt von subnetplan könnte so erweitert werden, dass er die Regeln wahlweise als Download liefert:

```python
from fastapi.responses import PlainTextResponse

@app.post("/api/v1/plans/firewall/download", response_class=PlainTextResponse)
def firewall_download(request: FirewallRequest) -> PlainTextResponse:
    _, plan = _make_plan(request.config, request.overrides)
    rules = render_firewall(plan, request.firewall, request.options.to_options())
    return PlainTextResponse(rules, headers={"Content-Disposition": f'attachment; filename="{request.firewall}.rules"'})
```

### 19.4 Statische Dateien

```python
from fastapi.staticfiles import StaticFiles
app.mount("/static", StaticFiles(directory="static"), name="static")
```

Typisch für ein Frontend-Build: `app.mount("/", StaticFiles(directory="dist", html=True), name="spa")` **nach** allen API-Routen, weil ein Mount auf `/` sonst alles abfängt.

---

## 20. Testen

### 20.1 `TestClient`

Der `TestClient` (Starlette, basiert auf `httpx`) ruft die App **im selben Prozess** auf – ohne Server, ohne Netzwerk, ohne Port. Er ist schnell und deterministisch.

```python
import pytest
from fastapi.testclient import TestClient
from web.api import app

pytestmark = pytest.mark.integration

CONFIG = {
    "base_network": "10.0.0.0/24",
    "groups": [
        {"name": "A", "hosts": 50, "access": ["B"]},
        {"name": "B", "hosts": 10},
    ],
}

client = TestClient(app)
```

Ein Modul-weiter Client reicht, solange die App **zustandslos** ist (wie hier). Bei Lifespan-Logik nutzt man `with TestClient(app) as client:` oder eine pytest-Fixture.

### 20.2 Ein typischer Test aus dem Projekt

```python
def test_meta_and_plan_analysis_routes():
    assert client.get("/api/v1/health").json() == {"status": "ok"}
    options = client.get("/api/v1/options").json()
    assert "nftables" in options["firewalls"]

    plan = client.post("/api/v1/plans", json={"config": CONFIG})
    assert plan.status_code == 200
    assert [item["group"] for item in plan.json()["subnets"]] == ["A", "B"]

    verified = client.post("/api/v1/plans/verify", json={"config": CONFIG}).json()
    assert verified["valid"] is True
```

Das Testmuster in fünf Punkten:

1. **Eine** gemeinsame Beispielkonfiguration (`CONFIG`) für alle Tests.
2. `client.post(url, json=…)` serialisiert und setzt den Content-Type.
3. Zuerst `status_code`, dann ausgewählte Inhalte prüfen.
4. Je Endpunkt mindestens ein **Erfolgsfall**.
5. Je Fehlerklasse mindestens ein **Fehlerfall** (422).

### 20.3 Hilfsfunktionen für Testdaten

```python
def _ports_config(ports):
    return {
        "base_network": "10.0.0.0/24",
        "groups": [
            {"name": "A", "hosts": 100, "access": [{"to": "B", "ports": ports}]},
            {"name": "B", "hosts": 100},
        ],
    }
```

Damit werden mehrere Tests lesbar: Sie variieren nur das, was sie testen wollen.

### 20.4 Fachliche Tests über die API

Die Dienst-Vergleichsfunktion wird mit allen vier Relationen getestet:

```python
def test_compare_services_subset_and_differences():
    narrow = _ports_config(["tcp/443"])
    wide = _ports_config(["tcp/443", "tcp/8000-8099"])
    body = client.post(
        "/api/v1/plans/compare-services", json={"config_a": narrow, "config_b": wide}
    ).json()
    assert body["relation"] == "a_in_b"
    assert body["a_in_b"] is True and body["b_in_a"] is False
    assert body["differences"] == [
        {"source": "A", "target": "B", "a": ["tcp/443"], "b": ["tcp/443", "tcp/8000-8099"]}
    ]
```

Hier wird eine **Teilmengenbeziehung** getestet: eine Richtung, die Gegenrichtung, Gleichheit, Unvergleichbarkeit und die Randfälle „kein Zugriff“ und „alle Dienste“. Gute API-Tests beschreiben **Verhalten**, nicht Implementierung.

### 20.5 Fehlerfälle

```python
def test_compare_services_invalid_config_returns_plan_error():
    response = client.post(
        "/api/v1/plans/compare-services",
        json={"config_a": _ports_config(["tcp/99999"]), "config_b": CONFIG},
    )
    assert response.status_code == 422
    assert response.json()["error"]["type"] == "plan_error"
```

### 20.6 pytest-Konfiguration des Projekts

```toml
[tool.pytest.ini_options]
minversion = "6.0"
testpaths = ["tests"]
pythonpath = [".", "src"]
addopts = "-q --cov=src --cov-report=html:doc/coverage --cov-report=term-missing"
markers = [
    "unit: unit test",
    "integration: component integration test",
    "slow: computationally expensive test",
]
```

- `pythonpath = [".", "src"]` macht `web.api` direkt importierbar, auch ohne Installation.
- **Marker** (`pytestmark = pytest.mark.integration`) erlauben `pytest -m integration` bzw. `pytest -m "not slow"`.
- Die API-Tests laufen mit Coverage (`--cov=src`, Zweigabdeckung) mit.

### 20.7 Asynchrone Tests

Für Code mit `async def` und Lifespan bietet sich `httpx.AsyncClient` mit `ASGITransport` an:

```python
import pytest, httpx

@pytest.mark.anyio
async def test_async():
    transport = httpx.ASGITransport(app=app)
    async with httpx.AsyncClient(transport=transport, base_url="http://test") as ac:
        r = await ac.get("/api/v1/health")
    assert r.status_code == 200
```

(Benötigt `anyio`-pytest-Plugin bzw. `pytest-asyncio`.)

### 20.8 Abhängigkeiten ersetzen

Siehe 11.6a. Fixture-Variante:

```python
@pytest.fixture
def client_as_admin():
    app.dependency_overrides[get_current_user] = lambda: User(id=1, is_admin=True)
    yield TestClient(app)
    app.dependency_overrides.clear()
```

### 20.9 Schema-Tests und Vertragstests

- Wie in 10.2 zeigt, kann man mit `app.openapi()` **die Doku selbst testen**.
- Für automatische Vertragsprüfung gegen das Schema: **Schemathesis** erzeugt aus dem OpenAPI-Schema zufällige Requests und findet 500er:

```bash
pip install schemathesis
schemathesis run http://127.0.0.1:8000/openapi.json
```

---

## 21. Betrieb und Deployment

### 21.1 Uvicorn

**Uvicorn** ist der ASGI-Server, der die FastAPI-App ausführt.

```bash
uvicorn web.api:app --host 127.0.0.1 --port 8000          # einfach
uvicorn web.api:app --reload                               # Entwicklung: Neustart bei Codeänderung
uvicorn web.api:app --workers 4                            # mehrere Prozesse
uvicorn web.api:app --proxy-headers --forwarded-allow-ips="10.0.0.5"   # hinter Reverse-Proxy
uvicorn web.api:app --log-level warning --no-access-log
```

| Option | Bedeutung |
| --- | --- |
| `--host` | Bind-Adresse; `127.0.0.1` = nur lokal, `0.0.0.0` = alle Schnittstellen |
| `--port` | Port |
| `--reload` | Dateiüberwachung, **nur** für Entwicklung |
| `--workers N` | N Prozesse (nicht kombinierbar mit `--reload`) |
| `--proxy-headers` | `X-Forwarded-*` beachten |
| `--root-path /api` | Pfadpräfix, wenn ein Proxy ihn abschneidet |
| `--ssl-keyfile`, `--ssl-certfile` | direktes TLS (meist übernimmt das der Proxy) |

### 21.2 Start über das Projekt-Kommando

subnetplan bringt einen **Konsolen-Entry-Point** mit:

```toml
[project.scripts]
subnetplan-api = "web.api:run"
```

```python
def run() -> None:
    import uvicorn
    uvicorn.run(
        "web.api:app",
        host=os.getenv("SUBNETPLAN_API_HOST", "127.0.0.1"),
        port=int(os.getenv("SUBNETPLAN_API_PORT", "8000")),
    )
```

Vorteil: `pip install .` genügt, danach funktioniert `subnetplan-api` überall. Der Import von `uvicorn` geschieht **innerhalb** von `run()`, damit das Modul auch ohne Uvicorn importierbar ist (z. B. in Tests). Der Import-String `"web.api:app"` (statt des Objekts) ist nötig, wenn man `workers`/`reload` nutzen will.

### 21.3 Mehrere Prozesse: Gunicorn mit Uvicorn-Workern

```bash
pip install gunicorn
gunicorn web.api:app -k uvicorn.workers.UvicornWorker -w 4 -b 0.0.0.0:8000 --timeout 60
```

Seit neueren Uvicorn-Versionen genügt auch `uvicorn --workers 4`, das seine Prozesse selbst überwacht. Faustformel für die Zahl: **ein Worker je CPU-Kern**, bei CPU-lastigen Aufgaben wie in subnetplan nicht deutlich mehr.

Da die App **zustandslos** ist, gibt es keine Synchronisationsprobleme zwischen Workern.

### 21.4 Docker

```dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /app

# git wird benötigt, da subgraph per Git-URL installiert wird
RUN apt-get update && apt-get install -y --no-install-recommends git \
    && rm -rf /var/lib/apt/lists/*

COPY pyproject.toml ./
COPY src ./src
COPY web ./web
RUN pip install --no-cache-dir .

RUN useradd --create-home appuser
USER appuser

ENV SUBNETPLAN_API_HOST=0.0.0.0 SUBNETPLAN_API_PORT=8000
EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=3s CMD python -c "import urllib.request,sys; urllib.request.urlopen('http://127.0.0.1:8000/api/v1/health', timeout=2)" || exit 1

CMD ["subnetplan-api"]
```

Hinweise:

- Im Container **muss** der Host `0.0.0.0` sein, sonst ist der Dienst von außen nicht erreichbar (im Gegensatz zum sicheren lokalen Standard). Das erledigt die Variable `SUBNETPLAN_API_HOST`.
- **Nicht als root** ausführen.
- Der `HEALTHCHECK` nutzt den vorhandenen Endpunkt `GET /api/v1/health`. Dessen Beschreibung weist korrekt darauf hin, dass er **keine Abhängigkeiten prüft**; er zeigt nur „Prozess antwortet“ (Liveness).

### 21.5 Liveness und Readiness

| Probe | Frage | Typische Antwort |
| --- | --- | --- |
| Liveness | Lebt der Prozess? | `GET /health` → 200 |
| Readiness | Ist er bereit für Traffic (DB erreichbar, Modelle geladen)? | `GET /ready` → 200 oder 503 |

subnetplan implementiert nur Liveness; der Endpunkt hat deshalb dokumentiert `503`, falls der Dienst nicht verfügbar ist, und führt keine Dependency-Prüfung aus.

### 21.6 Reverse-Proxy (nginx)

```nginx
server {
    listen 443 ssl http2;
    server_name api.example.org;

    client_max_body_size 1m;           # Body-Grenze (siehe 14.5)

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 60s;
    }
}
```

Läuft die API unter einem **Pfadpräfix** (`https://example.org/subnetplan/…`) und schneidet der Proxy es ab, gibt man `--root-path /subnetplan` bzw. `FastAPI(root_path="/subnetplan")` an, damit die Doku-URLs stimmen.

### 21.7 systemd-Dienst

```ini
[Unit]
Description=subnetplan API
After=network.target

[Service]
User=subnetplan
WorkingDirectory=/opt/subnetplan
Environment=SUBNETPLAN_API_HOST=127.0.0.1
Environment=SUBNETPLAN_API_PORT=8000
ExecStart=/opt/subnetplan/.venv/bin/subnetplan-api
Restart=on-failure
NoNewPrivileges=true
ProtectSystem=strict

[Install]
WantedBy=multi-user.target
```

### 21.8 Logging

```python
import logging
logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(name)s: %(message)s")
```

In Containern **auf stdout/stderr** loggen; Einsammeln übernimmt die Plattform. Für strukturierte (JSON-)Logs eignen sich `structlog` oder `python-json-logger`.

### 21.9 Checkliste vor dem Produktivbetrieb

- [ ] Abhängigkeiten gepinnt (Lock-Datei)
- [ ] Mehrere Worker, hinter Reverse-Proxy mit TLS
- [ ] Authentifizierung und CORS entschieden (Kapitel 14 und 17)
- [ ] Body-Größe und Timeouts begrenzt
- [ ] Health-Check eingerichtet
- [ ] Logs zentral gesammelt
- [ ] Tests und Schema-Test in der CI-Pipeline
- [ ] `/docs` bewusst offen oder geschützt

---

## 22. Performance und Skalierung

### 22.1 Wo die Zeit hingeht

Bei typischen FastAPI-Diensten liegt die Zeit in: (1) Fachlogik/Datenbank, (2) JSON-Validierung und -Serialisierung, (3) Netzwerk. Das Framework selbst ist selten der Engpass.

### 22.2 Hebel

| Maßnahme | Wirkung |
| --- | --- |
| Mehr Worker-Prozesse | nutzt mehrere Kerne (löst den GIL-Engpass) |
| `uvicorn[standard]` (uvloop, httptools) | schnellerer Event-Loop und HTTP-Parser |
| GZip für große Antworten | weniger Bandbreite, etwas mehr CPU |
| `response_model_exclude_none` | kleinere Antworten |
| Ergebnis-Caching (siehe unten) | vermeidet wiederholte Berechnung |
| Pagination | begrenzt Antwortgrößen |
| Async-I/O für Netzwerk/DB | viele gleichzeitige Verbindungen ohne Threads |
| Schnelleres Response-Serialisieren | `ORJSONResponse` als `default_response_class` (benötigt `orjson`) |

### 22.3 Doppelte Validierung vermeiden

subnetplan erzeugt den Plan als JSON-**Text** (`render_json(plan)`), lädt ihn mit `json.loads` zu einem `dict` und lässt FastAPI es nochmals gegen `PlanResponse` validieren. Das kostet etwas CPU, bringt aber die oben beschriebene Garantie „Doku = Wirklichkeit“. Bei sehr großen Plänen könnte man `render_json` durch eine Funktion ersetzen, die direkt ein `dict` liefert; der Gewinn lohnt sich erst nach Messung.

### 22.4 Caching zustandsloser Berechnungen

Weil Ergebnisse deterministisch von der Eingabe abhängen, kann man sie cachen – etwa über einen Hash der Konfiguration:

```python
import hashlib, json
from functools import lru_cache

def _key(config: dict, overrides: dict) -> str:
    return hashlib.sha256(json.dumps([config, overrides], sort_keys=True).encode()).hexdigest()

@lru_cache(maxsize=256)
def _cached_plan_json(key: str, payload: str) -> str:
    config, overrides = json.loads(payload)
    _, plan = _make_plan(config, overrides)
    return render_json(plan)
```

Ein Cache im Prozess ist pro Worker getrennt; für gemeinsamen Cache dient Redis. Caches nur einsetzen, wenn Messungen sie rechtfertigen.

### 22.5 Lastschutz

CPU-intensive Endpunkte (etwa exakte Politikvergleiche für sehr viele Gruppen) sollten **Eingabegrößen begrenzen** (`max_length` für Listen, Gruppen- und Hostzahl), sonst genügt ein einziger großer Request, um einen Worker lange zu blockieren. Zusätzlich: Timeouts am Proxy, Rate-Limiting (`slowapi` oder am Proxy).

### 22.6 Lasttests

```bash
pip install locust
# oder: wrk / hey / k6
hey -n 2000 -c 50 -m POST -T application/json -d @request.json http://127.0.0.1:8000/api/v1/plans
```

---

## 23. Best Practices und Anti-Patterns

### 23.1 Was subnetplan richtig macht – zum Übernehmen

| # | Praxis | Beleg im Projekt |
| --- | --- | --- |
| 1 | **Fachlogik ohne Web-Abhängigkeit** | `src/` importiert kein FastAPI; die API ist eine dünne Schicht |
| 2 | **Endpunkte klein halten** | 2–10 Zeilen; Arbeit steckt in Fachfunktionen |
| 3 | **Ein Fehlerformat für Fachfehler** | `PlanError` → ein Handler → `{"error": {...}}` |
| 4 | **Antwortmodelle für jeden Endpunkt** | `response_model=` überall; Antwort wird validiert |
| 5 | **Jedes Feld dokumentiert** | `Field(description=, examples=)` durchgängig |
| 6 | **Literal statt Freitext** | `Literal["iptables", "nftables", "cisco"]` |
| 7 | **Basisklassen für gemeinsame Felder** | `ConfigRequest` → fünf Unterklassen |
| 8 | **Wiederverwendbare `responses`-Konstante** | `COMMON_RESPONSES` |
| 9 | **Version im Pfad und im Schema** | `/api/v1`, `version="4.0.0"` |
| 10 | **Zustandslosigkeit** | Konfiguration in jedem Request |
| 11 | **`def` für CPU-Arbeit** | alle Endpunkte synchron |
| 12 | **Optionale Abhängigkeit abfangen** | lazy Import → 503 |
| 13 | **Sichere Standardwerte** | Bind an `127.0.0.1` |
| 14 | **Doku testen** | `test_openapi_has_descriptive_operations_and_examples` |
| 15 | **API über HTTP testen** | `TestClient`, keine Mocks von Interna |
| 16 | **Web-Modell ≠ Fachmodell** | `FirewallSettings.to_options()` |
| 17 | **Namen in Hilfsfunktionen mit `_`** | `_make_plan`, `_plan_payload`, `_pair_payload` |
| 18 | **Sprechende Docstrings + `description`** | für Entwickler *und* API-Nutzer |

### 23.2 Anti-Patterns, die man vermeiden sollte

| Anti-Pattern | Warum problematisch | Besser |
| --- | --- | --- |
| `async def` mit blockierendem Code | blockiert den Event-Loop für alle | `def` oder `run_in_threadpool` |
| Geschäftslogik im Endpunkt | nicht wiederverwendbar, schwer testbar | in `services/`-Modul auslagern |
| `except Exception: pass` | verschluckt Fehler | gezielte Exceptions + zentraler Handler |
| Dict statt Modell für Responses ohne `response_model` | unvalidiert, undokumentiert | `response_model` oder Rückgabetyp |
| Datenbankmodell direkt als Response | Datenleck-Gefahr (Hashes, interne Felder) | eigenes `…Public`-Modell |
| Veränderliche Defaults `= []` | historisch fehleranfällig | `Field(default_factory=list)` |
| Alles in einer riesigen Datei | unübersichtlich | `APIRouter` nach Themen |
| Globale Variablen als Zustand | Race Conditions mit mehreren Workern | Datenbank/Redis oder zustandslos |
| `allow_origins=["*"]` mit Credentials | unsicher/ungültig | konkrete Origins |
| Stacktrace in Antworten | Informationsleck | loggen, generische Meldung |
| Secrets im Code | Leck im Repository | Umgebung/Secret-Store |
| `--reload` in Produktion | instabil, langsam | nur in Entwicklung |
| Verben in Ressourcennamen (`/getPlans`) | inkonsistent | `GET /plans`, `POST /plans` |
| Statuscode 200 mit Fehlertext im Body | Clients erkennen Fehler nicht | passende 4xx/5xx |

### 23.3 Namens- und Stilkonventionen

- **JSON-Felder** in `snake_case` (wie im Projekt) oder per Alias in `camelCase`:

```python
from pydantic import ConfigDict
from pydantic.alias_generators import to_camel

class Base(BaseModel):
    model_config = ConfigDict(alias_generator=to_camel, populate_by_name=True)
```

- **Pfade** im Plural und kleingeschrieben mit Bindestrichen (`/compare-services` im Projekt).
- **Typannotationen überall**, auch am Rückgabewert; mypy/pyright im CI.
- **Docstrings** der Endpunkte beschreiben, *was* passiert (die Doku ist Teil des Produkts).
- Hilfsfunktionen, die nicht Teil der Schnittstelle sind, mit führendem Unterstrich.

### 23.4 Wie man Endpunkte entwirft – Faustregeln

1. **Idempotente Berechnung als `POST`**, wenn die Eingabe zu groß für die URL ist (so macht es das Projekt).
2. **Ressourcen** im Pfad, **Aktionen** als Unterpfad.
3. **Gleicher Name = gleiche Bedeutung** in allen Endpunkten (`config`, `overrides`, `firewall`, `options` tauchen identisch auf).
4. **Defaults** so wählen, dass der einfachste Request funktioniert (`overrides` und `options` sind optional).
5. **Keine Überraschungen bei Fehlern**: ein Format, dokumentierte Codes.
6. **Additive Änderungen**: neue Felder immer optional mit Default – so wie `service_mismatch` mit `default_factory=list` und `services`/`site` mit `default=None` nachträglich ergänzt wurden, ohne alte Clients zu brechen.

---

## 24. Verbesserungsvorschläge für subnetplan

Das Projekt nutzt FastAPI sehr konsequent. Die folgenden Punkte sind **keine Fehler**, sondern Ansatzpunkte für den nächsten Reifegrad; jeder davon ist durch die Kapitel oben abgedeckt.

| # | Beobachtung | Vorschlag | Kapitel |
| --- | --- | --- | --- |
| 1 | `pyproject.toml` nennt im Extra `dev` das Paket `httpx2`, die README und der `TestClient` brauchen `httpx`. | Eintrag prüfen und auf `httpx` korrigieren (bzw. bestätigen, dass `httpx2` bewusst gewählt wurde). | 3.2 |
| 2 | `config` ist `dict[str, Any]`; das OpenAPI-Schema zeigt dadurch nicht, welche Schlüssel (`groups[].hosts`, …) zulässig sind. | Ein Pydantic-Modell `ConfigModel` mit `groups: list[GroupModel]` für die **Doku** bereitstellen, die Validierung aber weiter in `build_config` belassen (z. B. `extra="allow"`). | 7.6 |
| 3 | `response_model_exclude_none=True` entfernt auch fachlich relevante `null`-Werte (z. B. `gateway`). | Bewusst dokumentieren oder auf `response_model_exclude_unset` umstellen und nur optionale Felder auslassen. | 8.2 |
| 4 | Es gibt kein Limit für Größe oder Gruppenanzahl in `config`. | `max_length` am Modell, Body-Größe am Proxy, Rate-Limit. | 14.5, 22.5 |
| 5 | Zwei 422-Formate (`error` und `detail`). | Optional `RequestValidationError`-Handler für ein einziges Format. | 9.4 |
| 6 | Alle Routen in einer Datei, URL-Präfix per f-String. | `APIRouter` je Tag, `include_router(prefix=API_PREFIX)`. | 13.3 |
| 7 | Keine Authentifizierung. | Für Betrieb außerhalb von `127.0.0.1`: API-Key oder OAuth2 als globale Abhängigkeit. | 17 |
| 8 | Keine CORS-Konfiguration. | Nur bei direktem Browser-Zugriff ergänzen. | 14.2 |
| 9 | `health` zeigt nur Liveness. | Optional `/ready`, das z. B. das Vorhandensein von `subnetplan.policy` meldet (Subgraph-Verfügbarkeit). | 21.5 |
| 10 | `operation_id`s sind automatisch erzeugt (lang, z. B. `create_plan_api_v1_plans_post`). | Eigene IDs oder generische Funktion für bessere Client-Generierung. | 10.6 |
| 11 | Mehrere Endpunkte parsen denselben Request und rufen `_make_plan` auf. | Falls Authentifizierung/Caching hinzukommt: gemeinsame `Depends`. | 11.6 |

---

## 25. Fehlersuche und FAQ

### 25.1 „422 Unprocessable Entity“ – was ist falsch?

Die Antwort nennt Feldpfad (`loc`) und Grund (`msg`, `type`). Häufige Ursachen:

| `type` | Ursache |
| --- | --- |
| `missing` | Pflichtfeld fehlt (z. B. `firewall` bei `/plans/firewall`) |
| `literal_error` | Wert nicht in `Literal`/Enum |
| `int_parsing`, `float_parsing` | falscher Zahlentyp |
| `less_than_equal`, `greater_than_equal` | Wertebereich verletzt (`threshold`) |
| `json_invalid` | kaputtes JSON (Komma, Anführungszeichen) |
| `model_attributes_type` | Objekt erwartet, anderes gesendet |

Unterscheide am **Antwortformat**: `{"detail": [...]}` = Struktur-/Typfehler (FastAPI); `{"error": {...}}` = fachlicher Fehler (`PlanError`, im Beispielprojekt).

### 25.2 „Mein Body wird nicht erkannt“

- Content-Type muss `application/json` sein (`curl -H "Content-Type: application/json"`, in `httpx`/`requests` `json=` statt `data=`).
- Ein einzelnes Pydantic-Modell erwartet das Objekt **direkt**; mehrere Body-Parameter erwarten ein Objekt mit deren Namen als Schlüssel.

### 25.3 „Mein Query-Parameter-Liste kommt als Body an“

Listen im Query brauchen `Annotated[list[str], Query()]`.

### 25.4 „Die Doku zeigt alte Felder“

`/openapi.json` wird beim ersten Aufruf **gecacht**. Nach Codeänderung Server neu starten (mit `--reload` automatisch).

### 25.5 „CORS-Fehler im Browser“

Siehe 14.2. Typisch: Origin fehlt in `allow_origins`, oder `Authorization` fehlt in `allow_headers`. Der Fehler wird vom **Browser** gemeldet; im Server-Log erscheint ein erfolgreicher oder fehlgeschlagener `OPTIONS`.

### 25.6 „Server hängt unter Last“

Wahrscheinlich blockiert ein `async def`-Endpunkt den Loop (Kapitel 12). Auf `def` umstellen oder `run_in_threadpool` verwenden.

### 25.7 „ImportError beim Start“

- `ModuleNotFoundError: web` – Projekt nicht installiert und nicht im `PYTHONPATH`; `pip install -e .` ausführen oder aus dem Projektverzeichnis starten.
- Zirkuläre Importe zwischen Router und Modellen – Modelle in eigenes Modul (`schemas`) auslagern.
- Wie im Projekt kann ein optionales Modul **lazy** importiert werden.

### 25.8 „Response validation error“ (HTTP 500)

Der Rückgabewert passt nicht zum `response_model`. Das ist ein **Programmierfehler** auf Serverseite (und der Grund, warum man `response_model` einsetzt: Fehler fallen auf, bevor falsche Daten beim Client landen). Die Fehlermeldung nennt das Feld.

### 25.9 „Pydantic v1 vs. v2“

Beispiele hier gelten für **v2** (`model_dump`, `field_validator`, `ConfigDict`). Alte v1-Muster (`.dict()`, `@validator`, `class Config`) sind in v2 veraltet. subnetplan nutzt v2 (`ConfigDict`, `Field(examples=[…])`, `json_schema_extra`).

### 25.10 „Wie gebe ich das Schema an Frontend-Entwickler?“

`openapi.json` exportieren (10.6) und Client generieren; oder `/docs` bereitstellen. Wird das Schema im CI auf Änderungen verglichen, sieht man **Breaking Changes** vor dem Release.

### 25.11 „Warum `from __future__ import annotations` im Projekt?“

Das Modul `api.py` beginnt damit. Es verschiebt die Auswertung von Typannotationen auf später. FastAPI und Pydantic v2 lösen die Strings korrekt auf, **solange die Typen auf Modulebene importiert sind** (z. B. `Any`, `Literal`, `Request`). Problematisch wird es nur bei Typen, die lokal in Funktionen definiert sind.

---

## 26. Spickzettel

```python
from typing import Annotated, Any, Literal
from fastapi import (FastAPI, APIRouter, Depends, HTTPException, Request, Response,
                     Query, Path, Header, Cookie, Body, Form, File, UploadFile,
                     BackgroundTasks, status)
from fastapi.responses import JSONResponse, PlainTextResponse, FileResponse, StreamingResponse
from pydantic import BaseModel, Field, ConfigDict, field_validator, model_validator

# App
app = FastAPI(title="…", summary="…", description="…", version="1.0.0",
              openapi_tags=[{"name": "x", "description": "…"}], lifespan=lifespan)

# Route
@app.post("/api/v1/things", tags=["x"], summary="…", description="…",
          response_model=ThingOut, response_model_exclude_none=True,
          responses={422: {"description": "…"}}, status_code=201)
def create(body: ThingIn, q: Annotated[int, Query(ge=1)] = 1) -> dict[str, Any]: ...

# Modell
class ThingIn(BaseModel):
    model_config = ConfigDict(extra="forbid", json_schema_extra={"examples": [{…}]})
    name: str = Field(min_length=1, description="…", examples=["x"])
    kind: Literal["a", "b"] = "a"
    tags: list[str] = Field(default_factory=list)

# Fehler
class DomainError(Exception): ...
@app.exception_handler(DomainError)
async def handler(_: Request, exc: DomainError) -> JSONResponse:
    return JSONResponse(status_code=422, content={"error": {"type": "domain_error", "message": str(exc)}})

# Abhängigkeit
def dep() -> Any: ...
Dep = Annotated[Any, Depends(dep)]

# Router
router = APIRouter(prefix="/things", tags=["x"]); app.include_router(router, prefix="/api/v1")

# Test
from fastapi.testclient import TestClient
client = TestClient(app); client.post("/api/v1/things", json={…}).json()

# Start
# fastapi dev main.py        |  uvicorn pkg.mod:app --reload  |  uvicorn pkg.mod:app --workers 4
```

| Aufgabe | Werkzeug |
| --- | --- |
| Pfadwert | `Path(...)` / Parameter im Pfad |
| Query-Wert | einfacher Parameter / `Query(...)` |
| JSON-Body | Pydantic-Modell / `Body(...)` |
| Header / Cookie | `Header()` / `Cookie()` |
| Formular / Datei | `Form()`, `File()`, `UploadFile` |
| Antwortform | `response_model` / Rückgabetyp |
| Fehler | `HTTPException` / `exception_handler` |
| Wiederverwendung | `Depends` |
| Gruppierung | `APIRouter` |
| Start/Ende | `lifespan` |
| Nachlauf | `BackgroundTasks` |
| Auth | `APIKeyHeader`, `OAuth2PasswordBearer` |
| Test | `TestClient`, `dependency_overrides` |

---

## 27. Glossar

| Begriff | Bedeutung |
| --- | --- |
| **ASGI** | Asynchronous Server Gateway Interface – Schnittstelle zwischen Python-Webservern (Uvicorn) und Anwendungen (FastAPI) |
| **Starlette** | das ASGI-Framework, auf dem FastAPI aufbaut |
| **Pydantic** | Bibliothek für Datenvalidierung anhand von Typannotationen |
| **Pfadoperation** | Endpunkt: Kombination aus HTTP-Methode, Pfad und Funktion |
| **OpenAPI** | Standard zur Beschreibung von HTTP-APIs (früher „Swagger“) |
| **Swagger UI / ReDoc** | Oberflächen, die OpenAPI-Schemata darstellen |
| **Dependency Injection** | Einspeisen von Bausteinen über `Depends` |
| **Lifespan** | Start-/Ende-Hook der Anwendung |
| **Middleware** | Code, der jeden Request/Response umschließt |
| **CORS** | Cross-Origin Resource Sharing – Regeln für Browser-Zugriffe über Origin-Grenzen |
| **Preflight** | `OPTIONS`-Vorabanfrage des Browsers bei CORS |
| **Router** | `APIRouter`: Gruppe zusammengehöriger Pfadoperationen |
| **Response-Modell** | Pydantic-Modell, das die Antwort filtert, validiert, dokumentiert |
| **Zustandslos (stateless)** | Server speichert nichts zwischen Requests; jeder Request enthält alle nötigen Daten |
| **Idempotent** | wiederholte Ausführung ergibt dasselbe Ergebnis |
| **Event-Loop** | Schleife, die asynchrone Aufgaben zeitlich verschachtelt |
| **GIL** | Global Interpreter Lock von CPython: nur ein Thread führt Python-Bytecode gleichzeitig aus |
| **Liveness / Readiness** | „läuft der Prozess“ / „ist er bereit für Last“ |
| **Roundtrip (im Projekt)** | erzeugte Firewall-Regeln zurücklesen und mit der Zugriffsmatrix vergleichen |
| **`PlanError` (im Projekt)** | Fachexception, die der Handler auf HTTP 422 abbildet |

---

## 28. Weiterführende Quellen

- FastAPI-Dokumentation: <https://fastapi.tiangolo.com> (Tutorial – Benutzerhandbuch, Fortgeschrittenes Benutzerhandbuch, Deployment)
- Pydantic: <https://docs.pydantic.dev>
- Starlette: <https://www.starlette.io>
- Uvicorn: <https://www.uvicorn.org>
- OpenAPI-Spezifikation: <https://spec.openapis.org>
- SQLModel: <https://sqlmodel.tiangolo.com>
- Schemathesis (Vertragstests): <https://schemathesis.readthedocs.io>
- Das Beispielprojekt: `web/api.py`, `tests/test_web_api.py`, Abschnitt „JSON API“ der `README.md`, https://github.com/naphets86/sbntgg

---

*Diese Dokumentation orientiert sich an Version 4.0.0 des Projekts subnetplan. Codeausschnitte aus dem Projekt sind gekürzt wiedergegeben; maßgeblich ist stets `web/api.py`.*
