# Design Pattern: Multi-Sensor Confidence Aggregation

## **1. Konzept**

Angenommen, wir haben folgende Sensoren:

| Sensor          | Item-Name    | Gewicht (Confidence) |
| --------------- | ------------ | -------------------- |
| Bewegungsmelder | MotionSensor | 0.7                  |
| Fensterkontakt  | WindowSensor | 0.5                  |
| Lichtsensor     | LightSensor  | 0.3                  |

**Ziel:** Wenn die aggregierte Konfidenz > 0.6, dann gilt „Anwesenheit erkannt“.

![Multi-Sensor Confidence Aggregation](https://raw.githubusercontent.com/Michdo93/openHAB-Design-Pattern-proposal/refs/heads/main/custom/multisensor.png)

---

## **2. Umsetzung in Rule DSL**

```java
// Items
// Switch MotionSensor "Bewegungsmelder" <motion>
// Switch WindowSensor "Fenster offen" <contact>
// Number LightSensor "Licht" <light>

// Regel in Rule DSL
rule "Multi-Sensor Confidence Aggregation"
when
    Item MotionSensor changed or
    Item WindowSensor changed or
    Item LightSensor changed
then
    // Einzelne Sensor-Konfidenzen
    var Number motionConfidence = if(MotionSensor.state == ON) 0.7 else 0.0
    var Number windowConfidence = if(WindowSensor.state == CLOSED) 0.5 else 0.0
    var Number lightConfidence  = if((LightSensor.state as Number) > 100) 0.3 else 0.0

    // Aggregation (Weighted Sum)
    var Number aggregatedConfidence = motionConfidence + windowConfidence + lightConfidence

    logInfo("MultiSensor", "Aggregierte Konfidenz: " + aggregatedConfidence)

    // Aktion nur bei Schwellenwert > 0.6
    if(aggregatedConfidence > 0.6) {
        logInfo("MultiSensor", "Anwesenheit erkannt!")
        // z.B. Licht einschalten:
        // LightSwitch.sendCommand(ON)
    } else {
        logInfo("MultiSensor", "Keine Anwesenheit.")
    }
end
```

**Erläuterung:**

* `if(... == ON)` prüft den Zustand des Sensors.
* `aggregatedConfidence` kann komplexer sein (z. B. gewichteter Durchschnitt, Bayes).
* Nur wenn ein Schwellenwert überschritten wird, erfolgt eine Aktion.

---

## **3. Umsetzung in Python (Jython Scripting)**

```python
from core.rules import rule
from core.triggers import when
from core.log import logging

log = logging.getLogger("MultiSensor")

@rule("Multi-Sensor Confidence Aggregation Python")
@when("Item MotionSensor changed")
@when("Item WindowSensor changed")
@when("Item LightSensor changed")
def multi_sensor_confidence(event):
    # Einzelne Sensor-Konfidenzen
    motion_conf = 0.7 if str(items["MotionSensor"]) == "ON" else 0.0
    window_conf = 0.5 if str(items["WindowSensor"]) == "CLOSED" else 0.0
    light_conf  = 0.3 if float(items["LightSensor"]) > 100 else 0.0

    # Aggregation
    aggregated_conf = motion_conf + window_conf + light_conf

    log.info(f"Aggregierte Konfidenz: {aggregated_conf}")

    if aggregated_conf > 0.6:
        log.info("Anwesenheit erkannt!")
        # z.B. Licht einschalten
        # events.sendCommand("LightSwitch", "ON")
    else:
        log.info("Keine Anwesenheit.")
```

**Erläuterung:**

* `items["MotionSensor"]` liefert den aktuellen Zustand.
* Bedingte Zuordnung für Konfidenz.
* Aggregation als Summe der Konfidenzen, Schwellenwert prüfen.
* Python ermöglicht flexiblere Berechnungen (z. B. Bayes, komplexe Formeln).

---

## **4. Erweiterungsmöglichkeiten**

1. **Dynamische Gewichtung:** Sensoren können unterschiedliche Vertrauenswerte haben.
2. **Bayesian Aggregation:** `P(Aktiv|Sensor1, Sensor2)` für probabilistische Entscheidungen.
3. **Historische Daten:** Mittelwert über Zeit, um Fehlalarme zu reduzieren.
4. **Flexible Schwellenwerte:** Verschiedene Schwellen für Aktionen (z. B. Licht, Alarm, Benachrichtigung).

# Erweiterung mit Bayesian Aggregation

---

## **1. Idee**

Wir benutzen **Bayes’ Theorem**:

$$
P(\text{Anwesenheit} \mid \text{Sensoren}) = \frac{P(\text{Sensoren} \mid \text{Anwesenheit}) \cdot P(\text{Anwesenheit})}{P(\text{Sensoren})}
$$

* Jeder Sensor liefert **Wahrscheinlichkeit für korrekte Erkennung** (True Positive / False Positive).
* Wir aggregieren über alle Sensoren unabhängig.

---

## **2. Beispiel in Python (Jython)**

```python
from core.rules import rule
from core.triggers import when
from core.log import logging

log = logging.getLogger("MultiSensorBayes")

# Sensorkennzahlen (Wahrscheinlichkeit korrekt)
SENSOR_CONFIDENCE = {
    "MotionSensor": {"tp": 0.9, "fp": 0.1},   # True Positive / False Positive
    "WindowSensor": {"tp": 0.8, "fp": 0.2},
    "LightSensor": {"tp": 0.7, "fp": 0.3}
}

# Basiswahrscheinlichkeit für Anwesenheit
PRIOR_PRESENCE = 0.5

@rule("Bayesian Multi-Sensor Aggregation")
@when("Item MotionSensor changed")
@when("Item WindowSensor changed")
@when("Item LightSensor changed")
def bayesian_aggregation(event):
    # Sensorzustände auslesen
    states = {
        "MotionSensor": str(items["MotionSensor"]) == "ON",
        "WindowSensor": str(items["WindowSensor"]) == "CLOSED",
        "LightSensor": float(items["LightSensor"]) > 100
    }

    # Bayes Wahrscheinlichkeit initial mit Prior
    prob_presence = PRIOR_PRESENCE
    prob_absence = 1 - PRIOR_PRESENCE

    for sensor, detected in states.items():
        tp = SENSOR_CONFIDENCE[sensor]["tp"]
        fp = SENSOR_CONFIDENCE[sensor]["fp"]

        if detected:
            prob_presence *= tp
            prob_absence *= fp
        else:
            prob_presence *= (1 - tp)
            prob_absence *= (1 - fp)

    # Normalisieren
    total = prob_presence + prob_absence
    prob_presence /= total

    log.info(f"Bayes Anwesenheitswahrscheinlichkeit: {prob_presence:.2f}")

    # Entscheidungsschwelle
    if prob_presence > 0.6:
        log.info("Anwesenheit erkannt!")
        # events.sendCommand("LightSwitch", "ON")
    else:
        log.info("Keine Anwesenheit.")
```

---

## **3. Erklärung**

1. **SENSOR\_CONFIDENCE:** Enthält, wie zuverlässig jeder Sensor ist (True Positive / False Positive).
2. **Zustandserkennung:** `True` wenn Sensor anzeigt „Anwesenheit“.
3. **Bayes-Kombination:** Multipliziert Wahrscheinlichkeiten für jeden Sensor.
4. **Normalisierung:** Damit `P(Anwesenheit) + P(Abwesenheit) = 1`.
5. **Schwelle:** Aktion erst, wenn z. B. > 0.6 Wahrscheinlichkeit.

---

## **4. Vorteile**

* Berücksichtigt Unsicherheiten der Sensoren.
* Skalierbar für beliebig viele Sensoren.
* Fehlalarme werden reduziert.
* Flexible Anpassung über `tp` / `fp` Werte.

---

