# Lösungsweg: Radhaus bekommt CI

Dieser Branch zeigt den Endstand nach allen Aufgaben und Zusatzaufgaben. Die fertigen Dateien:

| Datei | Aufgaben |
|---|---|
| `.github/workflows/hallo.yml` | 1 |
| `.github/workflows/ci.yml` | 2, 4, 5, Z1, Z2, Z3 |
| `shop/versand.py` (eine Zeile) | 3 |
| Ruleset in den Einstellungen | 6 |
| `pyproject.toml` | Z1 |
| `.github/dependabot.yml` | Z4 |
| `.github/workflows/terraform.yml`, `infra/main.tf` | Z5 |

## Aufgabe 1: Erster Workflow

```yaml
name: Hallo Radhaus

on: [push, workflow_dispatch]

jobs:
  hallo:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Hallo ${{ github.actor }}, willkommen bei Radhaus!"
```

- **Event:** `push` bzw. `workflow_dispatch` · **Workflow:** die Datei · **Job:** `hallo` · **Runner:** die frische Ubuntu-VM · **Step:** das `echo`.
- Nach dem Job wird der Runner gelöscht. Nichts bleibt übrig.

## Aufgabe 2: Die CI-Pipeline

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-python@v7
        with:
          python-version: '3.13'
      - run: python -m pip install -r requirements-dev.txt
      - run: python -m pytest -q
```

Erwartetes Ergebnis: **rot**. Im Log:

```text
FAILED tests/test_versand.py::test_versandkosten[50.00-0.00] - AssertionError
assert Decimal('4.90') == Decimal('0.00')
```

Für 50,00 € werden 0,00 € Versand erwartet, berechnet werden 4,90 €.

- `checkout` ist nötig, weil der Runner leer startet – der Workflow liegt zwar im Repository, der Code aber noch nicht auf der Maschine.
- Die feste Python-Version löst das „Bei mir läuft’s“-Problem.

## Aufgabe 3: Fehler beheben

In `shop/versand.py`:

```diff
-    if warenwert > VERSANDFREI_AB:
+    if warenwert >= VERSANDFREI_AB:
```

Beispiel für die PR-Beschreibung:

> **Problem:** `test_versandkosten[50.00-0.00]` schlägt fehl – bei genau 50,00 € wurden 4,90 € Versand berechnet.
> **Änderung:** Vergleich von `>` auf `>=` geändert, weil der Versand laut Vorgabe *ab 50,00 € einschließlich* kostenlos ist.
> **Prüfung:** Check `CI / test` ist grün (`python -m pytest -q`). Tests wurden nicht verändert.

Die Pipeline läuft im PR (Prüfung **vor** dem Merge) und nach dem Merge auf `main` (Prüfung des tatsächlich zusammengeführten Stands). Von Hand hätte man genau mit 50,00 € testen müssen – ein typischer Grenzwert, der leicht übersehen wird.

## Aufgabe 4: Zweiter Job und `needs`

```yaml
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-python@v7
        with:
          python-version: '3.13'
      - run: python -m pip install -r requirements-dev.txt
      - run: ruff check .

  startcheck:
    needs: [test, lint]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-python@v7
        with:
          python-version: '3.13'
      - run: python -m shop.versand
```

Log von `startcheck`: `Gesamtpreis: 50.00 EUR`. Beim „Ausprobieren“ mit falschem Testwert wird `test` rot und `startcheck` **übersprungen** (grauer Kreis) – das Gate.

## Aufgabe 5: Rechte und Pinning

```yaml
permissions:
  contents: read

# in jedem Job:
      - uses: actions/setup-python@6d4fd22b20465ccc0f7354476588e8df7c9ca0e0 # v7.1.0
