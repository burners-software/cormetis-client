# cor-metis-gitops und cor-metis-code

Stand 2026-10-13. Entschieden mit Chris im Gespräch, Variante (A).

## 1. Ziel

- Die Deploy-Skills für CGSS (Claude GitOps Support Service) werden im selben Marketplace verteilt wie die
  COR.metis-Skills. Sie funktionieren in **Claude Desktop und Claude Code**.
- In **Claude Code** wird der API-Key nicht mehr von Hand in eine Connector-URL kopiert, sondern einmal über
  die Plugin-Konfiguration maskiert abgefragt und im Credential-Store von Claude Code abgelegt — nicht in
  `settings.json`, nicht in einer URL, nicht im Repo. (Auf Linux ohne Keychain ist das eine Datei mit
  restriktiven Rechten, kein Tresor.)
- Jeder Skill liegt genau einmal im Repo.

Nicht-Ziele: kein Schreibzugriff auf Argo CD (CGSS ist read-only), kein Push durch den Skill, keine Änderung
an CGSS oder ESKS, kein OAuth.

## 2. Schnitt: drei Plugins

| Plugin | Inhalt | Desktop/Cowork | Claude Code | Wer installiert |
|---|---|---|---|---|
| `cor-metis` | Skills `orakel`, `burner-status`, `ingest`, `setup` | ja | ja | alle |
| `cor-metis-gitops` (neu) | Skills `deployment-status`, `deploy-and-wait`, `diagnose` | ja | ja | Entwickler (DEVELOPER/OPERATOR) |
| `cor-metis-code` (neu) | **keine Skills**, nur `.mcp.json` für ESKS und CGSS plus `userConfig` für den Key | nein | ja | Entwickler in Claude Code |

`cor-metis-code` bringt CGSS mit. Ohne Engineer-Rolle lehnt CGSS schon den Verbindungsaufbau mit 403 ab und
steht in `/mcp` als „failed“ — für Claude-Code-Nutzer, die ja Entwickler sind, ist das der Ausnahmefall und
wird in `setup` erklärt, statt ein viertes Plugin einzuführen. Beschreibung von Plugin und
Marketplace-Eintrag sagen ausdrücklich „nur Claude Code“, weil der Eintrag auch im Desktop-Marketplace
erscheint und Cowork ihn dort stillschweigend ignoriert.

Es gibt **einen** `setup`-Skill (`cor-metis:setup`); er prüft CGSS als optionalen Check mit, wenn
CGSS-Tools da sind oder der Nutzer danach fragt. Zwei `setup`-Skills hätten überlappende Auslöser.

`cor-metis-code` getrennt zu halten hat einen Grund: Cowork ignoriert MCP-Server, die `${user_config.*}`
ohne Default referenzieren, und fragt nichts ab; wie Claude Desktop (Chat) auf einen `${user_config.*}`-Header
reagiert, ist nicht eindeutig dokumentiert. Die Skill-Plugins bleiben dadurch frei von jeder Konfiguration
und laufen überall gleich.

## 3. Zugang je Oberfläche

**Claude Desktop / Cowork** (wie bisher, ein zweiter Connector):

| Connector-Name | URL |
|---|---|
| `ESKS` | `https://internal.cor.energy/esks/mcp?apikey=DEIN_KEY` |
| `CGSS` | `https://internal.cor.energy/cgss/mcp?apikey=DEIN_KEY` |

Derselbe persönliche Key für beide. CGSS verlangt zusätzlich eine Engineer-Rolle (DEVELOPER, OPERATOR,
ADMIN, MASTER); ohne sie antwortet CGSS mit 403.

Voraussetzung für `?apikey=` auf `/cgss`: der Query-String wird nicht geloggt. Laut Felix (2026-10-13)
protokolliert Envoy für alle Pfade `%PATH(NQ)%`, nginx loggt keine Query-Parameter mehr, und das
cormetis-frontend-nginx hat sein Access-Log abgeschaltet. Offen bleibt wie bei ESKS: APM/Tracing könnte die
volle URL anhängen.

**Claude Code** (`cor-metis-code`):

`plugin.json`:

```json
"userConfig": {
  "api_key": {
    "type": "string",
    "title": "COR.metis API-Key",
    "description": "Dein persönlicher COR.metis-Key (gilt für ESKS und CGSS).",
    "sensitive": true,
    "required": true
  }
}
```

`.mcp.json`:

```json
{
  "mcpServers": {
    "ESKS": {
      "type": "http",
      "url": "https://internal.cor.energy/esks/mcp",
      "headers": { "Authorization": "Bearer ${user_config.api_key}" }
    },
    "CGSS": {
      "type": "http",
      "url": "https://internal.cor.energy/cgss/mcp",
      "headers": { "Authorization": "Bearer ${user_config.api_key}" }
    }
  }
}
```

