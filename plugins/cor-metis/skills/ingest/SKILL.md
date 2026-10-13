---
name: ingest
description: "Spielt Dokumente in die COR.metis Knowledgebase ein (Ingest/Pflege). Nutze diesen Skill, wenn jemand ein Dokument (PDF, Datenblatt, Modbus-Tabelle, TÜV-Bericht, Präsentation, Anlagenbeschreibung, FAQ) in die KB aufnehmen, aktualisieren, neu klassifizieren oder versionieren will. Klassifiziert Vertraulichkeit und Typen, zerlegt in Chunks, baut Frontmatter, legt aus Tabellen (Fehlercode-Listen, Teile-/SKU-Listen, Hersteller-/Lieferantentabellen) nachschlagbare Tabellen-Einträge an, hängt Bilder an und gibt eine begutachtbare Review-Liste mit Deep-Links zurück. Greift auf den ESKS-MCP-Server zu."
---

# COR.metis Ingest-Skill

Nimmt Dokumente strukturiert in die Knowledgebase auf — sicher, nachvollziehbar, versionierbar.

Alle Tools in diesem Skill (`ingest_chunk`, `create_image_upload_ticket`, `lookup` …) kommen vom ESKS-Server. Ihr Präfix hängt von der Oberfläche ab (`mcp__ESKS__…`, `mcp__claude_ai_ESKS__…`, `mcp__plugin_cor-metis-code_ESKS__…`); maßgeblich ist das Segment unmittelbar vor `__<tool>`: `ESKS`, Groß-/Kleinschreibung egal. Deferred Tools per ToolSearch-Stichwortsuche nachladen („ESKS ingest_chunk“), nie per `select:` mit festem Namen.

## Leitprinzipien

- **„Maschine schlägt vor, Mensch committed."** Du klassifizierst und zerlegst, der Mensch gibt frei (Deep-Links, Vorschau-Manifeste, `review_status`).
- **Vier orthogonale Achsen, nie vermischen:** `confidentiality` (Sichtbarkeit) · `review_status` (Freigabe) · Versionsstand (`supersedes`/`document_lineage`) · `variant` (Hardware-Ausprägung).
- **Klassifikation per LLM-Reasoning, NICHT `suggest_type`** — Embeddings diskriminieren zu schlecht. `suggest_type` ist unbrauchbar.
- **Sicherheit vor Bequemlichkeit:** lieber über-restriktiv klassifizieren als leaken.
- **Bild-Bytes gehen nie durch den Context.** Upload ausschließlich binär per Ticket, niemals base64.
- **Tabellen doppelt ablegen:** lesbar im Chunk-Body **und** Zeile für Zeile als Tabellen-Einträge dieses Chunks. Nur Einträge machen Codes nachschlagbar — Vektorsuche ist für Kennungen blind.

---

## Schritt 0: Preflight (PFLICHT, vor allem anderen)

Der Ingest braucht zwei Voraussetzungen. Fehlt eine, funktionieren Bilder nicht — und das sieht dann so aus, als könnte COR.metis keine Bilder. Kann es. Es sind zwei Schalter.

**Beide Checks laufen, bevor du das erste Dokument anfasst. Das Ergebnis sagst du dem User aktiv** — auch wenn alles passt (ein Satz genügt). Niemals stillschweigend loslegen und erst beim Bild-Upload scheitern.

### Check 1 — Cloud-Modus aktiv?

Der Ingest muss in der Anthropic-VM laufen, nicht lokal. Prüfen: kannst du Shell-Kommandos ausführen und Dateien schreiben?

Wenn nein → **Computernutzung ist an und blockiert den Cloud-Modus.** Die beiden schließen sich gegenseitig aus.

### Check 2 — Ist `internal.cor.energy` erreichbar?

```bash
curl -sS -o /dev/null -w '%{http_code}\n' --max-time 15 https://internal.cor.energy/esks/health
```

Erwartet: `200`. Alles andere (DNS-Fehler, Timeout, Connection refused) → **die Domain ist für die Sandbox nicht freigegeben.**

