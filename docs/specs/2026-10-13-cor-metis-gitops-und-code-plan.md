# cor-metis-gitops und cor-metis-code — Umsetzungsplan

> **Historisch:** maßgeblich ist die Spec; die Präfixregel wurde nach dem Plan präzisiert (Spec §4).

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or
> superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for
> tracking.

**Goal:** Zwei neue Plugins im Marketplace `cor-metis-client` — CGSS-Skills für Desktop und Code, und ein
reines Claude-Code-Plugin, das ESKS und CGSS mit dem Key aus `userConfig` verbindet.

**Architecture:** Spec: [`2026-10-13-cor-metis-gitops-und-code.md`](2026-10-13-cor-metis-gitops-und-code.md).
Skills liegen genau einmal im Repo und sprechen Tools ohne festen Präfix an (Spec §4). Der Key lebt in Claude
Code nur in `userConfig` und geht als `Authorization: Bearer` raus.

**Tech Stack:** Claude-Plugin-Manifeste (JSON), Skills (Markdown mit YAML-Frontmatter), Claude Code 2.1.296
(`claude plugin validate`).

**Sprache:** Skills, README, ONBOARDING auf **Deutsch** wie der Rest des Repos; JSON-Schlüssel englisch.
Ton der Skills wie die bestehenden (`plugins/cor-metis/skills/*/SKILL.md`): direkt, begründend, mit
Gotchas-Abschnitt am Ende.

**Quellen, die jeder Task kennen muss:**
- CGSS-Tool-Oberfläche: `/home/chris/workspace/backend-monorepo/cormetis/cloud/ClaudeGitopsSupportService/README.md`
  (Abschnitte „Tools“, „Partial results“, „Errors“) und `…/Mcp/GitopsTools.cs` (die `[Description]`-Texte sind
  maßgeblich).
- Plugin-Doku: https://code.claude.com/docs/en/plugins/manifest-reference, https://code.claude.com/docs/en/mcp

**Tests:** Es gibt keinen Code. „Test“ heißt hier `claude plugin validate <pfad>` (muss fehlerfrei sein) und
`rg -n 'mcp__ESKS__|mcp__CGSS__' plugins` (Treffer nur in Beispielzeilen der Präfixregel). Der
Live-Test gegen central-1 ist Task 6 und läuft mit Chris.

---

## Dateien

| Datei | Task | Zweck |
|---|---|---|
| `plugins/cor-metis-code/.claude-plugin/plugin.json` | 1 | Manifest + `userConfig.api_key` |
| `plugins/cor-metis-code/.mcp.json` | 1 | Server `ESKS`, `CGSS` mit Bearer-Header |
| `plugins/cor-metis-code/README.md` | 1 | Was es ist, nur Claude Code |
| `.claude-plugin/marketplace.json` | 1, 2 | Task 1: Einträge für die neuen Plugins; Task 2: `cor-metis` → 0.3.0 |
| `plugins/cor-metis/skills/{orakel,burner-status,ingest}/SKILL.md` | 2 | Präfixregel |
| `plugins/cor-metis/skills/setup/SKILL.md` | 2 | Präfixregel, Claude-Code-Weg, CGSS-Check, Dubletten |
| `plugins/cor-metis/.claude-plugin/plugin.json`, `plugins/cor-metis/README.md` | 2 | 0.3.0, Setup-Zeile |
| `plugins/cor-metis-gitops/.claude-plugin/plugin.json` | 3 | Manifest |
| `plugins/cor-metis-gitops/README.md` | 3 | Skills-Tabelle, Voraussetzung |
| `plugins/cor-metis-gitops/skills/{deployment-status,deploy-and-wait,diagnose}/SKILL.md` | 3 | Die drei Skills |
| `README.md`, `docs/ONBOARDING.md` | 4 | Konfiguration je Oberfläche |
| `backend-monorepo/…/ClaudeGitopsSupportService/README.md` | 5 | „Connecting a client“ → Plugin |

Tasks 1, 2, 3 sind unabhängig voneinander, laufen aber im **selben Working Tree** auf
`feature/cor-metis-gitops`: jeder Task committet **nur seine eigenen Pfade** per `git add <pfad>` — nie
`git add -A` oder `commit -a`. `marketplace.json` fassen Task 1 (neue Einträge) und Task 2 (nur die
`cor-metis`-Version) an verschiedenen Zeilen an; Commits werden vom Hauptgespräch serialisiert. Task 4 nach 1–3. Task 5 unabhängig. Task 6
zuletzt.