Der Key steht damit nicht in einer URL. Beide Dienste nehmen `Authorization: Bearer` an
(`UserApiKeyEndpointFilter` in `Burners.Libraries.Mcp.Common`). Kein Default für `api_key` — ein Default
wäre ein geteilter Key.

## 4. Tool-Namen

Derselbe Server erscheint je nach Oberfläche unter verschiedenen Präfixen:

| Oberfläche | ESKS-Tool `get_stats` heißt |
|---|---|
| Claude Desktop / Cowork, Connector `ESKS` | `mcp__ESKS__get_stats` |
| Claude Code, claude.ai-Connector synchronisiert | `mcp__claude_ai_ESKS__get_stats` |
| Claude Code, Plugin `cor-metis-code` | `mcp__plugin_cor-metis-code_ESKS__get_stats` |

Dazu kommt ein manuell per `claude mcp add` angelegter Server, dessen Name frei wählbar ist (das CGSS-README
schlägt bisher klein `cgss` vor).

Die Skills nennen deshalb **keinen festen Präfix** mehr, sondern den Server und den nackten Tool-Namen
(„`get_stats` vom ESKS-Server“) mit der Regel: *der Teil zwischen `mcp__` und `__<tool>` ist `ESKS` bzw.
`CGSS` oder endet auf `_ESKS` bzw. `_CGSS`, Groß-/Kleinschreibung egal*. Nachgeladene („deferred“) Tools werden per ToolSearch-Stichwortsuche
gefunden („ESKS get_stats“), nie per `select:` mit festem Namen. Das gilt für die vier bestehenden Skills
ebenso wie für die neuen.

**Dubletten sind der Normalfall:** Wer Desktop-Connectoren hat und Claude Code mit demselben claude.ai-Konto
nutzt, bekommt die synchronisierten Connectoren automatisch. Mit `cor-metis-code` stehen dann rund 45 Tools
doppelt im Kontext, und der synchronisierte Weg trägt den Key in der URL. Die README weist an, in Claude Code
die synchronisierten Server `claude.ai ESKS` / `claude.ai CGSS` in `/mcp` zu deaktivieren; `setup` meldet
Dubletten als Hinweis.

## 5. Die Skills in `cor-metis-gitops`

Alle drei sprechen nur CGSS an, alle Tools sind read-only. CGSS-Fehler kommen als Ergebnis
(`{"error": {"code": …}}`), nicht als Tool-Fehler; die Skills lesen den `code`.

| Skill | Auslöser | Ablauf |
|---|---|---|
| `deployment-status` | „was läuft auf dev01?“, „welche Version hat ESKS auf central?“, „unterscheiden sich dev01 und dev02?“ | `list_environments` → `list_applications` (mit `name_contains`) bzw. `get_application_status`; Vergleich = zwei Status-Abfragen, der Skill vergleicht (CGSS-Entscheidung D8). Optional `get_deploy_history`, `search_images` („gibt es eine neuere Version?“). |
| `deploy-and-wait` | „ich hab gepusht, sag Bescheid wenn's läuft“, „warte bis 1.0.2 auf central ist“ | Revision bestimmen (s.u.) → `wait_for_revision` in Schleife **nur solange `state: pending`** → bei `reached` kurze Erfolgsmeldung, bei `failed` direkt in den Ablauf von `diagnose`. Jedes `error` beendet die Schleife sofort und wird gemeldet. `pending` mit `last_error` nennt diesen Fehler mit. Abbruch nach ca. 10 Minuten `pending` mit dem letzten `observed`-Stand. |
| `diagnose` | „warum ist X degraded?“, „warum startet Y neu?“ | `get_application_status` → `get_application_events` → `get_application_logs` (bei Neustarts `previous: true`) → ggf. `get_deploy_history`. `logs_not_released` ist eine Freigabeentscheidung, kein Fehler. |

**Revision für `wait_for_revision`** ist der Commit im **GitOps-Repo** (z.B. argocd), nicht der Code-Commit
und nicht der Image-Tag.

- *Claude Code:* der Skill liest zuerst README (und `.github/copilot-instructions.md`, falls vorhanden) des
  GitOps-Repos, prüft, dass `HEAD` gepusht ist (`git status -sb` / `git branch -r --contains HEAD`), nimmt
  `git rev-parse HEAD` und bestimmt aus den geänderten Pfaden die betroffenen Applications. Ein Commit kann
  mehrere Applications oder Environments treffen — dann wird je App gewartet. Pushen tut der Nutzer selbst.
- *Desktop:* kein Git — der Skill fragt nach Commit-SHA (mind. 7 Zeichen), Environment und Application.

