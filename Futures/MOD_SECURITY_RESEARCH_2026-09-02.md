# Bismut Mod Security Research — 2026-09-02

Status: `RESEARCH BASELINE / PHASE 1`

Scope: Sicherheitsmodell für Vintage-Story-Mods und Bismut-Launcher. Fokus auf reale Mod-Malware-/Sabotagefälle, Supply-Chain-Risiken, Schwachstellen in Mods, Grenzen klassischer AV-/Reputation-Scans sowie daraus ableitbare lokale Scanner- und Revocation-Mechanismen.

---

## 1. Executive Decision

Bismut darf Mod-Sicherheit nicht als einzelnen VirusTotal-/Antivirus-Check modellieren.

Die bisherige Evidenz zeigt drei unterschiedliche Bedrohungsklassen:

```text
A. System Malware
   Mod/Loader führt Schadcode gegen Betriebssystem/Nutzerkonten aus

B. Intentional In-Game Sabotage
   Mod manipuliert absichtlich Spiel-/Runtime-Zustand, ohne klassische OS-Malware zu sein

C. Vulnerable Mod / Exploitable Runtime
   Mod ist nicht absichtlich malicious, eröffnet aber RCE-/Deserialisierungs-/Netzwerk-Angriffsflächen
```

Ein klassischer AV-Scanner ist primär für Klasse A geeignet. Bismut braucht zusätzlich semantische .NET-/IL-Analyse, Provenance, Source↔Binary-Vergleich, Revocation und perspektivisch isolierte dynamische Analyse.

---

## 2. Reale Fallklasse A — Fractureiser / Minecraft

### Was passiert ist

2023 wurden mehrere Minecraft-Mod-/Plugin-Artefakte mit `fractureiser` infiziert und über bekannte Mod-Plattformen verteilt. Stage-0-Code wurde in Mod-JARs eingebettet und lud weitere Stufen nach.

Dokumentierte spätere Fähigkeiten umfassten unter anderem:

- weitere Payloads herunterladen/ausführen
- Persistenz
- Browser-Cookies/Login-Daten stehlen
- Discord-Credentials stehlen
- Microsoft-/Minecraft-Credentials stehlen
- Crypto-Adressen in der Zwischenablage ersetzen
- weitere JAR-Dateien infizieren

Quellen:

- https://github.com/trigram-mrp/fractureiser
- https://github.com/trigram-mrp/fractureiser/blob/main/docs/users.md
- https://github.com/trigram-mrp/fractureiser/blob/main/docs/tech.md

### Relevanz für Bismut

Der Fall zeigt:

1. Plattform-Reputation schützt nicht vor kompromittierten Uploads oder Accounts.
2. Neuartige, ökosystemspezifische Malware kann klassische AV-Erkennung zunächst umgehen.
3. Einmal installierte Mods müssen nachträglich gegen neue Revocation-/IOC-Daten geprüft werden können.
4. Bytecode-/IL-Struktur muss analysiert werden, nicht nur Dateiname oder Strings.
5. Staged Payloads verlangen Netzwerk-, Process-, Persistence- und Dynamic-Load-Detektion.

---

## 3. Reale Fallklasse B — Vintage Story ConfigLib

### Aktuelle öffentlich belegte Evidence

ConfigLib ist ein realer Vintage-Story-Fall von absichtlich disruptivem Mod-Code.

Auf der ModDB wurde für ConfigLib 1.12.0 detailliert berichtet, dass Code:

- über einen `[ModuleInitializer]` automatisch aktiv wurde,
- nur in Multiplayer-Client-Kontexten aktivierte,
- geladene Assemblies enumerierte,
- Assemblies über CRC32-/Heuristik-Checks fingerprintete,
- obfuskierten Code anhand kurzer Typ-/Membernamen heuristisch bewertete,
- verzögert/pseudozufällig aktivierte,
- anschließend fremden internen Vintage-Story-Clientzustand manipulierte und dadurch Crashes, Disconnects, Netzwerk-/UI-Probleme und andere Störungen verursachen konnte.

