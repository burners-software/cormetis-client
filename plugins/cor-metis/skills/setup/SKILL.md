---
name: "setup"
description: "Richtet COR.metis bei einem neuen Kollegen ein und prüft, ob es funktioniert. Nutze diesen Skill, wenn jemand COR.metis neu aufsetzen will („COR.metis einrichten\", „ESKS verbinden\", „wie komme ich an die Knowledgebase\"), wenn er wissen will ob sein Zugang klappt („funktioniert mein COR.metis\", „bin ich verbunden\", „welche Freigabe habe ich\"), oder wenn etwas nicht geht („COR.metis findet nichts\", „ESKS antwortet nicht\", „Ingest scheitert\"). Führt durch die Connector-Einrichtung, macht einen echten Testaufruf, benennt die erkannte Freigabestufe und prüft auf Wunsch die Zusatz-Voraussetzungen für den Dokumenten-Ingest."
---

# COR.metis Setup & Selbsttest

Zwei Aufgaben in einem Skill: **einrichten**, wenn noch nichts da ist — und **prüfen**, wenn es klemmt. Fang immer mit dem Prüfen an. Ein Diagnoselauf dauert zwei Tool-Calls und beantwortet meistens schon, was der Nutzer eigentlich wissen wollte.

## Die eine Regel, die nicht verhandelbar ist

**Lass dir den API-Key niemals in den Chat schreiben.**

Der Key ist die Identität des Nutzers — wer ihn hat, ist er. Er gehört in das Connector-Feld in den Einstellungen und sonst nirgendwohin. Fragt jemand „soll ich dir den Key schicken?", ist die Antwort nein, mit Begründung. Und wenn doch einer im Chat landet: sag es offen, bitte darum, ihn bei Chris rotieren zu lassen, und arbeite nicht mit ihm weiter.

Dasselbe gilt in die andere Richtung: du bekommst den Key auch nicht zu sehen. Ob er stimmt, weist du **indirekt** über einen echten Tool-Call nach — nie durch Hinschauen.

---

## Teil A — Diagnose

Arbeite die Checks der Reihe nach ab. Bricht einer, mach trotzdem weiter — **wo** es bricht, ist die eigentliche Information.

### Check 1 — Plugin geladen

Wenn du diesen Text liest: ja. Notiere, unter welchem Namen du aufgerufen wurdest; erwartet wird `cor-metis:setup`. Heißt es anders, stimmt das Namespacing im Paket nicht — das gehört gemeldet, ist für den Nutzer aber folgenlos.

### Check 2 — ESKS-Connector verbunden

Schau nach Tools mit dem Präfix `mcp__ESKS__` (z.B. `mcp__ESKS__search`, `mcp__ESKS__get_stats`). Sind sie als „deferred" gelistet, lade sie erst per ToolSearch nach — deferred heißt *vorhanden*, nicht *fehlend*. Das ist die häufigste Fehldiagnose an dieser Stelle.

Drei Fälle, die Verschiedenes bedeuten:

| Beobachtung | Bedeutung |
|---|---|
| Tools da | Connector existiert und ist verbunden → Check 3 |
| Kein `mcp__ESKS__*`, aber ein ähnlich benannter Server (`metis`, `cor-metis`, `esks-spike`) | Connector existiert, heißt aber **falsch** → siehe Teil B, Punkt „Name" |
| Gar nichts dergleichen | Connector fehlt → Teil B |

### Check 3 — Antwortet der Server, und greift der Key?

Ein billiger, lesender Call: `mcp__ESKS__get_stats`.

- **Strukturierte Antwort** → Connector *und* Key sind in Ordnung. Der Nutzer ist arbeitsfähig.
- **401 / 403 / „unauthorized"** → Server erreichbar, Key falsch, abgelaufen oder abgeschnitten. Häufigste Ursache: beim Kopieren ein Zeichen verloren, oder die URL enthält Leerzeichen/Zeilenumbruch.
- **DNS-Fehler, Timeout, „connection refused"** → der Nutzer ist nicht im Firmennetz bzw. nicht im VPN, oder die URL ist falsch. Sagt über den Key **nichts** aus.

