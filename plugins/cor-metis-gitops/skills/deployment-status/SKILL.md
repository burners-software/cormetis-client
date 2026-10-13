---
name: "deployment-status"
description: "Was läuft wo? Zustand der Deployments (Argo-CD-Applications) der COR-Dienste: welche Version bzw. welches Image eine App auf einem Environment fährt, ob sie Synced und Healthy ist, seit wann die Version läuft, was davor lief, ob sich zwei Environments unterscheiden und ob es in Nexus ein neueres Image gibt. Nutze diesen Skill bei Fragen wie „welche Version hat ESKS auf central?“, „was läuft auf dev01?“, „ist CGSS healthy?“, „seit wann ist 1.0.2 drauf?“, „unterscheiden sich dev01 und dev02?“, „gibt es schon ein neueres Image?“. Abgrenzung: Warten, bis ein gepushter GitOps-Commit ausgerollt ist, macht deploy-and-wait; die Ursache suchen, warum eine App degraded ist, neu startet oder der Sync scheitert, macht diagnose. Und: eine CORgen-Anlage (Burner, Seriennummer, Messwerte, Störungen) ist kein Deployment — dafür ist cor-metis:burner-status zuständig. Liest read-only über den CGSS-MCP-Server; braucht eine Engineer-Rolle."
---

# Deployment-Status — was läuft wo

Du liest den Zustand von Argo-CD-Applications und die Image-Tags in Nexus. Du **änderst nichts**:
kein Sync, kein Rollback, kein Restart, kein Push. CGSS kann das auch gar nicht — jedes Tool ist
read-only. Wer etwas ändern will, tut das selbst über das GitOps-Repo oder die Argo-CD-Oberfläche.

## Die CGSS-Tools finden

Derselbe Server erscheint je nach Oberfläche unter verschiedenen Präfixen. Maßgeblich ist der Servername im Tool-Namen:
**Der Teil zwischen `mcp__` und `__<tool>` ist `CGSS` oder endet auf `_CGSS`, Groß-/Kleinschreibung egal.** Beispiele für `list_environments`:

- `mcp__CGSS__list_environments` (Claude Desktop / Cowork, Connector `CGSS`)
- `mcp__claude_ai_CGSS__list_environments` (Claude Code, synchronisierter claude.ai-Connector)
- `mcp__plugin_cor-metis-code_CGSS__list_environments` (Claude Code, Plugin `cor-metis-code`)
- `mcp__cgss__list_environments` (Claude Code, von Hand per `claude mcp add` angelegt)

Stehen die Tools nur als „deferred“ da, lade sie per ToolSearch mit einer **Stichwortsuche**
(„CGSS list_environments“) nach — nie per `select:` mit einem fest angenommenen Namen. Deferred heißt
vorhanden. Siehst du dieselben Tools unter zwei Präfixen, ist das eine Dublette; nimm eins davon.

**Fehlen die CGSS-Tools ganz**, fehlt der Connector, oder CGSS hat die Verbindung schon beim Aufbau
abgelehnt — weil der Key falsch ist oder dem Nutzer die Engineer-Rolle (DEVELOPER, OPERATOR, ADMIN
oder MASTER) fehlt. Das kommt nicht als Tool-Ergebnis, die Tools sind dann einfach nicht da (in Claude
Code steht CGSS in `/mcp` auf „failed“). Nicht herumprobieren, sondern auf den Skill
**`cor-metis:setup`** verweisen: *„Ich sehe keinen Zugang zu CGSS — entweder ist der Connector nicht
eingerichtet, oder dein Key hat keine Engineer-Rolle. ‚COR.metis einrichten‘ prüft das.“*

## Environments nie raten

Die Tools je Application brauchen einen Environment-Namen; bei `list_applications` ist er
optional (weggelassen = alle Environments), `list_environments` und `search_images` kennen keinen.
**Ruf zuerst `list_environments` auf** und nimm die Namen von dort. Heute ist nur `central-1`
angebunden; dev01 und dev02 folgen. Fragt jemand nach einem Environment, das nicht in der Liste
steht, heißt das „ist an CGSS nicht angebunden“, nicht „gibt es nicht“. Produktion ist hier
grundsätzlich nicht lesbar.

