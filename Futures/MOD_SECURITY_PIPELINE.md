# Bismut Future — Mod Security Pipeline

Status: `FUTURE / DESIGN BASELINE`

## 1. Ziel

Bismut soll Mods nicht direkt aus einer Remote-Quelle in eine aktive Instanz übernehmen. Jede neue oder geänderte Mod-Version durchläuft zuerst eine lokale, reproduzierbare Sicherheits- und Provenance-Prüfung.

Ziel ist ausdrücklich **nicht**, einen Mod allein anhand eines einzelnen Antivirus-Ergebnisses als `safe` oder `malicious` zu deklarieren. Bismut kombiniert mehrere unabhängige Signale und materialisiert das Ergebnis pro Mod-Artefakt und SHA-256.

```text
Remote Source / ModDB
        ↓
Download Quarantine
        ↓
SHA-256 + Provenance
        ↓
Archive / Assembly Inspection
        ↓
Static Capability Analysis
        ↓
Local AV Provider
        ↓
Optional Reputation Provider
        ↓
Bismut Risk Evaluation
        ↓
ALLOW / WARN / BLOCK / MANUAL_REVIEW
        ↓
Lockfile / Instance Apply
```

## 2. Architekturregeln

1. **Hash-bound truth**  
   Sicherheitsstatus gilt immer nur für exakt ein Artefakt bzw. einen SHA-256. Ändert sich die Datei, ist der alte Trust Record ungültig.

2. **VirusTotal ist nur ein Signal**  
   Primär wird ein vorhandener Hash-/Reputationsreport abgefragt. Unbekannte Dateien werden nicht ungefragt an Dritte hochgeladen.

3. **Semantische Mod-Analyse zusätzlich zu AV**  
   Bismut analysiert Code-Mods auf für Vintage Story relevante Fähigkeiten und ungewöhnliche Verhaltensmuster. Ein AV-Scan allein erkennt absichtliche In-Game-Sabotage oder Supply-Chain-Abweichungen möglicherweise nicht.

4. **Source ↔ Binary Drift ist sicherheitsrelevant**  
   Wenn eine veröffentlichte DLL/ZIP Code enthält, der im angegebenen öffentlichen Source nicht nachvollziehbar ist, wird das als eigener Provenance-/Supply-Chain-Fund bewertet.

5. **Fail-closed bei kritischer Unsicherheit**  
   Kritische Findings, unklare Provenance oder nicht erklärbare Binärdrift dürfen nicht still als `trusted` durchgereicht werden.

## 3. Vorgesehene Prüfstufen

### 3.1 Provenance und Integrität

Erfassen:

- Mod-ID, Version, Quelle, Download-URL/Provider
- SHA-256 des Originaldownloads
- Archivstruktur und enthaltene Assemblies
- deklarierter Source-/Repository-Link
- Signaturen, falls vorhanden
- Lockfile-Referenz

### 3.2 Archive- und Assembly-Analyse

Prüfen unter anderem:

- `.dll`, `.exe`, native Libraries und unbekannte Binärdateien
- ungewöhnliche eingebettete Ressourcen
- `ModuleInitializer`
- `DllImport` / P/Invoke
- `Process.Start`
- `cmd.exe`, PowerShell oder Shell-Aufrufe
- Registry-/Autostart-/Scheduled-Task-Nutzung
- Netzwerk-/Socket-/HTTP-Code
- Dateizugriffe außerhalb der erwarteten VS-/Instanzpfade
- dynamisches Assembly Loading / Reflection Emit
- starke Obfuscation-Heuristiken
- Zugriff auf interne Engine-Typen und invasive Runtime-Manipulation

Ein Treffer bedeutet nicht automatisch Malware. Findings werden nach Kontext und Capability bewertet.

## 4. Bismut Capability Fingerprint

Vorgesehene Klassifikation:

```text
AssetsOnly
NormalCodeMod
NetworkedMod
NativeCode
ExternalProcess
FilesystemExtended
EngineInternalAccess
HarmonyPatch
DynamicAssemblyLoad
ObfuscatedAssembly
UnknownBinary
```

Daraus entsteht ein erklärbarer Risk Record statt eines simplen grünen/roten Symbols.

## 5. Lokaler AV-Provider

Unter Windows soll Bismut einen lokalen AV-Provider abstrahieren. Standardkandidat ist Microsoft Defender mit Custom Scan des Quarantänepfads.

Der AV-Status wird getrennt geführt:

```text
clean
threat_detected
scan_failed
provider_unavailable
unknown
```

`clean` bedeutet ausschließlich: der konfigurierte AV-Provider hat keinen Treffer geliefert. Es ist **kein** allgemeiner Sicherheitsbeweis.

## 6. Optionaler Reputation Provider

VirusTotal oder vergleichbare Dienste sind optionale Reputation Provider.

Standardpfad:

```text
SHA-256
  ↓
Reputation Lookup
  ↓
known / unknown / detections
```

Keine automatische Sample-Übermittlung ohne explizite Benutzerentscheidung und passende Nutzungs-/Lizenzbedingungen.

## 7. Source ↔ Binary Reproducibility / Drift

Für Mods mit öffentlichem Source soll Bismut perspektivisch prüfen können:

```text
Published Mod ZIP
      ↓
Assembly Metadata / IL Inventory
      ↓
Public Source Inventory
      ↓
Compare
      ↓
SOURCE_MATCH
SOURCE_DRIFT
SOURCE_DRIFT_CRITICAL
UNKNOWN
```

Mindestens folgende Fälle sind sicherheitsrelevant:

- Release enthält zusätzliche Typen/Methoden, die im angegebenen Source fehlen
- sicherheitsrelevante Datei wird im Source-Repository absichtlich nicht versioniert
- irreführende Namespaces oder versteckte Module
- Release-Verhalten kann aus der veröffentlichten Source-Baseline nicht reproduziert werden

## 8. Golden Security Fixture — ConfigLib

### Zweck

ConfigLib wird als **realer Vintage-Story-Supply-Chain-/Sabotage-Testfall** für die spätere Bismut Security Pipeline dokumentiert. Ziel ist nicht, dauerhaft jede Version pauschal als Malware zu labeln, sondern die Scanner so zu testen, dass bekannte problematische Muster reproduzierbar erkannt werden.

### Aktuell öffentlich belegbare Hinweise

Die Vintage Story ModDB führt für ConfigLib aktuell unter anderem folgende Versionshistorie:

- `1.13.0` wurde zurückgezogen.
- Als Retraction Reason steht: `actually, nvm, I'm leaving this code in`.
- Der zugehörige Changelog nennt zuvor: `Removed some code. If you dont know which one already, dont worry about it.`
- `1.13.1` nennt anschließend im Changelog: `Ok, no need for this piece of code anymore, tho it has fullfilled its purpose`.

Quelle:
- https://mods.vintagestory.at/configlib

Diese Aussagen belegen, dass ein umstrittener Codepfad in der Release-Historie selbst thematisiert wurde. Sie beweisen für sich allein noch **nicht** den exakten Binärinhalt jeder Version.

### Fixture-Regel

Für eine echte Golden Fixture dürfen nur lokal archivierte und gehashte Release-Artefakte verwendet werden. Vor Aufnahme in Tests müssen mindestens festgehalten werden:

```text
modId
version
originalDownloadSource
sha256
archiveManifest
assemblyManifest
staticFindings
sourceComparison
expectedVerdict
```

### Erwartete Scanner-Kategorien für den bekannten Fall

Die Pipeline soll bei entsprechend bestätigtem Binärinhalt insbesondere auf folgende Muster reagieren können:

```text
ModuleInitializer
assembly enumeration / foreign assembly inspection
internal Vintage Story runtime access
intentional mutation of foreign runtime/client state
randomized or delayed activation
obfuscation / misleading namespace heuristics
source-binary drift
hidden or non-versioned security-relevant source
```

Wichtig: Die konkrete Einstufung einer ConfigLib-Version als `MALICIOUS_CONFIRMED` darf erst erfolgen, wenn das betreffende Originalartefakt lokal gesichert, gehasht und statisch bzw. reproduzierbar analysiert wurde.

## 9. Trust Record

Beispielziel:

```json
{
  "modId": "examplemod",
  "version": "1.4.2",
  "sha256": "...",
  "source": "vintagestory-moddb",
  "security": {
    "localAv": "clean",
    "reputation": "known-clean",
    "staticRisk": "low",
    "sourceDrift": "none",
    "nativeCode": false,
    "externalProcess": false,
    "networkAccess": false,
    "writesOutsideInstance": false
  },
  "verdict": "trusted"
}
```

Dieser Record wird an den exakten Hash gebunden und kann in `mods.lock.json` oder einem separaten Security-Lock referenziert werden.

## 10. Verdict-Modell

Vorgesehene Zustände:

```text
TRUSTED
LOW_RISK
WARN
MANUAL_REVIEW
BLOCKED
MALICIOUS_CONFIRMED
UNKNOWN
```

`MALICIOUS_CONFIRMED` verlangt starke Evidence, z. B. reproduzierbar absichtliches Schad-/Sabotageverhalten oder belastbare Mehrquellen-Evidence. Heuristiken allein reichen dafür nicht.

## 11. Integration in Bismut

Zielpfad:

```text
ModDB Adapter
   ↓
Download Quarantine
   ↓
Mod Security Pipeline
   ↓
Dependency / Compatibility Resolver
   ↓
Security-aware Preflight
   ↓
mods.lock.json + Trust Record
   ↓
Snapshot → Apply → Verify → Rollback
```

UI später:

```text
🛡 Hash verified
✓ Local AV clean
✓ No native code
✓ No external process
🌐 Network access
⚠ Source/Binary drift
⛔ Critical behavior detected
```

Die UI darf niemals aus `0 detections` automatisch `safe` ableiten.

## 12. Spätere Acceptance Gates

Die Funktion gilt erst als releasefähig, wenn mindestens folgende Fixtures reproduzierbar bestehen:

1. normaler Asset-only-Mod → kein unnötiger Alarm
2. legitimer Network-Mod → Capability erkannt, nicht automatisch blockiert
3. legitimer Native-Code-Mod → erhöhte Prüfung, erklärbarer Warnstatus
4. EICAR/AV-Testfixture → lokaler AV-Treffer wird korrekt propagiert
5. bestätigte ConfigLib-Golden-Fixture → semantischer/Provenance-Fund auch dann, wenn klassische AV-Reputation unauffällig bleibt

## 13. Nicht tun

- unbekannte Mods ungefragt zu VirusTotal oder anderen Drittanbietern hochladen
- `0 detections = safe` anzeigen
- Netzwerkzugriff pauschal als Malware behandeln
- Obfuscation allein als Malwarebeweis verwenden
- eine Mod-Version ohne hashgebundene Evidence dauerhaft brandmarken
- Security Findings ohne Provenance und reproduzierbare Evidence in Lockfiles materialisieren
