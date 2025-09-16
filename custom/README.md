## 1. **Design Pattern: Conditional Sequence Trigger (CST)**

**Problem:** Man möchte eine Reihe von Aktionen nur dann ausführen, wenn eine komplexe Kombination von Bedingungen erfüllt ist, z. B. Sensorwerte, Tageszeit und Anwesenheit.

**Lösung:**

* Definiere ein „Trigger-Item“ oder eine Gruppe für die Bedingungskombination.
* Verwende eine Rule, die prüft, ob **alle Bedingungen gleichzeitig erfüllt sind**.
* Löst dann die definierte Sequenz von Aktionen aus.
* Änderungen in einer Bedingung setzen die Sequenz zurück.

**Vorteile:**

* Sauberes Handling komplexer, mehrfach abhängiger Regeln.
* Einfache Erweiterbarkeit durch Hinzufügen weiterer Bedingungen.

---

## 2. **Design Pattern: Event Debouncer for State Changes**

**Problem:** Manche Sensoren liefern sehr schnelle, mehrfach hintereinander auftretende State Changes, z. B. Bewegungsmelder oder Taster.
Ohne Filter führen sie zu mehrfachen Aktionen.

**Lösung:**

* Verwende ein Timer-basiertes „Debounce-Item“.
* Nach dem ersten Event wird ein Timer gestartet (z. B. 2 Sekunden).
* Während dieser Zeit werden keine neuen Aktionen ausgeführt.
* Erst danach werden neue State Changes wieder akzeptiert.

**Vorteile:**

* Reduziert ungewollte Doppelaktionen.
* Besonders nützlich bei instabilen Sensoren oder Tastern.

---

## 3. **Design Pattern: Multi-Sensor Confidence Aggregation**

**Problem:** Man möchte ein verlässliches Ergebnis aus mehreren Sensoren ableiten, z. B. Anwesenheit, Fensterstatus, Lichtsensoren. Einzelne Sensoren können falsch liegen.

**Lösung:**

* Berechne einen **Konfidenzwert** für jedes Sensor-Item.
* Verwende eine Regel, die eine aggregierte Entscheidung basierend auf allen Sensoren trifft (z. B. Weighted Average oder Bayesian).
* Trigger für Aktionen erfolgen nur, wenn ein bestimmter Schwellenwert überschritten wird.

**Vorteile:**

* Höhere Zuverlässigkeit.
* Flexibel für beliebige Sensor-Kombinationen.

---

## 4. **Design Pattern: Graceful Retry Actions**

**Problem:** Ein Aktor reagiert manchmal nicht auf Commands (z. B. MQTT-Gerät offline).
Standard-Rules führen dann zu Fehlermeldungen oder bleiben hängen.

**Lösung:**

* Definiere eine Rule, die Commands mit einem **Retry-Mechanismus** sendet.
* Nutze Timer, um die Wiederholungen zu steuern.
* Nach einem konfigurierbaren Timeout wird eine alternative Aktion (z. B. Notification) ausgelöst.

**Vorteile:**

* Robuster Umgang mit Netzwerkproblemen oder instabilen Geräten.
* Entkopplung von Fehlerbehandlung und normalem Ablauf.


