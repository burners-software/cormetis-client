---
name: "burner-status"
description: "Live-Status einer konkreten CORgen-Anlage. Nutze diesen Skill, wenn jemand zu einer bestimmten Seriennummer wissen will, wie es der Anlage geht, was sie gerade fährt, ob Störungen anliegen oder warum sie steht — alles, was an einer Seriennummer hängt. Erklärt, wie die Antwort von `get_burner_status` zu lesen ist und vor allem, was sie NICHT hergibt. Abgrenzung: Fragen zur Dokumentation, zu Datenblättern, Modbus-Registern oder zur Bedeutung eines Fehlercodes im Allgemeinen gehören in den Skill orakel — dieser Skill liest nur den Momentzustand einer einzelnen Maschine. Greift auf den ESKS-MCP-Server zu und ist auf interne Clearance (C2) beschränkt."
---

# COR.metis Burner-Status — Live-Blick auf eine Anlage

`get_burner_status(serial_number)` fragt alle verbundenen Cluster nach **einer** Anlage und liefert
einen Schnappschuss: Betriebszustand, Live-Messwerte, aktive Error- und Info-Codes, System-Ampeln.

Das Tool ist **read-only**. Du kannst von hier aus nichts an einer Anlage verändern. Und es gibt
**keine Historie** — du siehst diesen Moment, sonst nichts.

> Der Tool-Call heißt in der aktuellen Instanz technisch `mcp__ESKS__get_burner_status`. Einziger
> Parameter ist `serial_number` — es gibt keinen Clearance-Parameter, weil es keinen braucht: die
> Prüfung hängt am API-Key des Fragenden und passiert serverseitig.

## Preflight — ist ESKS überhaupt verbunden?

