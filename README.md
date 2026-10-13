# cormetis-client

Marketplace und Plugin-Paket für **COR.metis** — das Wissens-Orakel der COR Energy World GmbH in
Claude Desktop und Claude Code.

Dieses Repo ist gleichzeitig Marketplace *und* Plugin-Container. Kollegen fügen es einmal als
Marketplace hinzu, installieren die Plugins, die sie brauchen, und bekommen Skill-Updates danach
automatisch.

> **Du willst COR.metis nur benutzen?** → **[docs/ONBOARDING.md](docs/ONBOARDING.md)**
> Fünf Minuten, keine Vorkenntnisse nötig. Der Rest dieser Datei richtet sich an die Wartung.

---

## Aufbau

```
.claude-plugin/marketplace.json     Marketplace-Manifest (listet die drei Plugins)
plugins/
├── cor-metis/                      Für alle, Desktop/Cowork und Claude Code
│   ├── .claude-plugin/plugin.json  Name, Version, Beschreibung
│   ├── README.md
│   └── skills/
│       ├── orakel/                 Retrieval aus der Knowledgebase, clearance-abhängig
│       ├── burner-status/          Momentzustand einer Anlage per Seriennummer
│       ├── ingest/                 Dokumente in die KB einspielen
│       └── setup/                  Einrichtung + Selbsttest (inkl. optionalem CGSS-Check)
├── cor-metis-gitops/               Für Entwickler, Desktop/Cowork und Claude Code
│   ├── .claude-plugin/plugin.json
│   ├── README.md
│   └── skills/
│       ├── deployment-status/      Was läuft wo, in welcher Version (read-only über CGSS)
│       ├── deploy-and-wait/        Auf einen GitOps-Commit warten, bis er ausgerollt ist
│       └── diagnose/               Warum eine Application degraded ist oder neu startet
└── cor-metis-code/                 Nur Claude Code, keine Skills
    ├── .claude-plugin/plugin.json  userConfig: der persönliche Key
    ├── .mcp.json                   Server ESKS und CGSS, Key als Bearer-Header
    └── README.md
docs/ONBOARDING.md                  Die Anleitung für Kollegen
docs/specs/                         Entscheidungen und Begründungen
```

| Plugin | Desktop/Cowork | Claude Code | Wer installiert |
|---|---|---|---|
| `cor-metis` | ja | ja | alle |
| `cor-metis-gitops` | ja | ja | Entwickler mit Engineer-Rolle |
| `cor-metis-code` | nein | ja | Entwickler in Claude Code |

Details je Plugin stehen in dessen README: [`cor-metis`](plugins/cor-metis/README.md),
[`cor-metis-gitops`](plugins/cor-metis-gitops/README.md), [`cor-metis-code`](plugins/cor-metis-code/README.md).
Schnitt und Begründung: [`docs/specs/2026-10-13-cor-metis-gitops-und-code.md`](docs/specs/2026-10-13-cor-metis-gitops-und-code.md).

---

## Was ist zu konfigurieren

Alle Wege nutzen **denselben persönlichen Key** für ESKS und CGSS. CGSS verlangt zusätzlich eine
Engineer-Rolle (DEVELOPER, OPERATOR, ADMIN oder MASTER); ohne sie lehnt CGSS schon den
Verbindungsaufbau mit 403 ab.

| | Claude Desktop / Cowork | Claude Code |
|---|---|---|
| Marketplace | *Einstellungen → Plugins → Marketplace hinzufügen*, Repo-URL | `/plugin marketplace add burners-software/cormetis-client` |
| Plugins | `cor-metis`, für Entwickler `cor-metis-gitops` | `/plugin install cor-metis@cor-metis-client`<br>`/plugin install cor-metis-gitops@cor-metis-client`<br>`/plugin install cor-metis-code@cor-metis-client` |
| Zugang ESKS | Connector `ESKS`: `https://internal.cor.energy/esks/mcp?apikey=DEIN_KEY` | bringt `cor-metis-code` mit |
| Zugang CGSS (Entwickler) | Connector `CGSS`: `https://internal.cor.energy/cgss/mcp?apikey=DEIN_KEY` | bringt `cor-metis-code` mit |
| Key | steht in der Connector-URL | wird beim Aktivieren von `cor-metis-code` maskiert abgefragt; ändern mit `claude plugin configure cor-metis-code` |
| Aufräumen | — | synchronisierte Server `claude.ai ESKS` / `claude.ai CGSS` in `/mcp` deaktivieren |

