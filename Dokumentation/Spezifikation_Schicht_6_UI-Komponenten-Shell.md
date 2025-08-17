# Schicht 6: UI-Komponenten & Shell - Ultra-Feinspezifikation v2.0

**Dokumenten-Status:** Aktiv
**Version:** 2.0
**Letzte Aktualisierung:** 2025-08-16

## 1. Vision & Philosophie

### 1.1. Zweck und Geltungsbereich

Dieses Dokument definiert die **UI-Komponenten & Shell-Schicht (Schicht 6)**, das visuelle Fundament und das Herz des NovaDE-Design-Systems. Diese Schicht ist verantwortlich für die Erstellung aller wiederverwendbaren UI-Bausteine und der grundlegenden Anwendungs-Shell, in der die Core-Anwendungen (Schicht 7) leben.

Der Geltungsbereich umfasst sowohl die native Shell-Anwendung als auch die Bibliothek von Web-basierten UI-Komponenten. Das Ziel ist ein ästhetisch herausragendes, konsistentes und performantes Benutzererlebnis.

### 1.2. Architektonische Prinzipien

- **Hybrides UI-Modell:** NovaDE verfolgt einen hybriden Ansatz, um das Beste aus beiden Welten zu vereinen: eine hochperformante, native Shell für die Kern-UI (Panels, Fenster-Chrom) und eine flexible, moderne Web-Technologie für die eigentlichen Anwendungs-UIs.
- **Component-Driven Development (CDD):** Alle UI-Komponenten werden isoliert entwickelt, getestet und dokumentiert. Sie sind zustandslos und wiederverwendbar.
- **Pixel-Perfect-Anspruch:** Jede Komponente und jede Interaktion muss bis ins letzte Detail ausgefeilt sein, um ein hochwertiges, professionelles Erscheinungsbild zu gewährleisten.
- **Zentrales Design-System:** Das Theming ist nicht auf eine Technologie beschränkt. Ein zentrales System definiert Design-Tokens, die sowohl für die native GTK-Shell als auch für die Web-Komponenten kompiliert werden, um absolute visuelle Konsistenz zu garantieren.

---

## 2. Architektur-Übersicht: Das Hybride UI-Modell

Die UI von NovaDE besteht aus zwei primären Teilen, die nahtlos zusammenarbeiten:

1.  **Die Native Shell (GTK4 & Libadwaita):**
    - Dies ist eine in Rust geschriebene GTK4-Anwendung, die als Hauptprozess läuft.
    - Sie ist verantwortlich für das "Chrom" der Benutzeroberfläche: das Hauptfenster, die Menüleiste, die Seitenleiste, die Statusleiste und andere globale UI-Elemente.
    - Ihre Hauptaufgabe ist es, als **Container für WebViews** zu dienen. Jede Core-Anwendung (wie die Einstellungen oder der Datei-Browser aus Schicht 7) wird als eigenständige Web-Anwendung in einem von der Shell bereitgestellten WebView-Widget ausgeführt.

2.  **Die Web-basierten Anwendungen (Svelte & Tauri):**
    - Die eigentlichen Anwendungs-UIs werden mit modernen Web-Technologien (Svelte, TypeScript, SCSS) entwickelt.
    - Die Kommunikation zwischen dem Svelte-Frontend (JavaScript) und dem Rust-Backend (Domänen-Services aus Schicht 3) erfolgt ausschließlich über die von **Tauri** bereitgestellte, sichere `invoke`-Bridge.
    - Dies ermöglicht eine schnelle, iterative Entwicklung der UI, während die kritische Geschäftslogik sicher und performant in Rust verbleibt.

```mermaid
graph TD
    A[Native GTK4 Shell in Rust] -->|hostet| B(WebView Container);
    B --> C{Svelte-Anwendung (z.B. Einstellungen)};
    C -->|Tauri invoke()| D[Rust Backend];
    D -->|ruft auf| E(Domänen-Services Schicht 3);
```

---

## 3. Kernkomponenten-Übersicht

| Komponente                    | Technologie         | Verantwortlichkeit                                                                                              |
| ----------------------------- | ------------------- | --------------------------------------------------------------------------------------------------------------- |
| **GTK4 Shell**                | Rust, GTK4, Libadwaita | Implementiert die native Anwendungs-Shell (`MainWindow`, `SidePanel`, etc.) und verwaltet die WebViews.          |
| **Svelte Komponentenbibliothek** | Svelte, TypeScript, SCSS | Stellt eine umfassende Bibliothek von reinen, wiederverwendbaren UI-Komponenten (`Button`, `Input`, `Card`) bereit. |
| **Zentrales Theming-System**    | Rust (Build-Skript) | Definiert Design-Tokens (JSON/YAML) und kompiliert sie zu CSS für GTK und Web.                                  |
| **UI-Services**                 | Rust                | Stellt Hilfsdienste für die UI bereit, z.B. für das Laden und Cachen von Schriften (`FontService`) und Bildern. |

