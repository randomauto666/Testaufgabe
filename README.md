# Testaufgabe Java-Entwickler – Meldesystem „Elinski Hausen“

Stand: 04.10.2026 · Umfang: ca. 2–3 Arbeitstage · Sprache der Spielernachrichten: Deutsch

## 1. Ausgangslage

Elinski Hausen betreibt ein Minecraft-Netzwerk mit mehreren Paper-Servern hinter mehreren BungeeCord-Proxys. Spieler der Java Edition und der Bedrock Edition (über Geyser/Floodgate) spielen gemeinsam.

Deine Aufgabe ist ein **netzwerkweites Meldesystem** („EH-Meldungen“). Spieler melden andere Spieler, das Team bearbeitet die Fälle in einer Oberfläche im Spiel, und der Melder bekommt unabhängig von Server und Proxy eine Rückmeldung. Alles läuft asynchron, ist persistent und für den Produktivbetrieb gedacht.

## 2. Technische Rahmenbedingungen

### 2.1 Projektstruktur

Maven-Multi-Module-Projekt mit der Group-ID `de.elinskihausen` und der Artifact-ID `eh-meldungen`:

| Modul | Inhalt |
| --- | --- |
| `eh-api` | Interfaces, DTOs (Records), Enums, Event-Klassen. Keine Abhängigkeit auf Paper, Bungee, MongoDB oder Redis. |
| `eh-core` | Services, Repositories, Mongo-Zugriff, Redis-Messaging, Caching, Konfiguration, Guice-Module |
| `eh-paper` | Paper-Plugin: Commands, SmartInvs-GUIs, Floodgate-Formulare, Listener |
| `eh-proxy` | BungeeCord-Plugin: Login-Hinweis für das Team, Zustellung von Rückmeldungen an den Melder |
| `eh-testkit` | Test-Fixtures, In-Memory-Implementierungen der Repository-Interfaces, Testcontainers-Setup |

### 2.2 Technologie-Stack

- Java 21 (LTS)
- Maven mit Maven Wrapper
- Paper API 1.21.3 (`io.papermc.paper:paper-api:1.21.3-R0.1-SNAPSHOT`)
- BungeeCord API (`net.md-5:bungeecord-api:1.21-R0.1-SNAPSHOT`)
- MongoDB mit Morphia 2.x (`dev.morphia.morphia:morphia-core:2.4.14`)
- Redis mit Jedis 6 (`redis.clients:jedis:6.0.0`)
- Google Guice 7 (`com.google.inject:guice:7.0.0`) und Guava 33 (`com.google.guava:guava:33.0.0-jre`)
- Lombok 1.18.38, JetBrains Annotations 26.0.2
- SmartInvs 1.2.7 (`fr.minuskube.inv:smart-invs:1.2.7`)
- Floodgate API 2.2.4-SNAPSHOT (`org.geysermc.floodgate:api`)
- Testen: JUnit 5, Mockito, AssertJ, Testcontainers (MongoDB, Redis)

### 2.3 Harte Regeln

- Keine blockierenden Aufrufe im Main-Thread (Datenbank, Redis, HTTP).
- Keine eigenen `Thread`s, `ExecutorService`s oder `ForkJoinPool`s. Asynchrone Arbeit läuft ausschließlich über den `BukkitScheduler` bzw. den `TaskScheduler` von BungeeCord.
- Zugriffe auf die Bukkit-API (Inventare, Spieler, Nachrichten) passieren wieder im Main-Thread. Wie du zwischen den Threads wechselst, soll im Code erkennbar und an einer Stelle gebündelt sein.
- Alle Texte, die Spieler sehen, kommen aus einer Sprachdatei und nicht aus dem Code.

## 3. Funktionale Anforderungen

### 3.1 Meldung erstellen

**Command:** `/meldung [spieler]`, Alias `/report`

- Ohne Argument öffnet sich eine GUI mit den Köpfen aller Spieler, die gerade auf dem aktuellen Server online sind. Die Liste ist seitenweise durchblätterbar und hat eine Namenssuche per Chat- bzw. Formulareingabe.
- Mit Argument geht es direkt zur Auswahl des Meldegrunds.
- Man kann sich nicht selbst melden. Teammitglieder (`meldung.exempt`) können nicht gemeldet werden, es sei denn, das Team schaltet das in der Konfiguration frei.

