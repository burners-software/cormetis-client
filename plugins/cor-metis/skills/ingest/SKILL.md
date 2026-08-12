---
name: ingest
description: "Spielt Dokumente in die COR.metis Knowledgebase ein (Ingest/Pflege). Nutze diesen Skill, wenn jemand ein Dokument (PDF, Datenblatt, Modbus-Tabelle, TÜV-Bericht, Präsentation, Anlagenbeschreibung, FAQ) in die KB aufnehmen, aktualisieren, neu klassifizieren oder versionieren will. Klassifiziert Vertraulichkeit und Typen, zerlegt in Chunks, baut Frontmatter, hängt Bilder an und gibt eine begutachtbare Review-Liste mit Deep-Links zurück. Greift auf den ESKS-MCP-Server zu."
---

# COR.metis Ingest-Skill

Nimmt Dokumente strukturiert in die Knowledgebase auf — sicher, nachvollziehbar, versionierbar.

## Leitprinzipien

- **„Maschine schlägt vor, Mensch committed."** Du klassifizierst und zerlegst, der Mensch gibt frei (Deep-Links, Vorschau-Manifeste, `review_status`).
- **Vier orthogonale Achsen, nie vermischen:** `confidentiality` (Sichtbarkeit) · `review_status` (Freigabe) · Versionsstand (`supersedes`/`document_lineage`) · `variant` (Hardware-Ausprägung).
- **Klassifikation per LLM-Reasoning, NICHT `suggest_type`** — Embeddings diskriminieren zu schlecht. `suggest_type` ist unbrauchbar.
- **Sicherheit vor Bequemlichkeit:** lieber über-restriktiv klassifizieren als leaken.
- **Bild-Bytes gehen nie durch den Context.** Upload ausschließlich binär per Ticket, niemals base64.

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
4. **Frontmatter bauen** (Vorlage unten), pro Chunk.
5. **Einspielen** via `ingest_chunk` — ein Chunk je Call. Vorab `ingest_smart` (read-only) gegen Dubletten prüfen.
6. **Bilder anhängen** per Upload-Ticket (siehe unten).
7. **Verifizieren** (Stichprobe + Suche + Stats).
8. **Review-Liste** mit Deep-Links zurückgeben.

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
- **Neue Version:** `find_lineage` → `replace_document(commit=false)` (Alignment-Manifest) → Review → `replace_document(commit=true)`. Rollback nur bewusst via `revert_document`.

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
- `logical_key`-Regel beachten: pro Source-Doc namespacen zerstört das Versions-Alignment.

## Verifikation

- `get_chunk` an Stichprobe (confidentiality? Typen?), `search` auf erwartete Begriffe, `get_stats` (Zähler plausibel?).
- Bei Bildern: mindestens ein `image_id` via `get_image_url` auflösen und prüfen, dass eine URL zurückkommt.

## Review-Ergebnis

Pro Datei: `{ deep_link, Titel, #Chunks, #Bilder, confidentiality_label }` → begutachtbare, klickbare Liste.

## Gotchas (hart gelernt)

- **Preflight nicht überspringen.** Ohne Cloud-Modus + Domain-Freigabe scheitert der Ingest erst beim Bild-Upload — also nach der ganzen Arbeit.
- Einstellungsänderungen in Claude Desktop greifen erst in einer **neuen** Session.
- `suggest_type` unbrauchbar → Typen per Reasoning.
- `confidentiality` immer explizit (auch `0`).
- `doc_type`/`sub_doc_type` nicht via `ingest_chunk` änderbar → vorher richtig, sonst `set_*_type`.
- Versionen: `logical_key`s teilen, nicht namespacen.
- C3 (400) wird abgelehnt — Text **und** Bild, gar nicht erst einspielen.
- **`store_image` mit `file_path` gibt es nicht mehr.** Der serverseitige Dateipfad wurde entfernt und wird aktiv refused (LFI-Fläche, und auf einem Cloud-Service ohnehin sinnlos). Damit sind auch `KB_IMAGE_IMPORT_DIRS` und der Compose-Import-Ordner Geschichte — Bilder gehen nur noch per Ticket-Upload.
- **`ingest_file` und `ingest_directory` existieren nicht.** Es gibt nur `ingest_chunk` (ein Chunk je Call) und `replace_document` (ganze Version). Wer nach Batch-Tools sucht, sucht vergeblich.