---

## 4. Detail-Spezifikationen

### 4.1. GTK4 Shell (`shell/`, `window.rs`)
- **Zweck:** Bereitstellung der nativen, performanten Haupt-UI.
- **Implementierung:** Nutzt die `gtk4-rs`- und `libadwaita`-Bindings für Rust.
- **Komponenten:**
  - `MainWindow`: Das Hauptfenster, das die Anwendungsstruktur (HeaderBar, Seitenleiste, Content-Bereich) definiert.
  - `WebViewManager`: Ein Dienst, der für die Erstellung, Verwaltung und Kommunikation mit den Tauri/WebView-Instanzen verantwortlich ist.
  - `ShellUIController`: Verbindet die GTK-UI-Elemente mit den darunterliegenden Services (z.B. um den aktiven Workspace-Namen in der Statusleiste anzuzeigen).

### 4.2. Svelte Komponentenbibliothek (`components/`, `widgets/`)
- **Zweck:** Bildet das Herzstück des NovaDE-Design-Systems.
- **Prinzipien:**
  - **Zustandslos:** Komponenten erhalten ihren Zustand und ihre Callbacks ausschließlich über `props`.
  - **Strikt getypt:** Alle Komponenten werden mit TypeScript entwickelt.
  - **Stil-isoliert:** Nutzt Sveltes Scoped-Styles, um Stil-Konflikte zu vermeiden.
- **Beispiele:** `Button.svelte`, `Slider.svelte`, `DataGrid.svelte`, `Modal.svelte`.

### 4.3. Zentrales Theming-System (`theming/`, `build.rs`)
- **Zweck:** Gewährleistung eines 100% konsistenten Erscheinungsbildes zwischen nativer Shell und Web-Anwendungen.
- **Workflow:**
  1.  **Definition:** Design-Tokens (z.B. `primary-color`, `font-size-md`) werden in einem technologie-agnostischen Format wie YAML definiert.
  2.  **Kompilierung:** Ein `build.rs`-Skript wird während des Rust-Kompilierungsprozesses ausgeführt. Es liest die YAML-Dateien und generiert zwei separate CSS-Dateien:
      - `theme.css`: Enthält die Tokens als CSS-Variablen (z.B. `--primary-color: #...;`) für die Svelte-Komponenten.
      - `theme.gtk.css`: Enthält GTK-spezifisches CSS, das ebenfalls die CSS-Variablen nutzt, um die nativen Widgets zu stylen.
  3.  **Anwendung:** Die GTK-Shell lädt `theme.gtk.css` zur Laufzeit. Die Svelte-Anwendungen importieren `theme.css`.

---

## 5. Zukünftige Entwicklung & Strategie

- **Widget-Katalog / Storybook:** Entwicklung einer dedizierten Anwendung (ähnlich Storybook.js oder GTK Widget Factory), die alle Svelte- und GTK-Komponenten visuell darstellt. Dies beschleunigt die Entwicklung, verbessert die Dokumentation und ermöglicht visuelles Regression-Testing.
- **Animations-Bibliothek:** Erstellung einer standardisierten Animations-Bibliothek (sowohl für CSS als auch potenziell für GTK), um konsistente und flüssige Animationen im gesamten System zu gewährleisten.
- **Barrierefreiheit (Accessibility):** Systematische Überprüfung und Verbesserung aller Komponenten gemäß den WCAG- und ARIA-Standards, um sicherzustellen, dass NovaDE für alle Benutzer zugänglich ist.
- **Theming-Editor:** Entwicklung einer grafischen Anwendung, die es Benutzern ermöglicht, die Theme-Token-Dateien direkt zu bearbeiten und eine Live-Vorschau der Änderungen zu sehen.

---

## 6. Entwicklungs- und Qualitätsrichtlinien

- **Performance:** UI-Operationen dürfen niemals den UI-Thread blockieren. Alle langwierigen Aufgaben müssen in das Rust-Backend ausgelagert und asynchron aufgerufen werden.
- **Visuelle Konsistenz:** Jede neue Komponente muss sich nahtlos in das bestehende Design-System einfügen. Abweichungen müssen explizit genehmigt werden.
- **API-Design:** Die `props` der Svelte-Komponenten müssen klar, prägnant und gut dokumentiert sein.
- **Testbarkeit:** Die Logik in der GTK-Shell muss durch Unit-Tests abgedeckt sein. Für Svelte-Komponenten werden visuelle Snapshot-Tests und Interaktionstests (z.B. mit Vitest/Playwright) angestrebt.