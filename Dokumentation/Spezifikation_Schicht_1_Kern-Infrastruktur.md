# Schicht 1: Kern-Infrastruktur - Ultra-Feinspezifikation v2.0

**Dokumenten-Status:** Aktiv
**Version:** 2.0
**Letzte Aktualisierung:** 2025-08-16

## 1. Vision & Philosophie

### 1.1. Zweck und Geltungsbereich

Dieses Dokument definiert die **Kern-Infrastrukturschicht (Schicht 1)**, die fundamentalste Ebene der 7-Schichten-Architektur von NovaDE. Sie ist das unerschütterliche Fundament, auf dem alle anderen Schichten aufbauen. Ihr alleiniger Zweck ist die Bereitstellung eines stabilen, minimalen, hochgradig zuverlässigen und performanten Satzes von grundlegenden Werkzeugen, Typen und Diensten, die im gesamten System wiederverwendet werden.

Der Geltungsbereich dieser Schicht ist streng auf Funktionalitäten beschränkt, die **keinerlei Abhängigkeiten** zu anderen NovaDE-Schichten (Domäne, System, UI) aufweisen. Sie ist vollständig eigenständig und dient als die primäre interne Bibliothek für das gesamte NovaDE-Projekt.

### 1.2. Architektonische Prinzipien

- **Stabilität & Verlässlichkeit:** APIs in dieser Schicht sind heilig. Änderungen haben weitreichende Konsequenzen und erfordern höchste Sorgfalt. Jeder Code muss robust, getestet und produktionsreif sein.
- **Minimalismus & Kohäsion:** Die Schicht enthält nur das, was wirklich von mehreren anderen Schichten benötigt wird. Jedes Modul hat eine klare, abgegrenzte Verantwortung.
- **Performance-Obsession:** Jede Komponente ist auf minimalen Ressourcenverbrauch (CPU, Speicher) und maximale Performance ausgelegt. Overhead wird aktiv vermieden.
- **Null-Abhängigkeiten (Intern):** Das `novade-core`-Crate darf keine Abhängigkeiten zu `novade-domain`, `novade-system` oder `novade-ui` haben.
- **Sicherheit by Design:** Kein `unsafe` Code außerhalb von absolut notwendigen FFI-Grenzen. Sichere Standardwerte und rigorose Validierung sind die Norm.

---

## 2. Modul-Übersicht

Die Kern-Infrastrukturschicht wird im Rust-Crate `novade-core` implementiert. Sie besteht aus den folgenden Hauptmodulen:

| Modul                      | Verantwortlichkeit                                                                                             |
| -------------------------- | -------------------------------------------------------------------------------------------------------------- |
| `core::types`              | Definition fundamentaler, serialisierbarer Datentypen (Geometrie, IDs, Farben etc.).                           |
| `core::errors`             | Bereitstellung des zentralen Fehlerbehandlungs-Frameworks mit `CoreError` und `ErrorCode`.                     |
| `core::config`             | Laden, Parsen, Validieren und Verwalten der Kernkonfiguration, inklusive Hot-Reloading.                        |
| `core::logging`            | Initialisierung und Konfiguration des systemweiten, strukturierten Loggings (`tracing`).                       |
| `core::error_tracking`     | Integration mit externen Fehler-Tracking-Diensten (Sentry) zur Erfassung und Meldung von Fehlern.              |
| `core::in_memory_log_buffer` | Ein `tracing`-Layer, der die letzten Log-Nachrichten im Speicher puffert, um sie bei Fehlern anzuhängen.         |
| `core::clipboard`          | Definition der reinen Datenstrukturen und Fehler für die Zwischenablage (`ClipboardItem`, `ClipboardError`).     |
| `core::utils`              | Sammlung von zustandslosen, systemweit nützlichen Hilfsfunktionen (Hashing, Pfad- und String-Manipulation etc.). |

---

## 3. Detail-Spezifikationen

### 3.1. Modul: `core::types` – Fundamentale Datentypen

Dieses Modul definiert einen Satz von primitiven, wiederverwendbaren und serialisierbaren Datentypen. Alle Typen müssen `Debug`, `Clone`, `PartialEq` und, wo sinnvoll, `Copy`, `Eq`, `Hash`, `Default`, `Serialize`, `Deserialize` implementieren.

- **`types::geometry`**: Stellt geometrische Primitive bereit.
  - `Point<T>`, `Size<T>`, `Rect<T>` mit Typaliasen `PointI32`, `SizeU32`, `RectI32`.
  - Implementiert Methoden wie `distance()`, `scale()`, `intersects()`, `contains_point()`.
- **`types::id`**: Implementiert einen typsicheren ID-Wrapper.
  - `TypedId<T>`: Ein Wrapper um `uuid::Uuid`, der durch `PhantomData` Typsicherheit zwischen verschiedenen IDs (z.B. `WorkspaceId` vs. `WindowId`) gewährleistet.