**Meldegründe** (aus der Datenbank geladen, siehe 5.1). Als Startdaten legt das Plugin beim ersten Start diese Gründe an:

| Schlüssel | Anzeige | Besonderheit |
| --- | --- | --- |
| `HACKING` | Hacks / unerlaubte Mods | Priorität HIGH |
| `TOXIC` | Beleidigung / toxisches Verhalten |  |
| `EXPLOIT` | Ausnutzen von Bugs oder Dupes | Priorität HIGH |
| `GRIEF` | Zerstörung fremder Bauwerke | Ort wird automatisch erfasst |
| `ADVERTISING` | Werbung / Spam |  |
| `SCAM` | Betrug beim Handeln | Beweis-Hinweis wird angezeigt |
| `OTHER` | Sonstiges | Freitext Pflicht (10 bis 200 Zeichen) |

**Zwei Oberflächen, ein Ablauf:**

- Java Edition: SmartInvs-GUI (Spielerauswahl, Grundauswahl, Bestätigungsbildschirm).
- Bedrock Edition: Floodgate-Formulare (`SimpleForm` für Auswahl, `CustomForm` für Freitext und Bestätigung). Die Erkennung läuft über die Floodgate-API.
- Beide Oberflächen rufen denselben `ReportService` auf. Die Geschäftslogik darf nicht doppelt existieren.

**Ablauf:**

1. Spieler wählt das Ziel und den Grund.
2. Eingaben werden geprüft (Länge, Steuerzeichen, Cooldown, Duplikate, siehe 3.3).
3. Bestätigungsschritt: „Du meldest *Name* wegen *Grund*. Missbrauch wird bestraft.“
4. Meldung wird asynchron in MongoDB gespeichert.
5. Danach wird auf Redis veröffentlicht (`eh:meldung:neu`).
6. Der Melder bekommt: `[Meldung] Danke, deine Meldung #<kurzId> wurde aufgenommen.`

Schlägt Schritt 4 fehl, wird nichts veröffentlicht, und der Melder sieht eine verständliche Fehlermeldung. Schlägt nur Schritt 5 fehl, ist die Meldung trotzdem gespeichert, und ein Wiederholungsversuch über den Scheduler stellt die Veröffentlichung nach (maximal 3 Versuche, wachsende Wartezeit).

### 3.2 Identitäten auflösen (auch für Offline-Spieler)

Meldungen sollen auch gegen Spieler möglich sein, die gerade nicht online sind, z. B. wenn sie kurz vorher den Server verlassen haben.

- Zuerst wird in einem lokalen Cache nachgesehen, danach in der eigenen Spielerdatenbank (Collection `profiles`, falls vorhanden), zuletzt über eine externe API.
- Bedrock-Namen erkennst du am konfigurierbaren Floodgate-Präfix (Standard `.`). Je nach Edition wird ein anderer Endpunkt abgefragt (Mojang-Profil-API für Java, Floodgate-Global-API für Bedrock). Die Endpunkte stehen in der Konfiguration.
- HTTP-Anfragen laufen asynchron mit `java.net.http.HttpClient` (`sendAsync`) und Timeout. Das Ergebnis wird über den Scheduler zurück in den Main-Thread gegeben, wenn dort weitergearbeitet wird.
- Erfolgreiche Treffer werden zwischengespeichert (Guava `Cache`, konfigurierbare Dauer, Standard 6 Stunden). Auch **Fehlschläge** („Name existiert nicht“) werden kurz gespeichert (Standard 2 Minuten), damit niemand die API mit Tippfehlern flutet.
- Fällt die API aus oder antwortet mit 429/5xx, versucht das System es einmal erneut. Danach bekommt der Spieler eine klare Meldung („Spielersuche gerade nicht erreichbar, bitte später erneut versuchen“). Es gibt keine Exception im Chat.
- Namensformat wird vor dem Request geprüft (Java: 3–16 Zeichen, `[A-Za-z0-9_]`; Bedrock: Präfix plus Gamertag mit Leerzeichen-Ersetzung nach Floodgate-Regeln).

### 3.3 Missbrauchsschutz

