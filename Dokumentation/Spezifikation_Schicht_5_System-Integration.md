# Schicht 5: System-Integration - Ultra-Feinspezifikation v2.0

**Dokumenten-Status:** Aktiv
**Version:** 2.0
**Letzte Aktualisierung:** 2025-08-16

## 1. Vision & Philosophie

### 1.1. Zweck und Geltungsbereich

Dieses Dokument definiert die **System-Integrationsschicht (Schicht 5)**, den diplomatischen Arm und den "Anti-Corruption-Layer" von NovaDE. Ihre Mission ist es, die saubere, interne Welt der Domänen (Schicht 2 & 3) mit dem oft unstrukturierten, externen Linux-Ökosystem zu verbinden. Sie implementiert die von den unteren Schichten definierten abstrakten Schnittstellen (Traits) und übersetzt zwischen den internen Domänen-Modellen und den externen Datenformaten.

Der Geltungsbereich dieser Schicht umfasst die konkrete Implementierung von Persistenz, die Kommunikation mit System-Daemons über D-Bus, die Anbindung an XDG-Desktop-Portals und die Bereitstellung der konkreten Service-Implementierungen, die von der UI genutzt werden.

### 1.2. Architektonische Prinzipien

- **Anti-Corruption-Layer (ACL):** Das oberste Gebot ist der Schutz des Domänen-Modells. Kein externer Datentyp oder keine externe API-Logik darf in die Schichten 1-3 durchsickern. Diese Schicht agiert als Übersetzer und Puffer.
- **Adapter-Muster:** Für jede externe Interaktion (Datenbank, D-Bus-Service, Dateisystem) wird ein spezifischer "Adapter" implementiert. Dies isoliert die Komplexität und macht die externen Abhängigkeiten austauschbar.
- **Asynchronität und Resilienz:** Alle externen Aufrufe (I/O, Netzwerk, D-Bus) MÜSSEN asynchron sein, um die Reaktionsfähigkeit des Systems zu gewährleisten. Die Implementierungen müssen robust gegenüber Timeouts, Fehlern und unerwarteten Daten des externen Systems sein.
- **Abhängigkeiten:** Diese Schicht ist die erste, die das Gesamtsystem zusammenfügt. Sie hängt von den Schichten 1, 2, 3 und 4 ab und nutzt deren APIs und Modelle, um ihre Aufgaben zu erfüllen.

---

## 2. Kernkomponenten-Übersicht

Die Schicht 5 besteht aus mehreren logischen Kernkomponenten, die in einer Vielzahl von Modulen implementiert sind.

| Komponente                 | Verantwortlichkeit                                                                                                   |
| -------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| **Persistenz-Adapter**     | Implementiert die `Repository`-Traits aus Schicht 2. Speichert und lädt Domänen-Objekte.                             |
| **D-Bus-Adapter**          | Implementiert typsichere, asynchrone Clients für Systemdienste (UPower, NetworkManager etc.) via D-Bus.             |
| **Service-Implementierungen** | Stellt die konkreten `impl` für die `Service`-Traits aus Schicht 3 bereit. Orchestriert die Adapter.                |
| **Event-Publisher**        | Implementiert einen zentralen Event-Bus, über den die Domänen-Services (Schicht 3) Events publizieren können.         |
| **Portal-Integration**     | Implementiert die Backend-Logik für XDG-Desktop-Portals (z.B. für Datei-Öffnen-Dialoge).                             |
| **System-Health-Kollektoren** | Sammelt spezifische Systemmetriken (CPU, RAM) und stellt sie dem `SystemHealthMonitor`-Service zur Verfügung.      |

---

## 3. Detail-Spezifikationen

### 3.1. Persistenz-Adapter
- **Zweck:** Konkrete Implementierung der in Schicht 2 definierten `...Repository`-Traits.
- **Module:** `persistence/`, `global_settings_persistence/`, `workspace_persistence/`, `settings_storage.rs`.
- **Beispiel-Implementierungen:**
  - **`SettingsRepository`**: Wird durch das Modul `global_settings_persistence` implementiert. Nutzt den `ConfigServiceAsync` aus Schicht 1, um die `GlobalDesktopSettings` als TOML-Datei im XDG-Konfigurationsverzeichnis zu speichern.
  - **`WorkspaceRepository`**: Wird durch `workspace_persistence` implementiert. Speichert den Zustand der Workspaces als JSON-Datei.
  - **`ThemeRepository`**: Wird durch `theme_integration` implementiert. Liest Theme-Definitionen aus den XDG-Datenverzeichnissen (z.B. `/usr/share/novade/themes`).

