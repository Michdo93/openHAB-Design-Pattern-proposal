# **Gibt es Design Pattern für Sensor Fusion oder Kalman-Filter in openHAB?**

In **openHAB** gibt es offiziell keine dedizierten *Design Patterns* für **Sensor Fusion** oder **Kalman-Filter**. Die existierenden *Design Patterns* in der [openHAB Community](https://community.openhab.org/c/tutorials-examples/design-patterns/) drehen sich eher um Themen wie Proxy-Items, Group-Behaviors, Expire-Timer, Rule-Organisation, aber nicht um mathematische Signalverarbeitung.

## Warum gibt es das (noch) nicht?

openHAB selbst ist in erster Linie eine **Integrations- und Orchestrierungsplattform**. Es kümmert sich darum, viele Protokolle/Technologien (KNX, Zigbee, Z-Wave, MQTT etc.) zusammenzubringen und Automatisierungen zu ermöglichen.
Aufwendige Filter- oder Fusion-Algorithmen sind dort selten direkt eingebaut, weil sie typischerweise besser in einer **dedizierten Rechenumgebung** (z. B. Python, Node-RED, InfluxDB mit Kapazitätsabfragen, Edge-Devices wie ESP32) laufen.

## Aber: Man kann ein *Design Pattern* dafür erfinden

Man könnte ein „**Sensor Fusion Proxy Pattern**“ für openHAB vorschlagen. Idee:

1. **Rohsensoren** → Items mit Rohwerten (z. B. Temperatur von mehreren Fühlern).
2. **Pre-Processing** → kleine Skripte/Rules in openHAB oder extern (Jython/JS Scripting, Node-RED, MQTT-Broker, Python-Skript mit Kalman).
3. **Fusions-Item (Proxy)** → Ergebnis der Fusion (z. B. „Gefilterte\_Temperatur“) wird in ein neues Item geschrieben.
4. **Automationen** nutzen ausschließlich das Proxy-Item, niemals die Rohdaten.

Das wäre analog zu anderen Patterns (wie dem Proxy-Item-Pattern), nur dass der „Proxy“ hier ein **Fusions-/Filter-Output** ist.

## Beispiel für eine Umsetzungsidee

* Items: `Temp_Sensor1`, `Temp_Sensor2`, `Temp_Fused`
* Rule (Pseudocode in JS Scripting):

  ```javascript
  var s1 = items.getNumber("Temp_Sensor1");
  var s2 = items.getNumber("Temp_Sensor2");
  var fused = (s1 + s2) / 2; // simple fusion
  events.postUpdate("Temp_Fused", fused);
  ```
* Für komplexere Sachen (Kalman, Extended Kalman, Particle Filter):

  * Entweder **ein Binding** zu einem externen Prozess (Python-Skript via MQTT oder REST).
  * Oder ein **Rule Script** in GraalJS/Jython mit einer Kalman-Library.

## Vorschlag für ein neues Design Pattern

**„Sensor Fusion Proxy Pattern“**

* **Motivation**: Nutzung mehrerer unsicherer Sensoren, Glättung/Filterung von Messwerten.
* **Struktur**: Roh-Items → Fusion-Rule → Proxy-Item → Automationen.
* **Variante**: Einfache Fusion (Mittelwert, Median) vs. komplex (Kalman, ML-Ansätze).
* **Best Practice**: Rechenintensives extern auslagern, Proxy nur für Integration.

---

# **Design Pattern: Sensor Fusion Proxy Pattern**

## **Motivation / Problem**

Viele Smart Home Sensoren liefern Messwerte, die verrauscht, ungenau oder unterschiedlich sind. Um verlässliche Automationen zu erstellen, müssen diese Rohdaten gefiltert oder fusioniert werden. openHAB bietet von Haus aus keine speziellen Filter wie Kalman, aber man kann Sensor Fusion über Rules oder externe Dienste realisieren.

**Ziel:**

* Mehrere Rohsensoren zu einem einzigen zuverlässigen Wert zusammenführen.
* Automationen arbeiten nur mit den gefilterten Werten.
* Rechenintensive Filter (z. B. Kalman) können extern ausgelagert werden.

![Sensor Fusion Proxy Pattern](https://raw.githubusercontent.com/Michdo93/openHAB-Design-Pattern-proposal/refs/heads/main/custom/sensor%20fusion%20proxy.png)

---

## **Lösung / Pattern**

**„Sensor Fusion Proxy Pattern“**

**Struktur:**

1. **Roh-Items:** Repräsentieren die unverarbeiteten Sensorwerte.

   ```text
   Temp_Sensor1
   Temp_Sensor2
   Temp_Sensor3
   ```
2. **Fusion-Rule / Script / Prozess:** Berechnet den gefilterten Wert aus den Rohsensoren.
3. **Proxy-Item:** Speichert das Ergebnis der Fusion. Automationen greifen ausschließlich auf dieses Item zu.

   ```text
   Temp_Fused
   ```
4. **Automationen / Visualisierung:** Nutzt `Temp_Fused` anstelle der Rohsensoren.

---

## **Umsetzungsmöglichkeiten**

### **1. Interne openHAB DSL Rule**

Einfacher Mittelwert-Ansatz:

```java
rule "Fuse temperature sensors"
when
    Member of gTemperatureSensors changed
then
    var sum = 0.0
    var count = 0
    gTemperatureSensors.members.forEach[ sensor |
        val value = sensor.state as Number
        if(value != null) {
            sum += value.doubleValue
            count += 1
        }
    ]
    if(count > 0) {
        val fused = sum / count
        Temp_Fused.postUpdate(fused)
    }
end
```

* `gTemperatureSensors` ist eine Group aller Rohsensor-Items.
* Vorteil: Keine externe Abhängigkeit.
* Nachteil: Keine komplexen Filter (Kalman).

---

### **2. Jython Scripting (Rule Engine)**

Für etwas flexiblere Logik, z. B. gleitender Mittelwert oder Kalman-Filter-Integration:

```python
from core.rules import rule
from core.triggers import when
from core.actions import ScriptExecution

last_value = None
alpha = 0.5  # einfacher Low-Pass Filter

@rule("Fuse temperature sensors Jython")
@when("Member of gTemperatureSensors changed")
def fuse_temperature(event):
    global last_value
    values = [item.state for item in gTemperatureSensors.members if item.state is not None]
    if values:
        avg = sum(values)/len(values)
        if last_value is None:
            filtered = avg
        else:
            filtered = alpha * avg + (1-alpha) * last_value
        last_value = filtered
        Temp_Fused.postUpdate(filtered)
```

---

### **3. JavaScript Scripting (GraalJS)**

```javascript
var members = gTemperatureSensors.members;
var sum = 0;
var count = 0;

for (var i = 0; i < members.length; i++) {
    var value = members[i].state;
    if (value != null) {
        sum += value;
        count += 1;
    }
}

if (count > 0) {
    var fused = sum / count;
    Temp_Fused.postUpdate(fused);
}
```

---

### **4. Node-RED**

* Node-RED kann als externes Rule-Engine-System agieren.

* openHAB Nodes:

  * `events: state` → Trigger bei Sensoränderung.
  * Funktion-Node → Fusion/Filter:

    ```javascript
    let values = [msg.payload.Temp_Sensor1, msg.payload.Temp_Sensor2];
    let fused = values.reduce((a,b) => a+b,0) / values.length;
    msg.payload = {Temp_Fused: fused};
    return msg;
    ```
  * `openHAB out` → Update `Temp_Fused`.

* Vorteil: Visuell, externe Logik, leichter Einsatz von komplexen Filtern.

---

### **5. MQTT + Externes Python-Skript mit Kalman Filter**

**Setup:**

* Sensoren publizieren ihre Werte via MQTT:

  ```
  home/sensor/temperature1
  home/sensor/temperature2
  ```
* openHAB abonniert gefilterten Wert:

  ```
  Temp_Fused -> MQTT Topic: home/sensor/temperature/fused
  ```
* Python-Kalman Skript:

```python
import paho.mqtt.client as mqtt
from filterpy.kalman import KalmanFilter
import numpy as np

kf = KalmanFilter(dim_x=1, dim_z=1)
kf.x = np.array([[20]])  # Initial
kf.P *= 1000
kf.F = np.array([[1]])
kf.H = np.array([[1]])
kf.R = 2
kf.Q = 0.1

def fuse(values):
    for v in values:
        kf.predict()
        kf.update(v)
    return kf.x[0,0]

def on_message(client, userdata, msg):
    payload = [float(x) for x in msg.payload.decode().split(',')]
    fused = fuse(payload)
    client.publish("home/sensor/temperature/fused", fused)

client = mqtt.Client()
client.on_message = on_message
client.connect("localhost",1883,60)
client.subscribe("home/sensor/temperature")
client.loop_forever()
```

* openHAB erhält den bereits gefilterten Wert via MQTT-Thing:

  ```text
  Number Temp_Fused "Fused Temperature [%.1f °C]" {channel="mqtt:topic:broker:temp_fused"}
  ```

---

## **Best Practices**

1. **Proxy-Item nutzen:** Automationen greifen nie direkt auf Rohsensoren zu.
2. **Rechenintensive Filter extern auslagern:** Node-RED oder Python für Kalman/ML-Filter.
3. **Logging / Monitoring:** Fusionierte Werte in eine persistente Datenbank schreiben (InfluxDB), um Filterqualität zu prüfen.
4. **Fallbacks:** Wenn Sensoren ausfallen, Fusion sollte robust bleiben (z. B. Median statt Mittelwert).
5. **Dokumentation:** Rohsensoren, Fusion-Algorithmus und Proxy-Item klar benennen, z. B. `Temp_Sensor1` → `Temp_Fused`.

# **Unterscheidung / Abgrenzung zu anderen sensorbezogenen Pattern**

---

## **Sensor Fusion Proxy Pattern vs. Sensor Aggregation**

| Merkmal                       | Sensor Fusion Proxy Pattern                                                                 | Sensor Aggregation Pattern                                                                       |
| ----------------------------- | ------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| **Ziel**                      | Mehrere Sensoren zu einem gefilterten, stabilen Wert fusionieren (z. B. Mittelwert, Kalman) | Mehrere Sensoren logisch zusammenfassen, z. B. `OR`, `AND` für Anwesenheit oder Schalterzustände |
| **Art der Verarbeitung**      | Mathematische Filter, statistische Fusion, ggf. externe Algorithmen                         | Logische Kombination der Sensorzustände                                                          |
| **Komplexität**               | Mittel bis hoch, abhängig von Filter/Algorithmus                                            | Niedrig                                                                                          |
| **Handling von Unsicherheit** | Berücksichtigt Sensorunsicherheiten, Rauschen, Ausreißer                                    | Keine Unsicherheitsbehandlung, nur binär oder gruppenbasiert                                     |
| **Beispiele**                 | Temperaturfusion mit Kalman, kombinierte Luftfeuchte- oder Helligkeitswerte                 | Bewegungsmelder: „Wenn irgendeiner meldet → Anwesenheit“                                         |

**Fazit:**
Mein Pattern erweitert das klassische Sensor Aggregation Pattern um **Filterung, Glättung und Unsicherheitsbehandlung**. Während Sensor Aggregation einfach aggregiert, geht Sensor Fusion einen Schritt weiter: „**Wie kann ich die Daten intelligent zusammenführen, nicht nur kombinieren?**“

---

## **Sensor Fusion Proxy Pattern vs. Bayesian Sensor Aggregation**

| Merkmal                       | Sensor Fusion Proxy Pattern                                           | Bayesian Sensor Aggregation                                                                |
| ----------------------------- | --------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| **Ziel**                      | Allgemeine Sensor Fusion: Stabilisierung, Glättung, kombinierte Werte | Probabilistische Bewertung von Sensoren, um Wahrscheinlichkeiten für Zustände zu berechnen |
| **Art der Verarbeitung**      | Filterbasiert: Mittelwert, Median, Low-Pass, Kalman etc.              | Bayesianische Berechnungen, Gewichtung von Sensorwahrscheinlichkeiten                      |
| **Komplexität**               | Mittel bis hoch (bei Kalman)                                          | Hoch (Bayessche Logik, Wahrscheinlichkeitstheorie)                                         |
| **Handling von Unsicherheit** | Ja, über Filter/Statistik, aber nicht probabilistisch                 | Ja, probabilistisch; Sensorzuverlässigkeit explizit modelliert                             |
| **Externe Verarbeitung**      | Optional (Python, Node-RED, MQTT)                                     | Optional, meistens Jython oder externe Berechnungen nötig                                  |
| **Beispiele**                 | Gefilterte Temperatur oder Luftfeuchte                                | Anwesenheitserkennung mit Wahrscheinlichkeiten aus mehreren Sensoren                       |

**Fazit:**
Bayesian Sensor Aggregation ist **spezifischer**: Es geht um die **Wahrscheinlichkeit eines Zustands** basierend auf unsicheren Sensoren.
Sensor Fusion Proxy Pattern ist **allgemeiner**: Ziel ist die **Glättung, Filterung und Zusammenführung von Messwerten**. Man könnte Bayesian Aggregation sogar als **eine spezielle Variante von Sensor Fusion** sehen.

---

### **Zusammenfassung**

* **Sensor Aggregation:** Einfache logische Kombination → keine Unsicherheitsbehandlung.
* **Sensor Fusion Proxy Pattern (neu):** Mathematische Fusion/Filterung → stabilisierte Werte, Ausreißerbehandlung, Glättung.
* **Bayesian Sensor Aggregation:** Probabilistische Fusion → bewertet Wahrscheinlichkeit von Zuständen, explizit für unsichere Sensoren.

Also: **Mein Pattern ist ein flexibler „Allrounder“**, der einfache Mittelwerte, Kalman-Filter oder sogar Bayessche Logik aufnehmen kann. Es geht weniger um die Wahrscheinlichkeit eines Zustands und mehr um **verlässliche, fusionierte Messwerte**, die Automationen nutzen können.

---