Umgangssprache übersetzen: „central“, „intern“, „cor-metis-Cluster“ → das Environment aus der Liste,
dessen Beschreibung passt (heute `central-1`). Ist es nicht eindeutig, frag nach.

## Ablauf

**1. Die App finden.** `list_applications` mit `name_contains` — und **ohne** `environment`, wenn
die Frage nicht auf ein Environment festgelegt ist. Die Application-Namen in Argo sind nicht die
Dienstkürzel: „ESKS“ kann als `cor-metis-esks`, `expertsystemknowledgeservice` oder anders heißen.
Nimm einen kurzen, charakteristischen Teil (`esks`, `knowledge`, `cgss`, `gitops`) und probier bei
0 Treffern einen zweiten, bevor du aufgibst. Mehrere plausible Treffer → zeigen und fragen, nicht
den ersten nehmen.

`list_applications` liefert je App schon Sync, Health, Images, Revisionen und „deployed vor“. Für
„was läuft wo?“ und für einen Umgebungsvergleich reicht das oft schon — ein Aufruf über alle
Environments.

**2. Details, wenn gefragt.** `get_application_status(environment, application)` liefert zusätzlich
die Health-Meldung, die letzte Sync-Operation, Argo-Conditions und die nicht gesunden Ressourcen.
Nimm es, wenn jemand wissen will, *ob* eine App wirklich läuft, oder wenn `list_applications` etwas
anderes als `Synced`/`Healthy` zeigt.

**3. Vergleich zwischen Environments.** CGSS vergleicht nicht selbst; **du** vergleichst. Je
Environment den Status holen (aus Schritt 1 oder je ein `get_application_status`), dann Image-Tags
und Revisionen nebeneinanderlegen. Sag konkret, *was* sich unterscheidet (Tag `1.0.1` vs. `1.0.2`,
andere Revision, eines OutOfSync) — nicht nur „unterscheidet sich“. Gleiche Tags, aber verschiedene
GitOps-Revisionen sind normal, wenn je Cluster ein eigener Branch existiert.

**4. „Seit wann?“ / „Was lief davor?“** `get_deploy_history(environment, application)`. Die
Einträge sind GitOps-Revisionen mit Zeitpunkten, keine Image-Tags. Argo hält nur eine kurze
Historie (Standard 10); steht `note` in der Antwort, sag, dass ältere Deployments nicht mehr
sichtbar sind. „Seit wann läuft 1.0.2?“ beantwortest du mit `last_deployed_ago` aus dem Status —
das ist der letzte Deploy, und der muss nicht der Image-Wechsel gewesen sein, wenn danach nur
Konfiguration geändert wurde. Sag das, wenn es zur Frage tut.

**5. „Gibt es ein neueres Image?“** `search_images(name, version?)` in Nexus.

⚠ **Live beobachtet auf central-1 am 2026-10-13, entgegen der Tool-Beschreibung:** der Anfang
eines Image-Namens findet etwas (`claudegitops`, `expertsystem`), ein Teilstring mitten im Namen
nicht (`knowledge` → 0 Treffer). CGSS fragt Nexus per Stichwortsuche (`q`) und filtert erst danach
auf den Teilstring — was Nexus' Stichwortsuche nicht liefert, kann CGSS nicht finden. Deshalb:

- Mit dem **vollen Image-Namen** aus dem Status suchen — der Teil zwischen Registry-Host und `:`,
  ohne Tag; bei einem Pfad zuerst der letzte Pfadteil (`expertsystemknowledgeservice`).
- **0 Treffer bei einem Namensteil heißen nicht „gibt es nicht“.** Mit dem vollen Namen erneut
  suchen, bevor du etwas behauptest.
- `version` engt auf einen Tag-Anfang ein (`1.0`, `1.0.2`) — hilft auch, wenn `truncated` gesetzt ist.

