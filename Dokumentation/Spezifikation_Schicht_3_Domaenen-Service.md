# Schicht 3: Domänen-Service - Ultra-Feinspezifikation v2.0

**Dokumenten-Status:** Aktiv
**Version:** 2.0
**Letzte Aktualisierung:** 2025-08-16

## 1. Vision & Philosophie

### 1.1. Zweck und Geltungsbereich

Dieses Dokument definiert die **Domänen-Service-Schicht (Schicht 3)**, das Gehirn der NovaDE-Anwendung. Hier werden die "Verben" des Systems implementiert – die zustandslose Geschäftslogik, die die reinen Domänen-Modelle aus Schicht 2 zu kohärenten Anwendungsfällen ("Use Cases") orchestriert. Diese Schicht ist die einzige, die Domänen-Modelle verändern darf und die die Geschäftsregeln des Systems durchsetzt.

Der Geltungsbereich dieser Schicht ist die Kapselung der gesamten Anwendungslogik. Sie agiert als die primäre API für die darüberliegenden UI-Schichten (6 und 7) und delegiert Persistenz- und System-Interaktionen an die darunterliegenden Schichten (5).

### 1.2. Architektonische Prinzipien

- **Service-Orientierung:** Jeder Service hat eine klar abgegrenzte Verantwortung (Single Responsibility Principle), die sich an den Sub-Domänen aus Schicht 2 orientiert.
- **Zustandslosigkeit:** Domänen-Services halten keinen eigenen Zustand. Sie empfangen Daten (oft über Repository-Interfaces), operieren darauf und geben das Ergebnis zurück oder lösen Events aus. Der Zustand wird ausschließlich in den Domänen-Modellen und der Persistenzschicht gehalten.
- **Event-getriebene Architektur:** Jede signifikante Zustandsänderung, die von einem Service durchgeführt wird, MUSS ein Domänen-Event auslösen. Diese Events (z.B. `WorkspaceCreated`, `ThemeChanged`) sind der primäre Mechanismus zur Entkopplung der Systemkomponenten.
- **Abhängigkeiten:** Diese Schicht hängt nur von Schicht 1 (`novade-core`) und Schicht 2 (`novade-domain-model`) ab. Sie definiert die Schnittstellen (`Traits`), die von Schicht 5 implementiert werden, kennt aber deren konkrete Implementierung nicht (Dependency Inversion Principle).

---

## 2. Service-Modul-Übersicht

Jede Sub-Domäne aus Schicht 2 hat einen korrespondierenden Service in Schicht 3, der die Geschäftslogik für diese Domäne kapselt.

| Service-Trait (in `services.rs`) | Verantwortlichkeit                                                                                              |
| -------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| `WorkspaceManager`               | Orchestriert alle Operationen rund um virtuelle Workspaces.                                                     |
| `ThemingEngine`                  | Verwaltet die Anwendung von Themes und die Auflösung von Design-Tokens.                                         |
| `NotificationManager`            | Dient als zentraler Punkt für die Erstellung, Verwaltung und Interaktion mit System-Benachrichtigungen.         |
| `SettingsManager`                | Bietet eine validierte Schnittstelle zum Lesen und Schreiben der `GlobalDesktopSettings`.                       |
| `KeybindingManager`              | Lädt, validiert und verwaltet die aktiven Tastenkürzel des Systems.                                             |
| `PowerManager`                   | Kapselt die Logik für Energieprofile und reagiert auf Änderungen des Batteriestatus.                            |
| `AudioManager`                   | Verwaltet Audio-Geräte, steuert die Lautstärke und behandelt Audio-Streams.                                     |
| `SystemHealthMonitor`            | Überwacht Systemmetriken und implementiert die Logik zur Auslösung von Warnungen bei Schwellenwertüberschreitungen. |
| `ApplicationManager`             | Verwaltet den Lebenszyklus von Anwendungen (Starten, Stoppen, Abfragen von Informationen).                      |
| `WindowPolicyManager`            | Implementiert Regeln für die automatische Platzierung und das Tiling-Verhalten von Fenstern.                    |
| `SearchService`                  | Aggregiert Suchergebnisse von verschiedenen Anbietern und wendet kontextsensitive Gewichtungen an.              |
| `WorkflowEngine`                 | Orchestriert komplexe, benutzerdefinierte Workflows, die mehrere andere Services involvieren.                   |

---

## 3. Detail-Spezifikationen (Ausgewählte Service-Traits)

Die öffentliche API dieser Schicht wird durch asynchrone `async` Traits definiert, die in `services.rs` zusammengefasst sind.

### 3.1. `trait WorkspaceManager`
- **Verantwortlichkeiten:** Verwalten der Sammlung aller Workspaces, des aktiven Workspace und der Zuweisung von Fenstern.
- **Methoden (Beispiele):**
  - `async fn create_workspace(&self, name: Option<String>) -> Result<Workspace, WorkspaceError>;`
  - `async fn set_active_workspace(&self, id: &WorkspaceId) -> Result<(), WorkspaceError>;`
  - `async fn assign_window_to_workspace(&self, window_id: &WindowHandle, workspace_id: &WorkspaceId) -> Result<(), WorkspaceError>;`