- **Cooldown:** Pro Melder höchstens eine Meldung alle 60 Sekunden (konfigurierbar). Der Cooldown gilt netzwerkweit, also auch bei Serverwechsel. Hinweis: Speicherung dafür in Redis mit Ablaufzeit.
- **Duplikate:** Meldet derselbe Spieler dasselbe Ziel mit demselben Grund, solange noch eine offene Meldung existiert, wird keine neue angelegt. Stattdessen sieht der Melder: „Du hast diesen Spieler bereits gemeldet.“
- **Sammelmeldungen:** Melden mehrere verschiedene Spieler dasselbe Ziel innerhalb von 30 Minuten, wird die Meldung mit erhöhter Priorität markiert (Schwelle konfigurierbar, Standard 3). Das Team erhält einen Hinweis „Mehrfach gemeldet“.
- **Tageslimit:** Maximal 10 Meldungen pro Spieler und Tag (konfigurierbar).
- **Missbrauch:** Meldungen können als `MISSBRAUCH` abgelehnt werden. Ab 3 solcher Fälle innerhalb von 30 Tagen wird das Melden für den Spieler gesperrt (nur Markierung im Profil plus Hinweis an das Team, keine automatische Strafe).

### 3.4 Kontext und Beweise

Damit das Team Entscheidungen treffen kann, erfasst das System automatisch:

- Server, Welt und Koordinaten des Melders und, falls online auf demselben Server, des Gemeldeten
- Die letzten 10 Chatzeilen des Gemeldeten (nur bei den Gründen `TOXIC` und `ADVERTISING`, siehe Datenschutz in 6.3)
- Optional einen Beweislink (Pflicht für `SCAM`, freiwillig sonst), der nur gegen eine Whitelist erlaubter Domains angenommen wird (Konfiguration)

Das Feld `context` im Datenmodell ist bewusst flexibel gehalten, damit später weitere Informationen dazukommen können.

### 3.5 Team-Benachrichtigung in Echtzeit

- Alle Paper-Server abonnieren den Redis-Channel `eh:meldung:neu`.
- Spieler mit der Permission `meldung.team` bekommen:

```
[Meldung] #<kurzId> · <gemeldet> wurde von <melder> gemeldet (<grund>, Priorität <prio>)
Klicke hier, um die Meldung zu öffnen. Alle offenen Meldungen: /meldungen
```

- Der Klick auf die Nachricht führt `/meldungen details <id>` aus. Bei hoher Priorität wird zusätzlich ein Ton abgespielt.
- Teammitglieder können Benachrichtigungen mit `/meldungen stumm` für sich an- oder ausschalten. Die Einstellung bleibt über Neustarts erhalten.
- Der absendende Server verarbeitet die eigene Nachricht nicht doppelt (Nachrichten tragen die `sourceInstance`).

### 3.6 Meldungen bearbeiten

**Status-Modell:**

```
NEU → IN_PRUEFUNG → BERECHTIGT
                  → ABGELEHNT
                  → ESKALIERT → BERECHTIGT / ABGELEHNT
```

Ungültige Übergänge (z. B. `ABGELEHNT` → `IN_PRUEFUNG`) sind im Code unmöglich bzw. werden mit einem klaren Fehler abgewiesen.

**Übernehmen („Claim“):** Ein Teammitglied übernimmt eine Meldung. Die Übernahme muss atomar sein: Greifen zwei Teammitglieder gleichzeitig zu, bekommt genau eines den Zuschlag, das andere sieht „Wird bereits von X bearbeitet“. Umsetzung über ein bedingtes Update in MongoDB (`findAndModify` mit Status und `handledBy` in der Bedingung), nicht über Lesen-dann-Schreiben.

**GUI `/meldungen`:**

- Übersicht mit Paginierung
- Filter: Grund, Server, Status, Priorität, Zeitraum (heute / 7 Tage / 30 Tage), „nur meine“
- Sortierung: Alter, Priorität, Anzahl Meldungen gegen das Ziel
- Detailansicht mit allen Informationen aus 3.4 und der Historie früherer Meldungen gegen dieses Ziel

**Aktionen und Commands:**