### 3.2. D-Bus-Adapter
- **Zweck:** Typsichere, asynchrone Kommunikation mit externen Systemdiensten.
- **Technologie:** Nutzt das `zbus`-Crate.
- **Module:** `dbus_clients/`, `dbus_integration/`, `power_management/`, `network_manager/`.
- **Architektur:**
  1. Das `zbus`-Tooling wird verwendet, um aus den D-Bus-Introspektions-XML-Dateien der Dienste typsichere Rust-Proxies zu generieren.
  2. Ein Adapter-Modul (z.B. `power_management`) wrappt diesen Proxy.
  3. Der Adapter fängt D-Bus-Signale ab (z.B. `PropertiesChanged` von UPower) und übersetzt die D-Bus-Datentypen in die internen Domänen-Modelle aus Schicht 2 (z.B. `PowerState`, `BatteryStatus`).
  4. Der Adapter löst dann ein internes Domänen-Event aus (z.B. `PowerEvent::BatteryLevelChanged`).

### 3.3. Service-Implementierungen
- **Zweck:** Die konkrete Implementierung der in Schicht 3 definierten `Service`-Traits.
- **Module:** `system_services/`, `services/` und viele der Root-`.rs`-Dateien.
- **Beispiel `impl ThemingEngine for SystemServices`**:
  - Die Methode `set_active_theme` würde den `ThemeRepository`-Adapter (Persistenz) aufrufen, um das Theme zu laden, und den `SettingsRepository`-Adapter, um den Namen des neuen Themes in den globalen Einstellungen zu speichern. Anschließend würde es ein `ActiveThemeChanged`-Event über den `EventPublisher` publizieren.

### 3.4. Event-Publisher
- **Zweck:** Entkopplung der Systemkomponenten.
- **Modul:** `event_publisher/`.
- **Architektur:**
  1. Definiert einen `DomainEventPublisher`-Trait, der von den Services aus Schicht 3 per Dependency Injection erhalten wird.
  2. Implementiert diesen Trait mit einem asynchronen Multi-Producer-Multi-Consumer-Channel (z.B. `tokio::sync::broadcast`).
  3. Andere Adapter in Schicht 5 (z.B. D-Bus-Adapter, die auf interne Events lauschen) und die UI-Schichten können den `Receiver`-Teil des Channels abonnieren, um auf Events zu reagieren.

---

## 4. Zukünftige Entwicklung & Strategie

- **PolicyKit-Integration:** Implementierung eines Adapters für PolicyKit, um Aktionen, die administrative Rechte erfordern (z.B. Installation von System-Updates), sicher durchzuführen.
- **Erweiterte Portal-Unterstützung:** Integration weiterer XDG-Desktop-Portals wie `org.freedesktop.portal.ScreenCast` für die Bildschirmaufnahme und `org.freedesktop.portal.Secret` für eine standardisierte, sichere Speicherung von Passwörtern.
- **Account-Synchronisation:** Entwicklung eines Adapters für `org.freedesktop.portal.Account`, um Online-Konten (z.B. Google, Microsoft) in das System zu integrieren.
- **Resilienz erhöhen:** Implementierung von Caching-Strategien in den Adaptern, um die UI auch dann reaktionsfähig zu halten, wenn ein externer Dienst (z.B. NetworkManager) vorübergehend nicht verfügbar ist.

---

## 5. Entwicklungs- und Qualitätsrichtlinien

- **Integrationstests:** Diese Schicht ist der primäre Ort für Integrationstests. Wo immer möglich, sollten Tests gegen echte (aber gemockte oder temporäre) externe Dienste laufen. Zum Beispiel sollte ein Test für den `UPower`-Adapter einen echten D-Bus-Service auf einem Test-Bus starten.
- **Fehler-Mapping:** Jeder Adapter ist dafür verantwortlich, die spezifischen Fehler der externen Bibliothek (z.B. `zbus::Error`, `rusqlite::Error`) in die entsprechenden Varianten von `CoreError` oder spezifischere Fehler zu übersetzen.
- **Daten-Validierung:** Jeder Adapter muss die von externen Quellen empfangenen Daten validieren und normalisieren, bevor er sie in Domänen-Modelle umwandelt. Vertraue niemals den Daten von außerhalb.
- **Dokumentation:** Die Mapping-Logik (von externen zu internen Modellen) und die Fehlerbehandlungsstrategien für jeden Adapter müssen klar im Code (`// NARRATE:`) dokumentiert sein.