```

- Der SHA gehört zu `actions/setup-python` **v7.1.0** (Stand 20.11.2026).
- Bei tj-actions hätte ein per SHA eingebundener Workflow weiter den alten, sauberen Code ausgeführt – die Angreifer konnten nur die Tags verschieben, nicht den Inhalt eines bestehenden Commits ändern.
- Ein Tag wie `v7` bringt automatisch Fehlerbehebungen, kann aber vom Herausgeber – oder einem Angreifer – jederzeit auf anderen Code gesetzt werden.

## Aufgabe 6: Ruleset

Einstellungen unter **Settings → Rules → Rulesets**:

| Einstellung | Wert |
|---|---|
| Enforcement status | Active |
| Target branches | Default branch |
| Restrict deletions, Block force pushes | an |
| Require a pull request before merging | an, Required approvals: 0 |
| Require status checks to pass | an, Check `test` |

- Direkter Commit auf `main` im Browser: GitHub verweigert ihn und bietet an, einen Branch und PR anzulegen.
- PR mit Pauschale 5,90 €: `test` wird rot, weil die Tests 4,90 € erwarten. **Merge** ist gesperrt.
- Wäre die Preisänderung gewollt, müssten die Tests (also die dokumentierte Geschäftsregel) im selben PR mitgeändert werden – das macht die Änderung im Review sichtbar.

## Zusatz 1: Coverage

```yaml
      - run: python -m pytest -q --cov=shop --cov-report=term-missing --cov-fail-under=90
```

Erster Lauf: 80 %, rot. Nicht abgedeckt sind die Zeilen des Startblocks `if __name__ == "__main__":`. Sie laufen nur beim direkten Start (Job `startcheck`), nie in den Tests. Lösung ohne Schwelle zu senken – `pyproject.toml`:

```toml
[tool.coverage.report]
exclude_also = [
    "if __name__ == .__main__.:",
]
```

Danach 100 %. Der 50-€-Fehler hatte trotzdem volle Coverage: Die fehlerhafte Zeile wurde ausgeführt, nur das Ergebnis war für genau 50,00 € falsch. Coverage zeigt, **was ausgeführt** wurde – nicht, ob es **richtig** ist.

## Zusatz 2: Matrix

```yaml
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python: ['3.12', '3.13', '3.14']
    steps:
      # ...
      - uses: actions/setup-python@6d4fd22b20465ccc0f7354476588e8df7c9ca0e0 # v7.1.0
        with:
          python-version: ${{ matrix.python }}
```

Danach im Ruleset statt `test` die drei Checks `test (3.12)`, `test (3.13)`, `test (3.14)` als Pflicht eintragen.

## Zusatz 3: Zeitplan

```yaml
  schedule:
    - cron: '0 1 * * *'
  workflow_dispatch:
```

`1` statt `2`, weil GitHub in UTC rechnet und die deutsche Winterzeit UTC+1 ist. Im Sommer (UTC+2) läuft der Job dann um 3:00 Uhr deutscher Zeit.

## Zusatz 4: Dependabot

```yaml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: "/"
    schedule:
      interval: weekly
  - package-ecosystem: pip
    directory: "/"
    schedule:
      interval: weekly
```

Bei gepinnten Actions ersetzt Dependabot den SHA **und** den Versionskommentar. Ein Linter ist exakt festgelegt, weil neue Versionen neue Regeln mitbringen – sonst wird die Pipeline eines Tages rot, ohne dass jemand am Code etwas geändert hat.

## Zusatz 5: Terraform

```yaml
name: Terraform

on:
  push:
    branches: [main]
    paths: ['infra/**', '.github/workflows/terraform.yml']
  pull_request:
    branches: [main]
    paths: ['infra/**', '.github/workflows/terraform.yml']

permissions:
  contents: read

jobs:
  terraform:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: infra
    steps:
      - uses: actions/checkout@v7
      - uses: hashicorp/setup-terraform@dfe3c3f87815947d99a8997f908cb6525fc44e9e # v4.0.1
      - run: terraform fmt -check -diff
      - run: terraform init -backend=false
      - run: terraform validate
```

Fehler: In `infra/main.tf` sind im Block `resource "local_file"` die `=` nicht ausgerichtet. Korrektur:

```hcl
  filename        = "${path.module}/shop-${var.umgebung}.conf"
  content         = <<-KONFIG
  ...
  file_permission = "0644"
```

`plan`/`apply` brauchen in echt einen Remote-State (z. B. S3) und Cloud-Zugangsdaten – am besten per OIDC (Thema Abend 2). Die `.terraform.lock.hcl` hält die exakten Provider-Versionen und Prüfsummen fest – reproduzierbare Builds für Infrastruktur.