| Aktion | Command | Wirkung |
| --- | --- | --- |
| Übernehmen | `/meldungen claim <id>` | Status `IN_PRUEFUNG`, Bearbeiter gesetzt |
| Freigeben | `/meldungen release <id>` | zurück auf `NEU` |
| Berechtigt | `/meldungen accept <id> [notiz]` | Status `BERECHTIGT`, Melder wird benachrichtigt |
| Ablehnen | `/meldungen decline <id> <grund>` | Status `ABGELEHNT`, Begründung Pflicht |
| Eskalieren | `/meldungen escalate <id> <grund>` | Status `ESKALIERT`, Hinweis an `meldung.leitung` |
| Details | `/meldungen details <id>` | vollständige Informationen |
| Statistik | `/meldungen stats [spieler]` | siehe 3.9 |
| Hingehen | `/meldungen goto <id>` | siehe 3.10 |

IDs dürfen in Commands als Kurz-ID (die letzten 6 Zeichen der ObjectId) angegeben werden, solange sie eindeutig ist. Tab-Completion liefert offene Meldungen.

### 3.7 Rückmeldung an den Melder (Multi-Proxy)

Der Melder kann auf einem anderen Server und über einen anderen Proxy online sein als das Teammitglied, das die Meldung abschließt. Kein Proxy kennt die Spieler der anderen Proxys.

**Beispiel:** Lena meldet auf `citybuild-2` (Proxy A). Das Teammitglied schließt den Fall auf `lobby-1` (Proxy B) ab. Lena muss die Nachricht trotzdem sehen, und zwar genau einmal.

**Nachrichten an den Melder:**

- Berechtigt: `[Meldung] Deine Meldung #<kurzId> gegen <spieler> wurde geprüft und war berechtigt. Danke für deine Hilfe!`
- Abgelehnt: `[Meldung] Deine Meldung #<kurzId> gegen <spieler> wurde geprüft und abgelehnt. Begründung: <notiz>`

**Anforderungen:**

- Zustellung unabhängig von Server und Proxy, auch bei vielen Proxys
- Keine doppelte Zustellung, auch wenn alle Proxys dieselbe Redis-Nachricht erhalten
- Kein zentrales Spielerverzeichnis über alle Proxys hinweg als Voraussetzung (du darfst aber eines in Redis aufbauen, wenn du es begründest)
- Verpasste Nachrichten (Melder war offline) werden beim nächsten Login zugestellt und danach als zugestellt markiert
- Geht der Melder genau während der Zustellung offline, darf die Nachricht nicht verloren gehen
- Das Modul `eh-proxy` und das Plugin `eh-paper` dürfen nicht beide zustellen. Lege fest, wer verantwortlich ist, und dokumentiere die Entscheidung.

**Zu beantworten im README (Abschnitt „Entwurfsentscheidungen“):** Wie stellst du Exactly-once-Zustellung (soweit praktisch erreichbar) sicher? Welche Rolle spielen die Redis-Pub/Sub-Eigenschaften (keine Persistenz, kein Replay), und was ersetzt sie an dieser Stelle?

### 3.8 Team-Hinweis beim Login (Proxy)

Loggt sich ein Spieler mit `meldung.team` ein:

- Asynchrone Abfrage der offenen Meldungen (Status `NEU` und `ESKALIERT`, bei Leitung auch `IN_PRUEFUNG` älter als 24 Stunden)
- Nachricht: `[Meldung] Aktuell warten X Meldungen auf Bearbeitung (davon Y mit hoher Priorität). Nutze /meldungen.`
- Der Login darf durch die Abfrage nicht verzögert werden. Bei einem Fehler der Datenbank wird der Hinweis still ausgelassen und im Log vermerkt.
- Die Abfrage nutzt einen Zähler (`countDocuments` mit passendem Index), nicht das Laden aller Dokumente.

### 3.9 Statistiken

`/meldungen stats` zeigt netzwerkweit: offene Meldungen, durchschnittliche Bearbeitungszeit der letzten 7 Tage, Quote berechtigter Meldungen, häufigste Gründe.

`/meldungen stats <spieler>` zeigt für einen Spieler: wie oft gemeldet, wie oft berechtigt, wie oft selbst gemeldet und wie viele davon abgelehnt wurden. Berechnung per Aggregation in MongoDB, nicht im Plugin. Ergebnisse werden 60 Sekunden gecacht.

### 3.10 Zum Gemeldeten springen

