# Schicht 7: Anwendungs- & Präsentationsschicht - Ultra-Feinspezifikation v2.0

**Dokumenten-Status:** Aktiv
**Version:** 2.0
**Letzte Aktualisierung:** 2025-08-16

## 1. Vision & Philosophie

### 1.1. Zweck und Geltungsbereich

Dieses Dokument definiert die **Anwendungs- & Präsentationsschicht (Schicht 7)**, die oberste und für den Benutzer sichtbare Ebene der NovaDE-Architektur. Ihre Mission ist es, die wiederverwendbaren UI-Komponenten aus Schicht 6 zu vollständigen, intuitiven und ästhetisch ansprechenden Core-Anwendungen zusammenzusetzen. Hier werden die User-Workflows choreographiert und die Daten aus dem Backend für die Darstellung aufbereitet.

Der Geltungsbereich dieser Schicht ist die Implementierung der Frontend-Logik (Präsentationslogik) und die Komposition der UI. Sie enthält **keinerlei Geschäftslogik** – diese wird ausschließlich von den Domänen-Services in Schicht 3 ausgeführt.

### 1.2. Architektonische Prinzipien

- **User-Workflow-Fokus:** Das Design der Anwendungen orientiert sich nicht an Features, sondern an den Zielen und Arbeitsabläufen des Benutzers. Jede Interaktion soll reibungslos und intuitiv sein.
- **Strikte Trennung von UI und Logik (MVVM/Presenter):** Die UI (View) ist so "dumm" wie möglich. Die gesamte Präsentationslogik (z.B. das Aufrufen von Backend-Services, das Aufbereiten von Daten für die Anzeige) ist in einem separaten Presenter oder ViewModel gekapselt. Dies erhöht die Testbarkeit und Wartbarkeit.
- **Reaktive Datenbindung:** Der Zustand der UI wird in reaktiven Stores (z.B. Svelte Stores) gehalten. Die UI-Komponenten abonnieren diese Stores und aktualisieren sich automatisch, wenn sich die Daten ändern. Der Presenter ist für die Aktualisierung der Stores verantwortlich.
- **Asynchronität als Standard:** Jede Kommunikation mit dem Backend (über die Tauri-Bridge) MUSS asynchron erfolgen, um die UI niemals zu blockieren. Dem Benutzer werden währenddessen Ladezustände angezeigt.

---

## 2. Architektur-Übersicht: Das Presenter-Muster

Alle Anwendungen in dieser Schicht folgen konsequent einem Model-View-Presenter (MVP) oder Model-View-ViewModel (MVVM) ähnlichen Muster, um eine saubere Trennung der Verantwortlichkeiten zu gewährleisten.

**Der Datenfluss einer typischen Interaktion:**

1.  **View (Svelte-Komponente):** Ein Benutzer klickt auf einen Button. Der `@on:click`-Handler der Komponente ruft eine Methode im zuständigen Presenter auf.
2.  **Presenter (TypeScript-Modul):** Die Presenter-Methode wird ausgeführt. Sie zeigt einen Ladezustand im UI-Store an.
3.  **Backend-Aufruf:** Der Presenter ruft via `invoke` (Tauri-Bridge) die entsprechende `#[tauri::command]`-Funktion im Rust-Backend auf. Diese Rust-Funktion ist ein dünner Wrapper, der die Methode des zuständigen Domänen-Service (Schicht 3) aufruft.
4.  **Backend-Verarbeitung:** Der Domänen-Service führt die Geschäftslogik aus, interagiert mit Repositories (Schicht 5) und gibt ein Ergebnis (ein Domänen-Objekt aus Schicht 2) zurück.
5.  **Rückgabe & Transformation:** Das Ergebnis wird über die Tauri-Bridge an den Presenter zurückgegeben.
6.  **Store-Aktualisierung:** Der Presenter transformiert das Domänen-Objekt in ein reines, für die UI optimiertes ViewModel, aktualisiert den Svelte-Store mit diesem ViewModel und entfernt den Ladezustand.
7.  **Reaktive UI-Aktualisierung:** Da die View den Svelte-Store abonniert hat, rendert sie sich automatisch neu und zeigt die neuen Daten an.

```mermaid
graph TD
    subgraph Frontend (Svelte/TS in WebView)
        A(View: Svelte-Komponente) -- 1. User-Interaktion --> B(Presenter: TypeScript-Modul);
        B -- 2. invoke() --> C(Tauri-Bridge);
        B -- 6. aktualisiert --> E(Store: Svelte Store);
        E -- 7. Re-Render --> A;
    end
    subgraph Backend (Rust)
        D(Rust-Command) -- 4. ruft auf --> F(Domänen-Service Schicht 3);
    end
    C -- 3. leitet weiter --> D;
    D -- 5. gibt Result zurück --> C;
    C --> B;
```

---

## 3. Kernkomponenten-Übersicht