Der letzte Punkt betrifft jeden, der die Connectoren auch in Desktop hat: Claude Code übernimmt sie
über das claude.ai-Konto automatisch. Mit `cor-metis-code` ist ESKS dann zweimal konfiguriert. Claude Code
erkennt die Dublette und unterdrückt eine Seite — welche, ist nicht festgelegt (im Test 2026-10-13 verlor
in Headless-Läufen das Plugin). Der synchronisierte Weg trägt den Key in der URL; abschalten stellt sicher,
dass der Plugin-Weg mit dem Key im Header genommen wird.

Schritt für Schritt für Nutzer: [docs/ONBOARDING.md](docs/ONBOARDING.md). Key-Speicherort und
Engineer-Rolle in Claude Code: [`plugins/cor-metis-code/README.md`](plugins/cor-metis-code/README.md).
Geführt in Claude: `cor-metis:setup`, Teil B1 (Desktop/Cowork) bzw. B2 (Claude Code).

---

## `.mcp.json` nur in `cor-metis-code`

Der Zugang hängt an einem persönlichen Key. Ein Paket, das alle bekommen, kann nichts
Personenbezogenes transportieren — es kann den Key höchstens beim Nutzer abfragen (`userConfig`) und
per `${user_config.api_key}` einsetzen. Das funktioniert nicht überall:

- **Cowork** ignoriert MCP-Server, die `${user_config.*}` ohne Default referenzieren, und fragt nichts
  ab. Schon früher getestet und widerlegt: Cowork Desktop substituiert `${user_config.KEY}` auch in
  einer Connector-URL nicht, der Platzhalter landet wörtlich darin. Beleg:
  `cormetis-skills-raw/docs/COR.metis-Deployment-Spec.md`, Abschnitt 2–3.
- **Claude Desktop (Chat):** wie es auf einen `${user_config.*}`-Header reagiert, ist nicht eindeutig
  dokumentiert.
- **Claude Code** fragt den Key beim Installieren maskiert ab und legt ihn im eigenen
  Credential-Store ab — bestätigt am 2026-10-13 (Claude Code 2.1.296, Linux: `~/.claude/.credentials.json`,
  Rechte `0600`). Der Key stand danach weder in `~/.claude.json`, `settings.json`, den Debug-Logs noch im
  Repo.

Deshalb liegt das einzige `.mcp.json` in `cor-metis-code`, einem Plugin ohne Skills, das ausdrücklich
nur für Claude Code beschrieben ist. `cor-metis` und `cor-metis-gitops` bleiben frei von jeder
Konfiguration und laufen überall gleich. Ein Default für `api_key` ist tabu — er wäre ein geteilter Key.

Konsequenz für Desktop und Cowork: **Die Plugins liefern die Skills, den Connector richtet jeder
einmalig selbst ein.** Das Teil, das sich häufig ändert, wird verteilt; das Teil, das sich nie
ändert, wird einmal gesetzt.

---

## Voraussetzungen an das Repo

- **Das Repo muss öffentlich sein.** Private Marketplace-Repos hängen an den Git-Credentials jedes
  einzelnen Kollegen; die Hintergrund-Updates sind dort unzuverlässig. Inhaltlich unbedenklich —
  bewusst preisgegeben sind der Hostname `internal.cor.energy`, die Pfade `/esks/mcp` und
  `/cgss/mcp` und die Tatsache, dass es einen Argo-CD-Lesezugang gibt. Mit Chris abgestimmt; nichts
  davon ist geheimer als das, was die ESKS-Skills ohnehin zeigen.
- **Keine Keys im Repo.** Nirgends, auch nicht als Beispiel. Der Platzhalter heißt `DEIN_KEY`.

---

## Skill-Namespacing

Plugin-Skills werden mit dem Plugin-Namen präfigiert. Die Ordner heißen deshalb `orakel/`,
`burner-status/`, `ingest/`, `setup/` — daraus wird `cor-metis:orakel` usw., analog
`cor-metis-gitops:deploy-and-wait`. Die `name:`-Felder in den Frontmattern sind entsprechend gekürzt;
hießen sie weiter `cor-metis-orakel`, entstünde `cor-metis:cor-metis-orakel`.

Die `description:`-Felder steuern das Triggering und sind gegeneinander abgegrenzt — beim Ändern
darauf achten, dass die Abgrenzung zwischen `orakel` (Dokumentation) und `burner-status` (Maschine)
erhalten bleibt. Es gibt bewusst nur **einen** `setup`-Skill (`cor-metis:setup`); er prüft CGSS
optional mit. Ein zweiter hätte überlappende Auslöser.

---

## Tool-Namen: Server-Name statt Präfix

Derselbe MCP-Server erscheint je nach Oberfläche unter einem anderen Präfix:

| Oberfläche | ESKS-Tool `get_stats` heißt |
|---|---|
| Claude Desktop / Cowork, Connector `ESKS` | `mcp__ESKS__get_stats` |
| Claude Code, claude.ai-Connector synchronisiert | `mcp__claude_ai_ESKS__get_stats` |
| Claude Code, Plugin `cor-metis-code` | `mcp__plugin_cor-metis-code_ESKS__get_stats` |

Dazu kommen von Hand per `claude mcp add` angelegte Server mit frei gewähltem Namen (oft klein `cgss`).

Die Skills nennen deshalb **nie einen festen Präfix**, sondern den Server und den nackten Tool-Namen
(„`get_stats` vom ESKS-Server“). Die Regel: *der Teil zwischen `mcp__` und `__<tool>` ist `ESKS` bzw.
`CGSS` oder endet auf `_ESKS` bzw. `_CGSS`, Groß-/Kleinschreibung egal.* Nachgeladene („deferred“)
Tools werden per ToolSearch-Stichwortsuche gefunden („ESKS get_stats“), nie per `select:` mit festem
Namen. Wer einen Skill ändert oder neu schreibt, hält sich daran — ein fest verdrahteter Präfix
funktioniert genau auf einer Oberfläche.

---

## Eine Änderung ausliefern

1. Skill (bzw. Konfiguration) bearbeiten.
2. `version` des **betroffenen Plugins** an **beiden Stellen** hochziehen —
   `plugins/<plugin>/.claude-plugin/plugin.json` und der passende Eintrag in
   `.claude-plugin/marketplace.json`. Laufen die auseinander, zieht Cowork nicht sauber nach. Jedes
   Plugin hat seine eigene Version; unbeteiligte Plugins bleiben, wie sie sind.
3. `claude plugin validate .` laufen lassen.
4. Committen und pushen.

Cowork prüft gegen den Marketplace und aktualisiert im Hintergrund. Lokal veränderte Dateien werden
vor dem Überschreiben gemeldet.

**Semver-Disziplin:**

| Änderung | Bump |
|---|---|
| Formulierungen, Tippfehler, Beispiele | Patch |
| Neue Regel, neuer Skill, erweiterte Abdeckung | Minor |
| Verhaltensänderung, die jemand aktiv kennen muss | Major |

Ein späterer Split (z.B. `ingest` in ein eigenes Plugin) bleibt über `renames` im Marketplace-Manifest
verlustfrei.

---

## Vor dem Rollout offen

Aus der Deployment-Spec, Abschnitt 7 — serverseitig, blockierend:

- [x] **`apikey` aus den Query-Logs.** Erledigt (Felix, 2026-08-12) — nginx protokolliert Query-Parameter
      gar nicht mehr, statt nur `apikey` zu maskieren. Für `/cgss` bestätigt (Felix, 2026-10-13): Envoy
      loggt für alle Pfade ohne Query-String. Offen bleibt: Error-Tracing/APM hängt die volle URL gern
      an den Trace.
- [ ] **Keys URL-safe erzeugen** (base64url oder Hex) — `+`, `/`, `=` zerlegen den Query-String.
- [ ] **Keys pro Person, widerrufbar.** Ein geteilter Key hebelt das Clearance-Modell aus.
- [x] **MCP-Endpoint-Pfad bestätigen.** Erledigt (2026-10-13): `/esks/mcp` und `/cgss/mcp` laufen live auf central-1.

Weiter offen, nicht blockierend:

- [ ] `burner-status`: ob ein **gemappter** Fehlercode ein `label`-Feld mitbringt, ist gegen echte
      Daten unverifiziert.
- [x] Claude Code: `userConfig` funktioniert. Live-Test 2026-10-13: maskierte Key-Abfrage, ESKS und CGSS
      über den Bearer-Header verbunden, Key-Leck-Probe ohne Treffer, Skills `deployment-status`,
      `deploy-and-wait` und `setup` gegen central-1 grün.

---

## Migrationspfad OAuth

Bekommt ESKS OAuth, ändert sich das Bild grundlegend: das Plugin kann eine URL **ohne Geheimnis**
mitliefern, der Connector wandert zurück ins Paket, das Onboarding schrumpft von vier Schritten auf
zwei. Kein Blocker für jetzt — aber der Grund, warum OAuth mehr ist als Kosmetik.

---

## Verwandte Repos

| Repo | Inhalt |
|---|---|
| `cormetis-skills-raw` | Arbeitsstand der Skills, Deployment-Spec, UX-Dokument, Spike-Beleg |
| `claude-plugins-internal` | Interner Marketplace für Entwickler-Plugins (`cor-dev-guide`) |

## Lizenz

Proprietär — COR Energy World GmbH.