`/meldungen goto <id>` bringt ein Teammitglied zum gemeldeten Spieler, sofern er online ist. Ist er auf einem anderen Server, wird zuerst der Serverwechsel über den Proxy angestoßen (Plugin-Message oder Redis-Anfrage) und die Teleportation nach dem Join ausgeführt. Ist der Spieler offline, wird der letzte bekannte Ort aus dem Kontext angezeigt.

## 4. Datenmodelle und Messaging

### 4.1 MongoDB

**Collection `meldungen`:**

```java
@Entity("meldungen")
public class MeldungModel {
    @Id private ObjectId id;
    private UUID melderUuid;
    private String melderName;
    private UUID zielUuid;
    private String zielName;
    private String grundSchluessel;      // Template-Schlüssel
    private String freitext;             // nur bei OTHER Pflicht
    private MeldungStatus status;        // NEU, IN_PRUEFUNG, BERECHTIGT, ABGELEHNT, ESKALIERT
    private Prioritaet prioritaet;       // NIEDRIG, NORMAL, HOCH
    private UUID bearbeiterUuid;
    private String bearbeiterName;
    private String notiz;
    private String server;
    private Map<String, Object> kontext; // Ort, Chatauszug, Beweislink
    private Instant erstelltAm;
    private Instant geaendertAm;
    private Instant abgeschlossenAm;
    private int version;                 // optimistisches Locking
}
```

**Collection `meldungs_gruende`:** `schluessel`, `anzeigename`, `beschreibung`, `prioritaet`, `freitextPflicht`, `beweisPflicht`, `aktiv`.

**Collection `zustellungen`:** Offene Rückmeldungen an Melder. Felder: `meldungId`, `empfaengerUuid`, `status` (OFFEN, ZUGESTELLT), `nachricht`, `erstelltAm`, `zugestelltAm`. Ein TTL-Index entfernt zugestellte Einträge nach 30 Tagen.

**Collection `team_einstellungen`:** `uuid`, `benachrichtigungenStumm`.

**Indizes (mit Begründung im README):**

- `meldungen`: `{status, prioritaet, erstelltAm}` (Übersicht und Login-Zähler)
- `meldungen`: `{zielUuid, erstelltAm}` und `{melderUuid, erstelltAm}` (Historie, Duplikate, Statistik)
- `meldungen`: `{server, status}` (Filter)
- `zustellungen`: `{empfaengerUuid, status}`

### 4.2 Redis

Alle Nachrichten sind JSON, tragen eine `schemaVersion`, eine eindeutige `messageId` und die `sourceInstance`. Unbekannte Felder müssen ignoriert werden (Vorwärtskompatibilität).

**Channel `eh:meldung:neu`:**

```json
{
  "schemaVersion": 1,
  "messageId": "<UUID>",
  "meldungId": "<ObjectId>",
  "kurzId": "a1b2c3",
  "melder": "<Name>",
  "ziel": "<Name>",
  "grund": "<Anzeigename oder Freitext>",
  "prioritaet": "NORMAL",
  "server": "<Serverinstanz>",
  "sourceInstance": "<Instanz-ID>",
  "zeitpunkt": "<ISO-8601>"
}
```

**Channel `eh:meldung:status`:**

```json
{
  "schemaVersion": 1,
  "messageId": "<UUID>",
  "meldungId": "<ObjectId>",
  "kurzId": "a1b2c3",
  "melderUuid": "<UUID>",
  "melderName": "<Name>",
  "zielName": "<Name>",
  "status": "BERECHTIGT | ABGELEHNT",
  "bearbeiter": "<Name>",
  "notiz": "<Text oder null>",
  "sourceInstance": "<Instanz-ID>",
  "zeitpunkt": "<ISO-8601>"
}
```

**Channel `eh:config:reload`:** löst auf allen Instanzen das Neuladen der Meldegründe aus.

**Redis-Schlüssel:** `eh:cooldown:<uuid>` (Ablaufzeit), `eh:tageslimit:<uuid>:<datum>` (Zähler mit Ablaufzeit), `eh:zugestellt:<messageId>` (Deduplizierung, kurze Ablaufzeit).

