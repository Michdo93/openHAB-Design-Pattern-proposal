## **Design Pattern: Conditional Sequence Trigger (CST)**

**Problem:**
Man möchte eine Reihe von Aktionen nur dann ausführen, wenn eine komplexe Kombination von Bedingungen erfüllt ist (z. B. Sensorwerte, Tageszeit, Anwesenheit).

**Lösung:**

* Verwende ein **Trigger-Item** oder eine **Gruppe**, die alle relevanten Bedingungen enthält.
* Prüfe innerhalb der Rule, ob **alle Bedingungen gleichzeitig erfüllt** sind.
* Führe dann die gewünschte Aktionssequenz aus.
* Änderungen einer Bedingung setzen die Sequenz zurück.

![Conditional Sequence Trigger](https://raw.githubusercontent.com/Michdo93/openHAB-Design-Pattern-proposal/refs/heads/main/custom/cst.png)

---

### **1. Rule DSL Beispiel**

#### Items

```java
Group gCST "Conditional Sequence Trigger"

Switch motionSensor "Bewegungssensor" (gCST)
Switch presenceSensor "Anwesenheitssensor" (gCST)
DateTime currentTime "Aktuelle Zeit" (gCST)
Switch light "Licht"
```

#### Rule

```java
rule "Conditional Sequence Trigger"
when
    Member of gCST changed
then
    // Prüfen, ob alle Bedingungen erfüllt sind
    if(motionSensor.state == ON && presenceSensor.state == ON && now.getHourOfDay >= 18 && now.getHourOfDay <= 22) {
        
        // Aktion ausführen: z.B. Licht einschalten, Szene starten
        logInfo("CST", "Alle Bedingungen erfüllt – starte Sequenz")
        light.sendCommand(ON)
        
        // Weitere Aktionen in definierter Reihenfolge
        // ...
        
    } else {
        logInfo("CST", "Bedingungen nicht erfüllt – Sequenz zurücksetzen")
        light.sendCommand(OFF)
        // optional: alle Zwischenschritte abbrechen
    }
end
```

**Erläuterung:**

* `Member of gCST changed` sorgt dafür, dass die Rule jedes Mal prüft, wenn sich ein Element der Gruppe ändert.
* Die Bedingungen können beliebig erweitert werden (z. B. weitere Sensoren oder Zeitfenster).
* Die else-Bedingung setzt Aktionen zurück, wenn die Kombination nicht mehr erfüllt ist.

---

### **2. JSR223 Python (Jython) Beispiel**

#### Items

```python
from core.rules import rule
from core.triggers import when
from core.log import logging

log = logging.getLogger("org.openhab.rules.CST")

motionSensor = ir.getItem("motionSensor")
presenceSensor = ir.getItem("presenceSensor")
light = ir.getItem("light")
```

#### Rule

```python
@rule("Conditional Sequence Trigger")
@when("Member of gCST changed")
def conditional_sequence_trigger(event):
    hour = now.getHourOfDay()
    
    if motionSensor.state == ON and presenceSensor.state == ON and 18 <= hour <= 22:
        log.info("Alle Bedingungen erfüllt – starte Sequenz")
        light.sendCommand(ON)
        # Weitere Aktionen hier
        # z.B. Timer starten, weitere Geräte steuern, etc.
    else:
        log.info("Bedingungen nicht erfüllt – Sequenz zurücksetzen")
        light.sendCommand(OFF)
        # Optional: Zwischenschritte abbrechen
```

**Erläuterung Jython:**

* `now.getHourOfDay()` holt die aktuelle Stunde.
* Das Pattern ist leicht erweiterbar: weitere Sensoren oder komplexe Bedingungen lassen sich als zusätzliche `and`-Ketten einfügen.
* Aktionen können beliebig erweitert werden: Licht, Jalousien, Benachrichtigungen, Timer usw.

---

### **Vorteile des CST Patterns**

1. Sauberes Handling von **komplexen, mehrfach abhängigen Bedingungen**.
2. Einfache Erweiterbarkeit: weitere Sensoren oder Bedingungen können hinzugefügt werden.
3. Flexibles Zurücksetzen der Sequenz, wenn eine Bedingung nicht mehr erfüllt ist.
4. Kombination von Rule DSL und Python/Jython möglich.
Willst du, dass ich das mache?
