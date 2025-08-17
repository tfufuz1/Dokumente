# Schicht 2: Domänen-Modell - Ultra-Feinspezifikation v2.0

**Dokumenten-Status:** Aktiv
**Version:** 2.0
**Letzte Aktualisierung:** 2025-08-16

## 1. Vision & Philosophie

### 1.1. Zweck und Geltungsbereich

Dieses Dokument definiert das **Domänen-Modell (Schicht 2)**, das Herzstück der Anwendungslogik von NovaDE. Es repräsentiert die "Nomen" des Systems – die zentralen Konzepte, Entitäten, Wertobjekte und Aggregate, die die Geschäftsregeln und das Wissen der Domäne kapseln. Diese Schicht ist eine reine, ausdrucksstarke und technologie-agnostische Darstellung der NovaDE-Vision.

Der Geltungsbereich dieser Schicht ist die Definition der Datenstrukturen und ihrer inhärenten Invarianten. Sie enthält **keine Anwendungslogik** (diese gehört in Schicht 3) und hat **keinerlei Wissen** über Persistenz, UI oder externe Systeme.

### 1.2. Architektonische Prinzipien

- **Domain-Driven Design (DDD) Purismus:** Das Modell wird streng nach den Prinzipien von DDD entworfen. Entitäten haben eine eindeutige Identität, Wertobjekte sind unveränderlich und durch ihre Attribute definiert, und Aggregate schützen ihre internen Invarianten und dienen als Transaktionsgrenzen.
- **Reichhaltiges Domänen-Modell:** Das Modell soll nicht nur aus anämischen Datencontainern bestehen. Es soll Verhalten kapseln, das untrennbar mit den Daten verbunden ist (z.B. Validierungslogik in Konstruktoren).
- **Persistenz-Agnostizismus:** Das Modell ist vollständig von der Art der Speicherung entkoppelt. Die Kommunikation mit der Persistenzschicht erfolgt ausschließlich über die hier definierten `Repository`-Traits.
- **Minimale Abhängigkeiten:** Das `novade-domain-model`-Crate hängt nur von `novade-core` (für Basistypen und Fehler) und sorgfältig ausgewählten, fundamentalen Bibliotheken wie `serde` und `uuid` ab.

---

## 2. Domänen-Modul-Übersicht

Das Domänen-Modell ist in klar abgegrenzte Sub-Domänen ("Bounded Contexts") unterteilt, die jeweils ein spezifisches Geschäftsfeld abbilden.

| Modul-Pfad                      | Verantwortlichkeit                                                                    |
| ------------------------------- | --------------------------------------------------------------------------------------- |
| `domain::application`           | Repräsentiert installierte Anwendungen und deren Metadaten.                             |
| `domain::audio`                 | Modelle für Audio-Geräte, Lautstärke und Streams.                                     |
| `domain::clipboard`             | Definiert die Struktur von Einträgen in der Zwischenablage-Historie.                    |
| `domain::display`               | Modelle für physische Monitore, Auflösungen und deren Konfiguration.                    |
| `domain::filesystem`            | Abstraktionen für Dateien und Verzeichnisse als Domänenkonzepte.                        |
| `domain::global_settings`       | Definiert das zentrale Aggregat für alle benutzerspezifischen Einstellungen.             |
| `domain::keybindings`           | Modelle für Tastenkürzel, Akkorde und deren Aktionen.                                  |
| `domain::menu`                  | Strukturen für Anwendungs- oder Kontextmenüs.                                         |
| `domain::network`               | Modelle für Netzwerkverbindungen (WLAN, Ethernet) und deren Status.                     |
| `domain::notifications`         | Definiert die Struktur von System-Benachrichtigungen.                                   |
| `domain::power`                 | Modelle für den Batteriestatus, Energieprofile und Energiequellen.                      |
| `domain::search`                | Definiert Suchanfragen, Ergebnisse und durchsuchbare Quellen als Domänenkonzepte.       |
| `domain::system_health`         | Modelle für Systemmetriken wie CPU-, RAM- und Festplattenauslastung.                   |
| `domain::terminal`              | Domänenkonzepte für die Terminal-Emulation (z.B. PTY-Zustand).                         |
| `domain::theming`               | Definiert das Design-System mit Themes, Tokens und Stil-Definitionen.                   |
| `domain::user`                  | Repräsentiert den Benutzer und seine Präferenzen.                                      |
| `domain::window_management`     | Enthält Modelle für Fenster, deren Zustände und Eigenschaften.                          |
| `domain::workspaces`            | Definiert das Konzept der virtuellen Arbeitsbereiche (Workspaces) als Aggregate.          |

---

## 3. Detail-Spezifikationen (Ausgewählte Domänen)

### 3.1. Domäne: `workspaces`
- **`Workspace` (Aggregate Root):** Repräsentiert einen einzelnen virtuellen Desktop.
  - **Attribute:** `id: WorkspaceId`, `name: String`, `layout: WorkspaceLayoutType`, `windows: BTreeSet<WindowHandle>`, `creation_timestamp: TimestampNanoseconds`.
  - **Invarianten:** Der Name muss eindeutig sein. Ein Fenster kann nur in einem Workspace existieren.
- **`WindowHandle` (Value Object):** Eine typsichere ID für ein Anwendungsfenster.
- **`WorkspaceLayoutType` (Enum):** `Tiling`, `Floating`, `Monocle`.

### 3.2. Domäne: `theming`
- **`Theme` (Aggregate Root):** Repräsentiert ein komplettes Theme.
  - **Attribute:** `id: DomainId`, `name: String`, `author: Option<String>`, `tokens: HashMap<String, ThemeToken>`.