Sortiert ist nach `last_modified`, nicht nach Versionsnummer. „Neuester Push“ und „höchste Version“
sind nicht dasselbe (ein nachgebauter alter Tag steht oben). Vergleich den laufenden Tag mit den
gefundenen und sag, welche Tags **jünger gepusht** sind als der laufende. Ist `truncated` gesetzt,
kann der neueste Tag fehlen — das gehört in die Antwort.

## Teilergebnisse und Fehler

**`errors` heißt „unbekannt“, nicht „nichts“.** `list_applications` nennt in `errors` jedes
Environment, das nicht lesbar war. Die Liste ist dann unvollständig: Eine App, die dort fehlt, kann
sehr wohl laufen. Ebenso `unhealthy_resources: null` mit `unhealthy_resources_error` im Status — das
heißt „konnte ich nicht lesen“, nicht „alles gesund“. Beides gehört in die Antwort.

Fehler kommen als Ergebnis `{"error": {"code": …, "message": …}}`, nicht als Tool-Fehler. Lies den
`code`:

| Code | Bedeutung | Was du sagst / tust |
|---|---|---|
| `application_not_found` | Die App **gibt es nicht oder sie ist für CGSS nicht freigegeben** — bewusst nicht unterscheidbar | Genau so sagen. Nicht behaupten, sie existiere nicht. Namen per `list_applications` prüfen |
| `unknown_environment` | Environment nicht konfiguriert; die Meldung nennt die gültigen | Mit einem gültigen Namen weiter, oder fragen |
| `invalid_argument` | Argument fehlerhaft (leerer `name`, …) | Korrigieren, nicht blind wiederholen |
| `upstream_unavailable` | Argo oder Nexus gerade nicht erreichbar | Vorübergehend; einmal später erneut versuchen ist in Ordnung |
| `upstream_rejected` | Argo/Nexus lehnt CGSS ab — Token abgelaufen oder Rolle falsch konfiguriert | Kein Problem des Nutzers. Sagen, dass der Plattform-Betreiber das Token erneuern muss. Nicht wiederholen |
| `internal_error` | Unerwarteter Fehler in CGSS | Melden; ein erneuter Versuch kann klappen |

## Antwortformat

Eine kurze Tabelle, dann **ein Satz Befund**:

| Environment | App | Image:Tag | Sync | Health | deployed vor |
|---|---|---|---|---|---|
| central-1 | cor-metis-esks | expertsystemknowledgeservice:1.0.0-pre.398 | Synced | Healthy | 2 d 3 h |

*„ESKS läuft auf central-1 in pre.398, synchron und gesund.“*

Bei einer einzelnen, einfachen Frage („welche Version hat ESKS?“) reicht der Satz ohne Tabelle. Den
Registry-Host im Image darfst du weglassen, den Tag nie. Läuft eine App mit mehreren Images, alle
nennen. Keine Revisions-SHAs in die Tabelle, außer jemand fragt danach oder sie sind der
Unterschied.

## Gotchas

- Read-only. Bietet sich ein Sync oder Rollback an, sag, dass das der Nutzer selbst auslöst — nie
  so tun, als könntest du es.
- Environments aus `list_environments`, nie geraten. Fehlt eins: „nicht angebunden“, nicht „gibt es
  nicht“.
- `application_not_found` = „gibt es nicht oder ist nicht freigegeben“. Wörtlich so.
- `errors` und `unhealthy_resources: null` = unbekannt, nicht leer.
- `search_images`: voller Image-Name; 0 Treffer bei einem Namensteil beweisen nichts. Sortierung
  nach Push-Zeit, nicht nach Version.
- `get_deploy_history` zeigt GitOps-Revisionen, keine Image-Tags, und nur die letzten ~10.
- Anlage ≠ Deployment. „Wie geht's der cor-xc001?“ ist `cor-metis:burner-status`.
- Degraded, CrashLoop, Sync failed → nicht hier herumdiagnostizieren, sondern
  `cor-metis-gitops:diagnose`. „Sag Bescheid, wenn’s läuft“ → `cor-metis-gitops:deploy-and-wait`.