Fehlertexte **wörtlich** zitieren, nicht paraphrasieren. Der Unterschied zwischen 403 und Timeout ist die halbe Diagnose.

### Check 4 — Welche Freigabestufe hat der Zugang?

Nur laufen lassen, wenn Check 3 grün war. Ruf `mcp__ESKS__get_burner_status` mit der Testanlage **`cor-xc001`** auf.

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

### Ergebnis

Gib genau diese Tabelle aus — `PASS` / `FAIL` / `n/a`, je ein Satz Beobachtung:

| Check | Ergebnis | Beobachtung |
|---|---|---|
| 1 Plugin geladen | | |
| 2 ESKS-Connector verbunden | | |
| 3 Server antwortet, Key greift | | |
| 4 Freigabestufe | | |
| 5 Ingest-Voraussetzungen | | |

Danach **ein** Satz Klartext: was der Nutzer jetzt kann, und was — falls etwas fehlt — der eine nächste Schritt ist. Keine Schrittliste, wenn alles grün ist. Wer arbeitsfähig ist, will keine Anleitung lesen.

---

## Teil B — Connector einrichten

Nur nötig, wenn Check 2 fehlgeschlagen ist.

**Voraussetzung:** ein persönlicher ESKS-API-Key. Den gibt es bei Chris. Hat der Nutzer keinen, endet der Weg hier — ohne Key gibt es nichts einzurichten, und ein geliehener Key eines Kollegen ist keine Lösung: die Freigabestufe hängt am Key, ein geteilter Key hebelt das ganze Modell aus.

Führe den Nutzer durch diese Schritte. Ein Schritt pro Nachricht, wenn er unsicher wirkt; sonst am Stück.

1. **Einstellungen → Konnektoren → „Connector hinzufügen" → „Benutzerdefinierter Connector".**
2. **Name:** exakt `ESKS` — Großbuchstaben, sonst nichts.
3. **URL:** die Basis-URL mit angehängtem Key:
   ```
   https://internal.cor.energy/esks/mcp?apikey=DEIN_KEY
   ```
   Ohne Leerzeichen, ohne Zeilenumbruch, `DEIN_KEY` wörtlich durch den eigenen Key ersetzt.
4. **Speichern**, dann **neue Session starten.**
5. Zurückkommen und sagen: *„prüf mal"* → Teil A.

### Der Name ist keine Geschmacksfrage

Die Skills sprechen die Tools als `mcp__ESKS__*` an. Heißt der Connector „COR.metis", „Metis" oder „esks", heißen die Tools anders und die Skills laufen ins Leere — bei einem Nutzer, der schwören würde, dass alles verbunden ist. Das ist die teuerste Fehlerquelle im ganzen Setup.

Hat jemand den Connector schon unter falschem Namen angelegt: umbenennen geht nicht immer sauber. Trennen und neu anlegen ist der verlässlichere Weg.

### Was das Plugin bewusst *nicht* mitbringt

Falls jemand fragt, warum der Connector nicht einfach im Plugin steckt: weil der API-Key die Identität des Nutzers ist. Ein Paket, das alle bekommen, kann nichts Personenbezogenes enthalten. Das Teil, das sich häufig ändert (die Skills), kommt automatisch; das Teil, das sich nie ändert (die URL), wird einmal von Hand gesetzt.

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
- **Connector-Name exakt `ESKS`.** Häufigste stille Fehlerquelle.
- **403 ≠ Timeout.** Das eine ist der Key, das andere das Netz. Nie zusammenwerfen.
- **Abgelehnte Anlagendaten sind kein Defekt**, sondern eine Freigabestufe. Formuliere es als Berechtigung, nicht als Störung.
- **Einstellungen greifen erst in neuer Session.** Wer das nicht dazusagt, produziert einen zweiten Support-Fall.
- Läuft alles: **keine Anleitung ausgeben.** Ein Satz, fertig.