ModDB:
- https://mods.vintagestory.at/configlib

Öffentliche Diskussion / aktueller Untersuchungsstatus:
- https://www.vintagestory.at/forums/topic/22395-fire-maltiez/

### Wichtige Einordnung

Dieser Fall ist gerade deshalb wichtig, weil er nicht zwingend wie klassische OS-Malware aussieht.

Ein AV-Provider kann `clean` melden, obwohl ein Mod absichtlich andere Mods/Clients sabotiert. Für Bismut muss deshalb gelten:

```text
AV clean != behavior safe
```

### Golden-Fixture-Kandidat

ConfigLib 1.12.0 soll als Bismut-Golden-Fixture verwendet werden, sobald das Originalartefakt lokal archiviert, gehasht und statisch unabhängig reproduziert wurde.

Aktueller Research-Stand:

- ModDB liefert den Original-Release und dessen CDN-URL über die API.
- In dieser Research-Session konnte das Binärartefakt aufgrund der Tool-/CDN-Downloadgrenze nicht lokal heruntergeladen werden.
- Daher ist die binäre Golden-Fixture noch `OPEN` und darf nicht als abgeschlossen gelten.

Erforderliche nächste Evidence:

```text
original zip
sha256
archive manifest
assembly hashes
IL inventory
ModuleInitializer confirmation
source↔binary diff
static findings
expected verdict
```

---

## 4. Vintage-Story-spezifische Angriffsfläche — Server-Mod-Downloads

Vintage Story unterstützt serverbezogene Mod-Downloads und speichert solche Mods unter serverbezogenen Modpfaden.

Ein öffentliches Vintage-Story-Issue beschreibt einen Fall, in dem ehemals serverseitig benötigte, später entfernte Mods clientseitig als verwaiste Mods weiter geladen werden konnten. Der Reporter weist ausdrücklich darauf hin, dass dies im Worst Case auch ein zuvor ausgeliefertes malicious Artefakt betreffen könnte.

Quelle:
- https://github.com/anegostudios/VintageStory-Issues/issues/7273

### Konsequenz für Bismut

Server-Join muss eine eigene Security Boundary sein:

```text
Server requests mods
      ↓
Resolve exact versions
      ↓
Download to quarantine
      ↓
Security scan
      ↓
Create isolated server profile
      ↓
Allow join
```

Zusätzlich:

- servergebundene Mods dürfen nicht global in eine normale Instanz auslaufen
- verwaiste Server-Mods müssen erkannt werden
- Server-Modset-Diff muss vor jedem Join möglich sein
- entfernte/revoked Mods müssen deaktiviert bzw. quarantänisiert werden

---

## 5. ModDB-v2 liefert bereits Revocation-Signale

Der öffentliche `vsmoddb`-Source dokumentiert für API v2:

```text
/api/v2/mods/install-information
/api/v2/mods/{modid}/releases
/api/v2/mods/{modid}/releases/{releaseid}
```

Retracted Releases liefern `retractionReason`. Ohne `ignore-retraction` werden Downloadinformationen für zurückgezogene Releases standardmäßig nicht ausgegeben.

Quelle:
- https://github.com/anegostudios/vsmoddb/blob/master/README.md

### Bismut-Feature: Revocation Watch

Bismut kann daraus einen echten lokalen Security-Mechanismus machen:

```text
Installed Lockfile
      ↓
Periodic / launch-time ModDB revocation check
      ↓
release still valid?
├── yes → unchanged
└── retracted / force-retracted
      ↓
security state = REVOKED_REMOTE
      ↓
block update / warn / quarantine according to policy
```

Wichtig: Eine Retraction ist nicht automatisch Malware. Die Ursache muss getrennt gespeichert werden.

---

## 6. Existierendes Vintage-Story-Pattern — FairPlayGuardian

FairPlayGuardian zeigt, dass SHA-256-/Integrity-basierte Modprüfungen im VS-Ökosystem bereits praktisch eingesetzt werden. Die Mod beschreibt unter anderem:

- Allow-/Block-Modlisten
- SHA-256-Integritätsprüfung
- Erkennung von Harmony-Patches

Quelle:
- https://mods.vintagestory.at/show/mod/6366

### Relevanz

Bismut sollte diese Mechanismen nicht kopieren, aber die Capability bestätigen:

```text
exact artifact identity = SHA-256
runtime patch capability = security-relevant metadata
```

---

## 7. Plattformvergleich — CurseForge

CurseForge beschreibt seit dem Fractureiser-Vorfall einen mehrstufigen Mod-Review-Prozess:

```text
1. JAR decompilation
2. class hashes + cache / known-malicious matching
3. static analysis
4. deeper analysis for suspicious classes
5. reject or manual review
```

Quelle:
- https://blog.curseforge.com/our-mod-approval-process/

### Bismut-Übertragung auf .NET

```text
ZIP
 ↓
Assembly extraction
 ↓
ECMA-335 / CIL parser
 ↓
Type/method hash inventory
 ↓
known signatures / known-good cache
 ↓
static capability + dataflow rules
 ↓
contextual deep analysis
 ↓
verdict
```

Empfohlene .NET-Basis:

- Mono.Cecil
- https://github.com/jbevain/cecil

Mono.Cecil kann .NET-Binaries analysieren, ohne sie per Reflection in den Prozess zu laden. Das ist für einen sicheren lokalen Preflight deutlich geeigneter als unbekannte Mod-Assemblies direkt zu laden.

---

## 8. Plattformvergleich — Modrinth

Modrinth bestätigt, dass eigene Malware-Scans Projekte markieren und anschließend technische manuelle Reviews auslösen können.

Quelle:
- https://support.modrinth.com/en/articles/8793355-project-review-times

Ein öffentlich dokumentierter Malware-Fall war 2024 der Mod `Windows Borderless`. Nach mehreren malicious Releases wurde der Code dekompiliert, die Projekte entfernt und eine Malware-Hash-/Quarantine-API als Ziel angekündigt.

Quelle:
- https://modrinth.com/news/article/windows-borderless-malware-disclosure/

### Relevanz

Besonders wertvoll für Bismut ist das Muster:

```text
known malicious artifact database
        +
launcher-side hash query
```

Damit kann eine Datei nach ihrer Installation noch nachträglich widerrufen werden.

---

## 9. MMPA / spezialisierte Mod-Malware-Scanner

Die Minecraft Malware Prevention Alliance pflegt spezialisierte Scanner.

Beispiel `Concoction`:
- statische und dynamische Analyse
- wiederverwendbare Scan-Models
- CLI/Library-Betrieb
- archive-aware scanning

Quelle:
- https://github.com/Minecraft-Malware-Prevention-Alliance/concoction

### Pattern für Bismut

Bismut sollte Rule Packs/Scan Models versionieren können:

```text
security-rules/
├── generic-dotnet/
├── vintagestory/
├── known-malware/
├── supply-chain/
└── compatibility/
```

Scanner-Engine und Regeln bleiben getrennt.

Damit können neue IOCs/Verhaltensregeln aktualisiert werden, ohne den gesamten Launcher neu zu bauen.

---

## 10. Warum VirusTotal allein nicht reicht

### Geeignet für

- bekannte Hashes
- bekannte Malwarefamilien
- klassische PE/native Payloads
- zusätzliche Reputation

### Nicht ausreichend für

- erstmalige/unikale Mod-Malware
- absichtliche In-Game-Sabotage
- legitime APIs mit malicious Intent
- Source↔Binary-Drift
- unsichere Deserialisierung / verwundbare Mods
- mod-spezifische Engine-Manipulation

Darum bleibt VirusTotal ein Provider, keine Verdict Authority.

---

## 11. Bismut Static Analyzer — P0 Rule Families

### P0-A: Automatic execution

Detect:

- `[ModuleInitializer]`
- static constructors with significant behavior
- ModSystem lifecycle hooks with hidden side effects
- AssemblyResolve / AssemblyLoad handlers

