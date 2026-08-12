# COR.metis einrichten

Diese Anleitung richtet sich an alle Kolleginnen und Kollegen bei COR Energy. Sie setzt keine
Vorkenntnisse voraus und dauert etwa fünf Minuten.

Danach kannst du Claude Fragen zu unseren Produkten stellen — Datenblätter, Modbus-Register,
Fehlercodes, Installation, Zertifizierung — und bekommst Antworten aus **unseren** Unterlagen, mit
Link zur Quelle. Nicht aus dem Internet, nicht geraten.

---

## Was du brauchst

- **Claude Desktop** (die App, nicht die Website — Plugins laufen nur in der App).
- **Deinen persönlichen ESKS-API-Key.** Den bekommst du von Chris.

> **Zum Key:** Er ist dein Ausweis. Was du in COR.metis zu sehen bekommst, hängt an ihm — deshalb
> hat jeder seinen eigenen, und deshalb wird keiner weitergegeben. Auch nicht kurz, auch nicht an
> Kollegen. Und: **niemals in einen Chat tippen**, weder bei Claude noch woanders. Er gehört in genau
> ein Feld, und das kommt gleich in Schritt 1.

---

## Schritt 1 — Zugang zur Knowledgebase herstellen

1. In Claude Desktop: **Einstellungen → Konnektoren.**
2. **„Connector hinzufügen" → „Benutzerdefinierter Connector".**
3. **Name:** `ESKS`

   > ⚠️ **Exakt so, in Großbuchstaben.** Nicht „COR.metis", nicht „Metis", nicht „esks". Der Name ist
   > die Adresse, unter der COR.metis den Zugang später sucht — stimmt er nicht, ist alles verbunden
   > und trotzdem funktioniert nichts. Das ist mit Abstand der häufigste Stolperstein.

4. **URL:** deine Basis-URL mit angehängtem Key:

   ```
   https://internal.cor.energy/esks/mcp?apikey=DEIN_KEY
   ```

   `DEIN_KEY` ersetzt du durch deinen eigenen Key. Achte darauf, dass beim Kopieren kein Leerzeichen
   und kein Zeilenumbruch mitkommt.

5. **Speichern.**

---

## Schritt 2 — COR.metis installieren

1. **Einstellungen → Plugins** (unter „Anpassen").
2. **„Marketplace hinzufügen"**, dort eintragen:

   ```
   burners-software/cormetis-client
   ```

3. Im Marketplace das Plugin **`cor-metis`** auswählen und **installieren**.

---

## Schritt 3 — Neu starten und ausprobieren

Einstellungen greifen erst in einer **neuen** Session. Also: neue Aufgabe / neuen Chat starten und
schreiben:

> **prüf mal, ob COR.metis bei mir läuft**

Claude arbeitet dann eine kurze Checkliste ab und sagt dir in einem Satz, ob du startklar bist —
und falls nicht, woran es liegt.

---

## Und jetzt?

Einfach fragen. Beispiele:

- „Welche Modbus-Register hat die COR.gen TwinPower für die Energiezählung?"
- „Wo schließe ich die Pumpe an?"
- „Was bedeutet Fehlercode 17 in der Heizungssteuerung?"
- „Was macht die Anlage cor-xc001 gerade?" *(braucht interne Freigabe)*
- „Schreib mir eine Mail an einen Kunden über die Leistungsdaten der COR.gen."

Beim letzten Beispiel wird Claude einmal nachfragen, wer der Empfänger ist. Das ist Absicht: für
eine Mail nach draußen sucht COR.metis von vornherein nur in freigegebenem Material, damit nichts
Internes versehentlich mitrutscht.

### Zwei Dinge, die dir auffallen werden

**Antworten haben Quellen.** Unten steht, aus welchem Dokument etwas stammt, zum Anklicken. Wenn
kein Link dabei ist, ist es keine Antwort aus der Knowledgebase — dann nachhaken.

**Manchmal steht ein Hinweis zur Vertraulichkeit da**, etwa *„die Angabe zum Anschlussport ist
intern klassifiziert (C2)"*. Das ist kein Fehler, sondern die Antwort auf die Frage: *Was davon darf
ich weiterschicken?*

---

## Updates

Nichts zu tun. Verbesserungen an COR.metis kommen automatisch bei dir an — du musst nicht neu
installieren und nichts nachziehen.

---

## Nur für KB-Pflege: Dokumente einspielen

**Diesen Abschnitt kannst du überspringen**, wenn du COR.metis nur benutzt, um Fragen zu stellen.
Er betrifft die Handvoll Kolleginnen und Kollegen, die Dokumente **in** die Knowledgebase
einspielen.

Der Ingest liest PDFs, zerlegt sie, extrahiert Bilder und lädt sie hoch. Das passiert in der
Cloud-Sandbox und braucht deshalb drei Einstellungen, die fürs reine Fragen nicht nötig sind:

1. **Computerverwendung ausschalten** — *Einstellungen → Fähigkeiten → Computerverwendung → AUS.*
   Solange sie an ist, läuft die Aufgabe auf deinem Rechner statt in der Cloud, und der Ingest
   scheitert später beim Bild-Upload.

2. **Netzwerkzugriff freigeben** — *Einstellungen → Fähigkeiten:*
   - „Cloud-Code-Ausführung und Dateierstellung" → **AN**
   - „Ausgehenden Netzwerkverkehr erlauben" → **AN**
   - „Domain-Zulassungsliste" → **Alle Domains**

3. **In der Cloud arbeiten, nicht lokal** — beim Starten einer Aufgabe „In der Cloud" wählen.

Danach **neue Session** starten — vorher greifen die Einstellungen nicht.

Zum Prüfen:

> **prüf die Ingest-Voraussetzungen**

Und dann: Dokument anhängen, „spiel das bitte in die Knowledgebase ein". COR.metis klassifiziert,
zerlegt und legt dir am Ende eine Liste zum Gegenlesen vor — eingespielt wird nur, was du
freigibst.

---

## Wenn es klemmt

Der schnellste Weg ist immer: **„prüf mal, ob COR.metis bei mir läuft"**. Die Diagnose sagt dir, an
welcher Stelle es hakt. Zur Einordnung:

| Symptom | Wahrscheinliche Ursache |
|---|---|
| „Ich finde dazu nichts in der Knowledgebase", obwohl es das Dokument gibt | Connector heißt nicht exakt `ESKS` |
| Claude antwortet aus Allgemeinwissen statt aus unseren Unterlagen | Plugin nicht installiert, oder Session vor der Installation gestartet |
| Fehlermeldung mit „unauthorized" / 403 | Key falsch, abgelaufen oder beim Kopieren beschädigt → bei Chris melden |
| Zeitüberschreitung, „Server nicht erreichbar" | Du bist nicht im Firmennetz / nicht im VPN |
| „Dein Zugang hat keine interne Freigabe" | Kein Fehler. Anlagen-Livedaten sind intern; die Dokumentensuche funktioniert weiter |
| Ingest bricht beim Bild-Upload ab | Die drei Schalter oben — und danach eine **neue** Session |

Hilft das nicht weiter: bei Chris melden, am besten mit der wörtlichen Fehlermeldung.

---

## Sicherheit in drei Sätzen

Dein Key ist deine Identität — nicht weitergeben, nicht in Chats tippen, nicht in Tickets kleben.
Was du siehst, entscheidet der Server anhand deines Keys; COR.metis kann dir nichts zeigen, wofür du
nicht freigegeben bist. Wenn du einen Text nach draußen schreibst, sag es dazu — dann sucht COR.metis
von vornherein nur in freigegebenem Material.
