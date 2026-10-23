## Software und Wiederverwendung: `src/` als Kern, `web/` als Integration

### Beobachtung

Die Projektstruktur trennt zwei Schichten:

- `src/` enthält den **Kern**: den Subgraph Algorithmus mit seiner gesamten Fachlogik. Er kennt weder HTTP noch FastAPI und bleibt bei jeder Integration **unverändert erhalten**.
- `web/` enthält die **Integration**: FastAPI-Anwendung, Routen, Request-/Response-Modelle. Diese Schicht ruft den Kern nur auf und implementiert keine eigene Fachlogik.

Wiederverwendung heißt dann: Der Kern wird nicht kopiert oder umgebaut, sondern von beliebig vielen Schichten (Web, CLI, Bibliothek, Tests) aufgerufen.

```
projekt/
├── src/          # Kern: Algorithmus, unverändert wiederverwendbar
└── web/          # Integration: FastAPI ruft src/ auf
```

### Umsetzung mit FastAPI

Die Integrationsschicht ist ein dünner Adapter. Der Importpfad zeigt immer von `web/` nach `src/`, nie umgekehrt.

```python
# web/main.py  (Beispiel; Modul- und Funktionsnamen sind Platzhalter)
from fastapi import FastAPI
from pydantic import BaseModel

from src.subgraph import find_subgraph   # Kern, unverändert

app = FastAPI()


class Query(BaseModel):
    graph: dict[str, list[str]]
    pattern: dict[str, list[str]]


@app.post("/subgraph")
def subgraph(q: Query):
    return {"result": find_subgraph(q.graph, q.pattern)}
```

Regeln:

1. **Abhängigkeitsrichtung:** `web/` importiert `src/`. `src/` importiert nichts aus `web/`.
2. **Keine Fachlogik in `web/`:** Validierung des Transportformats ja, Algorithmus nein.
3. **Tests am Kern:** Die Korrektheit wird in `src/` getestet (pytest). Die Web-Tests prüfen nur die Anbindung.
4. **Austauschbarkeit der Schicht:** FastAPI kann durch eine andere Schicht ersetzt werden, ohne dass sich `src/` ändert.

### Mathematisches Analogon: die e-Funktion als Eigenfunktion

Die Beobachtung geht von der e-Funktion aus. Ihr korrektes Analogon zur Wiederverwendung ist die **Eigenfunktionseigenschaft**: Der Ableitungsoperator $D=\frac{d}{dt}$ verändert $e^{st}$ nicht in seiner Gestalt, er skaliert nur mit dem Faktor $s$. So bleibt auch der Kern bei der Integration in seiner Gestalt erhalten.

**Satz.** Für alle $s,t\in\mathbb{R}$ (ebenso $\mathbb{C}$) gilt
$$\frac{d}{dt}e^{st}=s\,e^{st},\qquad \frac{d^k}{dt^k}e^{st}=s^k e^{st}\quad(k\in\mathbb{N}_0).$$

**Beweis.**

*Schritt 1: Konvergenzradius.* Sei $x=st$ und $a_n=\frac{s^n}{n!}$ die Koeffizienten in $t$. Für $s\neq0$ gilt
$$\left|\frac{a_{n+1}}{a_n}\right|=\frac{|s|}{n+1}\xrightarrow{n\to\infty}0,$$
also ist der Konvergenzradius $R=\infty$ (Quotientenkriterium). Für $s=0$ ist die Reihe konstant $1$, die Aussage ist trivial.

*Schritt 2: Gliedweise Differentiation.* Nach dem Satz über Potenzreihen darf eine Potenzreihe im Inneren ihres Konvergenzradius gliedweise differenziert werden, und die abgeleitete Reihe hat denselben Konvergenzradius. Hier gilt also für alle $t\in\mathbb{R}$:
$$\frac{d}{dt}\sum_{n=0}^{\infty}\frac{s^n t^n}{n!}=\sum_{n=1}^{\infty}\frac{s^n\, n\, t^{n-1}}{n!}=\sum_{n=1}^{\infty}\frac{s^n t^{n-1}}{(n-1)!}.$$
Das konstante Glied ($n=0$) hat die Ableitung $0$ und entfällt. Es wird **kein weiteres Glied gestrichen**.

*Schritt 3: Indexverschiebung.* Mit $m=n-1$ ist $m\mapsto m+1$ eine Bijektion von $\mathbb{N}_0$ auf $\mathbb{N}_{\ge1}$. Da die Reihe absolut konvergiert, ist die Umnummerierung zulässig:
$$\sum_{n=1}^{\infty}\frac{s^n t^{n-1}}{(n-1)!}=\sum_{m=0}^{\infty}\frac{s^{m+1}t^{m}}{m!}=s\sum_{m=0}^{\infty}\frac{(st)^m}{m!}=s\,e^{st}.$$

*Schritt 4: Induktion.* Der Fall $k=0$ ist klar, und aus $\frac{d^k}{dt^k}e^{st}=s^ke^{st}$ folgt
$$\frac{d^{k+1}}{dt^{k+1}}e^{st}=s^k\frac{d}{dt}e^{st}=s^{k+1}e^{st}.\qquad\blacksquare$$

**Folgerung (keine Periodizität).** Für reelles $s\neq0$ ist $t\mapsto e^{st}$ streng monoton, also nicht periodisch. Es gibt demnach keine „Periode $n$" der Ableitungen: Jede Ableitung reproduziert die Funktion exakt bis auf den Faktor $s^k$. Für rein imaginäres $s=i\omega$ ist $e^{i\omega t}$ periodisch mit Periode $2\pi/\omega$, und diese Periode folgt aus der Eulerschen Formel, nicht aus einem Streichen von Reihengliedern.

**Hinweis zur Notation.** In der Reihe bezeichnet $n$ einen Gliedindex aus $\mathbb{N}_0$. Die Eulersche Zahl $e\approx2{,}718\ldots$ ist keine natürliche Zahl und kann daher nicht als Gliedindex auftreten. Ein Term wie $\frac{(st)^e}{e!}$ ist keine Reihenglied der Taylor-Reihe.

### Übertragung auf die Software

| Mathematik | Software |
|---|---|
| Operator $D=\frac{d}{dt}$ | Integrationsschicht (`web/`) |
| Eigenfunktion $e^{st}$ | Kern (`src/`) |
| Eigenwert $s$ | Transportformat (JSON, HTTP), das nur skaliert bzw. verpackt |
| $D^k e^{st}=s^k e^{st}$ | Beliebig viele Schichten, der Kern bleibt unverändert |

Die Grenze der Analogie: Sie beschreibt das Entwurfsprinzip (Invarianz des Kerns unter Anwendung einer Schicht), sie ist kein formaler Beweis für die Korrektheit einer Software. Die Korrektheit des Kerns wird durch die Tests in `src/` gesichert.

### Fazit

- `src/` ist der Kern und bleibt erhalten.
- `web/` ist die Integration und ruft den Kern auf.
- Wiederverwendung entsteht durch Schichten über einem unveränderten Kern, so wie $e^{st}$ unter der Ableitung seine Gestalt behält.