---

### Task 1: Plugin `cor-metis-code`

**Files:** Create `plugins/cor-metis-code/.claude-plugin/plugin.json`, `plugins/cor-metis-code/.mcp.json`,
`plugins/cor-metis-code/README.md`; Modify `.claude-plugin/marketplace.json`.

- [ ] **Step 1: `plugin.json`**

```json
{
  "name": "cor-metis-code",
  "version": "0.1.0",
  "description": "Nur Claude Code: verbindet ESKS und CGSS mit deinem persönlichen COR.metis-Key. Enthält keine Skills.",
  "author": { "name": "COR Energy World GmbH", "url": "https://cor.energy" },
  "homepage": "https://github.com/burners-software/cormetis-client",
  "repository": "https://github.com/burners-software/cormetis-client",
  "license": "UNLICENSED",
  "keywords": ["cor", "metis", "esks", "cgss", "claude-code"],
  "userConfig": {
    "api_key": {
      "type": "string",
      "title": "COR.metis API-Key",
      "description": "Dein persönlicher COR.metis-Key (gilt für ESKS und CGSS). Nie weitergeben, nie in einen Chat tippen.",
      "sensitive": true,
      "required": true
    }
  }
}
```

- [ ] **Step 2: `.mcp.json`** — exakt wie Spec §3 (Server `ESKS` und `CGSS`, `type: http`, URL ohne Query,
  `headers.Authorization = "Bearer ${user_config.api_key}"`).

- [ ] **Step 3: `README.md`** (kurz): Zweck, „nur Claude Code — in Desktop nicht installieren, dort Connectoren
  von Hand (siehe Repo-README)“, Installation (`/plugin install cor-metis-code@cor-metis-client`, Key-Abfrage beim
  Aktivieren, ändern über `/plugin` → Configure options bzw. `claude plugin configure cor-metis-code`),
  Dubletten-Hinweis (synchronisierte `claude.ai ESKS`/`claude.ai CGSS` in `/mcp` deaktivieren), CGSS braucht
  Engineer-Rolle, sonst steht CGSS in `/mcp` auf „failed“ — das ist dann kein Defekt.

- [ ] **Step 4: `marketplace.json`** — neue Einträge `cor-metis-gitops` (0.1.0,
  `./plugins/cor-metis-gitops`, Beschreibung „COR.metis Gitops — Deployment-Status, Warten auf Rollouts und
  Diagnose über CGSS (für Entwickler)“) und `cor-metis-code` (0.1.0, `./plugins/cor-metis-code`, Beschreibung
  wie plugin.json).

- [ ] **Step 5: validieren** — `claude plugin validate plugins/cor-metis-code` → keine Fehler.
  `claude plugin validate .` (Marketplace) → rc 0; ein fehlendes `cor-metis-gitops`-Verzeichnis oder eine
  Versionswarnung für `cor-metis` gehört zu Task 2/3 und wird hier **nicht** angefasst.

- [ ] **Step 6: Commit** `Add cor-metis-code plugin for Claude Code`

---

### Task 2: `cor-metis`-Skills präfixneutral, `setup` erweitert

**Files:** Modify `plugins/cor-metis/skills/{orakel,burner-status,ingest,setup}/SKILL.md`,
`plugins/cor-metis/.claude-plugin/plugin.json` (0.3.0), `plugins/cor-metis/README.md`.

- [ ] **Step 1: Präfixregel** — jede Erwähnung von `mcp__ESKS__<tool>` durch „`<tool>` vom ESKS-Server“
  ersetzen. Wo ein Skill erklärt, wie man die Tools findet (orakel :8/:29, burner-status :19–20/:37), die Regel
  aus Spec §4 einsetzen: *Segment unmittelbar vor `__<tool>` ist `ESKS`, Groß-/Kleinschreibung egal; Beispiele
  `mcp__ESKS__search`, `mcp__claude_ai_ESKS__search`, `mcp__plugin_cor-metis-code_ESKS__search`; deferred → per
  ToolSearch-Stichwortsuche „ESKS search“ nachladen, nie `select:` mit festem Namen.* Fachlicher Inhalt der
  Skills bleibt unverändert.

