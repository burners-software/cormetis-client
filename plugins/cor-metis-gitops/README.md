# cor-metis-gitops

Deployment-Status, Rollout-Beobachtung und Diagnose für die Dienste der COR Energy World GmbH —
über Argo CD und Nexus, read-only. Für Entwicklerinnen und Entwickler, in Claude Desktop, Cowork und
Claude Code.

## Was drin ist

| Skill | Aufruf | Zweck |
|---|---|---|
| `deployment-status` | `cor-metis-gitops:deployment-status` | Was läuft wo: Version/Image, Sync, Health, seit wann, Umgebungsvergleich, neuere Images in Nexus |
| `deploy-and-wait` | `cor-metis-gitops:deploy-and-wait` | Wartet, bis ein gepushter GitOps-Commit ausgerollt ist, und übergibt bei Fehlschlag an `diagnose` |
| `diagnose` | `cor-metis-gitops:diagnose` | Warum ist eine App degraded, startet neu oder scheitert beim Sync — Ursache, Belege, nächster Schritt |

Die Skills triggern von selbst. Man muss sie nicht aufrufen, sondern einfach fragen:
*„Welche Version hat ESKS auf central?“*, *„Ich hab gepusht, sag Bescheid, wenn’s läuft.“*,
*„Warum startet CGSS dauernd neu?“*

## Voraussetzung

Die Skills lesen über den MCP-Server **CGSS** (Claude GitOps Support Service). Das Plugin bringt
**keinen** Zugang mit:

| Oberfläche | Zugang |
|---|---|
| Claude Desktop / Cowork | Connector mit dem Namen `CGSS` und der URL `https://internal.cor.energy/cgss/mcp?apikey=DEIN_KEY` |
| Claude Code | Plugin `cor-metis-code` aus diesem Marketplace (fragt den Key einmal maskiert ab) |

Der Key ist derselbe persönliche COR.metis-Key wie für ESKS. CGSS verlangt zusätzlich eine
**Engineer-Rolle** (DEVELOPER, OPERATOR, ADMIN oder MASTER). Ohne sie lehnt CGSS schon den
Verbindungsaufbau ab — die Tools fehlen dann, in Claude Code steht CGSS in `/mcp` auf „failed“.

Installation in Claude Code (Marketplace `cor-metis-client`):

```
/plugin marketplace add burners-software/cormetis-client
/plugin install cor-metis-gitops@cor-metis-client
```

Der Connector muss `CGSS` heißen: Die Skills erkennen die Tools am Servernamen im Tool-Namen: Der Teil zwischen `mcp__` und
`__<tool>` ist `CGSS` oder endet auf `_CGSS`, Groß-/Kleinschreibung egal.

Anleitung: **[docs/ONBOARDING.md](../../docs/ONBOARDING.md)** im Repo-Root.
Oder direkt in Claude: *„COR.metis einrichten“* (Skill `cor-metis:setup` prüft auch den CGSS-Zugang).

## Read-only

Kein Skill löst einen Sync, Rollback oder Restart aus, und keiner pusht. CGSS kann das auch gar
nicht: Jeder Zugriff auf Argo CD und Nexus ist ein lesender. Änderungen laufen wie bisher über das
GitOps-Repo und Argo CD — durch dich.

Angebunden ist heute `central-1`; dev01 und dev02 folgen. Produktion ist nicht lesbar.

## Lizenz

Proprietär — COR Energy World GmbH.