- **Events:** Löst `WorkspaceCreated`, `WorkspaceDeleted`, `ActiveWorkspaceChanged` aus.

### 3.2. `trait ThemingEngine`
- **Verantwortlichkeiten:** Anwenden von Themes und Bereitstellen des aktuellen, aufgelösten Zustands für die UI.
- **Methoden (Beispiele):**
  - `async fn set_active_theme(&self, theme_id: &DomainId) -> Result<(), ThemingError>;`
  - `async fn get_current_applied_state(&self) -> Result<AppliedThemeState, ThemingError>;`
- **Events:** Löst `ActiveThemeChanged` aus.

### 3.3. `trait SettingsManager`
- **Verantwortlichkeiten:** Sicherer und validierter Zugriff auf die globalen Einstellungen.
- **Methoden (Beispiele):**
  - `async fn get_settings(&self) -> Result<GlobalDesktopSettings, SettingsError>;`
  - `async fn update_settings(&self, new_settings: &GlobalDesktopSettings) -> Result<(), SettingsError>;`
  - `async fn update_single_setting(&self, key_path: &str, value: serde_json::Value) -> Result<(), SettingsError>;`
- **Events:** Löst `SettingsChanged { changed_keys: Vec<String> }` aus.

### 3.4. `trait KeybindingManager`
- **Verantwortlichkeiten:** Laden und Aktivieren der benutzerdefinierten Tastenkürzel.
- **Methoden (Beispiele):**
  - `async fn reload_keybindings(&self) -> Result<Vec<Keybinding>, KeybindingError>;`
  - `async fn get_action_for_combination(&self, combination: &KeyCombination) -> Result<Option<KeybindingAction>, KeybindingError>;`
- **Events:** Löst `KeybindingsReloaded` aus.

---

## 4. Domänen-Events (`common_events.rs`)

Domänen-Events sind das Rückgrat der Entkopplung in NovaDE. Sie werden von den Services in dieser Schicht erzeugt und über einen `DomainEventPublisher`-Trait (implementiert in Schicht 5) im System verteilt. Andere Teile des Systems können diese Events abonnieren, um darauf zu reagieren, ohne den ursprünglichen Service kennen zu müssen.

- **Beispiel-Events:**
  - `enum WorkspaceEvent { Created(Workspace), Deleted(WorkspaceId), ... }`
  - `enum ThemingEvent { ThemeChanged { theme_id: DomainId, theme_name: String } }`
  - `enum SettingsEvent { SettingUpdated { key: String, new_value: serde_json::Value } }`
  - `enum PowerEvent { BatteryLevelChanged(u8), PowerSourceChanged(PowerSource), ... }`

- **Zweck:** Ein Event wie `PowerEvent::BatteryLevelChanged` kann von der UI abonniert werden, um das Batterie-Icon zu aktualisieren, und gleichzeitig vom `SystemHealthMonitor`, um bei niedrigem Akkustand eine Warnung auszulösen.

---

## 5. Zukünftige Entwicklung & Strategie

- **Proaktive Services:** Die Services werden zunehmend proaktive Logik enthalten. Beispiel: Der `SystemHealthMonitor` könnte nicht nur warnen, sondern dem `PowerManager` proaktiv vorschlagen, in ein Energiesparprofil zu wechseln.
- **Workflow-Orchestrierung:** Der `WorkflowEngine`-Service wird eine zentrale Rolle einnehmen. Er wird es Benutzern ermöglichen, komplexe Abläufe zu definieren (z.B. "Wenn ich Workspace 'Dev' öffne, starte VS Code und Docker Desktop"), die Aktionen über mehrere Domänen-Services hinweg koordinieren.
- **KI-Integration:** Der `AIConnector`-Service wird die Brücke zu externen KI-Modellen schlagen. Höherlevelige Services wie ein `ContextualSearchService` werden ihn nutzen, um Anfragen mit dem aktuellen Systemkontext (aktive Anwendung, offene Fenster etc.) anzureichern und intelligentere Ergebnisse zu liefern.
- **Feingranulare Fehler:** Die spezifischen Fehler-Enums (z.B. `WorkspaceError`) werden weiter verfeinert, um den UI-Schichten eine noch präzisere Fehlerbehandlung und Darstellung zu ermöglichen.

---

## 6. Entwicklungs- und Qualitätsrichtlinien

- **Fehlerbehandlung:** Jeder Service MUSS sein eigenes, spezifisches Fehler-Enum mit `thiserror` definieren, das von `CoreError` erbt. Dies ermöglicht eine klare Trennung der Fehlerdomänen.
- **Tests:** Die Geschäftslogik jedes Services muss durch Unit-Tests abgedeckt sein. Dabei werden die Abhängigkeiten zu den Repositories (definiert in Schicht 2) durch Mock-Implementierungen ersetzt, um die Logik isoliert zu testen.
- **Dokumentation:** Jeder Anwendungsfall (d.h. jede öffentliche Service-Methode) muss klar dokumentiert sein, einschließlich der Vor- und Nachbedingungen sowie der ausgelösten Domänen-Events.
- **Keine Seiteneffekte (außer Events):** Eine Service-Methode sollte entweder einen Wert zurückgeben oder ein Event auslösen. Direkte Aufrufe an andere Services sollten vermieden werden; die Kommunikation erfolgt primär über Events.