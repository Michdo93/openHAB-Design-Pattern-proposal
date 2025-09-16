# Design Pattern: Event Debouncer for State Changes

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

![Event Debounce for State Changes](https://raw.githubusercontent.com/Michdo93/openHAB-Design-Pattern-proposal/refs/heads/main/custom/debounce%20timer.png)

---

## **1. Rule DSL Beispiel**

### Items

```java
Switch motionSensor "Bewegungssensor"
Switch light "Licht"
```

### Rule

```java
var Timer debounceTimer = null

rule "Debounced Motion Sensor"
when
    Item motionSensor changed
then
    // Prüfen, ob Timer bereits läuft
    if(debounceTimer === null) {
        // Aktion ausführen
        logInfo("Debounce", "Bewegung erkannt – Licht einschalten")
        light.sendCommand(ON)
        
        // Timer starten (z.B. 2 Sekunden)
        debounceTimer = createTimer(now.plusSeconds(2), [|
            debounceTimer = null  // Timer zurücksetzen, neue Events akzeptieren
            logInfo("Debounce", "Debounce beendet – neue Events möglich")
        ])
    } else {
        logInfo("Debounce", "Event ignoriert – Timer läuft noch")
    }
end
```

**Erläuterung Rule DSL**:

* `debounceTimer` verhindert, dass innerhalb der 2 Sekunden mehrfach Aktionen ausgelöst werden.
* Nach Ablauf wird der Timer zurückgesetzt und neue Events werden wieder verarbeitet.
* Besonders nützlich bei Sensoren, die sehr schnell hintereinander Signale liefern.

---

## **2. JSR223 Python (Jython) Beispiel**

### Items

```python
from core.rules import rule
from core.triggers import when
from core.actions import ScriptExecution
from core.log import logging

log = logging.getLogger("org.openhab.rules.Debounce")

motionSensor = ir.getItem("motionSensor")
light = ir.getItem("light")

debounce_timer = None
```

### Rule

```python
@rule("Debounced Motion Sensor")
@when("Item motionSensor changed")
def debounced_motion_sensor(event):
    global debounce_timer
    
    if debounce_timer is None:
        # Aktion ausführen
        log.info("Bewegung erkannt – Licht einschalten")
        light.sendCommand("ON")
        
        # Timer starten (2 Sekunden)
        def reset_timer():
            global debounce_timer
            debounce_timer = None
            log.info("Debounce beendet – neue Events möglich")
        
        debounce_timer = ScriptExecution.createTimer(now.plusSeconds(2), reset_timer)
    else:
        log.info("Event ignoriert – Timer läuft noch")
```

**Erläuterung Jython**:

* `debounce_timer` sorgt dafür, dass innerhalb der Timerperiode keine neue Aktion ausgeführt wird.
* Nach Ablauf wird der Timer zurückgesetzt und neue Sensorereignisse werden verarbeitet.
* Kann leicht für mehrere Sensoren wiederverwendet werden.

---

## **Vorteile des Debounce Patterns**

1. Vermeidet ungewollte Doppelaktionen durch instabile Sensoren oder Taster.
2. Reduziert unnötige Logeinträge oder Befehle an Aktoren.
3. Einfach erweiterbar durch Anpassung der Timerdauer oder Nutzung mehrerer Sensoren.

---

### **Weiterführender Link**

OpenHAB Community Thread zum Debounce Design Pattern:
[https://community.openhab.org/t/design-pattern-debounce/101566}](https://community.openhab.org/t/design-pattern-debounce/101566)

---

Wenn du willst, kann ich daraus direkt auch eine **LaTeX-Dokumentation** wie bei deinem ersten CST-Pattern erstellen – fertig zum Kompilieren mit Syntax-Highlighting für Rule DSL und Python.

Willst du, dass ich das mache?
