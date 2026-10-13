# cor-metis

Das Wissens-Orakel der COR Energy World GmbH für Claude Desktop.

## Was drin ist

| Skill | Aufruf | Zweck |
|---|---|---|
| `orakel` | `cor-metis:orakel` | Produkt-, Doku- und Technikfragen aus der Knowledgebase — mit Quellenlink und Vertraulichkeits-Kennzeichnung |
| `burner-status` | `cor-metis:burner-status` | Momentzustand einer CORgen-Anlage über die Seriennummer |
| `ingest` | `cor-metis:ingest` | Dokumente klassifizieren, zerlegen und in die Knowledgebase einspielen |
| `setup` | `cor-metis:setup` | Einrichtung und Selbsttest — „prüf mal, ob COR.metis läuft" |

Die Skills triggern von selbst. Man muss sie nicht aufrufen, sondern einfach fragen.

## Voraussetzung

Das Plugin bringt **keinen** Connector mit. Jeder Nutzer richtet den ESKS-Zugang einmalig mit
seinem persönlichen API-Key ein — er ist die Identität, an der die Freigabestufe hängt.

- **Claude Desktop / Cowork:** benutzerdefinierter Connector namens `ESKS` mit dem Key in der URL.
- **Claude Code:** zusätzlich das Plugin `cor-metis-code` installieren
  (`/plugin install cor-metis-code@cor-metis-client`). Es bringt die Server ESKS und CGSS mit und
  fragt den Key beim Aktivieren maskiert ab. Wer dieselben Connectoren auch in Desktop hat, deaktiviert
  in Claude Code unter `/mcp` die synchronisierten Server `claude.ai ESKS` / `claude.ai CGSS`, sonst
  stehen alle Tools doppelt da.

Die Skills finden die ESKS-Tools unter jedem dieser Wege: maßgeblich ist der Server-Name `ESKS`
im Tool-Namen, nicht der Präfix davor.

Anleitung: **[docs/ONBOARDING.md](../../docs/ONBOARDING.md)** im Repo-Root.
Oder direkt in Claude: *„COR.metis einrichten"*.

## Freigabestufen

Was jemand sieht, entscheidet der Server anhand seines Keys — nicht das Plugin. Die Skills
kennzeichnen, was eingestuft ist, und fragen bei ausgehender Kommunikation einmal nach dem
Empfänger, damit internes Material für einen Kundentext gar nicht erst geladen wird.

Skala: `C0` public · `C1C` confidential · `C1N` NDA · `C2` internal · `C3` (wird nicht eingespielt).

## Lizenz

Proprietär — COR Energy World GmbH.
