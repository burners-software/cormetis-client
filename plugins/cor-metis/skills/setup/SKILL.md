---
name: "setup"
description: "Richtet COR.metis bei einem neuen Kollegen ein und prüft, ob es funktioniert. Nutze diesen Skill, wenn jemand COR.metis neu aufsetzen will („COR.metis einrichten\", „ESKS verbinden\", „wie komme ich an die Knowledgebase\"), wenn er wissen will ob sein Zugang klappt („funktioniert mein COR.metis\", „bin ich verbunden\", „welche Freigabe habe ich\"), oder wenn etwas nicht geht („COR.metis findet nichts\", „ESKS antwortet nicht\", „Ingest scheitert\"). Führt durch die Connector-Einrichtung, macht einen echten Testaufruf, benennt die erkannte Freigabestufe, prüft auf Wunsch die Zusatz-Voraussetzungen für den Dokumenten-Ingest und den Deploy-Zugang/CGSS („Deploy-Zugang prüfen\", „CGSS verbinden\", „sehe ich die Deployments\"). Kennt beide Wege: Connector in Claude Desktop/Cowork und Plugin `cor-metis-code` in Claude Code."
---

# COR.metis Setup & Selbsttest

Zwei Aufgaben in einem Skill: **einrichten**, wenn noch nichts da ist — und **prüfen**, wenn es klemmt. Fang immer mit dem Prüfen an. Ein Diagnoselauf dauert zwei Tool-Calls und beantwortet meistens schon, was der Nutzer eigentlich wissen wollte.

## Die eine Regel, die nicht verhandelbar ist

**Lass dir den API-Key niemals in den Chat schreiben.**

Der Key ist die Identität des Nutzers — wer ihn hat, ist er. Er gehört in das Connector-Feld in den Einstellungen (Desktop/Cowork) bzw. in die maskierte Plugin-Abfrage von `cor-metis-code` (Claude Code) und sonst nirgendwohin. Fragt jemand „soll ich dir den Key schicken?", ist die Antwort nein, mit Begründung. Und wenn doch einer im Chat landet: sag es offen, bitte darum, ihn bei Chris rotieren zu lassen, und arbeite nicht mit ihm weiter.

Dasselbe gilt in die andere Richtung: du bekommst den Key auch nicht zu sehen. Ob er stimmt, weist du **indirekt** über einen echten Tool-Call nach — nie durch Hinschauen.

---

## Teil A — Diagnose

Arbeite die Checks der Reihe nach ab. Bricht einer, mach trotzdem weiter — **wo** es bricht, ist die eigentliche Information.

### Check 1 — Plugin geladen

Wenn du diesen Text liest: ja. Notiere, unter welchem Namen du aufgerufen wurdest; erwartet wird `cor-metis:setup`. Heißt es anders, stimmt das Namespacing im Paket nicht — das gehört gemeldet, ist für den Nutzer aber folgenlos.

### Check 2 — ESKS-Connector verbunden

Derselbe Server heißt je nach Oberfläche verschieden. Das Tool `get_stats` erscheint etwa als:

| Oberfläche | Tool-Name |
|---|---|
| Claude Desktop / Cowork, Connector `ESKS` | `mcp__ESKS__get_stats` |
| Claude Code, synchronisierter claude.ai-Connector | `mcp__claude_ai_ESKS__get_stats` |
| Claude Code, Plugin `cor-metis-code` | `mcp__plugin_cor-metis-code_ESKS__get_stats` |

Die Regel: ein Tool gehört zum ESKS-Server, wenn der Teil zwischen `mcp__` und `__<tool>` `ESKS` ist oder auf `_ESKS` endet — Groß-/Kleinschreibung egal. Alle drei Varianten oben sind richtig, ebenso ein Connector, der klein `esks` heißt (`mcp__esks__get_stats`). Sind die Tools als „deferred" gelistet, lade sie erst per ToolSearch-Stichwortsuche nach („ESKS get_stats“), nie per `select:` mit festem Namen — deferred heißt *vorhanden*, nicht *fehlend*. Das ist die häufigste Fehldiagnose an dieser Stelle.

Vier Fälle, die Verschiedenes bedeuten:

| Beobachtung | Bedeutung |
|---|---|
| ESKS-Tools da (unter einem Präfix) | Connector existiert und ist verbunden → Check 3 |
| ESKS-Tools unter **zwei** Präfixen (z.B. `mcp__claude_ai_ESKS__…` und `mcp__plugin_cor-metis-code_ESKS__…`) | Dublette, **kein FAIL** → Check 3, aber als Hinweis melden: in Claude Code den synchronisierten Server `claude.ai ESKS` unter `/mcp` deaktivieren. Sonst unterdrückt Claude Code eine Seite nach eigenem Ermessen, und der synchronisierte Weg trägt den Key in der URL. Fehlen die Plugin-Tools, obwohl `cor-metis-code` installiert ist, und nur `mcp__claude_ai_ESKS__…` ist da, ist das dieselbe Ursache, kein Defekt. |
| Keine ESKS-Tools, aber ein ähnlich benannter Server (`metis`, `cor-metis`, `esks-spike`) | Connector existiert, heißt aber **falsch** → siehe Teil B, Punkt „Name" |
| Gar nichts dergleichen | Connector fehlt → Teil B |

### Check 3 — Antwortet der Server, und greift der Key?

Ein billiger, lesender Call: `get_stats` vom ESKS-Server.

- **Strukturierte Antwort** → Connector *und* Key sind in Ordnung. Der Nutzer ist arbeitsfähig.
- **401 / 403 / „unauthorized"** → Server erreichbar, Key falsch, abgelaufen oder abgeschnitten. Häufigste Ursache: beim Kopieren ein Zeichen verloren, oder die URL enthält Leerzeichen/Zeilenumbruch. In Claude Code den Key neu setzen: `/plugin` → `cor-metis-code` → „Configure options" bzw. `claude plugin configure cor-metis-code`.
- **DNS-Fehler, Timeout, „connection refused"** → der Nutzer ist nicht im Firmennetz bzw. nicht im VPN, oder die URL ist falsch. Sagt über den Key **nichts** aus.

Fehlertexte **wörtlich** zitieren, nicht paraphrasieren. Der Unterschied zwischen 403 und Timeout ist die halbe Diagnose.

### Check 4 — Welche Freigabestufe hat der Zugang?

Nur laufen lassen, wenn Check 3 grün war. Ruf `get_burner_status` vom ESKS-Server mit der Testanlage **`cor-xc001`** auf.

- **Antwort kommt** → der Zugang hat interne Freigabe (C2). Alle drei Fach-Skills stehen offen.
- **Abgelehnt** → der Zugang ist auf Dokumentation beschränkt. Das ist **kein Fehler** und darf auch nicht so klingen: Anlagen-Livedaten sind intern, die Knowledgebase-Suche funktioniert trotzdem — nur eben gedeckelt auf das, was der Key sieht.

Den Anlagenzustand hier **nicht** interpretieren. `cor-xc001` ist ein Testgerät auf dem Cluster `dev01`; es zählt nur, dass überhaupt etwas zurückkommt.

### Check 5 — Ingest-Voraussetzungen (nur auf Nachfrage)

Diesen Check nur laufen lassen, wenn der Nutzer Dokumente **einspielen** will. Für reines Fragen ist er irrelevant und würde nur verunsichern.

1. **Läuft die Session in der Cloud?** Prüfen: kannst du Shell-Kommandos ausführen und Dateien schreiben?
2. **Ist `internal.cor.energy` aus der Sandbox erreichbar?**
   ```bash
   curl -sS -o /dev/null -w '%{http_code}\n' --max-time 15 https://internal.cor.energy/esks/health
   ```
   Erwartet: `200`.

Schlägt einer fehl → die drei Schalter aus Teil C. Und dazu der Satz, den man leicht vergisst: **Einstellungsänderungen greifen erst in einer neuen Session.**

### Check 6 — Deploy-Zugang (CGSS) (optional)

Nur laufen lassen, wenn CGSS-Tools da sind **oder** der Nutzer nach Deployments, Gitops oder CGSS fragt. Wer nur die Knowledgebase nutzt, braucht CGSS nicht — ein fehlendes CGSS ist für ihn `n/a`, kein `FAIL`.

CGSS-Tools erkennst du nach derselben Regel wie in Check 2: der Teil zwischen `mcp__` und `__<tool>` ist `CGSS` oder endet auf `_CGSS`, Groß-/Kleinschreibung egal (von Hand per `claude mcp add` angelegt, heißt er oft klein `cgss` — auch das zählt). Deferred → per ToolSearch-Stichwortsuche nachladen („CGSS list_environments“).

Ruf `list_environments` vom CGSS-Server auf:

- **Liste kommt** → PASS. Nenne die Environments, die tatsächlich zurückkommen — nur diese, keine aus der Erinnerung ergänzen (heute `central-1`, `dev01` und `dev02`).
- **CGSS-Tools fehlen, obwohl eingerichtet**, bzw. in Claude Code zeigt `/mcp` den Server CGSS als „failed" → entweder ist der Key falsch, **oder** dem Nutzer fehlt eine Engineer-Rolle (DEVELOPER, OPERATOR, ADMIN oder MASTER, oder `*`). CGSS lehnt ohne Rolle schon den Verbindungsaufbau mit 403 ab, deshalb tauchen die Tools gar nicht erst auf. Funktioniert ESKS mit demselben Key (Check 3 grün), ist es fast sicher die Rolle → bei Chris anfragen.
- **Noch nicht eingerichtet** → Teil B1 (zweiter Connector `CGSS`) bzw. B2 (`cor-metis-code` bringt CGSS mit).