### P0-B: Process / Shell

Detect:

- `System.Diagnostics.Process.Start`
- cmd / powershell / shell execution
- script extraction + execution

### P0-C: Native boundary

Detect:

- `DllImport`
- `NativeLibrary.Load`
- unmanaged delegates/function pointers
- bundled PE/ELF/Mach-O files

### P0-D: Network

Detect:

- HttpClient / WebClient
- sockets
- custom DNS
- hardcoded IP/URL/domain
- download→write→load/execute chains

### P0-E: Filesystem / persistence

Detect:

- writes outside instance/DataPath
- browser/profile directories
- `%APPDATA%`, `%LOCALAPPDATA%`, startup folders
- registry writes
- scheduled tasks/services

### P0-F: Runtime manipulation

Detect:

- Harmony usage
- Reflection against internal Vintage Story types
- dynamic method/assembly generation
- unsafe pointer/memory APIs
- modification of foreign mod/runtime state

### P0-G: Obfuscation / anti-analysis

Detect as heuristic only:

- high ratio of tiny/random identifiers
- encoded/encrypted strings
- runtime decryption loops
- reflection-based method resolution
- exception swallowing around sensitive operations

Obfuscation alone must never result in `MALICIOUS_CONFIRMED`.

### P0-H: Supply-chain drift

Detect:

- public source declared but release contains extra types/methods
- embedded dependency differs from known upstream
- release includes ignored/untracked security-relevant source
- unexplained native payloads
- modinfo/source metadata mismatch

---

## 12. Source → Sink / Dataflow statt Keyword-Scanner

Bismut sollte nicht nur API-Aufrufe zählen.

Beispiel:

```text
HttpClient.GetByteArrayAsync(url)
        ↓
File.WriteAllBytes(path)
        ↓
Assembly.LoadFrom(path)
```

ist erheblich riskanter als:

```text
HttpClient.GetStringAsync(ModDBApi)
        ↓
JSON parse
        ↓
UI update
```

Daher braucht die spätere Engine mindestens begrenzte interprozedurale Source→Sink-Regeln.

---

## 13. Verdict und Confidence getrennt halten

Vorschlag:

```text
verdict:
  TRUSTED
  LOW_RISK
  WARN
  MANUAL_REVIEW
  BLOCKED
  MALICIOUS_CONFIRMED
  UNKNOWN

confidence:
  LOW
  MEDIUM
  HIGH
```

Zusätzlich:

```text
reasonCodes[]
evidence[]
ruleVersion
scannerVersion
artifactSha256
scanTimestamp
```

Damit ist jeder Status reproduzierbar und später neu berechenbar.

---

## 14. Kein globales "Author Trust"

Ein Autor darf Reputation als Signal erhalten, aber niemals als Security-Bypass.

Fractureiser zeigte kompromittierte Accounts; Modrinths Windows-Borderless-Fall zeigte, dass ein zuvor akzeptiertes Projekt später malicious Releases erhalten kann.

Regel:

```text
trust artifact, not author
```

Jede Version wird unabhängig an ihren SHA-256 gebunden.

---

## 15. Dynamic Analysis — FUTURE, aber eigener Sicherheitsbereich

Statische Analyse reicht nicht gegen:

- verschlüsselte Runtime-Strings
- zeitverzögerte Payloads
- environment-triggered behavior
- server-triggered behavior
- Reflection-/Dynamic-Code-Ketten

Perspektivische Bismut Sandbox:

```text
Disposable VM / Sandbox
├── isolated fake VS profile
├── fake credentials / honey files
├── network capture
├── process monitoring
├── filesystem diff
├── registry diff
├── child-process capture
└── timeout / destroy
```

Wichtig:

- niemals verdächtigen Modcode im normalen Launcher-Prozess laden
- Docker allein ist für Windows-/Vintage-Story-Verhalten nicht automatisch ausreichend
- für echte VS-Clientanalyse ist eine disposable Windows-VM/Sandbox wahrscheinlich realistischer

---

## 16. Recommended Architecture V1

