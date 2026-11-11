# Datamb-fire
# DataMB-Daten mit Firecrawl – automatisch alle 24 Stunden aktuell

Liest Team- und Spielerdaten von [datamb.football](https://datamb.football) über
[Firecrawl](https://github.com/firecrawl/firecrawl) aus, prüft sie und legt sie als
CSV, JSON, Excel, SQLite und HTML-Übersicht ab. Ein Zeitplan wiederholt das alle 24 Stunden.

## Welche Ligen enthalten sind

| Gewünschte Liga | Stand 11.11.2026 | Teams | Spielerprofile |
|---|---|---:|---:|
| 1. Bundesliga | ✔ enthalten | 18 | 252 |
| La Liga | ✔ enthalten | 20 | 336 |
| Ligue 1 | ✔ enthalten | 18 | 252 |
| Premier League | ✔ enthalten | 20 | 285 |
| 2. Bundesliga | ✘ nicht frei verfügbar | – | – |
| Championship | ✘ nicht frei verfügbar | – | – |

**2. Bundesliga und Championship** gibt es bei DataMB nur im kostenpflichtigen Pro-Bereich hinter
einem Login. Die frei zugängliche Seite enthält ausschließlich die „Top 7“-Ligen. Die Pipeline
umgeht diese Bezahlschranke nicht. Beide Ligen sind aber fertig konfiguriert: Sobald DataMB sie
auf der freien Teamübersicht aufführt, werden sie beim nächsten Lauf automatisch mit abgerufen.
In allen Ausgaben stehen sie bis dahin sichtbar als „nicht verfügbar“ statt stillschweigend zu fehlen.

## Schnellstart (Docker)

Voraussetzung: Docker mit Compose, ca. 2 GB RAM, ausgehender Internetzugang.

```bash
cp .env.example .env
docker compose up -d --build      # startet Firecrawl und den 24-h-Dienst
docker compose logs -f datamb     # dem ersten Lauf zusehen (rund 10 Minuten)
```

Danach liegen die Daten in `data/latest/`. Der Dienst `datamb` läuft weiter und aktualisiert
alle 24 Stunden; nach einem Neustart des Servers startet er von selbst wieder
(`restart: unless-stopped`).

```bash
docker compose run --rm datamb status   # Datenstand und nächster Lauf
docker compose run --rm datamb check    # Verbindung und Seitenstruktur prüfen
docker compose run --rm datamb run      # sofort aktualisieren
```

## Was herauskommt

Alles in `data/latest/` (wird bei jedem erfolgreichen Lauf in einem Schritt ersetzt):

| Datei | Inhalt |
|---|---|
| `report.html` | Übersicht zum Anklicken: Ligen, Teams, Spieler mit Filter, Suche und Sortierung. Funktioniert offline. |
| `datamb.xlsx` | Excel: Übersicht, Teams und je ein Blatt pro Position, mit Filtern und Farbskala |
| `teams.csv` | Ein Team pro Zeile: 7 Team-Kennzahlen, Tore und Gegentore pro 90 Minuten |
| `players.csv` | Ein Spielerprofil pro Zeile, eine Spalte je Kennzahl (leer, wenn die Position sie nicht hat) |
| `player_metrics.csv` | Dieselben Werte im Langformat: eine Zeile je Spieler und Kennzahl |
| `leagues.csv` | Die sechs gewünschten Ligen mit Verfügbarkeit |
| `dataset.json` | Alles in einer Datei, inklusive Metadaten |
| `metadata.json` | Zeitpunkt, Stand der Quelle, Zähler, Hinweise des letzten Laufs |
| `datamb.sqlite` | Datenbank mit aktuellen Tabellen und Verlauf (`history_*`) über die Wochen |

Außerdem: `data/history/` (komprimierte Kopie je Datenstand, die letzten 60),
`data/runs.jsonl` und `data/last_run.json` (Protokoll aller Läufe), `data/state.json` (Merker der Pipeline).

### Die Werte

Alle Kennzahlen sind **Perzentile von 0 bis 100**: 80 heißt „besser als 80 % der Vergleichsgruppe“
(Spieler derselben Position bzw. alle Teams der Top-7-Ligen). Absolute Werte zeigt die freie
Seite nur für zwei Team-Größen (Tore und Gegentore pro 90 Minuten). Enthalten sind Spieler ab der
von DataMB gesetzten Mindesteinsatzzeit (aktuell 350 Minuten).

| Position | Kennzahlen |
|---|---|
| Team | Goals, Attacking, Possession, Counters, Defending, Physicality, Pressing |
| Torhüter | Save percentage %, Aerial duels won, Interceptions (PAdj), Passes completed, Long pass accuracy %, Short pass accuracy %, PSxG − GA |
| Innenverteidiger | Passes completed, Forward pass completion %, Progressive passes completed, Possessions won, Defensive duels won %, Aerial duels won %, Progressive carries |
| Außenverteidiger | Accurate crosses, Expected assists, Progressive passes completed, Possessions won, Defensive duels won %, Aerial duels won %, Progressive carries |
| Mittelfeld | Duels won %, Possessions won, Progressive carries, Forward passes completed, Forward pass completion %, Key passes, Progressive passes completed |
| Flügelspieler | Progressive carries, Successful dribbles, Non-penalty goals, Non-penalty xG + xA, Assists, Key passes, Accurate crosses |
| Stürmer | Non-penalty goals, Non-penalty xG, Goal conversion, Aerial duels won %, Touches in box, Expected assists, Offensive duels won |

Zwei Besonderheiten der Quelle:

- **Zwei Profile pro Spieler.** Rund 65 Spieler führt DataMB auf zwei Positionen (z. B. Mittelfeld
  und Flügel) mit unterschiedlichen Perzentilen. Die Teamseite verlinkt bei jedem Aufruf zufällig
  eines davon. Die Pipeline merkt sich jedes gefundene Profil; beim ersten Lauf ruft sie jede
  Teamseite fünfmal auf und findet so fast alle, die restlichen kommen in den Folgetagen dazu.
  Deshalb gibt es mehr Profile (1.125) als Spieler (1.060).
- **Tote Links.** Einzelne Profile, die DataMB verlinkt, existieren dort nicht (aktuell Mama Baldé
  von Brest und Y. Titraoui von Lens). Sie fehlen im Datensatz und stehen namentlich in den Hinweisen des Laufs.

## So bleibt es aktuell

DataMB aktualisiert seine Datenbank laut eigener Angabe wöchentlich (zuletzt am 09.11.2026).
Die Pipeline prüft trotzdem täglich:

1. **Jeden Tag** werden die Teamübersicht und alle 76 Teamseiten neu geladen – ca. 85 Abrufe, unter 2 Minuten.
2. Haben sich Teamwerte, Kader oder Einsatzminuten geändert, hat die Quelle ein Update bekommen.
   Dann werden **alle Spielerseiten** neu geladen – ca. 1.200 Abrufe, rund 3 Minuten.
3. Spätestens nach 7 Tagen wird ohnehin alles neu geladen (`max_age_days`).

Wer lieber täglich alles neu laden möchte, setzt `refresh_policy: always`.

Ein Lauf wird nur veröffentlicht, wenn er die Prüfungen besteht. Andernfalls bleiben die bisherigen
Dateien unverändert, und der Zeitplan versucht es nach 60 Minuten erneut (höchstens dreimal).

| Fall | Verhalten |
|---|---|
| Firecrawl liefert eine Seite ohne ausgeführtes JavaScript | wird erkannt, bis zu 4 Versuche |
| Einzelne Seite scheitert endgültig | bisheriger Wert bleibt, als „älter“ markiert; nächster Lauf lädt komplett neu |
| Mehr als 10 % der Teams oder 5 % der Spieler scheitern | Lauf wird verworfen |
| 8 Seiten in Folge scheitern (Firecrawl aus, Netz weg) | sofortiger Abbruch statt Hunderter sinnloser Abrufe |
| Liga verliert plötzlich über 40 % ihrer Profile, oder die Teamübersicht ist lückenhaft | Lauf wird verworfen |
| DataMB benennt Kennzahlen um oder ändert die Reihenfolge | die Namen von der Seite gewinnen, der Lauf meldet es |
| Zwei Läufe gleichzeitig | der zweite bricht sofort ab (Sperrdatei) |

`datamb status` endet mit Exit-Code 1, wenn der letzte erfolgreiche Abruf älter als
2 × Intervall + 1 Stunde ist – geeignet für eigenes Monitoring.

## Betriebsvarianten

Die Aktualisierung braucht einen Rechner, der dauerhaft läuft. Vier Wege:

**1. Docker-Dienst (empfohlen)** – siehe Schnellstart. Feste Uhrzeit statt Intervall:
`DATAMB_SCHEDULE_AT=04:30` in `.env`.

**2. GitHub Actions, ohne eigenen Server** – `.github/workflows/refresh.yml` startet täglich um
03:17 UTC Firecrawl direkt im GitHub-Runner, führt einen Lauf aus und schreibt `data/` zurück ins
Repository. Dafür das Projekt in ein **privates** Repository legen und in `.gitignore` die Zeile
`data/` entfernen. Ein fehlgeschlagener Lauf erscheint dort als fehlgeschlagener Workflow. Dieser Weg ist hier nicht
auf GitHub selbst getestet, nur die einzelnen Schritte lokal.

**3. systemd-Timer** – `deploy/systemd/` enthält Service und Timer (täglich 04:30 Uhr, holt
verpasste Läufe nach).

**4. cron** – siehe `deploy/crontab.example`.

Für 3 und 4 einmalig: `python3 -m venv .venv && .venv/bin/pip install .` und Firecrawl starten mit
`docker compose -f docker-compose.firecrawl.yml -f docker-compose.firecrawl.lean.yml up -d`.

### Firecrawl Cloud statt Self-Hosting

`FIRECRAWL_API_KEY=fc-…` setzen und `FIRECRAWL_API_URL` weglassen (Varianten 2 bis 4; der
Docker-Dienst aus dem Schnellstart ist auf die eigene Instanz verdrahtet). Jeder Abruf kostet einen Credit:
grob 85 pro Tag plus 1.200 je Quell-Update, also etwa **8.000 Credits im Monat**. Beim kleinsten
Tarif zusätzlich `requests_per_minute` passend begrenzen. Die Cloud-Variante ist hier nicht
getestet; alle Messwerte stammen von der selbst gehosteten Instanz.

## Befehle

```text
datamb run        einmal aktualisieren        --force  --league ID  --limit-teams N
datamb schedule   dauerhaft laufen            --run-now  --every-hours H  --at HH:MM
datamb status     Datenstand anzeigen         --json  --max-age-hours H
datamb check      Verbindung, robots.txt und Seitenstruktur prüfen
datamb export     Ausgabedateien aus dem vorhandenen Datensatz neu erzeugen
```

Gemeinsame Optionen: `--config`, `--data-dir`, `-v`, `-q`.
Exit-Codes: 0 in Ordnung, 1 Lauf fehlgeschlagen bzw. Daten veraltet, 2 Konfigurationsfehler, 3 anderer Lauf aktiv.

## Konfiguration

Ohne Konfigurationsdatei gelten sinnvolle Standardwerte. Alle Einstellungen stehen kommentiert in
`config.example.yaml` (kopieren nach `config.yaml`). Reihenfolge:
Standardwerte < `config.yaml` < Umgebungsvariablen (`DATAMB_*`, `FIRECRAWL_*`) < Optionen auf der Kommandozeile.

Weitere frei verfügbare Ligen (Serie A, Eredivisie, Liga Portugal) lassen sich dort mit drei Zeilen ergänzen.

## Firecrawl-Setup

`docker-compose.firecrawl.yml` startet Firecrawl aus den offiziellen, fertigen Images
(API, Playwright-Dienst, Redis, RabbitMQ, PostgreSQL). Drei Anpassungen gegenüber dem Original:

- **Nur lokal erreichbar.** Die API ist an `127.0.0.1` gebunden, weil selbst gehostetes Firecrawl
  keine Anmeldung hat.
- **Sparmodus** (`docker-compose.firecrawl.lean.yml`): startet nur den API-Prozess. Das reicht,
  weil die Pipeline ausschließlich `POST /v2/scrape` benutzt, und senkt den Bedarf auf rund 650 MB.
  `/v2/crawl` und `/v2/batch/scrape` funktionieren in diesem Modus nicht. Auf größeren Servern
  die Datei in `.env` aus `COMPOSE_FILE` streichen.
- **Render-Proxy** (`docker/render-proxy`): Der Browser darf nur die Adressen laden, die die
  Seite zum Darstellen braucht. Ohne ihn blieben Werbe- und Tracking-Skripte hängen, und Firecrawl
  lieferte in rund jedem vierten Fall nach 30 Sekunden eine Seite ohne Tabellen. Mit Proxy:
  2 bis 3 Sekunden pro Seite, keine Ausfälle. Wer dieselbe Firecrawl-Instanz auch für andere
  Websites nutzt, schaltet ihn mit `FIRECRAWL_RENDER_PROXY=` ab oder ergänzt `docker/render-proxy/allowlist`.

Die Images stehen auf `:latest`. Für stabilen Betrieb die getesteten Stände aus `.env.example` aktivieren.

## Umgang mit der Quelle

- **robots.txt** wird bei jedem Lauf gelesen und befolgt. Sie untersagt den Abruf der CSV- und
  XLSX-Datenbankdateien; die Pipeline fordert solche Adressen nie an und liest nur HTML-Seiten.
- **Einschränkung dazu:** Teamseiten müssen in einem Browser dargestellt werden, weil Kaderliste
  und Diagramm erst per JavaScript entstehen. Dabei lädt die Seite ihre eigene Datendatei nach,
  wie bei jedem Besucher. Das lässt sich mit Firecrawl nicht abstellen. Für Spielerseiten ist es
  vermieden (reines HTML), mit Ausnahme von sechs Kontrollabrufen pro Komplettabruf
  (`calibration: false` schaltet sie ab).
- **Last:** höchstens 2 Browser- und 4 einfache Abrufe gleichzeitig, an normalen Tagen ca. 85 Abrufe.
- **Nutzungsrechte:** Die Inhalte gehören DataMB. Die Nutzungsbedingungen erlauben kein
  Vervielfältigen oder Weiterverbreiten ohne Zustimmung. Die Daten daher nur für eigene Auswertungen
  verwenden und nicht veröffentlichen. Für eine weitergehende Nutzung oder die Pro-Ligen DataMB direkt fragen.

## Entwicklung

```bash
python3 -m venv .venv && .venv/bin/pip install -e ".[dev]"
.venv/bin/pytest          # 97 Tests, ohne Netzwerk
.venv/bin/ruff check src tests
```

Die Tests laufen gegen aufgezeichnete Seiten (`tests/fixtures/`) und eine nachgebaute Mini-Version
der Website. Aufbau des Codes in `src/datamb_pipeline/`:

| Modul | Aufgabe |
|---|---|
| `firecrawl_client.py`, `robots.py` | Abruf über Firecrawl, Wiederholungen, robots.txt |
| `parsers/` | HTML → Daten (Teamseite, Spielerseite, Übersicht, Diagrammwerte) |
| `pipeline.py` | Ablauf eines Laufs, Änderungserkennung, Prüfungen |
| `exporters.py`, `storage.py` | Ausgabeformate, Veröffentlichen in einem Schritt, Verlauf |
| `runner.py`, `scheduler.py`, `cli.py` | ein Lauf, Zeitplan, Kommandozeile |

## Wenn etwas nicht läuft

| Meldung | Ursache und Abhilfe |
|---|---|
| `preflight failed: … Connection refused` | Firecrawl läuft nicht: `docker compose ps`, `docker compose logs firecrawl-api` |
| `8 pages in a row failed … page not rendered` | Browser kann die Seite nicht laden: `docker compose logs playwright-service render-proxy`; evtl. braucht die Seite eine neue Adresse in der `allowlist` |
| `radar labels changed on the site` | DataMB hat Kennzahlen geändert. Daten stimmen weiterhin, Spaltennamen folgen der Seite. |
| `unexpected page structure` | Seitenaufbau geändert: `datamb check` zeigt, welcher Seitentyp betroffen ist; Parser in `parsers/` anpassen |
| Liga plötzlich „nicht verfügbar“ | Überschrift auf `/teams/` heißt anders: `site_names` der Liga in `config.yaml` ergänzen |