### Ergebnis

Gib genau diese Tabelle aus — `PASS` / `FAIL` / `n/a`, je ein Satz Beobachtung:

| Check | Ergebnis | Beobachtung |
|---|---|---|
| 1 Plugin geladen | | |
| 2 ESKS-Connector verbunden | | |
| 3 Server antwortet, Key greift | | |
| 4 Freigabestufe | | |
| 5 Ingest-Voraussetzungen | | |
| 6 Deploy-Zugang (CGSS) | | |

Danach **ein** Satz Klartext: was der Nutzer jetzt kann, und was — falls etwas fehlt — der eine nächste Schritt ist. Keine Schrittliste, wenn alles grün ist. Wer arbeitsfähig ist, will keine Anleitung lesen.

---

## Teil B — Connector einrichten

Nur nötig, wenn Check 2 fehlgeschlagen ist (bzw. Check 6, wenn ein Entwickler den Deploy-Zugang braucht).

**Voraussetzung:** ein persönlicher COR.metis-API-Key. Den gibt es bei Chris. Hat der Nutzer keinen, endet der Weg hier — ohne Key gibt es nichts einzurichten, und ein geliehener Key eines Kollegen ist keine Lösung: die Freigabestufe hängt am Key, ein geteilter Key hebelt das ganze Modell aus.

Der Weg hängt von der Oberfläche ab: **B1** für Claude Desktop und Cowork, **B2** für Claude Code. Führe den Nutzer durch die Schritte. Ein Schritt pro Nachricht, wenn er unsicher wirkt; sonst am Stück.

### B1 — Claude Desktop / Cowork

1. **Einstellungen → Konnektoren → „Connector hinzufügen" → „Benutzerdefinierter Connector".**
2. **Name:** `ESKS` — Großbuchstaben empfohlen, sonst nichts dazu.
3. **URL:** die Basis-URL mit angehängtem Key:
   ```
   https://internal.cor.energy/esks/mcp?apikey=DEIN_KEY
   ```
   Ohne Leerzeichen, ohne Zeilenumbruch, `DEIN_KEY` wörtlich durch den eigenen Key ersetzt.
4. **Speichern**, dann **neue Session starten.**
5. Zurückkommen und sagen: *„prüf mal"* → Teil A.

**Optional, nur für Entwickler:** ein zweiter Connector für den Deploy-Zugang, gleiche Schritte:

- **Name:** `CGSS` (Großbuchstaben empfohlen)
- **URL:** `https://internal.cor.energy/cgss/mcp?apikey=DEIN_KEY` — derselbe persönliche Key wie für ESKS.

CGSS verlangt zusätzlich eine Engineer-Rolle (DEVELOPER, OPERATOR, ADMIN oder MASTER, oder `*`). Ohne sie lehnt der Server die Verbindung ab — siehe Check 6.

### B2 — Claude Code

In Claude Code gibt es keinen Connector zum Abtippen. Das Plugin `cor-metis-code` bringt ESKS und CGSS mit; der Key wird einmal abgefragt und im Credential-Store von Claude Code abgelegt, nicht in einer URL.

1. Marketplace hinzufügen:
   ```
   /plugin marketplace add burners-software/cormetis-client
   ```
2. Plugins installieren:
   ```
   /plugin install cor-metis@cor-metis-client
   /plugin install cor-metis-code@cor-metis-client
   ```
   Entwickler zusätzlich `/plugin install cor-metis-gitops@cor-metis-client` (die Deploy-Skills).
3. **Key eingeben**, wenn Claude Code beim Aktivieren von `cor-metis-code` danach fragt. Die Eingabe ist maskiert — der Key gehört dort hinein, **nicht** in den Chat.
4. Claude Code neu starten, mit `/mcp` prüfen, dass ESKS (und CGSS) verbunden sind.
5. Nutzt derselbe claude.ai-Account auch Desktop-Connectoren, tauchen sie in Claude Code zusätzlich als `claude.ai ESKS` / `claude.ai CGSS` auf. Die in `/mcp` deaktivieren — sonst wählt Claude Code selbst, welche Seite es nimmt (Check 2).
6. Zurückkommen und sagen: *„prüf mal"* → Teil A.

`/mcp` zeigt CGSS als „failed", ESKS aber verbunden → fehlende Engineer-Rolle, siehe Check 6. Für Entwickler ist das der Ausnahmefall.