- [ ] **Step 2: `setup` — Check 2** auf die Regel umstellen; die Tabelle „falsch benannter Server“ bleibt, aber
  `claude_ai_ESKS` und `plugin_cor-metis-code_ESKS` sind **richtig**. Neue Zeile: ESKS-Tools unter **zwei**
  Präfixen → Dublette, Hinweis (in Claude Code den synchronisierten `claude.ai ESKS` in `/mcp` deaktivieren),
  kein FAIL.

- [ ] **Step 3: `setup` — Check 6 „Deploy-Zugang (CGSS)“** nach Check 5, nur wenn CGSS-Tools da sind oder der
  Nutzer nach Deployments/Gitops/CGSS fragt. `list_environments` vom CGSS-Server aufrufen: Liste → PASS, Namen
  nennen. CGSS-Tools fehlen, obwohl eingerichtet, bzw. in Claude Code `/mcp` zeigt CGSS „failed“ → Key falsch
  **oder** keine Engineer-Rolle (DEVELOPER/OPERATOR/ADMIN/MASTER) — CGSS lehnt dann schon die Verbindung mit
  403 ab; Rolle bei Chris anfragen. Ergebnis-Tabelle um Zeile 6 erweitern; Description-Frontmatter um
  „Deploy-Zugang/CGSS prüfen“ ergänzen.

- [ ] **Step 4: `setup` — Teil B** in „B1 Claude Desktop / Cowork“ (bestehender Text, plus optional zweiter
  Connector `CGSS`, URL `https://internal.cor.energy/cgss/mcp?apikey=DEIN_KEY`, nur für Entwickler) und
  „B2 Claude Code“ aufteilen: `/plugin marketplace add burners-software/cormetis-client`,
  `/plugin install cor-metis@cor-metis-client` und `cor-metis-code@cor-metis-client` (Entwickler zusätzlich
  `cor-metis-gitops@cor-metis-client`), Key wird beim Aktivieren maskiert abgefragt — **nicht** in den Chat.
  Abschnitt „Was das Plugin bewusst nicht mitbringt“ entsprechend korrigieren (Desktop: kein Connector im
  Paket; Code: `cor-metis-code`). Abschnitt „Der Name ist keine Geschmacksfrage“ (heute :108) neu begründen:
  der Connector muss `ESKS` bzw. `CGSS` heißen, weil die Skills die Tools am Server-Segment erkennen — nicht
  mehr „weil die Skills `mcp__ESKS__*` ansprechen“. Ebenso :30/:37.

- [ ] **Step 5: Version + README** — `plugin.json` 0.3.0 und in `.claude-plugin/marketplace.json` nur die
  `version` des Eintrags `cor-metis` auf 0.3.0; `plugins/cor-metis/README.md` Abschnitt
  „Voraussetzung“ um Claude Code (`cor-metis-code`) ergänzen.

- [ ] **Step 6: prüfen** — `rg -n 'mcp__ESKS__' plugins/cor-metis` → nur Beispielzeilen der Regel;
  `claude plugin validate plugins/cor-metis` → fehlerfrei.

- [ ] **Step 7: Commit** `Address ESKS tools by server name, not prefix, and teach setup Claude Code and CGSS`

---

### Task 3: Plugin `cor-metis-gitops` mit drei Skills

**Files:** Create `plugins/cor-metis-gitops/.claude-plugin/plugin.json` (Schema wie `cor-metis`, 0.1.0,
Keywords `cor, metis, gitops, argocd, cgss`), `plugins/cor-metis-gitops/README.md`,
`plugins/cor-metis-gitops/skills/{deployment-status,deploy-and-wait,diagnose}/SKILL.md`.

Gemeinsam für alle drei Skills (je Skill ein kurzer Abschnitt, nicht ausgelagert):
- Frontmatter `name:` ohne Präfix (`deployment-status` → `cor-metis-gitops:deployment-status`); `description`
  triggert präzise und grenzt gegen die beiden anderen ab und gegen `cor-metis:burner-status` (Anlage ≠
  Deployment).
- Tools vom CGSS-Server, Präfixregel wie Spec §4 (Segment `CGSS`).
- Alles read-only; der Skill löst nie einen Sync aus und pusht nie.
- CGSS-Fehler kommen als Ergebnis `{"error": {"code", "message"}}`: Bedeutung je Code wie Spec §5 /
  CGSS-README „Errors“, `application_not_found` wörtlich als „gibt es nicht oder ist nicht freigegeben“.
  `errors`-Liste und `unhealthy_resources: null` heißen *unbekannt*, nicht *nichts*.
