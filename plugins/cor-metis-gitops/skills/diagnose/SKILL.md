---
name: "diagnose"
description: "Findet heraus, warum ein Deployment (Argo-CD-Application) eines COR-Dienstes nicht läuft: Degraded, CrashLoopBackOff, ImagePullBackOff, ständige Neustarts, fehlgeschlagener Sync, Progressing ohne Ende. Nutze diesen Skill bei „warum ist ESKS degraded?“, „warum startet CGSS dauernd neu?“, „der Sync auf central ist failed“, „warum geht X nicht?“, „was steht in den Logs von Y?“ — und wenn deploy-and-wait mit failed endet. Liest Status, Kubernetes-Events, Logs und Deploy-Historie; liefert gerankte Ursache, Belege und nächsten Schritt. Behebt nichts (kein Rollback, Sync, Restart). Abgrenzung: „was läuft wo / welche Version“ ohne Problem ist deployment-status; auf einen frisch gepushten Commit warten ist deploy-and-wait; klemmt der Zugang/Connector selbst (ESKS/CGSS antwortet nicht, Tools fehlen) → cor-metis:setup; eine CORgen-Anlage mit Störung (Burner, Seriennummer, Fehlercode) ist kein Deployment, dafür ist cor-metis:burner-status da. Liest read-only über den CGSS-MCP-Server; braucht eine Engineer-Rolle."
---

# Diagnose — warum läuft das Deployment nicht

Du suchst die Ursache, belegst sie und sagst, was als Nächstes zu tun ist. Du **behebst nichts**:
kein Rollback, kein Sync, kein Restart, kein Push. CGSS ist read-only. Das sagst du am Ende klar,
statt einen Fix anzudeuten, den es von hier aus nicht gibt.

## Die CGSS-Tools finden

**Der Teil zwischen `mcp__` und `__<tool>` ist `CGSS` oder endet auf `_CGSS`, Groß-/Kleinschreibung egal.**
Beispiele: `mcp__CGSS__get_application_status` (Desktop/Cowork),
`mcp__claude_ai_CGSS__get_application_status` (Claude Code, synchronisierter Connector),
`mcp__plugin_cor-metis-code_CGSS__get_application_status` (Plugin `cor-metis-code`),
`mcp__cgss__get_application_status` (von Hand angelegt). Deferred → per ToolSearch-Stichwortsuche
„CGSS get_application_logs“ nachladen, nie per `select:` mit festem Namen.

**Fehlen die CGSS-Tools ganz**, ist der Connector nicht eingerichtet oder CGSS hat die Verbindung
abgelehnt (falscher Key oder keine Engineer-Rolle; in Claude Code steht CGSS in `/mcp` auf
„failed“). Nicht herumprobieren — auf **`cor-metis:setup`** verweisen.

## Environment und App

Environment-Namen aus `list_environments`, nie geraten. Heute ist nur `central-1` angebunden;
dev01 und dev02 folgen. Fehlt ein Environment dort, ist es nicht angebunden — nicht „gibt es
nicht“. Den exakten App-Namen bei Bedarf über `list_applications` mit `name_contains`.

Kommst du aus `cor-metis-gitops:deploy-and-wait`, hast du Environment, App, Revision und die
unhealthy resources schon — dann mit diesen einsteigen und Schritt 1 nur für das holen, was fehlt.

## Ablauf

Erst breit, dann gezielt. Jeder Schritt soll eine Hypothese stützen oder verwerfen; hör auf, wenn
die Ursache belegt ist.

**1. Status — `get_application_status(environment, application)`.** Lies:

- `health` (Status und Meldung) und `sync`,
- `last_operation` — `phase` `Failed`/`Error` und deren `message` sind bei Sync-Problemen oft
  schon die Antwort (Hook gescheitert, Manifest ungültig, Ressource abgelehnt),
- `conditions` — `ComparisonError`, `SyncError` u.ä. zeigen Probleme, bevor überhaupt ein Pod läuft,
- `unhealthy_resources` — welche Ressource woran krankt (Degraded zuerst, höchstens 20,
  `unhealthy_resources_truncated`). Ist es `null` mit `unhealthy_resources_error`, ist der Zustand
  der Ressourcen **unbekannt**, nicht gesund.