### Wenn ein Check fehlschlägt: abbrechen und das hier ausgeben

> **Ingest kann noch nicht starten — es fehlen zwei Einstellungen in Claude Desktop.**
>
> **1. Computernutzung ausschalten** — *Einstellungen → Desktop-App → Allgemein → Computernutzung → „Computernutzung aktivieren" auf AUS.*
> Solange Computernutzung an ist, kann Cowork nicht in der Cloud-VM laufen. Die beiden Modi schließen sich gegenseitig aus.
>
> **2. Netzwerkzugriff freigeben** — *Einstellungen → Fähigkeiten:*
> - „Cloud-Code-Ausführung und Dateierstellung" auf AN
> - „Ausgehenden Netzwerkverkehr erlauben" auf AN
> - „Domain-Zulassungsliste" auf **Alle Domains** (alternativ eine Allowlist, die `internal.cor.energy` enthält)
>
> Danach die Aufgabe neu starten — Einstellungen greifen erst für eine **neue** Session.
>
> Hintergrund: Bilder werden absichtlich binär hochgeladen, nicht als base64 durch den Chat. Das ist der Grund, warum der Netzwerkzugriff nötig ist — und der Grund, warum der Ingest auch bei einem PDF mit 30 Diagrammen nicht aus dem Ruder läuft.

Nicht weiterarbeiten, bis beide Checks grün sind. Kein Teil-Ingest „schon mal ohne Bilder", außer der User verlangt es nach dem Hinweis ausdrücklich.

---

## Ablauf (ein Dokument)