- **`types::aliases`**: Definiert konkrete Typaliase für IDs zur besseren Lesbarkeit.
  - `DomainId`, `WorkspaceId`, `WindowId`.
- **`types::color`**: Definiert einen Farbentyp.
  - `RgbaColor`: Repräsentiert eine RGBA-Farbe mit Methoden zur Konvertierung von/zu Hex-Strings.
- **`types::app_identifier`**:
  - `AppIdentifier`: Ein validierter Wrapper für Anwendungs-IDs (z.B. "org-novade-settings").
- **`types::status`**:
  - `Status`: Ein Enum (`Enabled`, `Disabled`, `Pending`, `Error(i32)`) zur Darstellung des Zustands von Komponenten.
- **`types::orientation`**:
  - `Orientation`: Enum für `Horizontal` / `Vertical`.
  - `Direction`: Enum für die vier Himmelsrichtungen (`North`, `South`, `East`, `West`).
- **`types::time`**:
  - `TimestampNanoseconds`: Zeitstempel mit Nanosekunden-Präzision.
  - `DurationNanoseconds`: Dauer mit Nanosekunden-Präzision.
- **`types::entity`**:
  - `Version`: Eine Struktur zur Versionierung von Entitäten.
  - `Identifiable`, `Versionable`: Traits für Entitäten mit ID und Version.

### 3.2. Modul: `core::errors` – Fehlerbehandlungs-Framework

- **`CoreError` Enum**: Definiert mit `thiserror` als die Basis für alle Fehler im System. Es wrappt spezifischere Fehler aus dieser Schicht.
  - **Varianten:** `Io`, `Logging`, `Serialization`, `Config`, `Clipboard`, `Initialization`, `InvalidState`, `NotFound`, `NotImplemented`, `UserFeedbackFailed`.
- **`ErrorCode` Enum**: Ein strukturierter, maschinenlesbarer Fehlercode, der eine programmatische Behandlung von Fehlern ermöglicht. Die `code()`-Methode auf `CoreError` liefert den entsprechenden `ErrorCode`.
- **Richtlinie:** Jedes Modul in jeder Schicht definiert sein eigenes spezifisches Fehler-Enum (z.B. `ConfigError`), das dann mittels `#[from]` in `CoreError` oder einen Fehler einer höheren Schicht konvertiert wird.

### 3.3. Modul: `core::config` – Konfigurations-Management

- **Strukturen**: Definiert die gesamte Konfigurationsstruktur des Cores.
  - `CoreConfig`: Die Wurzel-Konfiguration, die alle anderen umfasst.
  - `LoggingConfig`, `ErrorTrackingConfig`, `ConfigWatcherConfig`, `InMemoryLogBufferConfig`.
- **Globales Management**:
  - `initialize_core_config()`: Funktion, die einmalig beim Start aufgerufen wird, um die Konfiguration zu laden und global verfügbar zu machen.
  - `get_core_config()`: Gibt eine statische Referenz auf die geladene Konfiguration zurück.
  - `subscribe_to_config_changes()`: Ermöglicht anderen Diensten, auf Konfigurationsänderungen zur Laufzeit zu reagieren (Hot-Reloading).
- **Laden & Speichern**:
  - `ConfigLoader`: Eine Struktur, die den `ConfigServiceAsync`-Trait nutzt, um Konfigurationen zu laden und zu speichern.
- **Service-Abstraktion**:
  - `ConfigServiceAsync` Trait: Definiert eine asynchrone Schnittstelle zum Lesen/Schreiben von Konfigurationen, um die Logik von der Speicherung (z.B. Dateisystem) zu entkoppeln.
  - `FilesystemConfigService`: Eine konkrete Implementierung, die Konfigurationen aus dem Dateisystem unter Beachtung der XDG-Spezifikation liest/schreibt.

### 3.4. Modul: `core::logging` – Logging-Infrastruktur

- **Technologie:** Basiert vollständig auf dem `tracing`-Crate.
- **Initialisierung:**
  - `init_logging()`: Eine Funktion, die den globalen `tracing` Subscriber konfiguriert. Sie kombiniert verschiedene Layer:
    - `EnvFilter`: Erlaubt das Überschreiben des Loglevels via `RUST_LOG`.
    - `SentryLayer`: Leitet Logs an Sentry weiter.
    - `InMemoryLogBufferLayer`: Puffert Logs im Speicher.
    - Format-Layer (`json` oder `text`).
    - Output-Layer (`stdout`, `stderr`, `file` mit Rotation, `journald`).
- **Rückgabewerte:** Gibt eine `WorkerGuard` (für file-based logging) und einen `JoinHandle` (für den Config-Update-Task) zurück, deren Lebensdauer vom Aufrufer verwaltet werden muss.

### 3.5. Modul: `core::error_tracking` – Externes Fehler-Tracking

