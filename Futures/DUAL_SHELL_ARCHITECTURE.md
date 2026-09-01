# Bismut Future — Dual-Shell Launcher Architecture

Status: `FUTURE / ARCHITECTURE BASELINE`

## 1. Ziel

Bismut soll langfristig zwei Desktop-Shells mit **einer gemeinsamen Anwendungsschicht** unterstützen:

```text
Bismut User Launcher  → Tauri 2 / WebView2
Bismut Dev Launcher   → Electron / Chromium / Node

Shared Application
├── React / Vite / TypeScript
├── Contracts
├── ReadModels
├── Services
├── Normalizer
├── Provider Interfaces
└── BismutHost Boundary

Bismut Engine
└── separater nativer Prozess / Sidecar
```

Ziel ist ausdrücklich **nicht**, zwei unabhängige Launcher-Codebasen zu pflegen.

Der User Launcher soll möglichst klein, schnell und ressourcenschonend sein. Der Dev Launcher darf bewusst schwerer sein und Electron für Debugging, Automation, DevTools, Puppeteer-/E2E-Tests und experimentelle Integrationen verwenden.

---

## 2. Grundentscheidung

Langfristige Zielrichtung:

```text
USER BUILD
React/Vite
   ↓
Tauri 2 / WebView2
   ↓
TauriBismutHost
   ↓
Canonical BismutHost Contract
   ↓
Bismut Engine

DEV BUILD
React/Vite
   ↓
Electron
   ↓
ElectronBismutHost
   ↓
Canonical BismutHost Contract
   ↓
Bismut Engine
```

Beide Shells konsumieren dieselben Produktverträge.

---

## 3. Shared Frontend ist Shell-agnostisch

Das gemeinsame Frontend darf keine direkte Kenntnis über die konkrete Desktop-Shell benötigen.

Insbesondere sollen produktive Shared-Dateien langfristig **keine direkten Imports** aus folgenden APIs enthalten:

```text
@tauri-apps/*
electron
Electron Main internals
Node-only shell APIs
```

Stattdessen konsumiert das Frontend nur einen kanonischen Host-Contract.

Beispielziel:

```ts
interface BismutHost {
  worlds: {
    list(): Promise<WorldReadModel[]>;
  };

  profiles: {
    list(): Promise<ProfileReadModel[]>;
  };

  servers: {
    list(): Promise<ServerReadModel[]>;
  };

  engine: {
    status(): Promise<EngineStatus>;
    start(): Promise<void>;
    stop(): Promise<void>;
  };

  system: {
    openPath(path: string): Promise<void>;
    selectDirectory(): Promise<string | null>;
  };
}
```

Die konkrete Form darf sich noch ändern. Entscheidend ist die Boundary:

```text
UI / Services
     ↓
BismutHost
     ↓
Shell Adapter
```

---

## 4. Electron bleibt Dev-Shell

Electron soll nicht vorschnell entfernt werden.

Der Dev Launcher profitiert weiterhin von:

- kontrolliertem Chromium
- Electron Main / Preload
- Node-Integration hinter klarer Boundary
- DevTools
- Puppeteer-/Automation-Tests
- Debug-/Diagnostics-Surfaces
- experimentellen Host-/Engine-Adaptern
- schnellem Instrumentieren von Runtime- und IPC-Verhalten

Electron ist damit perspektivisch die **Entwicklungswerkbank**, nicht zwingend die endgültige User-Runtime.

---

## 5. Tauri 2 als User-Shell-Kandidat

Der User Launcher soll langfristig auf eine kleinere Shell umgestellt werden können.

Bevorzugter Kandidat:

```text
Tauri 2
```

Auf Windows kann die UI über WebView2 laufen, während der native Host dünn bleibt.

Der Tauri-Teil soll möglichst nur besitzen:

```text
Window Lifecycle
Filesystem Boundary
Process / Sidecar Lifecycle
Updater
OS Integration
Dialog / Path Selection
Shell-specific IPC
```

Produktlogik gehört nicht in Rust, nur weil Rust dort verfügbar ist.

---

## 6. Bismut Engine bleibt separater Prozess

Die Desktop-Shell darf nicht zur neuen Engine werden.

Ziel:

```text
Launcher Shell
     ↓
BismutHost / Engine Contract
     ↓
Bismut Engine Process
     ↓
Renderer / Runtime / Vulkan / OpenGL
```

Für Tauri kann der Engine-Prozess perspektivisch als Sidecar gebündelt werden.

Für Electron wird derselbe Engine-Contract über den Electron Host konsumiert.

Die Engine bleibt dadurch unabhängig von der gewählten UI-Shell.

---

## 7. Kein doppeltes Frontend

Verbotenes Zielbild:

```text
Electron Launcher
└── eigenes Frontend

Tauri Launcher
└── eigenes Frontend
```

Bevorzugtes Zielbild:

```text
Bismut Launcher
├── frontend/
│   ├── views/
│   ├── components/
│   ├── readmodels/
│   ├── services/
│   └── contracts/
│
├── shells/
│   ├── electron/
│   └── tauri/
│
└── host/
    └── canonical BismutHost boundary
```

Die genaue Verzeichnisstruktur ist noch nicht kanonisch. Die Ownership-Grenze ist es.

---

## 8. Shell-spezifische Imports

Langfristige Regel:

```text
@tauri-apps/*
→ nur Tauri Adapter / Shell

electron / Electron Main APIs
→ nur Electron Adapter / Shell

Shared React / ReadModels / Services
→ keine direkte Shell-Abhängigkeit
```

Damit kann derselbe produktive Launcher-Code in beiden Shells laufen.

---

## 9. Provider- und ReadModel-Modell

Die aktuelle Bismut-Migration in Richtung:

```text
Provider
→ Normalizer / Adapter
→ Canonical ReadModel
→ UI
```

ist bereits die gewünschte Vorarbeit für die Dual-Shell-Architektur.

World-, Profile-, Server-, ModDB- und spätere Engine-Daten sollen über diese gemeinsame Schicht laufen.

Die Shell ist Transport-/OS-Grenze, nicht Truth Layer.

---

## 10. Migration erst nach stabiler Produktgrenze

Diese Future darf die laufende Bismut-Migration nicht destabilisieren.

Vor einer aktiven Tauri-Migration müssen mindestens gelten:

```text
Launcher Canon stabil
World/Profile/Server ReadModels stabil
Host Contract stabil
Engine Contract stabil
Electron produktiv reproduzierbar
Fresh-clone Build reproduzierbar
Shell-spezifische Imports inventarisiert
```

Erst danach wird Tauri als zweiter Consumer des Host-Contracts aufgebaut.

Nicht vorher das aktuelle Launcher-Frontend gleichzeitig migrieren und neu strukturieren.

---

## 11. Vorgesehene Phasen

```text
Phase 0
Canonical Launcher / ReadModels / Provider stabilisieren

Phase 1
BismutHost Contract kanonisieren

Phase 2
Electron vollständig hinter ElectronBismutHost isolieren

Phase 3
Shared Frontend auf Shell-Abhängigkeiten = 0 prüfen

Phase 4
Minimalen TauriBismutHost implementieren

Phase 5
User Launcher Smoke / Build / Updater / Sidecar

Phase 6
Releasevergleich Electron vs Tauri

Phase 7
Tauri als User-Default nur nach messbarer Qualifikation
```

---

## 12. Acceptance Gates für Tauri User Launcher

Tauri darf erst User-Default werden, wenn mindestens folgende Gates bestehen:

```text
World / Profile / Server parity
ModDB parity
Publisher / Modpack parity soweit release-relevant
Engine lifecycle parity
Updater parity
Settings parity
Fresh install PASS
Fresh clone/build PASS
Crash diagnostics PASS
Rollback PASS
No shell-specific imports in Shared Frontend
```

Zusätzlich messen:

```text
Installer size
Cold start
Warm start
Idle RAM
CPU idle
Update size
Runtime stability
```

Die Entscheidung soll datenbasiert erfolgen, nicht weil eine Shell auf dem Papier moderner aussieht.

---

## 13. Dev Launcher bleibt unabhängig davon erhalten

Auch wenn Tauri später User-Default wird, kann Electron dauerhaft als Dev-Shell bestehen bleiben.

Beispiel:

```text
Bismut.exe
→ User / Tauri

Bismut-Dev.exe
→ Electron / Diagnostics / DevTools / Experiments
```

Beide sind Consumers desselben kanonischen Produkt- und Engine-Contracts.

---

## 14. Nicht tun

- zwei getrennte Launcher-Produkte mit driftender Produktlogik erzeugen
- Shared UI direkt an Tauri koppeln
- Shared UI direkt an Electron koppeln
- Rust zur zweiten Produktlogik-Schicht machen
- Electron vor abgeschlossener Host-Boundary entfernen
- Tauri während laufender Ownership-/Migration-Slices opportunistisch einbauen
- Engine-Funktionalität in die Desktop-Shell verschieben
- Tauri als User-Default ohne Build-/Parity-/Performance-Evidence festlegen

---

## 15. Zielzustand

```text
                    Shared Bismut Application
                  React / TS / Contracts / ReadModels
                              │
                         BismutHost
                        ┌─────┴─────┐
                        │           │
                 Electron Host   Tauri Host
                    DEV             USER
                        │           │
                        └─────┬─────┘
                              │
                       Bismut Engine
                              │
                    Vulkan / OpenGL Runtime
```

Kurzfassung:

```text
Electron = Development Workbench
Tauri    = Lightweight User Shell Candidate
Frontend = Shared Product
Engine   = Independent Runtime
```

Status bleibt `FUTURE`, bis die laufende Launcher-/Engine-Migration abgeschlossen und die Host-Boundary kanonisch stabil ist.
