---
name: "burner-status"
description: "Live-Status einer konkreten CORgen-Anlage. \"Burner\", \"Brenner\", \"COR.gen\", \"cor.gen\" und \"corgen\" meinen dasselbe Gerät. Nutze diesen Skill, wenn jemand wissen will, wie es einer Anlage geht, was sie gerade fährt, ob Störungen anliegen oder warum sie steht — egal ob er eine Seriennummer nennt, sie schludrig schreibt, nur den internen Namen kennt (\"chriseins\") oder die Anlage bloß umschreibt (\"der Kessel in der Technikhalle\"). Erklärt, wie man von einer unscharfen Angabe über `find_burner` zur Seriennummer kommt und wie die Antwort von `get_burner_status` zu lesen ist — vor allem, was sie NICHT hergibt. Abgrenzung: Fragen zur Dokumentation, zu Datenblättern, Modbus-Registern oder zur Bedeutung eines Fehlercodes im Allgemeinen gehören in den Skill orakel — dieser Skill liest nur den Momentzustand einer einzelnen Maschine. Greift auf den ESKS-MCP-Server zu und ist auf interne Clearance (C2) beschränkt."
---

# COR.metis Burner-Status — Live-Blick auf eine Anlage

Zwei Werkzeuge, eine Aufgabe:

- **`find_burner(query)`** — übersetzt eine unscharfe Angabe in Seriennummern. Sucht über alle
  verbundenen Cluster. Liefert Treffer, keine Zustände.
- **`get_burner_status(serial_number)`** — fragt alle verbundenen Cluster nach **einer** Anlage und
  liefert einen Schnappschuss: Betriebszustand, Live-Messwerte, aktive Error- und Info-Codes,
  System-Ampeln.

Beide sind **read-only**. Du kannst von hier aus nichts an einer Anlage verändern. Und es gibt
**keine Historie** — du siehst diesen Moment, sonst nichts.

> Technisch heißen die Calls in der aktuellen Instanz `mcp__ESKS__find_burner` und
> `mcp__ESKS__get_burner_status`. Beide haben genau einen Parameter (`query` bzw. `serial_number`)
> und keinen Clearance-Parameter, weil es keinen braucht: die Prüfung hängt am API-Key des
> Fragenden und passiert serverseitig.

## Ein Gerät, viele Namen

**`burner`, `Brenner`, `COR.gen`, `cor.gen` und `corgen` bezeichnen dasselbe Produkt.** Die
Schreibweise sagt nichts über die Sache aus — sie hängt daran, wer gerade spricht: im Marketing
`COR.gen`, im Code und in den Tool-Namen `burner`, im Betrieb oft nur „der Brenner" oder „die
Anlage". Behandle alle Varianten gleich und frag nicht nach, welche gemeint ist.

Praktisch heißt das: „Wie geht's dem Burner in Halle 3?", „Status der corgen 4711?" und „Läuft die
COR.gen noch?" führen alle hierher. Und wenn du selbst eine Anlage benennst, nimm die Schreibweise,
die dein Gegenüber benutzt hat, statt sie zu korrigieren.

## Preflight — ist ESKS überhaupt verbunden?