| Komponente         | Technologie         | Verantwortlichkeit                                                                                              |
| ------------------ | ------------------- | --------------------------------------------------------------------------------------------------------------- |
| **Anwendungen**    | Svelte, TypeScript  | Implementiert die vollständigen, sichtbaren Core-Anwendungen (z.B. `Einstellungen`, `Dashboard`) durch Komposition. |
| **App-Kit**        | TypeScript          | Stellt ein gemeinsames Toolkit für alle Anwendungen bereit (z.B. Basis-Presenter, Store-Factories).             |
| **Features**       | Svelte, TypeScript  | Kapselt die UI und Präsentationslogik für anwendungsübergreifende Features (z.B. globale Suche, Kommando-Palette). |
| **Tauri-Brücke**   | Rust, `tauri`       | Definiert die `#[tauri::command]`-Funktionen, die als sichere Brücke zu den Domänen-Services dienen.             |

---

## 4. Detail-Spezifikationen

### 4.1. Anwendungen (`apps/`)
- **Zweck:** Das Zuhause der Core-Anwendungen von NovaDE.
- **Struktur:** Jede Anwendung hat ihr eigenes Unterverzeichnis (z.B. `apps/settings/`).
- **Implementierung:** Eine Anwendung besteht typischerweise aus mehreren "Seiten" (z.B. `Appearance.svelte`, `Sound.svelte`), die jeweils ihre eigenen Presenter und Stores haben und aus den generischen Komponenten aus Schicht 6 zusammengesetzt sind.
- **Beispiele:** `apps/settings`, `apps/file-browser`, `apps/dashboard`.

### 4.2. App-Kit (`app_kit/`)
- **Zweck:** Vermeidung von Code-Duplizierung und Sicherstellung eines konsistenten Architektur-Musters über alle Anwendungen hinweg.
- **Inhalt:**
  - `BasePresenter`: Eine Klasse oder ein Satz von Funktionen, die allgemeine Logik für Presenter enthalten (z.B. Fehlerbehandlung, Ladezustands-Management).
  - `createStore()`: Eine Factory-Funktion, die einen Svelte-Store mit einer standardisierten Struktur (z.B. `{ data, isLoading, error }`) erstellt.
  - `tauriApi.ts`: Ein typsicheres Modul, das die `invoke`-Aufrufe an das Backend kapselt.

### 4.3. Tauri-Brücke (`lib.rs`)
- **Zweck:** Definition der sicheren API, die das Frontend aufrufen kann.
- **Implementierung:** Die `lib.rs` des `novade-application-presentation`-Crates enthält die `main`-Funktion, die den Tauri-Anwendungs-Builder konfiguriert.
- **Commands:** Für jeden Service aus Schicht 3 wird hier eine entsprechende `#[tauri::command]`-Funktion definiert. Diese Funktion ist typischerweise ein einfacher Wrapper, der die Eingabeparameter vom Frontend entgegennimmt, die Methode des entsprechenden Service-Traits aufruft und das Ergebnis zurückgibt.

---

## 5. Zukünftige Entwicklung & Strategie

- **Anwendungsübergreifende Workflows:** Entwicklung von Mechanismen, die eine nahtlose Navigation und Datenübergabe zwischen den verschiedenen Core-Anwendungen ermöglichen (z.B. Klick auf eine Datei im `file-browser` öffnet diese in einer entsprechenden Editor-Anwendung).
- **State-Management-Optimierung:** Evaluierung von fortgeschritteneren State-Management-Bibliotheken (z.B. XState für komplexe Zustandsautomaten), um die Präsentationslogik für sehr komplexe Anwendungen (wie einen potenziellen Workflow-Editor) beherrschbar zu halten.
- **Offline-Fähigkeit:** Verbesserung der Anwendungen, sodass sie auch bei vorübergehendem Verlust der Verbindung zum Backend (in einem verteilten Szenario) oder bei langsamen Operationen einen sinnvollen, zwischengespeicherten Zustand anzeigen können.
- **Neue Core-Anwendungen:** Entwicklung weiterer strategischer Anwendungen wie:
  - `Workflow-Editor`: Eine grafische Oberfläche zur Erstellung und Verwaltung von Automatisierungen.
  - `AI-Assistant`: Eine Konversations-UI zur Interaktion mit den KI-Fähigkeiten des Systems.

---

## 6. Entwicklungs- und Qualitätsrichtlinien

- **Keine Geschäftslogik:** Es ist strikt verboten, in dieser Schicht Logik zu implementieren, die über die reine Darstellung und das Aufrufen von Backend-Diensten hinausgeht. Validierung, Berechnungen etc. gehören in Schicht 3.
- **Testbarkeit:** Die Präsentationslogik in den TypeScript-Modulen muss durch Unit-Tests (z.B. mit Vitest) abgedeckt sein. Dabei werden die `invoke`-Aufrufe an das Backend gemockt.
- **Komponenten wiederverwenden:** Es dürfen nur Komponenten aus Schicht 6 verwendet werden. Wird eine neue, generische Komponente benötigt, muss sie in Schicht 6 entwickelt und nicht lokal in einer Anwendung erstellt werden.
- **Performance:** Die Lade- und Reaktionszeiten der Anwendungen sind kritisch. Große Datenmengen müssen paginiert oder virtualisiert werden. Der initiale Payload jeder Anwendung muss so klein wie möglich sein.