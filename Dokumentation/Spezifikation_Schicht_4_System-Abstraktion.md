# Schicht 4: System-Abstraktion - Ultra-Feinspezifikation v2.0

**Dokumenten-Status:** Aktiv
**Version:** 2.0
**Letzte Aktualisierung:** 2025-08-16

## 1. Vision & Philosophie

### 1.1. Zweck und Geltungsbereich

Dieses Dokument definiert die **System-Abstraktionsschicht (Schicht 4)**, das Herzstück von NovaDE. Diese Schicht ist die direkte Schnittstelle zum Betriebssystem-Kernel und zur Hardware. Sie abstrahiert die Komplexität von Grafik-Rendering, Eingabeverarbeitung, Fenster-Management und Hardware-Interaktionen und stellt den höheren Schichten eine saubere, stabile und performante API zur Verfügung. Ihr Kernstück ist der Wayland-Compositor.

Der Geltungsbereich dieser Schicht ist die vollständige Kapselung aller plattformspezifischen Details. Die darüberliegenden Schichten sollen so geschrieben sein, als ob sie auf einer idealisierten "NovaDE-Maschine" laufen, ohne Kenntnis von Wayland, libinput oder Vulkan zu haben.

### 1.2. Architektonische Prinzipien

- **Performance & Latenz über alles:** Die Latenz vom Input zum Photon ist die kritischste Metrik. Jede Codezeile in dieser Schicht muss auf minimalen Overhead und maximale Geschwindigkeit optimiert sein. Blockierende Operationen sind verboten.
- **Stabilität als Fundament:** Ein Absturz in dieser Schicht führt zum Absturz des gesamten Systems. Der Code muss absolut robust, fehlerresistent und sicher sein. Client-Fehler dürfen niemals den Compositor destabilisieren.
- **Strikte Abstraktion:** Die Schicht wird um einen Satz von `IPlatform`-Traits herum entworfen, die die Funktionalität definieren. Die konkrete Implementierung (für Linux/Wayland) realisiert diese Traits. Dies ermöglicht zukünftige Portierbarkeit und Testbarkeit.
- **Kontrollierter `unsafe`-Einsatz:** `unsafe`-Code ist nur an den unumgänglichen FFI-Grenzen zu C-Bibliotheken erlaubt. Jeder `unsafe`-Block MUSS mit einem `// SAFETY:`-Kommentar versehen sein, der die Sicherheit der Operation detailliert begründet.

---

## 2. System Abstraction Layer (SAL) - Trait-basierte Architektur

Das Design der Schicht 4 basiert auf einer Reihe von abstrakten Traits, die eine plattformunabhängige Schnittstelle definieren.

- **`trait IPlatform`**: Das übergreifende Trait, das die gesamte Plattform-Implementierung kapselt und Zugriff auf die spezialisierten Manager gewährt.
- **`trait IWindowManager`**: Definiert die Kernfunktionalitäten eines Fenster-Managers: Fenster erstellen, zerstören, fokussieren, anordnen etc. Der Wayland-Compositor ist die Linux-Implementierung dieses Traits.
- **`trait IInputManager`**: Abstrahiert die Verarbeitung von Eingabeereignissen. Wandelt rohe Hardware-Events in normalisierte, systemweite Events um (z.B. `PointerMotion`, `KeyPress`).
- **`trait IRenderer`**: Entkoppelt die Compositor-Logik von der konkreten Grafik-API. Definiert Methoden wie `begin_frame`, `render_texture`, `end_frame`.
- **`trait IHardwareManager`**: Bietet eine Schnittstelle zur Abfrage und Steuerung von Hardware-Komponenten (z.B. Display-Helligkeit, Batteriestatus).

---

## 3. Linux-Implementierung: Modul-Übersicht

Die konkrete Implementierung für Linux/Wayland ist in den folgenden Modulen organisiert:

| Modul-Pfad                      | Implementiert Trait(s) | Verantwortlichkeit                                                                                             |
| ------------------------------- | ---------------------- | -------------------------------------------------------------------------------------------------------------- |
| `compositor` / `wayland_compositor_core` | `IWindowManager`       | Kern des Wayland-Compositors, basierend auf `smithay`. Verwaltet Protokolle, Clients und den Compositor-State. |
| `nova_compositor_logic`         | `IWindowManager`       | Implementiert die NovaDE-spezifische Fenster-Management-Logik (Tiling, Stacking) auf dem `smithay`-Fundament. |
| `input`                         | `IInputManager`        | Nutzt `libinput` und `udev`, um rohe Kernel-Events in saubere `InputEvent`-Typen zu übersetzen.                |
| `renderer` / `graphics`         | `IRenderer` (definiert) | Definiert den `FrameRenderer`-Trait und andere Grafik-Primitive.                                               |
| `renderers`                     | `IRenderer` (impl)     | Enthält konkrete `FrameRenderer`-Implementierungen für `WGPU` und/oder `Vulkan`.                               |
| `hardware`                      | `IHardwareManager`     | Implementiert die Steuerung der Display-Helligkeit und das Auslesen des Batteriestatus via D-Bus/sysfs.      |
| `pty`                           | -                      | Stellt eine Abstraktion für Pseudo-Terminals (PTYs) bereit, die von der Terminal-Anwendung benötigt wird.     |
| `ipc`                           | -                      | Implementiert Low-Level-Mechanismen für die Inter-Prozess-Kommunikation.                                       |