0. **Preflight** (siehe oben) — beide Checks grün, Ergebnis dem User gesagt.
1. **Lesen.** `pdfinfo` → Seitenzahl; PDF in Etappen (≤20 Seiten je Read) **vollständig** lesen — inkl. Deckblatt/Klassifikationsseite.
2. **Klassifizieren** (confidentiality + doc_type + sub_doc_type).
3. **Zerlegen** in sinnvolle Chunks (ein Thema je Chunk; Tabellen/Register-Bereiche zusammenhalten).
4. **Tabellen erkennen & inspizieren** — enthält das Dokument Tabellendaten oder Key/Value-Artiges? Dann Namespaces prüfen, bei einer neuen Version den Ist-Stand der Einträge holen und die Differenz berichten (siehe [Tabellen-Einträge](#tabellen-einträge-entries)). Neuer Namespace nur nach Zustimmung.
5. **Frontmatter bauen** (Vorlage unten), pro Chunk — Chunks mit Einträgen brauchen `logical_key` und `document_lineage`.
6. **Einspielen** via `ingest_chunk` — ein Chunk je Call, Einträge im `entries`-Argument. Vorab `ingest_smart` (read-only) gegen Dubletten prüfen. Neue Version: `replace_document` mit `entries_by_logical_key`.
7. **Bilder anhängen** per Upload-Ticket (siehe unten).
8. **Verifizieren** (Stichprobe + Suche + Stats + Einträge).
9. **Review-Liste** mit Deep-Links und Eintragszahlen zurückgeben.

## Confidentiality (sicherheitskritisch)

Stufe **aus dem Dokument** lesen (Deckblatt/Fußzeile), in Integer übersetzen:

| Label im Dok | `confidentiality` |
|---|---|
| C0 / public / öffentlich | `0` |
| C1C / confidential | `100` |
| C1N / NDA | `200` |
| C2 / internal | `300` |
| C3 | `400` → **wird abgelehnt**, nicht einspielen |

**Immer explizit setzen.** Keine Stufe im Dokument → konservativ **C2 (300)** + Hinweis.

**C3 gilt für Bilder genauso wie für Text.** Ein C3-Bild wird nicht extrahiert, nicht hochgeladen, nicht angehängt. Typische Fälle: Foto einer Demo-Installation bei einem Geschäftspartner, ein noch vertrauliches Diagramm. Im Zweifel: nicht hochladen und den User fragen.

## Typen (organisch wachsen lassen)

- `doc_type` (Source-Doc-Eigenschaft) **und** `sub_doc_type` (Sub-Doc-Eigenschaft) per **Reasoning** gegen `list_types` bestimmen — beide Registries (`doc`/`subdoc`) vorher listen.
- Passt nichts → `create_type(registry, id, name, description, examples[2-3], rationale)` als **provisional**.
- **Beim Ingest gleich richtig setzen.** Nachträglich nur via `set_document_type` / `set_subdocument_type` — **nicht** über `ingest_chunk`.

## IDs, Lineage & Versionen

- **Auto-ID:** `id` weglassen → Server leitet aus `source_document_id` + slug(`logical_key`) ab.
- `document_lineage` bleibt **stabil über alle Versionen**; `source_document_id` ist pro Version verschieden.
- **`logical_key` — wichtigste Regel:** Für verschiedene Versionen DESSELBEN Dokuments **müssen** die `logical_key`s übereinstimmen (NICHT pro Version namespacen!). Nur so greifen `replace_document`, `diff_versions`, Supersede. Verschiedene Dokumente = verschiedene `document_lineage`.
- **Neue Version:** `find_lineage` → Einträge-Ist-Stand + Differenzbericht (siehe Tabellen-Einträge) → `replace_document(commit=false)` (Alignment-Manifest, zeigt `entry_count` je Chunk) → Review → `replace_document(commit=true)`. Rollback nur bewusst via `revert_document`.

## Doc-Metadaten

**`doc_description`** ist Retrieval-Gold (wird pro Treffer ausgeliefert) — **immer** setzen: 1–2 Sätze mit *was · Produkt · Version · Scope/Variante · Vertraulichkeit*. Weitere Felder: `doc_version` (Key heißt `doc_version`), `language`, `product`, `source_file`, `title`, `logical_key`. `review_status`: Prototyp-Default `approved`, Ziel `preliminary`. `authoritative: true` nur für kanonische Quellen.

### Frontmatter-Vorlage (pro Chunk)

```markdown
---
title: "Modbus-Register – Energiezählung (Reg 39–52)"
document_lineage: rust_thor_modbus
source_document_id: rust_thor_modbus_thor
logical_key: rust_thor_modbus/energy-metering   # GLEICH über Versionen halten!
doc_type: technical_spec
sub_doc_type: modbus_register_map
product: COR.gen TwinPower
confidentiality: 300            # C2 internal — IMMER explizit
doc_description: "Modbus-Registerkarte COR.gen TwinPower (THOR), Energiezählung Reg 39–52. Vertraulichkeit C2 (intern)."
doc_version: THOR
language: de
source_file: "RUST AUX Logic Control Modbus Interface Registers - THOR.xlsx"
review_status: approved
authoritative: true
supersedes: null
---

# Modbus-Register – Energiezählung
Body als Markdown …
```

---

## Tabellen-Einträge (entries)

Ein **Eintrag** ist ein kleiner, strukturierter Fakt aus einer Tabelle des Quelldokuments. Das
Orakel findet ihn per `lookup` (exakt, bei Tippfehlern fuzzy) und über die `entries`-Liste von
`search` — auch aus einer Symptombeschreibung. Ohne Eintrag bleibt „Fehlercode 123" für die KB
unauffindbar, egal wie sauber der Chunk ist.

| Feld | Inhalt | Beispiel |
|---|---|---|
| `namespace` | welche Art Fakt | `device_error_codes`, `primary_manufacturers` |
| `key` | die Kennung, **wie in der Quelle geschrieben** | `123`, `E-05`, `AB531183XYZ`, `side channel compressor` |
| `value` | kurze Antwort, englisch, ≤ 500 Zeichen | `side channel compressor overheated` |
| `notes` | optional: Ursache, Abhilfe, Erläuterung, englisch, ≤ 4000 Zeichen | `Check the cooling air intake …` |
| `embedding_text` | ein natürlicher englischer Satz für die semantische Suche, mit Produkt | `error code 123 of the COR.gen: the side channel compressor has overheated` |

Ein Eintrag gehört **immer zu dem Chunk, aus dem er stammt**, und über ihn zu genau einer
Dokumentversion. Vertraulichkeit, `review_status` und Produkt erbt er vom Dokument — er hat keine
eigenen. Einträge ohne Quell-Chunk gibt es nicht.

### Wann Einträge anlegen

Sobald eine Quelle Tabellendaten enthält oder etwas, das von Natur aus Key und Value ist:

- Fehlercode- und Meldungslisten
- Teile-, Artikel- und SKU-Listen
- Hersteller- und Lieferantentabellen
- Registertabellen, **wenn die Registernummer das ist, wonach Leute fragen**

Der Chunk bleibt, wie er ist — die Tabelle steht weiter lesbar im Body — **und** ihre Zeilen kommen
als Einträge dazu.

**Nie** Einträge für Geräte-Seriennummern. Die leben in der Live-Anlagenverwaltung (`find_burner`).

### Vorher inspizieren (Pflicht)

1. **`list_namespaces`** aufrufen. Passt ein vorhandener Namespace, **wiederverwenden** — Beschreibung
   und `key_type` lesen, Normalisierungsregeln (`strip_prefixes`, `strip_leading_zeros`) gegen die
   Keys der Quelle halten.
2. **Bei einer neuen Version eines vorhandenen Dokuments:** `find_lineage` → `source_document_id` der
   aktuellen Version → `list_entries(source_document_id: …, limit: 500)`. **`truncated` prüfen** —
   ist es `true`, pro `namespace` oder `chunk_id` nachladen, bis der Ist-Stand vollständig ist.
   Dann dem User den **Differenzbericht** zeigen, bevor irgendetwas geschrieben wird:

   ```
   Einträge device_error_codes — v3 → v4
   + neu (4):       130, 131, 132, E-07
   − entfallen (1): 118            ← nach dem Replace aus der Suche verschwunden
   ~ geändert (2):  123  value: "compressor overheated" → "side channel compressor overheated"
                    155  notes geändert
   = unverändert:   41
   ```

   Entfallene Keys ausdrücklich benennen — das ist die Stelle, an der ein versehentlich
   abgeschnittener Tabellenteil auffällt.
3. **Nur wenn nichts passt:** einen neuen Namespace **vorschlagen** und auf Zustimmung warten:
   - `id` in `lower_snake_case`, englische Beschreibung (1–2 Sätze: was die Keys identifizieren,
     woher sie stammen), `key_type`:
     - `code` — Kennungen; exakt, dann fuzzy (ab 3 Zeichen) gefunden.
     - `text` — Namen und Begriffe, **englisch**; exakt, dann semantisch gefunden.
     `key_type` ist später **nicht** änderbar.
   - bei `code`: sollen Präfixe wie `E` oder `F` entfernt werden (`strip_prefixes`, längere zuerst),
     sollen führende Nullen fallen (`strip_leading_zeros`)? Mit echten Keys aus der Quelle
     vorrechnen, was dabei herauskommt.
   - **Präfix nur strippen, wenn er nichts bedeutet:** sind `E05` und `F05` verschiedene Meldungen,
     dürfen `E` und `F` nicht weg — sonst werden beide zu `05`.
   - `id` und Beschreibung sieht **jeder** Mitarbeiter, die `id` auch der Kundensupport-Bot — nichts
     Vertrauliches hinein.
   - Erst nach „ja" `create_namespace` aufrufen. Er startet als **provisional**; ein Administrator
     gibt ihn frei.
   - **Die Normalisierungsregeln frieren mit dem ersten Eintrag ein.** Keine Einträge in einen neuen
     Namespace schreiben, bevor der User die Regeln gegen die echte Quelle bestätigt hat.

### Wie schreiben

- **`ingest_chunk`** nimmt ein optionales `entries`-Array; **`replace_document`** nimmt
  `entries_by_logical_key`: Objekt von `logical_key` des Chunks → seine Einträge.
- **Eine neue Version ist eine neue Liste.** Bei `replace_document` die **vollständigen** Einträge
  für **jeden** Chunk mitgeben, der welche hat — nicht nur die geänderten. Die Einträge der alten
  Version gehen mit ihr in Rente; was fehlt, ist aus der Suche verschwunden.
- **`entries` bei `ingest_chunk` auf einen bestehenden Chunk:**

  | Argument | Wirkung |
  |---|---|
  | weggelassen | vorhandene Einträge bleiben unverändert |
  | `[]` | alle Einträge des Chunks werden **entfernt** |
  | Liste | ersetzt die Einträge des Chunks komplett |

  `[]` nur bewusst, nie aus Versehen.
- **Neuer Chunk mit `supersedes:`** ist eine neue Version *dieses* Chunks: die Einträge des alten
  gehen mit ihm in Rente, der neue hat **nur**, was du mitgibst. `entries` weglassen heißt dort:
  der neue Chunk hat keine.
- Ein Chunk mit Einträgen **muss** einen `logical_key` haben und zu einer `document_lineage` gehören
  (eigene oder die seines Dokuments) — sonst lehnt der Server ab, auch bei einem späteren Re-Ingest,
  der den `logical_key` eines Chunks mit Einträgen fallen ließe. Grund: ohne `logical_key` könnte ein
  späteres `replace_document` den Chunk nicht ablösen, und seine alten Werte blieben neben der neuen
  Version auffindbar.
- Neue Dokumente tragen **immer** ein explizites `confidentiality` im Frontmatter — Tabellendokumente
  auch. Die Einträge erben es.
- **Höchstens 500 Einträge pro Chunk.** Längere Tabellen entlang ihrer natürlichen Abschnitte auf
  mehrere Chunks verteilen (z.B. nach Baugruppe oder Codebereich), jeder mit eigenem stabilen
  `logical_key`.
- **Ein Key höchstens einmal pro Namespace und Dokumentversion** — und zwar nach Normalisierung
  (`E-05` und `5` können derselbe Key sein). Führt die Quelle selbst einen Key doppelt mit
  unterschiedlicher Bedeutung: **stoppen und dem User zeigen** — das ist ein Fehler in der Quelle,
  nicht etwas, das du auflöst.
- **Keys wie in der Quelle** (`E-05` bleibt `E-05`); der Server normalisiert.
- **`value` und `notes` englisch.** `value` kurz; Erklärungen, Ursachen, Abhilfe gehören in `notes`.
- **Keine URLs in `value` oder `notes`.** Der Kundensupport-Bot entfernt jeden Text mit Link — die
  ganze Notiz wäre für Kunden weg. Auch keine Seriennummern.
- **`embedding_text` für jeden Eintrag schreiben:** ein Satz, in dem jemand mit dem Problem es
  wiedererkennt, **mit Produkt**. Er ersetzt den Key nicht — der wird separat gematcht. Beispiel:
  `"COR.gen error 42: the fuel store is empty and no pellets reach the burner"`.

Beispiel `ingest_chunk` (Werte erfunden):

```json
{
  "markdown": "---\ntitle: \"Fehlercodes 100–199\"\ndocument_lineage: corgen_error_codes\nlogical_key: corgen_error_codes/100-199\nconfidentiality: 300\n...\n---\n\n| Code | Meldung | Abhilfe |\n|---|---|---|\n| 123 | Seitenkanalverdichter überhitzt | … |",
  "entries": [
    {
      "namespace": "device_error_codes",
      "key": "123",
      "value": "Side channel compressor overheated",
      "notes": "Check the cooling air intake for blockage; let the unit cool down before restarting.",
      "embedding_text": "COR.gen error 123: the side channel compressor has overheated"
    }
  ]
}
```

### Danach

Pro Chunk berichten, **wie viele Einträge in welchen Namespaces** geschrieben wurden — zusammen mit
der üblichen Review-Liste. Einträge werden erst auffindbar, wenn das Dokument freigegeben ist, genau
wie Chunks.

---

## Bilder (Upload-Ticket-Flow)

### base64 ist verboten

`store_image` mit `base64_data` wird **nicht** verwendet — auch nicht „nur schnell für das eine Bild". Keine Ausnahme, kein Fallback.

Warum: base64 bläht die Bytes um ~33% auf und schiebt sie durch den LLM-Context. Ein Datenblatt mit 30 Diagrammen frisst so den Context, bevor die eigentliche Arbeit anfängt — und kostet Tokens pro Bild. Der Ticket-Weg schiebt dieselben Bytes per HTTP daran vorbei.

### Ablauf pro Bild

1. **Extrahieren** — `pdfimages -png` für eingebettete Bilder, `pdftoppm` für ganzseitige Renderings. Nur **inhaltlich wertvolle** Abbildungen: Diagramme, Schaltbilder, P&ID, Zeichnungen, Produktfotos. Logos und Deko überspringen.
2. **Optimieren** — längste Kante auf ~2000px, als WebP oder PNG. Spart Upload-Zeit und Storage, kostet bei technischen Zeichnungen nichts an Lesbarkeit.
3. **Ticket minten** — `create_image_upload_ticket(chunk_id, description, max_bytes)`. Alle drei Parameter optional. Ein Ticket pro Bild; die Calls sind billig (kein Embedding). `description` **reichhaltig** — sie ist das, was später gefunden wird, und sie wird ins Ticket eingebacken: der Uploader kann sie nicht mehr ändern. Zurück kommen `upload_url` (absolut) und `ticket`.
4. **Bytes hochladen** — Ticket geht als **Header**, nicht in der URL:
   ```bash
   curl -sS -X POST "$UPLOAD_URL" \
        -H "X-Upload-Ticket: $TICKET" \
        -F "file=@diagramm.webp" \
        -H "Accept: application/json"
   # → {"image_id":"...","mime_type":"image/webp","url":"https://...","length":83421,...}
   ```
   Ohne den `X-Upload-Ticket`-Header schlägt der Upload fehl — das ist die häufigste Stolperstelle. Kein Bearer-Token nötig, das Ticket **ist** die Autorisierung.

   Statuscodes richtig lesen:

   | Code | Bedeutung | Reaktion |
   |---|---|---|
   | `201` | gespeichert | weiter, `image_id` + `url` im Body |
   | `400` | kein lesbares Multipart, leeres `file`-Feld, oder kein unterstützter Bildtyp | Datei prüfen — nicht das Ticket |
   | `404` | Ticket fehlt, ist unbekannt, abgelaufen oder verbraucht (bewusst ununterscheidbar) | **neu minten**, nicht retrien |
   | `409` | Ticket und Bytes ok, aber der Ziel-Chunk wurde nach dem Minten gelöscht oder umklassifiziert | **Stopp.** Nicht blind neu minten — erst klären, was mit dem Chunk passiert ist. Bei Umklassifizierung kann das Bild jetzt zu hoch eingestuft sein. |
   | `413` | größer als das Ticket erlaubt | kleiner rechnen, neu minten |
5. **Fertig.** Das `url`-Feld im Response ist bereits die signierte URL — `get_image_url` brauchst du nur, wenn die Signatur später abgelaufen ist.

### Regeln

- Das Bild erbt die `confidentiality` seines Chunks. Ohne Chunk gilt C2 internal.
- Ein Ticket = ein Bild, single-use, wenige Minuten gültig. Abgelaufen → neu minten, nicht retrien.
- Tickets erst minten, wenn der Chunk existiert und die `chunk_id` bekannt ist.
- **Sollte `create_image_upload_ticket` (noch) nicht verfügbar sein:** nicht auf base64 ausweichen. Stattdessen Text-Ingest sauber abschließen und pro Chunk eine Nachtrag-Liste ausgeben (`chunk_id`, Quelldatei, vorgesehene `description`), damit die Bilder später in einem Rutsch nachgezogen werden können. Den User darauf hinweisen.

## Bulk-Ingest (viele Dokumente)

- **Ein Dokument pro Subagent**, parallel — 8 gleichzeitig bewährt.
- **Preflight nur einmal** im Hauptlauf, nicht pro Subagent.
- **Typ-Anlage zentralisieren (Race-Schutz):** Subagenten *melden* fehlende Typen, legen sie **nicht selbst** an. Haupt-Lauf legt Typen einmal zentral an, dann ingesten die Subagenten.
- **Namespaces genauso:** Subagenten melden fehlende Namespaces samt Beispiel-Keys, der Haupt-Lauf schlägt sie dem User vor und legt sie nach Zustimmung einmal an — erst danach schreiben Subagenten Einträge.
- `logical_key`-Regel beachten: pro Source-Doc namespacen zerstört das Versions-Alignment.

## Verifikation

- `get_chunk` an Stichprobe (confidentiality? Typen?), `search` auf erwartete Begriffe, `get_stats` (Zähler plausibel?).
- Bei Bildern: mindestens ein `image_id` via `get_image_url` auflösen und prüfen, dass eine URL zurückkommt.
- Bei Einträgen: `list_entries(source_document_id: …)` — Anzahl je Namespace gegen die Quelltabelle halten (`truncated` beachten). Ist das Dokument bereits freigegeben, ein, zwei Keys per `lookup` gegenprüfen; bei `preliminary` sind sie für `lookup` noch unsichtbar, dann reicht `list_entries`.

## Review-Ergebnis

Pro Datei: `{ deep_link, Titel, #Chunks, #Bilder, #Einträge je Namespace, confidentiality_label }` → begutachtbare, klickbare Liste. Bei einer neuen Version zusätzlich der Differenzbericht der Einträge.

## Gotchas (hart gelernt)

- **Preflight nicht überspringen.** Ohne Cloud-Modus + Domain-Freigabe scheitert der Ingest erst beim Bild-Upload — also nach der ganzen Arbeit.
- Einstellungsänderungen in Claude Desktop greifen erst in einer **neuen** Session.
- `suggest_type` unbrauchbar → Typen per Reasoning.
- `confidentiality` immer explizit (auch `0`).
- `doc_type`/`sub_doc_type` nicht via `ingest_chunk` änderbar → vorher richtig, sonst `set_*_type`.
- Versionen: `logical_key`s teilen, nicht namespacen.
- C3 (400) wird abgelehnt — Text **und** Bild, gar nicht erst einspielen.
- **`store_image` mit `file_path` gibt es nicht mehr.** Der serverseitige Dateipfad wurde entfernt und wird aktiv refused (LFI-Fläche, und auf einem Cloud-Service ohnehin sinnlos). Damit sind auch `KB_IMAGE_IMPORT_DIRS` und der Compose-Import-Ordner Geschichte — Bilder gehen nur noch per Ticket-Upload.
- **Tabellen → Einträge.** Chunk allein macht Codes nicht nachschlagbar.
- Namespace erst vorschlagen, dann anlegen; Normalisierung frieren mit dem ersten Eintrag ein. Präfixe nur strippen, wenn sie nichts bedeuten.
- `replace_document`: **vollständige** Einträge aller Chunks mitgeben — Weggelassenes ist weg.
- `ingest_chunk`: `entries` weglassen = behalten, `[]` = löschen. Bei `supersedes:` heißt Weglassen: keine.
- Chunks mit Einträgen brauchen `logical_key` + `document_lineage`.
- Einträge englisch, keine URLs, keine Seriennummern, ≤ 500 je Chunk, Key einmal pro Namespace und Version.
- **`ingest_file` und `ingest_directory` existieren nicht.** Es gibt nur `ingest_chunk` (ein Chunk je Call) und `replace_document` (ganze Version). Wer nach Batch-Tools sucht, sucht vergeblich.