Sind die `mcp__ESKS__*`-Tools nicht verfügbar (auch nicht als „deferred" nachladbar), fehlt dem Nutzer der Connector. Das ist etwas völlig anderes als eine abgelehnte Berechtigung — und muss auch anders klingen:

> „Der ESKS-Connector ist bei dir nicht eingerichtet — ohne den sehe ich keine Anlage. Sag ‚COR.metis einrichten', dann führe ich dich durch."

Der Skill `setup` aus diesem Plugin übernimmt das. Erst wenn der Connector da ist und der Server *antwortet*, greift die Berechtigungslogik aus Schritt 0.

## Leitprinzipien

- **Der Server schützt, du kennzeichnest.** Anlagendaten sind intern (C2). Ob jemand sie sehen darf,
  entscheidet sein Key — nicht du. Frag ihn nicht nach seiner Rolle; ruf auf und arbeite mit dem,
  was zurückkommt.
- **Bei mehreren Treffern nicht raten.** Lieber eine Rückfrage zu viel als der Status einer fremden
  Anlage. Der erste Treffer ist kein Vorschlag des Servers, sondern nur der erste.
- **Erst der Zustand der Daten, dann die Daten.** Veraltet, abgeschaltet, Cluster stumm: das gehört
  vor die Interpretation, nicht in eine Fußnote.
- **Beantworte die Frage, die gestellt wurde.** Nicht die, für die du die meisten Felder hast.
- **Ehrlich über Grenzen.** Eine Verbrennungsstörung diagnostiziert man nicht aus einem Schnappschuss.

## Schritt 0 — Von dem, was gesagt wurde, zur Seriennummer

Es gibt hier **keine** Clearance-Vorabfrage. Anlagendaten erfordern interne Freigabe (C2), aber das
prüft der Server am Key des Fragenden. Wer nicht darf, bekommt eine Ablehnung — und wer fragen darf,
soll nicht erst ein Formular ausfüllen.

**Wenn ein Tool ablehnt:** sag es klar und hör auf. Nachfassen hilft nicht, ein zweiter Versuch
ändert nichts. Und formuliere es als das, was es ist — eine Frage der Berechtigung, nicht ein
Problem mit der Anlage. *„Für Anlagen-Livedaten brauchst du interne Freigabe; dein Zugang hat sie
nicht"* ist richtig, *„die Anlage konnte nicht abgefragt werden"* ist irreführend.

### Der Ablauf

**1. Sieht die Eingabe wie eine Seriennummer aus?** Also: nur Buchstaben, Ziffern und Bindestriche,
4–32 Zeichen. Dann **direkt `get_burner_status`** aufrufen.

**Normalisiere nicht selbst.** Groß-/Kleinschreibung und führende oder nachfolgende Leerzeichen sind
dem Server egal — `"  COR-XC001  "` findet dieselbe Anlage wie `"cor-xc001"`. Gib weiter, was der
Benutzer geschrieben hat. Wenn du selbst am String herumschraubst, machst du aus einer Seriennummer
im Zweifel eine andere.

**2. Kein Treffer oder gar keine Seriennummer?** `find_burner(query)` aufrufen, wenn

- `get_burner_status` mit `found: false` zurückkam, **oder**
- die Eingabe von vornherein nicht wie eine Seriennummer aussah: Leerzeichen mittendrin, Umlaute,
  Klartext wie *„das Gerät in der Technikhalle"*, *„Kessel Halle 3"*.

Als `query` gehört hinein, **was der Benutzer gesagt hat** — nicht deine Interpretation davon.

**3. Antwort von `find_burner` behandeln:**

| Antwort | Was zu tun ist |
|---|---|
| `match_count: 0` | Kein passendes Gerät. **Erst `warnings` prüfen** (siehe unten), dann so sagen. |
| `match_count: 1` | `matches[0].serial_number` nehmen und damit `get_burner_status` aufrufen. Ohne Rückfrage. |
| `action_required: "choose_one"` | **Nicht raten, nicht den ersten nehmen.** Liste zeigen, Benutzer wählen lassen, erst dann `get_burner_status` mit der gewählten Nummer. |
| `error: "too_many_matches"` | Sagen, dass die Angabe zu unscharf war, und um eine genauere bitten. **Kein zweiter Versuch mit derselben Query.** |

Bei der Auswahlliste: pro Eintrag die **Seriennummer** zeigen, dazu den **`text`**, falls er gefüllt
ist, und den **`cluster`**, wenn er bei der Unterscheidung hilft (ein `dev01` neben einem
Produktivcluster ist genau die Information, die die Wahl leicht macht).

**`text` trägt den internen Gerätenamen** — den Namen, unter dem die Techniker die Anlage in der
Halle und auf den Prüfständen führen, weil sich niemand Seriennummern merken will. Genau dafür ist
`find_burner` da: der Name ist das, was der Mensch sagt, die Seriennummer das, was das System
braucht.

Zwei Dinge dazu:

- **Der Name ist keine Kennung.** Er wird von Hand vergeben und kann sich ändern — ein Gerät, das
  gestern „chriseins" hieß, kann heute anders heißen, und derselbe Name kann später an einer
  anderen Maschine hängen. Verlass dich nie auf ihn, um eine Anlage über die Zeit
  wiederzuerkennen; dafür gibt es die Seriennummer. Merk dir auch keinen Namen aus einem früheren
  Gespräch — frag lieber neu ab.
- **Beides zeigen, wenn du eine Anlage benennst**: Name *und* Seriennummer, also
  *„cor-xc001 (`chriseins`)"*. Der Name macht die Antwort lesbar, die Nummer macht sie
  überprüfbar.

Ist `text` doch mal `null`, gilt weiter: nichts erfinden, nicht als „unbenannt" verkaufen — dann
steht in der Liste eben nur die Seriennummer.

**4. `warnings` gilt für ALLE diese Antworten.** Das Feld kann in jeder Antwort von `find_burner`
*und* von `get_burner_status` auftauchen und bedeutet immer dasselbe: **ein Cluster war nicht
erreichbar.** Also wurde dieser Cluster nicht durchsucht. Wenn es gefüllt ist, **muss** es in deine
Antwort:

- bei `match_count: 0` → „nicht gefunden" ist nicht die ganze Wahrheit. Die Anlage kann sehr wohl im
  stummen Cluster stehen. Sag, welcher.
- bei `found: false` aus `get_burner_status` → dasselbe. Erst wenn **alle** Cluster geantwortet haben
  und keiner die Anlage kennt, darfst du sie als unbekannt bezeichnen.
- bei einer Auswahlliste → die Liste kann unvollständig sein. Das gesuchte Gerät liegt
  möglicherweise im nicht erreichbaren Cluster. Sag das **bevor** der Benutzer wählt, nicht danach.

`warnings` fehlt bei einem Treffer ganz und ist `null`, wenn es nichts zu melden gibt — beides
heißt: keine Warnungen.

### Grenzen der Unschärfe (Stand heute)

`find_burner` ist heute deutlich weniger unscharf, als der Name verspricht: Groß-/Kleinschreibung und
umschließende Leerzeichen werden weggebügelt, andere Schreibvarianten aber nicht — `"COR XC 001"`
und `"COR-XC-001"` finden `cor-xc001` **nicht**. Auch Teilstrings ziehen nicht (`"cor-xc"` → 0
Treffer), und Clusternamen sind keine Suchbegriffe.

**Bei internen Namen ist die Suche großzügiger.** Eine Teilangabe reicht dort: `"chris"` findet
`chriseins` und `chriszwei`. Das ist beobachtet, nicht zugesichert — verlass dich nicht darauf,
aber probier bei einem Namen ruhig die Kurzform, bevor du nachfragst. Rechne dabei mit mehreren
Treffern (siehe `choose_one` oben): gerade Namensfragmente treffen leicht eine ganze Familie.

Praktische Folge: **`match_count: 0` ist kein Beweis, dass es die Anlage nicht gibt.** Wenn die
Eingabe nach einer verunstalteten Seriennummer aussah, sag ehrlich, dass du nichts gefunden hast,
und bitte um die Nummer, wie sie am Typenschild steht. Das ist besser, als sich Varianten
auszudenken und den Server damit zu bombardieren.

### Mehrere Anlagen in einer Frage

Pro Anlage ein eigener Weg durch diesen Ablauf, und die Ergebnisse getrennt halten. Nicht zwei
Anlagen in einen Absatz mischen.

### Geht die Antwort nach draußen?

Falls jemand erkennbar für einen externen Empfänger schreibt — Kunde, Partner, Servicebericht, der
das Haus verlässt — dann sind diese Daten intern und gehören nicht ungefiltert hinein. Weise darauf
hin, statt es stillschweigend zu übernehmen. Für die Dokumentationsseite gilt die Regelung im
Skill **`cor-metis:orakel`**.

## Schritt 1 — Neun Dinge, die man an `get_burner_status` leicht falsch liest

**„Nicht gefunden" heißt nicht immer „gibt es nicht."** Schau **zuerst** in `warnings`, dann versuch
es über `find_burner` (siehe Schritt 0). Eine Anlage für nicht existent zu erklären, ist der teuerste
Fehler, den dieser Skill machen kann.

**`data_stale: true` heißt, die Messwerte sind alt.** Sag das, **bevor** du irgendetwas
interpretierst. Eine Kesseltemperatur von vor vier Stunden ist nicht, was die Maschine jetzt tut —
sie ist, was die Maschine vor vier Stunden getan hat. `last_seen_age_minutes` sagt dir, wie alt,
`stale_threshold_minutes`, was in diesem Deployment als alt gilt.

**`data_stale: false` heißt aber NICHT, dass jeder Wert frisch ist.** Der Flag bezieht sich auf
`last_seen_utc` — den letzten Kontakt zur Anlage insgesamt. Jeder einzelne Eintrag in `physics` hat
sein **eigenes** `timestamp_utc`, und die laufen weit auseinander: in echten Antworten stehen neben
sekundenaktuellen Werten problemlos Zähler, die seit Monaten nicht mehr aktualisiert wurden. Bevor
du einen einzelnen Wert berichtest, schau auf **dessen** Zeitstempel. Ist er deutlich älter als
`last_seen_utc`, sag es dazu.

**Sentinelwerte sind keine Messwerte.** `-273.2 °C` ist der absolute Nullpunkt — das ist die
Kodierung für „kein Messwert / Fühler liefert nichts", nicht „die Anlage ist eiskalt". Solche Werte
gehören **nicht** in die Antwort; nenne sie höchstens als „kein Wert von diesem Fühler". Sei auch
bei verdächtig glatten Extremen misstrauisch (z.B. Vor- und Rücklauf beide exakt `300 °C`) — melde
den Wert dann als auffällig, statt ihn als Betriebszustand zu verkaufen. Und schließe daraus
**nicht** auf einen defekten Sensor, das gibt die Antwort nicht her.

**`state` hat zwei Seiten.** `current` ist der Ist-, `desired` der Sollzustand (dazu `engine` für
den Motorzustand). Laufen die beiden auseinander — z.B. `current: EMERGENCY-STOPPING` bei
`desired: IDLE` — ist die Anlage entweder gerade unterwegs dorthin oder sie kommt nicht an. Nenne
beide, nicht nur `current`.

**`enabled: false` erklärt fast alles andere.** Eine in der Registry abgeschaltete Anlage hat alte
Messwerte, keinen Zustand und keine Fehler. Führe mit dieser Tatsache, statt die Folgen als
Symptome zu berichten.

**Errors und Infos sind zwei verschiedene Dinge.** `errors` sind Störungen. `infos` sind
Betriebsmeldungen — „Entaschung läuft" ist die Maschine bei der Arbeit, nicht die Maschine beim
Klagen. Die beiden Listen **nie** zusammenwerfen, und einen Info-Code nie als Problem beschreiben.

**Ein unmappter Code ist kein unbekannter Code.** Hat ein Code kein Label, kommt er zerlegt in
`component`, `sub_component` und `error`. Berichte die. `component` und `sub_component` sind
sprechende Namen (`DeviceHardware`, `HeatingControlSystemSensors`) — zitiere sie wörtlich, statt sie
einzudeutschen. „Fehler 2354 in `DeviceHardware / HeatingControlSystemSensors`" ist brauchbar;
„unbekannter Fehler 1073809714" nicht — und du hattest genug in der Hand, um es besser zu machen.

**Die Messwertliste (`physics`) ist gefiltert, nicht vollständig.** Eine Anlage meldet weit mehr
Werte, als hier auftauchen; das Deployment entscheidet, welche gezeigt werden. Sag **nie** „die
Anlage misst nur diese Werte" und schließe **nie** auf einen defekten Sensor, weil ein Wert fehlt —
beides kannst du aus dieser Antwort nicht unterscheiden.

## Schritt 2 — Ampeln

`traffic_lights` deckt drei Subsysteme ab: `hardware`, `software`, `raspi` (das Controller-Board).

- `"Green"` — nichts zu melden, es kommt kein Detail mit. Lies nichts in das Schweigen hinein.
- `null` — **keine Daten**, keine Störung. Entweder hat das Subsystem nie gemeldet oder sein Zustand
  ist unbekannt. Sag „keine Daten", statt dir eine Farbe auszudenken.
- `"Yellow"` / `"Red"` — schau in **`traffic_lights.issues`** (die Liste liegt *innerhalb* von
  `traffic_lights`, nicht daneben). Dort steht jeder nicht-grüne Knoten mit einem Pfad wie
  `software > docker > container[bdgs]`. Diese Pfade sind präzise: **zitiere sie wörtlich**, statt
  sie zu umschreiben. Genau danach grept ein Techniker.

Sind alle drei Ampeln grün bzw. `null` und `issues` leer, heißt das **nicht**, dass die Anlage in
Ordnung ist — Störungen stehen in `errors`, nicht in den Ampeln. Die beiden Kanäle sind unabhängig.

## Schritt 3 — Fehlercode gefunden? Ans Orakel übergeben

Der häufigste nächste Schritt ist: *Was bedeutet dieser Code überhaupt?* Das steht in der
Knowledgebase, nicht in dieser Antwort.

- Aktiver Error-Code, dessen Bedeutung unklar ist → biete an, ihn im Skill **`cor-metis:orakel`**
  nachzuschlagen (`search` mit Produkt-Scope und dem Code bzw. der `component`/`error`-Zerlegung).
- Dabei gilt die Clearance des **Fragenden** weiter, nicht die dieses Tools. Ein Code aus einer
  internen Anlage rechtfertigt keine internen KB-Inhalte für jemand anderen.
- Trenne in der Antwort sauber: **was die Anlage meldet** (aus diesem Tool) vs. **was das laut
  Doku bedeutet** (aus der KB, mit Deep-Link).

## Antwortstil

Beantworte die Frage, die gestellt wurde. „Läuft sie?" will einen Satz, keine Tabelle mit sechzig
Messwerten. Biete das Detail an, kipp es nicht aus.

**Den Weg zur Anlage kurz halten.** Wenn du über `find_burner` gegangen bist und es genau einen
Treffer gab, reicht ein Halbsatz („für *Kessel Halle 3* → `cor-xc001`"). Kein Protokoll deiner
Tool-Calls. Interessant wird der Weg erst, wenn er mehrdeutig war oder ein Cluster gefehlt hat.

**Einheiten mitliefern.** Das Tool gibt sie meist her (`unit`) — aber nicht immer: Zähler wie
`FUEL_USED`, `PELLETS_USED` oder `ENGINE_RUNTIME_SINCE_START` kommen ohne. Wo eine Einheit fehlt,
nenne den Wert roh und **erfinde keine**. Wo eine da ist, gehört sie dran; eine Temperatur ohne
Einheit ist eine Zahl, mit der niemand etwas anfangen kann.

**Den Cluster nennen, wenn er zur Sache tut.** `cluster` sagt dir, wo die Anlage hängt — und kommt
auch schon in den Treffern von `find_burner` mit. Bei einem Namen wie `dev01` ist der Hinweis, dass
es sich um ein Entwicklungs-/Testsystem handelt, für die Einordnung oft wichtiger als jeder Messwert.

**Nicht spekulieren, wem die Anlage gehört oder wo sie steht.** Diese Information ist bewusst nicht
in der Antwort, und sie sich aus einer Seriennummer zusammenzureimen ist schlimmer als zu sagen,
dass du sie nicht hast.

Wenn etwas nach Störung aussieht: sag, **was du siehst** und **was es bedeuten könnte** — dann hör
auf. Eine Verbrennungsstörung aus einem Schnappschuss zu diagnostizieren gibt diese Datenlage nicht
her, und eine selbstbewusst falsche Vermutung über eine Industrieanlage ist schlimmer als ein
ehrliches „das ist, was ich sehe".

## Gotchas

- **Nicht** nach der Rolle des Fragenden fragen. Das prüft der Server am Key. Lehnt er ab, sag es
  als Berechtigungsfrage — nicht als Anlagenproblem.
- Seriennummer nicht selbst normalisieren. Case und umschließende Leerzeichen macht der Server.
- `found: false` ist kein Endzustand — erst `find_burner` versuchen, dann aufgeben.
- Bei `action_required: "choose_one"` **nie** den ersten Treffer nehmen. Fragen kostet eine Zeile,
  die falsche Anlage kostet mehr.
- Bei `too_many_matches` nicht dieselbe Query nochmal schicken. Um eine genauere Angabe bitten.
- `warnings` gilt für **beide** Tools und für **jede** Antwortform — auch für Auswahllisten und für
  `match_count: 0`. Ungelesen erklärst du eine existierende Anlage für nicht existent.
- `burner`, `Brenner`, `COR.gen`, `cor.gen`, `corgen` sind dasselbe. Keine Rückfrage, welche
  Variante gemeint ist.
- `text` ist der interne Name: von Hand vergeben, änderbar. Zum Anzeigen ja, zum Identifizieren
  nein — dafür die Seriennummer. Ist er `null`: nicht füllen, nicht erraten, nicht als
  „unbenannt" verkaufen.
- `data_stale` und `enabled: false` gehören **vor** die Interpretation, nicht dahinter.
- `data_stale: false` gilt für die Anlage, nicht für jeden Einzelwert — `timestamp_utc` pro Wert prüfen.
- `-273.2 °C` ist kein Messwert. Nie als Temperatur weitergeben.
- `infos` sind keine Fehler.
- `null` bei einer Ampel ist keine Farbe. Grüne Ampeln sind kein „alles in Ordnung" — dafür `errors` lesen.
- Keine Historie, kein Schreibzugriff. Wer einen Verlauf braucht, ist hier falsch — sag das.

## Feldreferenz

### `find_burner(query)`

```
{status, match_count, matches[{serial_number, text, cluster}], warnings}
```

- **verifiziert:** `match_count: 0` (dann fehlt `matches` komplett), `match_count: 1` sowie
  `match_count: n` zusammen mit `action_required: "choose_one"` und einem erklärenden `message`.
  `text` trägt den internen Gerätenamen (in früheren Ständen immer `null`), `cluster` ist immer
  gefüllt.
- **noch nicht gegen echte Daten gesehen:** `error: "too_many_matches"`. Behandle es wie oben
  beschrieben, aber sei nicht überrascht, wenn daneben weitere Felder stehen — und verlass dich
  nicht darauf, dass `match_count` dabei gesetzt ist.

### `get_burner_status(serial_number)`

Treffer:

```
{status, burner{
  serial_number, cluster, enabled, last_seen_utc, last_seen_age_minutes,
  data_stale, stale_threshold_minutes,
  state{current, desired, engine, filling_day_container},
  physics[{key, label, value, unit?, instance?, timestamp_utc}],
  errors[{code, component, sub_component, error}],
  infos[],
  traffic_lights{hardware, software, raspi, issues[]}
}}
```

Kein Treffer:

```
{status, found: false, serial_number, message, warnings}
```

> Ob ein **gemappter** Code zusätzlich ein `label`-Feld mitbringt, ist noch nicht gegen echte Daten
> geprüft — bislang wurden nur unmappte Codes beobachtet. Verhalte dich defensiv: ist ein Label da,
> nimm es; ist keins da, zerlege wie oben beschrieben.

## Mini-Beispiele

### A — Seriennummer genannt

> **Frage:** „Was macht die cor-xc001 gerade?"

1. Sieht aus wie eine Seriennummer → direkt `get_burner_status(serial_number="cor-xc001")`,
   unverändert, keine Rückfrage.
2. Antwort: `cluster: dev01`, `enabled: true`, `data_stale: false` (letzter Kontakt vor 1 min),
   `state.current: IDLE` = `desired: IDLE`, zwei unmappte `errors` in `DeviceHardware`,
   `infos: []`, Hardware- und Raspi-Ampel `null`, Software grün.
3. Antwort: *„Die Anlage läuft nicht, sie steht im Leerlauf — Ist- und Sollzustand sind beide IDLE,
   also so gewollt. Es liegen aber zwei Störungen an, beide in `DeviceHardware`: Fehler 12345 in
   `HeatingControlSystemHardware` und Fehler 2354 in `HeatingControlSystemSensors`. Die Codes sind
   hier nicht gemappt. Keine Betriebsmeldungen. Software-Ampel grün, Hardware und Raspi melden keine
   Daten. Läuft auf dem Cluster dev01, also einem Testsystem. Soll ich die beiden Codes in der KB
   nachschlagen?"*

### B — Nur eine Umschreibung

> **Frage:** „Wie geht's dem Kessel in der Technikhalle?"

1. Sieht **nicht** wie eine Seriennummer aus (Leerzeichen, Klartext) → gar nicht erst
   `get_burner_status` versuchen, direkt `find_burner(query="der Kessel in der Technikhalle")`.
2. `match_count: 3`, `action_required: "choose_one"`, `warnings` meldet Cluster `prod02` als
   nicht erreichbar.
3. Antwort: *„Auf die Angabe passen drei Anlagen — welche meinst du? `cor-xc001` (`chriseins`,
   dev01), `cor-xc002` (`chriszwei`, dev01), `cor-ab042` (`halle3-nord`, prod01). Ein Hinweis
   dazu: der Cluster prod02 war gerade nicht erreichbar — falls deine Anlage dort hängt, fehlt sie
   in der Liste."*
4. Erst nach der Wahl: `get_burner_status` mit der gewählten Nummer.

### C — Seriennummer, aber verunstaltet

> **Frage:** „Status von COR XC 001?"

1. Leerzeichen mittendrin → keine Seriennummer im Sinne der Regel → `find_burner(query="COR XC 001")`.
2. `match_count: 0`, `warnings: null`.
3. Antwort: *„Unter *COR XC 001* finde ich keine Anlage — alle Cluster haben geantwortet. Die Suche
   verkraftet zwar Groß-/Kleinschreibung, aber keine abweichende Schreibweise mit Leerzeichen. Wenn
   du die Nummer so hast, wie sie am Typenschild steht, versuche ich es damit nochmal."*
   (Nicht: eigenmächtig `corxc001`, `cor-xc-001`, `cor-xc001` durchprobieren.)

> **Derselbe Aufruf mit einem Zugang ohne interne Freigabe:** Der Server lehnt ab — bei `find_burner`
> genauso wie bei `get_burner_status`. Antwort: *„Anlagen-Livedaten brauchen interne Freigabe, die
> dein Zugang nicht hat."* — nicht „die Anlage konnte nicht abgefragt werden". Geht es allgemein um
> das Produkt, übernimmt das Orakel.
