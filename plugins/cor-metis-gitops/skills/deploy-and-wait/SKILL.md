---
name: "deploy-and-wait"
description: "Wartet, bis ein gepushter GitOps-Commit in Argo CD ausgerollt ist, und meldet das Ergebnis. Nutze diesen Skill, wenn jemand einen Commit im GitOps-Repo (z.B. argocd-platform-configuration) gepusht hat und wissen will, wann er läuft: „ich hab gepusht, sag Bescheid wenn’s läuft“, „warte bis 1.0.2 auf central ist“, „ist mein Deploy durch?“, „beobachte den Rollout“. Bestimmt die richtige Revision (den GitOps-Commit, nicht den Code-Commit und nicht den Image-Tag) und die betroffenen Applications, wartet per wait_for_revision und übergibt bei einem Fehlschlag an cor-metis-gitops:diagnose. Pusht nie selbst und löst nie einen Sync aus. Abgrenzung: eine reine Momentaufnahme „was läuft wo / welche Version“ ist deployment-status; die Ursachensuche bei einer kaputten App ohne laufenden Rollout ist diagnose; eine CORgen-Anlage (Burner, Seriennummer) ist kein Deployment, dafür ist cor-metis:burner-status da. Liest read-only über den CGSS-MCP-Server; braucht eine Engineer-Rolle."
---

# Deploy and wait — bis der Commit läuft

Jemand hat einen Commit ins GitOps-Repo gepusht und will wissen, wann er läuft — oder warum nicht.
Du bestimmst, **worauf** gewartet wird, wartest geduldig und knapp, und meldest das Ergebnis.

Du **pushst nicht** und **löst keinen Sync aus**. CGSS ist read-only, `wait_for_revision` schaut nur
zu. Ist der Commit noch nicht gepusht, sagst du das und wartest, bis der Nutzer es getan hat.

## Die CGSS-Tools finden

Maßgeblich ist **das Segment unmittelbar vor `__<tool>`: es heißt `CGSS`, Groß-/Kleinschreibung
egal** — z.B. `mcp__CGSS__wait_for_revision` (Desktop/Cowork), `mcp__claude_ai_CGSS__wait_for_revision`
(Claude Code, synchronisierter Connector), `mcp__plugin_cor-metis-code_CGSS__wait_for_revision`
(Plugin `cor-metis-code`), `mcp__cgss__wait_for_revision` (von Hand angelegt). Deferred → per
ToolSearch-Stichwortsuche „CGSS wait_for_revision“ nachladen, nie per `select:` mit festem Namen.

**Fehlen die CGSS-Tools ganz**, ist der Connector nicht eingerichtet oder CGSS hat die Verbindung
abgelehnt (falscher Key oder keine Engineer-Rolle; in Claude Code steht CGSS in `/mcp` auf
„failed“). Nicht herumprobieren — auf **`cor-metis:setup`** verweisen.

## Schritt 1 — Die Revision bestimmen

**Die Revision ist der Commit im GitOps-Repo** (z.B. `argocd-platform-configuration`). Nicht der
Commit im Code-Repo, nicht der Image-Tag. Argo kennt nur GitOps-Commits. Sagt jemand „warte bis
1.0.2 auf central ist“, dann ist 1.0.2 der Image-Tag, und gewartet wird auf den GitOps-Commit, der
diesen Tag einträgt.

### In Claude Code (Git verfügbar)

1. **Das GitOps-Repo verstehen, bevor du rätst.** Lies dessen `README.md` und, falls vorhanden,
   `.github/copilot-instructions.md`. Dort steht, wie Branches auf Cluster abbilden (in
   `argocd-platform-configuration` ist **jeder Branch ein Cluster**, z.B. `central-1`) und wie
   Verzeichnisse auf Applications abbilden (`apps/<name>.yaml` deklariert die Application, die
   Werte liegen in einem eigenen Verzeichnis).
2. **Ist `HEAD` gepusht?** `git status -sb` (kein `ahead`) und `git branch -r --contains HEAD`
   (enthält den Remote-Branch des Clusters). Nicht gepusht → sagen und aufhören. **Nicht selbst
   pushen**, auch nicht auf Nachfrage im Vorbeigehen; der Nutzer pusht.
3. **Revision:** `git rev-parse HEAD`.
4. **Environment:** aus dem Branch (Cluster) ableiten und gegen `list_environments` prüfen. Steht
   das Environment dort nicht, ist es an CGSS nicht angebunden (heute nur `central-1`; dev01 und
   dev02 folgen) — dann kannst du nicht warten, sag das.
5. **Applications:** aus den geänderten Pfaden. `git show --name-only --format= HEAD`; hat der Nutzer
   mehrere Commits auf einmal gepusht, die Pfade seit dem Stand, den Argo gerade zeigt. Pfade über
   die Application-Manifeste den Application-Namen zuordnen und mit `list_applications`
   (`environment`, `name_contains`) bestätigen. **Ein Commit kann mehrere Applications oder
   Environments treffen** — dann wird je App gewartet. Ist die Zuordnung unklar, frag, statt eine
   App zu raten.

### In Claude Desktop / Cowork (kein Git)

Frag nach dem, was du nicht selbst ermitteln kannst — in einer Nachricht:

- **Commit-SHA** im GitOps-Repo, mindestens 7 Hex-Zeichen,
- **Environment** (zeig die Namen aus `list_environments`),
- **Application** (bei Unsicherheit `list_applications` mit `name_contains` anbieten).

Ein Image-Tag oder ein Code-Commit ist keine Antwort auf die SHA-Frage; erklär kurz, warum.

## Schritt 2 — Warten

`wait_for_revision(environment, application, revision, timeout_seconds: 25)` in einer Schleife.