### Der Name ist keine Geschmacksfrage

In Claude Desktop und Cowork muss der Connector `ESKS` bzw. `CGSS` heißen (Großschreibung empfohlen, klein geht auch), weil die Skills die Tools am Server-Namen erkennen — dem Teil zwischen `mcp__` und `__<tool>`. Heißt der Connector „COR.metis", „Metis" oder „esks-spike", steht dort etwas anderes und die Skills laufen ins Leere — bei einem Nutzer, der schwören würde, dass alles verbunden ist. Das ist die teuerste Fehlerquelle im ganzen Setup. (In Claude Code setzt das Plugin die Namen selbst.)

Hat jemand den Connector schon unter falschem Namen angelegt: umbenennen geht nicht immer sauber. Trennen und neu anlegen ist der verlässlichere Weg.

### Was das Plugin bewusst *nicht* mitbringt

Falls jemand fragt, warum der Connector in Claude Desktop nicht einfach im Plugin steckt: weil der API-Key die Identität des Nutzers ist. Ein Paket, das alle bekommen, kann nichts Personenbezogenes enthalten. Das Teil, das sich häufig ändert (die Skills), kommt automatisch; das Teil, das sich nie ändert (die URL), wird einmal von Hand gesetzt.

In Claude Code ist das anders gelöst: `cor-metis-code` enthält die Server, aber keinen Key — den fragt Claude Code beim Aktivieren maskiert ab und legt ihn im eigenen Credential-Store ab. Getrennt von `cor-metis` ist es, weil Cowork einen Server mit abgefragtem Key ohne Default stillschweigend ignoriert; die Skill-Plugins bleiben so frei von Konfiguration und laufen überall gleich.

Bekommt ESKS später OAuth, wandert der Connector zurück ins Paket und das Onboarding schrumpft auf zwei Schritte.

---

## Teil C — Zusatzeinstellungen für den Ingest

Nur für Kollegen, die Dokumente **einspielen**. Wer nur fragt, braucht davon nichts.

Drei Schalter in Claude Desktop:

1. **Computernutzung AUS** — *Einstellungen → Fähigkeiten → Computerverwendung.*
   Solange sie an ist, läuft die Aufgabe nicht in der Cloud-VM. Die beiden Modi schließen sich aus.
2. **Netzwerk frei** — *Einstellungen → Fähigkeiten:*
   - „Cloud-Code-Ausführung und Dateierstellung" → **AN**
   - „Ausgehenden Netzwerkverkehr erlauben" → **AN**
   - „Domain-Zulassungsliste" → **Alle Domains** (oder eine Allowlist mit `internal.cor.energy`)
3. **In der Cloud arbeiten, nicht lokal** — die Aufgabe muss in der Sandbox laufen, nicht auf dem eigenen Rechner.

Danach: **neue Session.** Die Einstellungen greifen nicht rückwirkend.

Warum das nötig ist, falls jemand fragt: Bilder werden binär hochgeladen statt als base64 durch den Chat. Deshalb braucht die Sandbox Netzzugang — und deshalb läuft ein Datenblatt mit dreißig Diagrammen nicht aus dem Ruder.

---

## Gotchas

- **Key nie im Chat.** Weder rein noch raus. Landet doch einer drin: rotieren lassen.
- **„Deferred" heißt vorhanden.** Erst nachladen, dann urteilen — sonst diagnostizierst du einen fehlenden Connector, der da ist.
- **Connector-Name `ESKS` bzw. `CGSS`** (klein geht auch, alles andere nicht). Häufigste stille Fehlerquelle.
- **Präfix egal, Server-Name zählt** (`ESKS` oder `…_ESKS`). `mcp__ESKS__…`, `mcp__claude_ai_ESKS__…` und `mcp__plugin_cor-metis-code_ESKS__…` sind alle richtig. Nachladen per Stichwortsuche, nie per `select:` mit festem Namen.
- **Zwei Präfixe = Dublette, kein Fehler.** Hinweis geben (`claude.ai ESKS` in `/mcp` deaktivieren), nicht als FAIL werten.
- **CGSS „failed", ESKS grün** → fast immer die fehlende Engineer-Rolle, nicht der Key.
- **403 ≠ Timeout.** Das eine ist der Key, das andere das Netz. Nie zusammenwerfen.
- **Abgelehnte Anlagendaten sind kein Defekt**, sondern eine Freigabestufe. Formuliere es als Berechtigung, nicht als Störung.
- **Einstellungen greifen erst in neuer Session.** Wer das nicht dazusagt, produziert einen zweiten Support-Fall.
- Läuft alles: **keine Anleitung ausgeben.** Ein Satz, fertig.