- CGSS-Tools fehlen ganz → auf `cor-metis:setup` verweisen (Key oder Engineer-Rolle), nicht herumprobieren.
- Environments nie raten: `list_environments`; heute ist nur central-1 angebunden, dev01/dev02 folgen.

- [ ] **Step 1: `deployment-status`** — Auslöser: „was läuft wo“, Version/Image einer App, healthy?, seit wann
  deployed, Umgebungsvergleich, „gibt es eine neuere Version in Nexus?“. Ablauf: `list_environments` →
  `list_applications` (`name_contains`) → `get_application_status` für Details; Vergleich = Status je
  Environment, Skill vergleicht Images/Revisionen (D8); `get_deploy_history` für „seit wann/was davor“;
  `search_images` für Nexus — ⚠ live beobachtet beim CGSS-E2E auf central-1 am 2026-10-13 (entgegen der
  Tool-Beschreibung): Namensanfang findet (`claudegitops`, `expertsystem`), Teilstring mitten im Namen nicht
  (`knowledge` → 0), weil CGSS Nexus per Keyword-`q` fragt. Darum mit dem vollen Image-Namen aus dem Status
  suchen; 0 Treffer bei einem Namensteil heißt nicht „gibt es nicht“. Antwortformat: kurze
  Tabelle (Environment | App | Image:Tag | Sync | Health | deployed vor), dann ein Satz Befund.

- [ ] **Step 2: `deploy-and-wait`** — exakt Spec §5 „Revision für `wait_for_revision`“ und die
  Fehlerbehandlung: Revision = GitOps-Commit; Claude Code: README + `.github/copilot-instructions.md` des
  GitOps-Repos lesen, gepusht prüfen, `git rev-parse HEAD`, geänderte Pfade → Applications (mehrere möglich);
  Desktop: nach SHA (≥ 7 Hex), Environment, App fragen. Schleife nur auf `pending`, `timeout_seconds` 25, nach
  jedem Aufruf knapp den `observed`-Stand nennen (nicht jede Runde ausführlich), Abbruch nach ~10 min;
  Abstammungs-Check (`git fetch` + `git merge-base --is-ancestor`) bzw. Rückfrage in Desktop; `OutOfSync` ohne
  laufende Operation über zwei Aufrufe → „vermutlich kein Auto-Sync“; jedes `error` beendet sofort;
  `reached` → ein Satz; `failed` → Skill `cor-metis-gitops:diagnose` aufrufen (unhealthy resources aus der
  Antwort als Einstieg mitgeben); der Ablauf wird nicht dupliziert.

- [ ] **Step 3: `diagnose`** — Auslöser: degraded, CrashLoop, startet neu, Sync failed, „warum geht X nicht“.
  Ablauf: `get_application_status` (Health, letzte Operation, Conditions, unhealthy resources) →
  `get_application_events` (ohne `resource_name` zuerst; Events leben ~1 h) → `get_application_logs`
  (bei Neustart `previous: true`; `filter` für Fehlerwörter; Grenzen 5 Pods/64 KiB) → `get_deploy_history`
  („was hat sich geändert?“, Image-Wechsel). `logs_not_released` = bewusste Freigabeentscheidung, so
  formulieren; dann mit Events/Status weiterarbeiten. Ergebnis: Ursache (oder wahrscheinlichste Ursachen,
  gerankt), Belege mit Zitat aus Events/Logs, nächster Schritt — und klar sagen, dass ein Fix (Rollback, Sync,
  Restart) nicht von hier aus geht. Log-Inhalte nicht ungefragt seitenweise ausgeben.

- [ ] **Step 4: `README.md`** des Plugins — Skills-Tabelle wie `plugins/cor-metis/README.md`, Voraussetzung
  (CGSS-Zugang: Desktop Connector `CGSS`, Code `cor-metis-code`; Engineer-Rolle), read-only-Hinweis.

- [ ] **Step 5: prüfen** — `claude plugin validate plugins/cor-metis-gitops` fehlerfrei;
  `rg -n 'mcp__CGSS__' plugins/cor-metis-gitops` → nur Beispielzeilen der Regel; jeder genannte Tool- und
  Parametername existiert in `GitopsTools.cs` (`rg -n 'Name = "<tool>"'`).

- [ ] **Step 6: Commit** `Add cor-metis-gitops plugin with deployment-status, deploy-and-wait and diagnose`

---