Der Redis-Subscriber muss eine unterbrochene Verbindung selbstständig wiederherstellen (wachsende Wartezeit, Obergrenze, Logeintrag je Versuch). Der Pub/Sub-Aufruf blockiert, deshalb muss klar sein, wie du ihn ohne eigenen Thread-Pool betreibst. Begründe deine Lösung im README.

## 5. Konfiguration und Erweiterbarkeit

### 5.1 Meldegründe

- Liegen in MongoDB (`meldungs_gruende`), werden beim Start geladen und im Speicher gehalten.
- Hot-Reload: `/meldungen reload` lädt neu und sendet `eh:config:reload`, damit alle Server nachziehen.
- Ein neuer Grund muss ohne Neustart und ohne Codeänderung nutzbar sein, in beiden Oberflächen (SmartInvs und Floodgate).

### 5.2 Konfigurationsdatei

Beispiel `config.yml` (Struktur ist verbindlich, Werte sind Beispiele):

```yaml
instance-id: "citybuild-2"

mongodb:
  uri: "mongodb://localhost:27017"
  database: "eh_meldungen"

redis:
  host: "localhost"
  port: 6379
  password: ""
  reconnect:
    initial-delay-ticks: 20
    max-delay-ticks: 1200

meldung:
  cooldown-seconds: 60
  daily-limit: 10
  cluster-threshold: 3
  cluster-window-minutes: 30
  freitext:
    min-length: 10
    max-length: 200
  allowed-evidence-domains:
    - "imgur.com"
    - "prnt.sc"
    - "youtube.com"
  team-reportable: false

identity:
  bedrock-prefix: "."
  java-endpoint: "https://api.mojang.com/users/profiles/minecraft/%s"
  bedrock-endpoint: "https://api.geysermc.org/v2/xbox/xuid/%s"
  timeout-ms: 3000
  cache-hours: 6
  negative-cache-minutes: 2
```

Die Sprachdatei (`messages_de.yml`) enthält alle Nachrichten mit Platzhaltern und einem einheitlichen Prefix.

## 6. Nicht-funktionale Anforderungen

### 6.1 Threading und Performance

- Alle Datenbank-, Redis- und HTTP-Operationen laufen asynchron. Das Ergebnis wird als `CompletableFuture` oder über einen eigenen kleinen Callback-Typ zurückgegeben.
- Der Wechsel zwischen asynchronem und synchronem Kontext ist an einer Stelle gekapselt (z. B. `SchedulerBridge`), damit er sich testen lässt.
- Es gibt eine Übersicht, welche Methode in welchem Thread laufen darf (als JavaDoc-Hinweis oder Annotation).
- Datenbankabfragen in Listen verwenden Projektionen und Limits. Das Plugin lädt nie unbegrenzt viele Dokumente.
- GUI-Inhalte werden vorab asynchron geladen und beim Öffnen nicht nachgeladen.

### 6.2 Code-Qualität

- Clean-Code-Prinzipien, kleine Klassen, klare Verantwortlichkeiten
- Java-21-Features sinnvoll einsetzen (Records, `switch` mit Pattern Matching, Sealed Interfaces für Ergebnistypen wie `MeldungErgebnis`, Textblöcke)
- Dependency Injection mit Guice, keine statischen Singletons außer der Plugin-Hauptklasse
- Sinnvolle Package-Struktur je Modul
- JavaDoc für alle öffentlichen Typen und Methoden der API
- Conventional Commits für alle Commits (`feat:`, `fix:`, `refactor:`, `test:`, `docs:`, `build:`)

### 6.3 Sicherheit, Datenschutz und Stabilität

- Alle Eingaben werden geprüft und bereinigt (Länge, Steuerzeichen, Formatierungscodes). Freitext darf keine Chat-Formatierung einschleusen.
- Jede Aktion prüft die Permission, auch Klicks in der GUI, nicht nur der Command.
- Chatauszüge (3.4) werden nur im Speicher als Ringpuffer gehalten und nur bei einer Meldung übernommen. Sie werden nach 90 Tagen automatisch gelöscht (TTL-Index oder geplanter Job).
- Es werden keine IP-Adressen gespeichert.
- Externe Dienste (MongoDB, Redis, HTTP) dürfen ausfallen, ohne den Server zu beeinträchtigen. Fehler werden geloggt, Spieler sehen verständliche Meldungen.
- Das Plugin darf beim Start ohne Datenbank nicht abstürzen. Es wechselt in einen eingeschränkten Modus und versucht die Verbindung erneut.
- Beim Herunterfahren werden offene Schreibvorgänge abgeschlossen (soweit ohne eigene Threads möglich), und Redis-Verbindungen werden sauber geschlossen.