- **Technologie:** Integration mit Sentry.
- **Funktionen:**
  - `init_error_tracking()`: Initialisiert den Sentry-Client mit DSN und Metadaten (Release, Environment).
  - `get_sentry_tracing_layer()`: Stellt den `tracing`-Layer für die Logging-Pipeline bereit.
  - `capture_error()`: Sendet einen Fehler an Sentry.
  - `add_breadcrumb()`: Fügt einen Breadcrumb zum Sentry-Kontext hinzu.
  - `submit_user_feedback()`: Sendet Benutzer-Feedback zu einem spezifischen Fehler-Event.

### 3.6. Modul: `core::in_memory_log_buffer` – Log-Pufferung

- **`InMemoryLogBuffer`**: Eine thread-sichere Struktur, die eine konfigurierbare Anzahl von Log-Einträgen in einer `VecDeque` speichert.
- **`InMemoryLogBufferLayer`**: Ein `tracing::Layer`, das Events an den Puffer weiterleitet.
- **`GLOBAL_LOG_BUFFER`**: Eine `OnceCell`, die eine globale Instanz des Puffers hält.
- **Funktionalität**: Kann Logs bei Bedarf in eine Datei (`.txt` oder `.json`) exportieren, z.B. für Fehlerberichte.

### 3.7. Modul: `core::clipboard` – Datenstrukturen für die Zwischenablage

- **Zweck**: Definiert die reinen, serialisierbaren Datenstrukturen für die Zwischenablage. Die Logik zur Interaktion mit dem System-Clipboard liegt in höheren Schichten.
- **Typen**:
  - `ClipboardItem`: Repräsentiert einen einzelnen Eintrag in der Historie (Inhalt, MIME-Typ, Metadaten).
  - `ClipboardItemId`: Typsicherer ID-Wrapper.
- **Fehler**:
  - `ClipboardError`: Definiert Fehler, die bei Clipboard-Operationen auftreten können (`ItemNotFound`, `PersistenceFailed` etc.).

### 3.8. Modul: `core::utils` – Allgemeine Hilfsfunktionen

- **`utils::hash`**: Stellt Hashing-Funktionen bereit.
  - `hash_string()` mit `HashAlgorithm`-Enum (`Sha256`, `Blake3`).
- **`utils::path_utils`**: Bietet Funktionen zur Pfadvalidierung und -manipulation.
  - `validate_path()` mit `PathRequirements`.
  - `normalize_path()`, `get_relative_path()`, `join_paths()`.
- **`utils::string_utils`**: Nützliche Funktionen zur String-Konvertierung und -Manipulation.
  - `truncate()`, `is_numeric()`, `camel_case_to_snake_case()`, etc.
- **`utils::time_utils`**: Hilfsfunktionen für Zeitstempel.
  - `now_utc()`, `to_iso8601()`, `from_iso8601()`.
- **`utils::uuid_utils`**: Wrapper für die `uuid`-Bibliothek.
  - `generate_v4()`, `from_string()`.

---

## 4. Zukünftige Entwicklung & Strategie

- **Feature-Flags:** Einführung eines `FeatureFlag`-Systems auf dieser Ebene, um experimentelle Funktionen systemweit zu steuern, ohne den Code mit `#cfg` zu überladen.
- **Kryptographie-Utils:** Erweiterung von `core::utils` um ein `crypto`-Modul für grundlegende symmetrische/asymmetrische Verschlüsselungsoperationen, die für sichere Speicherung oder IPC benötigt werden könnten.
- **Metrik-Fassade:** Einführung eines `core::metrics`-Moduls, das eine Fassade für Metrik-Backends (wie Prometheus) bietet, ähnlich wie `core::logging` für `tracing`.
- **Internationalisierung (i18n):** Bereitstellung von grundlegenden Primitiven für die Internationalisierung, z.B. ein `LocalizedString`-Typ, der eine ID und optionale Parameter enthält.

---

## 5. Entwicklungs- und Qualitätsrichtlinien

- **Tests:** Jede öffentliche Funktion und jeder Typ in `novade-core` muss mit Unit-Tests abgedeckt sein. Die Testabdeckung sollte nahe 100% liegen.
- **Dokumentation:** Jede öffentliche API muss umfassend mit Rustdoc-Kommentaren dokumentiert sein, einschließlich Code-Beispielen, die als `Doc-Tests` ausgeführt werden.
- **Kein `unsafe` Code:** Die Verwendung von `unsafe`-Blöcken ist in dieser Schicht strengstens verboten, außer es ist für FFI unumgänglich und wird durch einen `// SAFETY:`-Kommentar gerechtfertigt.
- **Abhängigkeiten:** Externe Abhängigkeiten sind auf ein absolutes Minimum zu beschränken und müssen sorgfältig geprüft werden. Jede neue Abhängigkeit erfordert eine explizite Begründung.
- **API-Stabilität:** Änderungen an der öffentlichen API von `novade-core` erfordern einen formalen Review-Prozess und müssen in einem `CHANGELOG.md` dokumentiert werden.