### Task 4: Repo-README und ONBOARDING

**Files:** Modify `README.md`, `docs/ONBOARDING.md`.

- [ ] **Step 1: `README.md`** — Abschnitt „Aufbau“ mit drei Plugins; neuer Abschnitt **„Was ist zu
  konfigurieren“** mit Tabelle je Oberfläche (Desktop/Cowork: Connector `ESKS`, für Entwickler `CGSS`, URLs aus
  Spec §3; Claude Code: Marketplace hinzufügen, drei Install-Befehle, Key-Abfrage, Ändern per
  `claude plugin configure cor-metis-code`, synchronisierte Connectoren in `/mcp` deaktivieren); Abschnitt
  „Kein `.mcp.json` — bewusst“ → „`.mcp.json` nur in `cor-metis-code`“ mit Begründung (Cowork ignoriert
  `${user_config}` ohne Default; Desktop-Verhalten unklar); Abschnitt Tool-Namen-Regel (Spec §4); „Eine Änderung
  ausliefern“: Version je Plugin an **beiden** Stellen; Satz „einzige bewusste Preisgabe“ korrigieren
  (Spec §6); offener Punkt „Claude Code: ob `userConfig` funktioniert“ bleibt offen (wird in Task 6 abgehakt).

- [ ] **Step 2: `docs/ONBOARDING.md`** — zwei kurze Abschnitte am Ende im Stil der Datei: „Für Entwickler:
  Deployments im Blick“ (Plugin `cor-metis-gitops`, Connector `CGSS`, Engineer-Rolle, Beispielfragen) und „Claude
  Code statt Desktop“ (drei Befehle, Key-Abfrage, kein Connector nötig, Dubletten abschalten).

- [ ] **Step 3: Commit** `Document configuration for Desktop and Claude Code`

---

### Task 5: CGSS-README im backend-monorepo

**Repo:** `/home/chris/workspace/backend-monorepo`, direkt auf `master` (von Chris im Gespräch so
freigegeben, abweichend von der ersten Spec-Fassung). Läuft im Hauptgespräch: `git switch master`,
`git pull --ff-only`; ist der Tree nicht sauber (außer dem bekannten `tsconfig.tsbuildinfo`), stoppen. Nur
diese Datei committen. Optional dabei die pauschale Access-Log-Warnung zu `?apikey=` an den Stand von Spec §3
anpassen.

**File:** Modify `cormetis/cloud/ClaudeGitopsSupportService/README.md`, Abschnitt „Connecting a client“
(:405–428).

- [ ] **Step 1:** „**Claude Code.**“-Block ersetzen: Plugin `cor-metis-code` aus dem Marketplace
  `burners-software/cormetis-client` (Befehle), Key per `userConfig` als Bearer-Header; Begründung in einem Satz,
  warum nicht `claude mcp add … --header` (Key in Shell-History und Klartext in `~/.claude.json`). Englisch wie
  das README.

- [ ] **Step 2: Commit** `Point Claude Code users at the cor-metis-code plugin`

---

### Task 6: Live-Test mit Chris, Merge, Push

Läuft im Hauptgespräch, nicht als Subagent — Chris gibt den Key ein, nie Claude.

- [ ] `claude plugin validate --strict .` und je Plugin → fehlerfrei.
- [ ] Chris: `/plugin marketplace add /home/chris/workspace/cormetis-client` (lokaler Pfad), dann
  `cor-metis-code` installieren, Key eingeben; `/mcp` zeigt ESKS + CGSS verbunden.
- [ ] Key-Leck-Probe (Spec §7.3): ein `claude --debug`-Lauf mit dem Plugin; danach führt Chris selbst
  `rg -F "<key>"` über `~/.claude/settings.json`, `~/.claude.json`, die Debug-Logs (`~/.claude/debug/`) und das
  Repo aus (Key nie an Claude). Erwartet: kein Treffer.
- [ ] `search_images`-Beobachtung (Präfix ja, Teilstring nein) noch einmal bestätigen.
- [ ] README: offenen Punkt „Claude Code: `userConfig`“ abhaken.
- [ ] Je Skill ein echter Aufruf gegen central-1 (Spec §7.4).
- [ ] Desktop-Test (Spec §7.5) durch Chris.
- [ ] Befunde fixen, Versionen prüfen, `feature/cor-metis-gitops` nach `master` mergen, `cormetis-client`
  pushen; `backend-monorepo` master pushen.