## 7. Tests

Mindestens:

- Unit-Tests für den Statusautomaten (alle erlaubten und verbotenen Übergänge)
- Unit-Tests für Cooldown, Duplikat- und Sammelmeldungs-Logik mit In-Memory-Implementierungen aus `eh-testkit`
- Test der Zustelllogik aus 3.7 (inklusive: Melder offline, Melder wechselt Proxy, Nachricht kommt zweimal an)
- Integrationstests mit Testcontainers für Repository und Redis-Messaging
- Test, dass zwei gleichzeitige `claim`-Aufrufe genau einen Gewinner haben

Die Tests müssen mit `mvn verify` laufen. Das Projekt soll ohne installierte Datenbanken bauen können, sofern Docker vorhanden ist.

## 8. Bewertungskriterien

| Bereich | Gewichtung | Worauf geachtet wird |
| --- | --- | --- |
| Architektur und Modularität | 20 % | Modultrennung, Schnittstellen, Austauschbarkeit der Speicherung |
| Threading und Asynchronität | 20 % | Scheduler-Nutzung, keine Verstöße gegen die harten Regeln, Thread-Sicherheit |
| Datenbank und Messaging | 15 % | Indizes, atomare Updates, Fehlerbehandlung, Reconnect |
| Multi-Proxy-Zustellung | 15 % | Korrektheit, Deduplizierung, Offline-Zustellung, Begründung im README |
| GUI und Bedienung | 10 % | SmartInvs und Floodgate-Formulare, Verständlichkeit |
| Codequalität und Tests | 15 % | Lesbarkeit, Testabdeckung der Kernlogik, JavaDoc |
| Git und Dokumentation | 5 % | Commit-Historie, README, Docker-Setup |

## 9. Abgabe

- **Git-Repository** mit funktionierendem Multi-Module-Build, sauberer Commit-Historie (Conventional Commits) und nachvollziehbarer Branch-Strategie (`main`, Feature-Branches, Pull-Requests an dich selbst sind ausdrücklich erwünscht)
- **README.md** mit Architekturübersicht (Diagramm), Setup für MongoDB und Redis, Build und Deployment, Konfigurationsbeispielen, Abschnitt „Entwurfsentscheidungen“ (siehe 3.7 und 4.2) und einer Liste bekannter Einschränkungen
- **docker-compose.yml** mit MongoDB und Redis, jeweils mit Healthcheck und Volume. Optional: ein zweiter Redis-Client-Container, um Pub/Sub von Hand zu testen.
- **Dokumentation** als JavaDoc, kurze API-Beschreibung (`eh-api`) und ein Abschnitt „Performance-Überlegungen“ (was ist gecacht, was ist indiziert, was würde bei 10-facher Last zuerst brechen)
- **Beispiel-Daten:** Skript oder Command, das Testmeldungen erzeugt, damit die GUI ohne Handarbeit prüfbar ist

## 10. Bonusaufgaben (optional)

- `/meldungen export` erzeugt eine CSV der letzten 30 Tage (asynchron, ohne den Server zu belasten)
- Metriken (Anzahl neuer Meldungen, durchschnittliche Zeit bis zur Bearbeitung) als einfacher Prometheus-Endpunkt oder als Log-Zeile im festen Format
- Discord-Webhook für Meldungen mit hoher Priorität (asynchron, mit Rate-Limit-Beachtung)
- Zweisprachige Nachrichten (Deutsch und Englisch) mit Sprachwahl pro Spieler
- GitHub-Actions-Workflow, der Build und Tests ausführt

## 11. Hinweise

- Bei Unklarheiten bitte nachfragen. Getroffene Annahmen gehören ins Ticket.
- Wichtiger als Vollständigkeit aller Bonuspunkte ist ein sauberer, nachvollziehbarer und getesteter Kern.
- Keine eigenen Thread-Pools, ausschließlich die Scheduler der Plattform.
- Alle I/O-Operationen laufen asynchron.