Sind die `mcp__ESKS__*`-Tools nicht verfügbar (auch nicht als „deferred" nachladbar), fehlt dem Nutzer der Connector. Das ist etwas völlig anderes als eine abgelehnte Berechtigung — und muss auch anders klingen:

> „Der ESKS-Connector ist bei dir nicht eingerichtet — ohne den sehe ich keine Anlage. Sag ‚COR.metis einrichten', dann führe ich dich durch."

Der Skill `setup` aus diesem Plugin übernimmt das. Erst wenn der Connector da ist und der Server *antwortet*, greift die Berechtigungslogik aus Schritt 0.

## Leitprinzipien

- **Der Server schützt, du kennzeichnest.** Anlagendaten sind intern (C2). Ob jemand sie sehen darf,
  entscheidet sein Key — nicht du. Frag ihn nicht nach seiner Rolle; ruf auf und arbeite mit dem,
  was zurückkommt.
- **Erst der Zustand der Daten, dann die Daten.** Veraltet, abgeschaltet, Cluster stumm: das gehört
  vor die Interpretation, nicht in eine Fußnote.
- **Beantworte die Frage, die gestellt wurde.** Nicht die, für die du die meisten Felder hast.
- **Ehrlich über Grenzen.** Eine Verbrennungsstörung diagnostiziert man nicht aus einem Schnappschuss.

## Schritt 0 — Seriennummer klären, dann aufrufen

Es gibt hier **keine** Clearance-Vorabfrage. Anlagendaten erfordern interne Freigabe (C2), aber das
prüft der Server am Key des Fragenden. Wer nicht darf, bekommt eine Ablehnung — und wer fragen darf,
soll nicht erst ein Formular ausfüllen.

**Wenn das Tool ablehnt:** sag es klar und hör auf. Nachfassen hilft nicht, ein zweiter Versuch
ändert nichts. Und formuliere es als das, was es ist — eine Frage der Berechtigung, nicht ein
Problem mit der Anlage. *„Für Anlagen-Livedaten brauchst du interne Freigabe; dein Zugang hat sie
nicht"* ist richtig, *„die Anlage konnte nicht abgefragt werden"* ist irreführend.

**Keine Seriennummer genannt?** Nachfragen, nicht raten. Eine geratene Seriennummer trifft im
Zweifel eine fremde Anlage. Bei mehreren Seriennummern in einer Frage: pro Nummer ein Call, und die
Ergebnisse getrennt halten.

**Geht die Antwort nach draußen?** Falls jemand erkennbar für einen externen Empfänger schreibt —
Kunde, Partner, Servicebericht, der das Haus verlässt — dann sind diese Daten intern und gehören
nicht ungefiltert hinein. Weise darauf hin, statt es stillschweigend zu übernehmen. Für die
Dokumentationsseite gilt die Regelung im Skill **`cor-metis:orakel`**.

## Schritt 1 — Neun Dinge, die man leicht falsch liest

**„Nicht gefunden" heißt nicht immer „gibt es nicht."** Schau **zuerst** in `warnings`. Hat ein
Cluster nicht geantwortet, wurde dieser Cluster nicht durchsucht — und die Anlage kann sehr wohl
dort stehen. Sag, welcher Cluster stumm war. Erst wenn alle Cluster geantwortet haben und keiner die
Anlage kennt, darfst du sie als unbekannt bezeichnen. (`warnings` fehlt bei einem Treffer ganz und
ist `null`, wenn es nichts zu melden gibt — beides heißt: keine Warnungen.)

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
`component`, `sub_component` und `error`. Berichte die. „Fehler 17 in der Sensorik der
Heizungssteuerung" ist brauchbar; „unbekannter Fehler 1090584593" nicht — und du hattest genug in
der Hand, um es besser zu machen.

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

**Einheiten mitliefern.** Das Tool gibt sie meist her (`unit`) — aber nicht immer: Zähler wie
`FUEL_USED`, `PELLETS_USED` oder `ENGINE_RUNTIME_SINCE_START` kommen ohne. Wo eine Einheit fehlt,
nenne den Wert roh und **erfinde keine**. Wo eine da ist, gehört sie dran; eine Temperatur ohne
Einheit ist eine Zahl, mit der niemand etwas anfangen kann.

**Den Cluster nennen, wenn er zur Sache tut.** `cluster` sagt dir, wo die Anlage hängt. Bei einem
Namen wie `dev01` ist der Hinweis, dass es sich um ein Entwicklungs-/Testsystem handelt, für die
Einordnung oft wichtiger als jeder Messwert.

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
- `warnings` vor „nicht gefunden" lesen. Sonst erklärst du eine existierende Anlage für nicht existent.
- `data_stale` und `enabled: false` gehören **vor** die Interpretation, nicht dahinter.
- `data_stale: false` gilt für die Anlage, nicht für jeden Einzelwert — `timestamp_utc` pro Wert prüfen.
- `-273.2 °C` ist kein Messwert. Nie als Temperatur weitergeben.
- `infos` sind keine Fehler.
- `null` bei einer Ampel ist keine Farbe. Grüne Ampeln sind kein „alles in Ordnung" — dafür `errors` lesen.
- Keine Historie, kein Schreibzugriff. Wer einen Verlauf braucht, ist hier falsch — sag das.

## Feldreferenz (gegen eine echte Antwort verifiziert)

`cluster` · `enabled` · `last_seen_utc` · `last_seen_age_minutes` · `data_stale` ·
`stale_threshold_minutes` · `state{current, desired, engine, filling_day_container}` ·
`physics[{key, label, value, unit?, instance?, timestamp_utc}]` ·
`errors[{code, component, sub_component, error}]` · `infos[]` ·
`traffic_lights{hardware, software, raspi, issues[]}` · `warnings` (nur bei Nicht-Treffer)

> Ob ein **gemappter** Code zusätzlich ein `label`-Feld mitbringt, ist noch nicht gegen echte Daten
> geprüft — bislang wurden nur unmappte Codes beobachtet. Verhalte dich defensiv: ist ein Label da,
> nimm es; ist keins da, zerlege wie oben beschrieben.

## Mini-Beispiel

> **Frage:** „Was macht die cor-xc001 gerade?"

1. Seriennummer ist genannt → direkt aufrufen, keine Rückfrage.
2. `get_burner_status(serial_number="cor-xc001")`.
3. Antwort: `cluster: dev01`, `data_stale: false` (letzter Kontakt vor 1 min),
   `state.current: EMERGENCY-STOPPING` bei `desired: IDLE`, zwei unmappte `errors` in
   `DeviceHardware`, `infos: []`, Hardware- und Raspi-Ampel `null`, mehrere Fühler auf `-273.2 °C`.
4. Antwort: *„Die Anlage steht im Not-Aus (EMERGENCY-STOPPING), Sollzustand wäre IDLE. Zwei
   Störungen, beide in DeviceHardware, Sub-Komponente 5 (Fehler 1 und 2) — die Codes sind hier nicht
   gemappt. Keine Betriebsmeldungen. Software-Ampel grün, Hardware und Raspi melden keine Daten.
   Mehrere Fühler liefern gar keinen Wert. Läuft auf dem Cluster dev01, also einem Testsystem. Soll
   ich die beiden Codes in der KB nachschlagen?"*

> **Derselbe Aufruf mit einem Zugang ohne interne Freigabe:** Der Server lehnt ab. Antwort:
> *„Anlagen-Livedaten brauchen interne Freigabe, die dein Zugang nicht hat."* — nicht „die Anlage
> konnte nicht abgefragt werden". Geht es allgemein um das Produkt, übernimmt das Orakel.