- **`ThemeToken` (Value Object):** Ein einzelner Design-Token (z.B. eine Farbe oder Schriftgröße).
  - **Attribute:** `token_type: ThemeTokenType`, `value: String`.
- **`ThemeTokenType` (Enum):** `Color`, `FontSize`, `Spacing`, `BorderRadius`, `FontFamily`.

### 3.3. Domäne: `notifications`
- **`Notification` (Entity):** Eine einzelne System-Benachrichtigung.
  - **Attribute:** `id: DomainId`, `app_name: String`, `summary: String`, `body: Option<String>`, `actions: Vec<NotificationAction>`, `urgency: NotificationUrgency`, `timestamp: TimestampNanoseconds`.
- **`NotificationAction` (Value Object):** Eine Aktion, die mit einer Benachrichtigung verknüpft ist.
- **`NotificationUrgency` (Enum):** `Low`, `Normal`, `Critical`.

### 3.4. Domäne: `global_settings`
- **`GlobalDesktopSettings` (Aggregate Root):** Bündelt alle globalen, benutzerspezifischen Einstellungen an einem Ort. Dient als zentrale Anlaufstelle für die Konfiguration.
  - **Attribute:** `appearance: AppearanceSettings`, `workspace_config: WorkspaceSettings`, `input_behavior: InputBehaviorSettings`, `power_management: PowerManagementPolicySettings`, `keybinding_config: KeybindingSettings`.
- **Beispiel-Wertobjekte:** `AppearanceSettings { theme_name: String, ... }`, `WorkspaceSettings { dynamic_workspaces: bool, ... }`.

### 3.5. Domäne: `keybindings`
- **`Keybinding` (Entity):** Repräsentiert eine einzelne Tastenkombination und die damit verbundene Aktion.
  - **Attribute:** `id: DomainId`, `combination: KeyCombination`, `action: KeybindingAction`.
- **`KeyCombination` (Value Object):** Stellt die Tastenkombination dar (z.B. `Ctrl+Shift+T`).
- **`KeybindingAction` (Value Object):** Definiert die auszuführende Aktion (z.B. `SpawnCommand("alacritty")`, `SwitchToWorkspace(WorkspaceId)`).

---

## 4. Repository-Schnittstellen (Traits)

Diese `async` Traits definieren die Verträge, die die Persistenzschicht (Schicht 5) implementieren muss, um die Domänenobjekte zu speichern und zu laden. Sie stellen sicher, dass das Domänen-Modell agnostisch gegenüber der konkreten Speichertechnologie (z.B. SQLite, TOML-Dateien) bleibt.

- **`trait ThemeRepository`**
  - `async fn get_by_id(&self, id: &DomainId) -> Result<Option<Theme>, CoreError>;`
  - `async fn save(&self, theme: &Theme) -> Result<(), CoreError>;`
  - `async fn get_all(&self) -> Result<Vec<Theme>, CoreError>;`

- **`trait WorkspaceRepository`**
  - `async fn get_all_workspaces(&self) -> Result<Vec<Workspace>, CoreError>;`
  - `async fn save_all_workspaces(&self, workspaces: &[Workspace]) -> Result<(), CoreError>;`

- **`trait SettingsRepository`**
  - `async fn load_settings(&self) -> Result<GlobalDesktopSettings, CoreError>;`
  - `async fn save_settings(&self, settings: &GlobalDesktopSettings) -> Result<(), CoreError>;`

- **`trait KeybindingRepository`**
  - `async fn get_all_keybindings(&self) -> Result<Vec<Keybinding>, CoreError>;`
  - `async fn save_all_keybindings(&self, keybindings: &[Keybinding]) -> Result<(), CoreError>;`

---

## 5. Zukünftige Entwicklung & Strategie

- **Domäne `ai`:** Dieses Modell wird entscheidend sein. Es wird Strukturen für `AIContext`, `PromptTemplate`, `ModelResponse` und `ConversationHistory` definieren, um eine tiefe, kontextsensitive KI-Integration zu ermöglichen.
- **Domäne `tracking`:** Hier werden Modelle für die Erfassung von Nutzungsmustern (z.B. `AppUsageFrequency`, `WorkflowExecution`) definiert, um die Grundlage für proaktive Vorschläge und Automatisierungen zu schaffen.
- **Validierung stärken:** Die Validierungslogik innerhalb der Domänenobjekte wird weiter ausgebaut, um ungültige Zustände zur Kompilierzeit oder so früh wie möglich zur Laufzeit zu verhindern (z.B. durch die Einführung von `NonEmptyString`-Typen).
- **Domänen-Events:** Formalisierung eines Musters für Domänen-Events. Wenn ein Aggregat seinen Zustand ändert (z.B. `Workspace::add_window`), soll es ein Event (`WindowAddedToWorkspace`) generieren, das von Schicht 3 publiziert werden kann.

---

## 6. Entwicklungs- und Qualitätsrichtlinien

- **Serialisierung:** Alle Entitäten und Wertobjekte MÜSSEN `serde::Serialize` und `serde::Deserialize` implementieren, um Persistenz und IPC zu ermöglichen.
- **Invarianten durchsetzen:** Das Typsystem ist das primäre Werkzeug, um Geschäftsregeln durchzusetzen. Konstruktoren und Factory-Methoden sind die Wächter, die sicherstellen, dass keine ungültigen Objekte erstellt werden können.
- **Tests:** Die Geschäftslogik, die in den Domänenobjekten gekapselt ist (insbesondere Validierungen), muss zu 100% durch Unit-Tests abgedeckt sein.
- **Dokumentation:** Jedes Domänen-Modul, jede Entität und jedes Wertobjekt muss klar dokumentiert sein und seinen Zweck und seine Invarianten erläutern.