- `out_of_sync_resources` — welche Ressourcen vom Repo abweichen (Kind, Namespace, Name, Status,
  `requires_pruning`; höchstens 20, `out_of_sync_resources_truncated`). Bei `OutOfSync` **zuerst
  hier hinschauen.** `requires_pruning: true` heißt: steht im Cluster, aber nicht mehr im Repo. Den
  Diff selbst liefert CGSS bewusst nicht (Live-Manifeste können Secrets enthalten) — die Ursache
  einer Abweichung findest du im Repo bzw. in der Argo-CD-Oberfläche. Fehlt das Feld in der Antwort,
  läuft eine ältere CGSS-Version; dann die Abweichung über das Repo erschließen.

**OutOfSync bei Healthy, und Argo synct immer wieder „erfolgreich“** (Events `OperationCompleted`
*„Partial sync operation … succeeded“* im Abstand weniger Minuten, danach sofort wieder OutOfSync):
das ist **dauerhafte Drift**, kein kaputter Rollout. Argo wendet das Manifest an, aber der Cluster
schreibt Felder zurück, die im Repo fehlen. Typische Ursache: eine Ressource mit vom Controller
ergänzten Feldern (z.B. HTTPRoute mit CRD-Defaults und Gateway-Write-backs) ohne
`argocd.argoproj.io/compare-options: ServerSideDiff=true` an der Application. Vergleich mit
gleichartigen Apps im GitOps-Repo — fehlt die Option nur hier, ist das der Befund.

**2. Events — `get_application_events(environment, application)`**, zuerst **ohne**
`resource_name`. Das liefert die Events der Application plus die von bis zu fünf kranken
Ressourcen; meistens reicht das für `BackOff`, `Failed` (ImagePull), `Unhealthy` (Probes),
`FailedScheduling`, `Killing`. Erst wenn eine bestimmte Ressource genauer interessiert, mit
`resource_name` (Name aus Schritt 1). **Kubernetes hält Events nur etwa eine Stunde.** Keine Events
heißt bei einem älteren Problem nicht „nichts passiert“.

**3. Logs — `get_application_logs(environment, application, …)`**, wenn Events auf den Container
selbst zeigen (Absturz, Probe schlägt fehl, App startet nicht).

- **Neustarts / CrashLoopBackOff: `previous: true`.** Der Grund für den Absturz steht im Log der
  *vorigen* Container-Instanz, nicht im frisch gestarteten.
- Einen bestimmten Pod mit `pod` (Name aus Status oder Events); ohne `pod` werden die fünf jüngsten
  Pods gelesen.
- `filter` für Fehlerwörter (`exception`, `error`, `fatal`, `unhandled`) — ein Wort pro Aufruf. Der
  Filter wirkt nur auf die gelesenen `tail_lines`; er sucht nicht weiter zurück. Wer filtert, nimmt
  ruhig `tail_lines` bis 1000.
- Grenzen: höchstens 5 Pods, 64 KiB Text (die neuesten Zeilen bleiben). `truncated`,
  `dropped_lines` und `timed_out` heißen: es gibt mehr, als du siehst. `errors` nennt Pods, die
  nicht lesbar waren.

**`logs_not_released` ist eine bewusste Freigabeentscheidung, kein Fehler.** Die Logs dieser App
sind für CGSS absichtlich gesperrt (auf central-1 z.B. mongo-express und der Customer-Support-Bot).
So formulieren — *„Die Logs dieser App sind für mich bewusst nicht freigegeben.“* —, nicht erneut
versuchen und mit Status und Events weiterarbeiten.

**4. Was hat sich geändert? — `get_deploy_history(environment, application)`.** Wenn das Problem mit
einem Deploy zusammenfallen könnte: wann der letzte Deploy war, welche Revision davor lief. Passt der
Zeitpunkt des Problems zum letzten Deploy, ist die Änderung der erste Verdächtige. Die Historie führt
GitOps-Revisionen, keine Tags, und nur die letzten ~10. `images` im Status zeigt nur die **aktuellen**
Images, nicht die früheren. Für „hat sich das Image (oder die Konfiguration) geändert?“:

- **Claude Code:** im GitOps-Repo `git diff <vorige Revision> <aktuelle Revision>` — beide aus
  `get_deploy_history` (ggf. vorher `git fetch`). Der Diff zeigt Tag-Wechsel und
  Konfigurationsänderungen gleichermaßen.
- **Desktop:** nachfragen, was mit dem letzten Deploy geändert wurde.

## Teilergebnisse und Fehler

`errors` in Events und Logs und `unhealthy_resources: null` heißen **unbekannt**, nicht „nichts“.
Was fehlt, gehört in die Antwort, sonst klingt ein Befund sicherer, als er ist.

Fehler kommen als Ergebnis `{"error": {"code": …, "message": …}}`:

| Code | Bedeutung | Umgang |
|---|---|---|
| `application_not_found` | Die App **gibt es nicht oder sie ist für CGSS nicht freigegeben** (bewusst nicht unterscheidbar) | Genau so sagen; Namen mit `list_applications` prüfen |
| `logs_not_released` | Logs dieser App bewusst gesperrt | Freigabeentscheidung, s.o.; ohne Logs weiter |
| `invalid_argument` | Meist ein `pod`/`resource_name`, der nicht im Ressourcenbaum steht, oder `since_seconds` < 1 | Namen aus Status/Events nehmen |
| `unknown_environment` | Environment nicht konfiguriert; Meldung nennt die gültigen | Korrigieren |
| `upstream_unavailable` | Argo gerade nicht erreichbar | Vorübergehend; einmal später erneut |
| `upstream_rejected` | Argo lehnt CGSS ab (Token abgelaufen o.ä.) | Plattform-Betreiber muss das Token erneuern; nicht wiederholen |
| `internal_error` | Unerwarteter Fehler in CGSS | Melden |

## Ergebnis

Drei Teile, in dieser Reihenfolge:

1. **Ursache** — oder, wenn nicht eindeutig, die **wahrscheinlichsten Ursachen, gerankt**, mit
   einem Wort zur Sicherheit („belegt“, „wahrscheinlich“, „möglich“).
2. **Belege** — knappe, wörtliche Zitate aus Events, Status oder Logs, mit Quelle
   (*Event `BackOff` an Pod `…-7d9f`, 14× in 10 min*; *Log: `MongoAuthenticationException: …`*).
   Zwei, drei Zeilen, die die Ursache tragen — nicht seitenweise Log.
3. **Nächster Schritt** — was der Nutzer tun kann (Konfiguration im GitOps-Repo korrigieren, auf
   die vorige Revision zurück, Secret prüfen, Image neu bauen …). Und ausdrücklich: **ein Rollback,
   Sync oder Restart geht nicht von hier aus**, das macht der Nutzer über das GitOps-Repo bzw. Argo
   CD.

Log-Inhalte nicht ungefragt ausgeben, schon gar nicht seitenweise — sie können personenbezogene
Daten enthalten. Zitiere, was die Ursache belegt; den Rest auf Nachfrage. Sieht eine Zeile nach
Token, Passwort oder Key aus, gib sie nicht wieder, sondern sag, dass dort etwas Geheimes steht.

## Gotchas

- Behebt nichts. Rollback, Sync, Restart: nicht von hier aus — klar sagen.
- Events zuerst ohne `resource_name`; Events leben ~1 h, leer heißt bei alten Problemen nichts.
- Neustart / CrashLoop → Logs mit `previous: true`, sonst siehst du nur den frischen Start.
- `filter` durchsucht nur die gelesenen `tail_lines`, nicht die ganze Vergangenheit.
- `logs_not_released` = Freigabeentscheidung. So sagen, nicht wiederholen, ohne Logs weiter.
- `application_not_found` = „gibt es nicht oder ist nicht freigegeben“.
- `errors`, `unhealthy_resources: null`, `truncated` = unvollständig. In die Antwort damit.
- Belege zitieren, Logs nicht auskippen, Geheimnisse nie wiedergeben.
- CORgen-Anlage mit Störung ≠ kaputtes Deployment → `cor-metis:burner-status`.