**`wait_for_revision` vergleicht exakt (Präfix), nicht nach Abstammung.** Argo zeigt den aufgelösten HEAD des
Ziel-Branches; landet nach dem Push ein weiterer Commit, wird der angefragte SHA nie `reached`. Der Skill
vergleicht deshalb bei `pending` die beobachtete Revision mit der angefragten:

- Claude Code: `git fetch` und `git merge-base --is-ancestor <angefragt> <beobachtet>` — ist der angefragte
  Commit enthalten, auf die beobachtete Revision weiterwarten und das sagen.
- Desktop: dem Nutzer die abweichende Revision melden und fragen.
- `OutOfSync` ohne laufende Operation über zwei Aufrufe hinweg → als „Argo synct das vermutlich nicht
  automatisch“ melden, statt bis zum Timeout zu warten (CGSS kann keinen Sync auslösen).

**Fehlercodes, die die Skills unterscheiden müssen:** `application_not_found` (gibt es nicht *oder* nicht
freigegeben — bewusst ununterscheidbar), `logs_not_released`, `upstream_rejected` (Token abgelaufen →
Plattform-Betreiber), `upstream_unavailable` (vorübergehend; in `wait_for_revision` kein Fehler, sondern
`pending` mit `last_error`), `unknown_environment` (Meldung nennt die gültigen Namen), `invalid_argument`,
`internal_error`.

Ein 403 wegen fehlender Rolle kommt **nicht** als Ergebnis, sondern als HTTP-Ablehnung beim Verbinden — die
CGSS-Tools fehlen dann ganz bzw. der Server steht in `/mcp` auf „failed“. Teilergebnisse über `errors` heißen *unbekannt*, nicht *nichts*.

## 6. Änderungen an Bestehendem

- `cor-metis`-Skills: feste `mcp__ESKS__`-Präfixe durch die Regel aus §4 ersetzen. `setup` bekommt den
  Claude-Code-Weg (`cor-metis-code`) als zweite Einrichtungsvariante, einen optionalen CGSS-Check
  (`list_environments`; fehlende Tools/„failed“ = Key oder Engineer-Rolle) und den Dubletten-Hinweis.
  Version 0.2.0 → 0.3.0.
- `marketplace.json`: drei Einträge; neue Plugins starten mit 0.1.0.
- `README.md`: neuer Aufbau, „Was ist zu konfigurieren“ je Oberfläche, Tool-Namen-Regel, Installation in
  Claude Code (`/plugin marketplace add …`, `/plugin install …@cor-metis-client`). Der Abschnitt
  „Kein `.mcp.json` — bewusst“ wird für Claude Code korrigiert, ebenso „einzige bewusste Preisgabe ist der
  Hostname“ (öffentlich werden zusätzlich die Pfade `/esks/mcp`, `/cgss/mcp` und dass es einen
  Argo-Lesezugang gibt — mit Chris abgestimmt, nicht geheimer als die ESKS-Skills).
- Folgeänderung außerhalb dieses Repos: Abschnitt „Connecting a client“ im CGSS-README
  (`backend-monorepo`) auf das Plugin verweisen statt auf `claude mcp add … --header "Bearer <key>"` (Key in
  Shell-History und Klartext in `~/.claude.json`). Direkt auf master, Push durch Claude (Chris,
  2026-10-13).
- `docs/ONBOARDING.md`: kurzer Abschnitt „Für Entwickler: Deployments“ (CGSS-Connector) und
  „Claude Code“.

## 7. Test (heute, vor dem Push)

1. `claude plugin validate` auf alle drei Plugins.
2. Claude Code: Marketplace lokal hinzufügen, `cor-metis-code` installieren, Key eingeben, `/mcp` zeigt
   ESKS und CGSS verbunden.
3. **Key-Leck-Probe:** den Key mit `rg -F` in `~/.claude/settings.json`, `~/.claude.json`, den Debug-Logs
   eines `claude --debug`-Laufs und im Repo suchen. Erwartet: kein Treffer.
4. Je Skill ein echter Aufruf gegen central-1: `cor-metis:setup` (inkl. CGSS-Check),
   `deployment-status` („welche Version läuft von ESKS?“), `diagnose` an einer gesunden App,
   `deploy-and-wait` mit der aktuell laufenden Revision (muss sofort `reached` liefern).
5. Desktop: Connector `CGSS` anlegen, `cor-metis:setup` und eine Statusfrage.

Ein Key ohne Engineer-Rolle wird nicht eigens getestet (keiner zur Hand); das Verhalten ist durch die
CGSS-Routentests belegt, `setup` beschreibt es.

Erst danach Merge nach master und Push (von Chris freigegeben).
