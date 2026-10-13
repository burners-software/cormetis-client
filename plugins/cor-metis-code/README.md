# cor-metis-code

Verbindet in **Claude Code** die MCP-Server **ESKS** (Knowledgebase, Anlagenstatus) und **CGSS**
(Argo-CD-Lesezugang) mit deinem persönlichen COR.metis-Key. Das Plugin enthält keine Skills — die kommen
aus `cor-metis` und `cor-metis-gitops`.

## Nur Claude Code

Dieses Plugin ist ausschließlich für Claude Code gedacht. In Claude Desktop und Cowork **nicht
installieren**: Cowork ignoriert den Eintrag stillschweigend und fragt keinen Key ab; wie Claude Desktop (Chat) reagiert, ist nicht eindeutig dokumentiert. Für Desktop richtest du die
Connectoren von Hand ein, siehe [README im Repo-Root](../../README.md) und
[docs/ONBOARDING.md](../../docs/ONBOARDING.md).

## Installation

```
/plugin marketplace add burners-software/cormetis-client
/plugin install cor-metis-code@cor-metis-client
```

Beim Aktivieren fragt Claude Code einmal nach dem API-Key (maskiert). Er liegt danach im Credential-Store von
Claude Code — nicht in `settings.json`, nicht in einer URL und nicht in diesem Repo. Unter Linux ohne Keychain
ist das eine Datei mit restriktiven Rechten, kein Tresor.

Key später ändern: `/plugin` → `cor-metis-code` → *Configure options*, oder

```
claude plugin configure cor-metis-code
```

Den Key gibt es nur persönlich (`DEIN_KEY` ist immer nur ein Platzhalter). Nie weitergeben und nie in einen
Chat tippen.

## Dubletten abschalten

Wer Desktop-Connectoren mit demselben claude.ai-Konto nutzt, bekommt in Claude Code automatisch die
synchronisierten Server `claude.ai ESKS` und `claude.ai CGSS`. Claude Code erkennt die Dublette und
unterdrückt eine der beiden Seiten — welche, ist nicht festgelegt; im Test verlor in Headless-Läufen
(`claude -p`) das Plugin. Der synchronisierte Weg trägt den Key in der URL. Deaktiviere die beiden in
`/mcp`, dann nimmt Claude Code sicher das Plugin.

## CGSS braucht eine Engineer-Rolle

CGSS verlangt eine Engineer-Rolle (DEVELOPER, OPERATOR, ADMIN oder MASTER). Ohne sie lehnt CGSS schon den
Verbindungsaufbau mit 403 ab und steht in `/mcp` auf „failed". Das ist dann kein Defekt; ESKS läuft trotzdem.

## Lizenz

Proprietär — COR Energy World GmbH.
