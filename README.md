# cormetis-client

Marketplace und Plugin-Paket für **COR.metis** — das Wissens-Orakel der COR Energy World GmbH in
Claude Desktop.

Dieses Repo ist gleichzeitig Marketplace *und* Plugin-Container. Kollegen fügen es einmal als
Marketplace hinzu, installieren `cor-metis`, und bekommen Skill-Updates danach automatisch.

> **Du willst COR.metis nur benutzen?** → **[docs/ONBOARDING.md](docs/ONBOARDING.md)**
> Fünf Minuten, keine Vorkenntnisse nötig. Der Rest dieser Datei richtet sich an die Wartung.

---

## Aufbau

```
.claude-plugin/marketplace.json     Marketplace-Manifest (listet die Plugins)
plugins/
└── cor-metis/
    ├── .claude-plugin/plugin.json  Name, Version, Beschreibung
    ├── README.md
    └── skills/
        ├── orakel/                 Retrieval aus der Knowledgebase, clearance-abhängig
        ├── burner-status/          Momentzustand einer Anlage per Seriennummer
        ├── ingest/                 Dokumente in die KB einspielen
        └── setup/                  Einrichtung + Selbsttest
docs/ONBOARDING.md                  Die Anleitung für Kollegen
```

**Kein `.mcp.json` — bewusst.** Der ESKS-Zugang hängt an einem persönlichen API-Key, der in der
Connector-URL steht. Ein Paket, das alle bekommen, kann nichts Personenbezogenes transportieren.
Getestet und widerlegt wurde der Weg über `userConfig`: Cowork Desktop substituiert
`${user_config.KEY}` nicht, der Platzhalter landet wörtlich in der URL. Beleg und Begründung:
`cormetis-skills-raw/docs/COR.metis-Deployment-Spec.md`, Abschnitt 2–3.

Konsequenz: **Das Plugin liefert die Skills, den Connector richtet jeder einmalig selbst ein.**
Das Teil, das sich häufig ändert, wird verteilt; das Teil, das sich nie ändert, wird einmal gesetzt.

---

## Voraussetzungen an das Repo

- **Das Repo muss öffentlich sein.** Private Marketplace-Repos hängen an den Git-Credentials jedes
  einzelnen Kollegen; die Hintergrund-Updates sind dort unzuverlässig. Inhaltlich unbedenklich —
  einzige bewusste Preisgabe ist der Hostname `internal.cor.energy` in der Onboarding-Doku.
- **Keine Keys im Repo.** Nirgends, auch nicht als Beispiel. Der Platzhalter heißt `DEIN_KEY`.

---

## Skill-Namespacing

Plugin-Skills werden mit dem Plugin-Namen präfigiert. Die Ordner heißen deshalb `orakel/`,
`burner-status/`, `ingest/`, `setup/` — daraus wird `cor-metis:orakel` usw. Die `name:`-Felder in
den Frontmattern sind entsprechend gekürzt; hießen sie weiter `cor-metis-orakel`, entstünde
`cor-metis:cor-metis-orakel`.

Die `description:`-Felder steuern das Triggering und sind gegeneinander abgegrenzt — beim Ändern
darauf achten, dass die Abgrenzung zwischen `orakel` (Dokumentation) und `burner-status` (Maschine)
erhalten bleibt.

---

## Eine Änderung ausliefern

1. Skill bearbeiten.
2. `version` **an beiden Stellen** hochziehen — `plugins/cor-metis/.claude-plugin/plugin.json` und
   `.claude-plugin/marketplace.json`. Laufen die auseinander, zieht Cowork nicht sauber nach.
3. Committen und pushen.

Cowork prüft gegen den Marketplace und aktualisiert im Hintergrund. Lokal veränderte Dateien werden
vor dem Überschreiben gemeldet.

**Semver-Disziplin:**

| Änderung | Bump |
|---|---|
| Formulierungen, Tippfehler, Beispiele | Patch |
| Neue Regel, neuer Skill, erweiterte Abdeckung | Minor |
| Verhaltensänderung, die jemand aktiv kennen muss | Major |

Ein späterer Split (z.B. `ingest` in ein eigenes Plugin) bleibt über `renames` im Marketplace-Manifest
verlustfrei.

---

## Vor dem Rollout offen

Aus der Deployment-Spec, Abschnitt 7 — serverseitig, blockierend:

- [ ] **`apikey` aus den Query-Logs maskieren.** Der Key steht in der URL und landet damit in
      Access-Logs, Proxy-Logs und Error-Traces.
- [ ] **Keys URL-safe erzeugen** (base64url oder Hex) — `+`, `/`, `=` zerlegen den Query-String.
- [ ] **Keys pro Person, widerrufbar.** Ein geteilter Key hebelt das Clearance-Modell aus.
- [ ] **MCP-Endpoint-Pfad bestätigen.** `/esks/mcp` ist aus `/esks/health` abgeleitet, nicht verifiziert.

Weiter offen, nicht blockierend:

- [ ] `burner-status`: ob ein **gemappter** Fehlercode ein `label`-Feld mitbringt, ist gegen echte
      Daten unverifiziert.
- [ ] Claude Code: ob `userConfig` dort funktioniert, ist ungeprüft. Falls ja, könnte das Repo für
      die Entwickler im Team zusätzlich ein `.mcp.json` mitliefern.

---

## Migrationspfad OAuth

Bekommt ESKS OAuth, ändert sich das Bild grundlegend: das Plugin kann eine URL **ohne Geheimnis**
mitliefern, der Connector wandert zurück ins Paket, das Onboarding schrumpft von vier Schritten auf
zwei. Kein Blocker für jetzt — aber der Grund, warum OAuth mehr ist als Kosmetik.

---

## Verwandte Repos

| Repo | Inhalt |
|---|---|
| `cormetis-skills-raw` | Arbeitsstand der Skills, Deployment-Spec, UX-Dokument, Spike-Beleg |
| `claude-plugins-internal` | Interner Marketplace für Entwickler-Plugins (`cor-dev-guide`) |

## Lizenz

Proprietär — COR Energy World GmbH.