- **Nur bei `state: pending` erneut aufrufen.** `reached` und `failed` sind Endzustände.
- **Nach jedem Aufruf eine knappe Zeile** zum `observed`-Stand: Sync, Health, Operationsphase. Etwa
  *„… OutOfSync, Sync läuft (Running)“*, *„… Synced, Progressing — Pods starten“*. Keine Absätze pro
  Runde; das Interessante kommt am Ende.
- **`pending` mit `last_error`**: Argo war bei der letzten Abfrage nicht erreichbar
  (`upstream_unavailable`). Das ist kein Abbruchgrund, aber nenn es in der Zeile mit.
- **Abbruch nach etwa 10 Minuten** `pending` (rund 24 Aufrufe). Dann den letzten `observed`-Stand
  melden und sagen, woran es hängt, so weit `observed` das hergibt. Kein stilles Weiterwarten.
- Mehrere Applications: je App eine eigene Schleife, abwechselnd aufrufen, Ergebnisse getrennt
  melden.

### Die beobachtete Revision ist eine andere

`wait_for_revision` vergleicht **exakt per Präfix, nicht nach Abstammung.** Argo zeigt den
aufgelösten HEAD des Ziel-Branches. Landet nach dem Push noch ein Commit (von jemand anderem oder
ein zweiter eigener), wird die angefragte SHA nie `reached`, obwohl sie längst drin ist. Vergleich
deshalb bei `pending` `observed.revisions` mit der angefragten Revision:

- **Claude Code:** `git fetch`, dann `git merge-base --is-ancestor <angefragt> <beobachtet>`.
  Exit 0 → der Commit ist in der beobachteten Revision enthalten: **auf die beobachtete Revision
  weiterwarten** und das sagen (*„Argo steht schon auf `abc1234`, das deinen Commit enthält — ich
  warte darauf.“*). Ist die beobachtete Revision ein **Vorgänger** der angefragten, hat Argo den
  Commit nur noch nicht abgeholt — normal weiterwarten. Weder noch → Branch oder App passen nicht;
  melden und aufhören.
- **Desktop:** die abweichende Revision nennen und fragen, ob sie den Commit enthält oder ob
  weitergewartet werden soll.

### OutOfSync, und nichts passiert

Zeigt `observed` über **zwei Aufrufe hinweg** `OutOfSync` ohne laufende Operation (keine
`operation_phase` `Running`), dann synct Argo diese App vermutlich nicht automatisch — mit
Auto-Sync startet die Operation Sekunden nach dem Erkennen der Änderung. Nicht bis zum Timeout warten, sondern
melden: *„Argo synct `<app>` vermutlich nicht automatisch — der Sync muss in Argo CD ausgelöst
werden. Das kann ich von hier aus nicht.“*

### Fehler beenden die Schleife sofort

Kommt statt eines Zustands `{"error": {"code": …}}`, **sofort aufhören** und melden. Kein zweiter
Aufruf; kein späterer kann das Ergebnis ändern.

| Code | Bedeutung |
|---|---|
| `application_not_found` | Die App **gibt es nicht oder sie ist für CGSS nicht freigegeben** (bewusst nicht unterscheidbar). Namen mit `list_applications` prüfen |
| `unknown_environment` | Environment nicht konfiguriert; die Meldung nennt die gültigen Namen |
| `invalid_argument` | Meist die Revision: keine 7–40 Hex-Zeichen |
| `upstream_rejected` | Argo lehnt CGSS ab (Token abgelaufen o.ä.). Kein Problem des Nutzers — der Plattform-Betreiber muss das Token erneuern |
| `internal_error` | Unerwarteter Fehler in CGSS; melden |

`upstream_unavailable` kommt hier nicht als Fehler, sondern als `pending` mit `last_error` (s.o.).

## Schritt 3 — Ergebnis

- **`reached`:** ein Satz. *„`cor-metis-esks` auf central-1 läuft mit `abc1234`: Synced, Healthy.“*
  Mehr nicht, außer jemand fragt.
- **`failed`:** den Skill **`cor-metis-gitops:diagnose`** aufrufen und dabei als Einstieg mitgeben:
  Environment, Application, die angefragte Revision, `observed` (Operationsphase und -meldung,
  Health-Meldung) und die `unhealthy_resources` aus der Antwort. Den Diagnose-Ablauf **nicht hier
  nachbauen**. Ist `unhealthy_resources` `null` mit `unhealthy_resources_error`, heißt das
  „unbekannt“, nicht „keine“ — so weitergeben.
- **Abbruch nach ~10 min `pending`:** letzter `observed`-Stand, eine Einschätzung (Pods starten
  noch? Sync hängt? nie abgeholt?) und das Angebot, später erneut zu warten oder mit `diagnose`
  nachzusehen.

## Gotchas

- Revision = GitOps-Commit. Nie den Code-Commit, nie den Image-Tag an `wait_for_revision` geben.
- Nie pushen, nie einen Sync auslösen. Nicht gepusht → sagen und warten, bis der Nutzer pusht.
- Nur auf `pending` erneut aufrufen; jedes `error` beendet sofort.
- `timeout_seconds` 25, Abbruch nach ~10 Minuten. Eine Zeile pro Runde, nicht mehr.
- Präfixvergleich, keine Abstammung: abweichende beobachtete Revision prüfen
  (`git merge-base --is-ancestor`), bevor du bis zum Timeout wartest.
- `OutOfSync` ohne laufende Operation über zwei Aufrufe → „vermutlich kein Auto-Sync“, melden.
- Ein Commit kann mehrere Apps treffen — je App warten, getrennt berichten.
- Environment nicht in `list_environments` → nicht angebunden; nicht raten.
- `failed` → `cor-metis-gitops:diagnose`, nicht selbst diagnostizieren.