```text
Bismut Download Broker
        ↓
Quarantine Store
        ↓
Artifact Identity (SHA-256)
        ↓
Archive Inspector
        ↓
.NET IL Scanner (Mono.Cecil)
        ↓
Rule Engine
        ├── generic .NET
        ├── Vintage Story specific
        ├── known malware/IOC
        └── supply-chain
        ↓
Local AV Provider
        ↓
Remote Reputation Providers
        ├── VirusTotal hash lookup
        ├── future community revocation feed
        └── ModDB retraction status
        ↓
Risk Aggregator
        ↓
Trust Record
        ↓
Security-aware Resolver / Preflight
```

---

## 17. Priorisierte Implementierungsreihenfolge

### P0 — Artifact + Revocation Foundation

1. SHA-256 für jedes Mod-Artefakt
2. Quarantine Store
3. Trust Record Schema
4. ModDB retraction/revocation ingestion
5. Security-aware lockfile binding

### P1 — .NET Static Analyzer

1. Mono.Cecil Reader
2. type/method/reference inventory
3. ModuleInitializer/static ctor rules
4. Process/network/native/filesystem rules
5. Harmony/internal-engine capability detection

### P2 — Source/Release Provenance

1. Source URL normalization
2. source snapshot identity
3. public-source inventory
4. release IL inventory
5. drift report

### P3 — Reputation Providers

1. Windows Defender adapter
2. optional VirusTotal hash lookup
3. known-malicious local database
4. rule/feed updates

### P4 — Dynamic Sandbox

Erst nach stabiler Static-/Quarantine-Basis.

---

## 18. Required Test Corpus

Bismut braucht kein ausschließlich künstliches Testset.

```text
SAFE
- asset-only VS mod
- normal C# code mod
- legitimate network mod
- legitimate Harmony mod
- legitimate native helper mod

SUSPICIOUS
- obfuscated but benign fixture
- Process.Start benign fixture
- network downloader without execution
- source/release drift fixture

MALICIOUS / KNOWN
- EICAR only for AV integration
- ConfigLib 1.12.0 after independent artifact capture/verification
- synthetic staged downloader fixture
- synthetic credential-path access fixture
```

Keine reale Malware wird für normale Unit Tests ausgeführt.

---

## 19. Phase-1 Ergebnis

### Sicher belegt

- reale modbezogene System-Malware existiert und kann Plattformen/Accounts als Supply-Chain nutzen
- klassische AV-Erkennung allein reicht bei neuartiger Mod-Malware nicht zuverlässig
- Vintage Story besitzt mit ConfigLib einen realen Fall von absichtlich disruptivem Mod-Code
- Vintage Story Server-Mod-Verteilung ist eine eigene Sicherheitsgrenze
- ModDB v2 besitzt bereits Retraction-Signale, die Bismut konsumieren kann
- SHA-256-/Harmony-Integritätsprüfung existiert bereits als Pattern im VS-Ökosystem
- etablierte Mod-Plattformen kombinieren Decompilation, Hashing, Static Analysis und Manual Review

### Noch offen

- unabhängige lokale Binäranalyse von ConfigLib 1.12.0
- Inventar weiterer Vintage-Story-spezifischer problematischer Mods/Incidents
- exakte False-Positive-Baseline für typische VS-Mod-APIs
- Design der ersten Mono.Cecil-Regeln
- Security-Feed-/Revocation-Format
- Entscheidung Windows Sandbox vs. VM für spätere dynamische Analyse

---

## 20. Nächster Research-Slice

`PT-SEC-RESEARCH-02 — Vintage Story .NET Mod Capability Corpus`

Ziel:

```text
20–30 repräsentative bekannte VS-Code-Mods
        ↓
IL/API inventory
        ↓
benign capability baseline
        ↓
false-positive map
        ↓
first Bismut static rule pack
```

Ohne diese Baseline würde ein Scanner schnell alles mit `HttpClient`, Harmony oder Reflection als Weltuntergang markieren. Das wäre technisch laut, aber sicherheitlich ziemlich nutzlos.