---

## 4. Detail-Spezifikationen (Ausgewählte Module)

### 4.1. Compositor (`compositor`, `wayland_compositor_core`, `nova_compositor_logic`)
- **Technologie:** Basiert auf dem `smithay`-Framework.
- **Architektur:**
  - **`wayland_compositor_core`**: Enthält das `smithay`-Boilerplate, die Initialisierung des Wayland-Sockets, die Verwaltung der Protokoll-Globals und die grundlegende Event-Schleife.
  - **`compositor`**: Integriert die Kern-Logik mit den anderen Teilen von Schicht 4 (Input, Renderer) und stellt die `IWindowManager`-Implementierung bereit.
  - **`nova_compositor_logic`**: Hier befindet sich die eigentliche "Intelligenz" des Fenster-Managers. Es implementiert die Tiling-Algorithmen, die Workspace-Logik (in Zusammenarbeit mit Schicht 3) und die Regeln für Fensterplatzierung und -fokus.

### 4.2. Rendering-Pipeline (`renderer`, `renderers`, `graphics`)
- **Abstraktion:** Der `renderer`-Modul definiert den zentralen `trait FrameRenderer`. Dieser entkoppelt die gesamte Compositor-Logik von der konkreten Grafik-API.
- **Implementierungen:** Der `renderers`-Modul enthält die konkreten Implementierungen.
  - **`wgpu_renderer`**: Eine moderne, sichere Implementierung, die auf `wgpu` basiert und somit verschiedene Backends (Vulkan, Metal, DX12) ansprechen kann.
- **Shader:** Der `assets/shaders`-Ordner enthält GLSL- oder WGSL-Shader-Code, der zur Kompilierzeit oder Laufzeit geladen wird.

### 4.3. Eingabeverarbeitung (`input`)
- **Technologie:** Nutzt `libinput` für eine robuste und hardware-unabhängige Verarbeitung von Eingabe-Events und `udev` zur Erkennung von Geräten.
- **Pipeline:**
  1. `udev`-Monitor erkennt Hot-Plugging von Geräten.
  2. `libinput` wird mit den Geräten initialisiert.
  3. Die Event-Schleife liest rohe Events von `libinput`.
  4. Der `input`-Manager wandelt diese in normalisierte `NovaInputEvent`-Typen um (z.B. `PointerMotion { delta: PointI32 }`, `KeyPress { keycode: u32, state: KeyState }`).
  5. Diese Events werden an den `nova_compositor_logic`-Teil weitergeleitet, um Aktionen auszulösen.

---

## 5. Zukünftige Entwicklung & Strategie

- **PipeWire-Integration:** Integration von PipeWire für performantes und sicheres Screen-Sharing und Audio-Routing direkt im Compositor.
- **Headless-Backend:** Entwicklung eines "Headless"-Backends für den Compositor, das kein physisches Display benötigt. Dies ist entscheidend für automatisierte Integrationstests der Fenster-Management-Logik.
- **Unterstützung für neue Hardware-Features:** Proaktive Planung für die Unterstützung von Technologien wie High Dynamic Range (HDR), Variable Refresh Rate (VRR) und modernen Touch/Stift-Gesten.
- **Alternative Compositor-Backends:** Obwohl der Fokus auf Wayland liegt, wird die `IPlatform`-Architektur so gepflegt, dass theoretisch ein Backend für X11 oder sogar Windows/macOS (für Testzwecke) implementiert werden könnte.

---

## 6. Entwicklungs- und Qualitätsrichtlinien

- **Latenz-Tests:** Implementierung von spezifischen Tests, die die Latenz von der Eingabe bis zum gerenderten Frame messen, um Performance-Regressionen zu verhindern.
- **Wayland-Protokoll-Konformität:** Regelmäßige Validierung des Compositors gegen die offiziellen Wayland-Test-Suiten, um die Kompatibilität mit einer breiten Palette von Wayland-Clients sicherzustellen.
- **Stresstests:** Entwicklung von Tests, die den Compositor mit einer großen Anzahl von Fenstern, schnellen Eingaben und häufigen Zustandsänderungen belasten, um die Stabilität zu gewährleisten.
- **Dokumentation von `unsafe`:** Jeder `unsafe`-Block muss nicht nur einen `// SAFETY:`-Kommentar haben, sondern auch im Pull Request explizit erwähnt und begründet werden.