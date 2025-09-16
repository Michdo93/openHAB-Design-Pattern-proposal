# **Graceful Retry Actions**

## **1. Idee**

Ziel: Ein Command soll an ein Gerät gesendet werden. Wenn das Gerät nicht reagiert (z. B. offline), versuchen wir das Command erneut, bis entweder:

* das Gerät antwortet, oder
* eine maximale Anzahl an Versuchen / Timeout erreicht wird.

Optional kann nach dem Timeout eine alternative Aktion erfolgen (z. B. Notification, Logging, Alarm).

![Graceful Retry Actions](https://raw.githubusercontent.com/Michdo93/openHAB-Design-Pattern-proposal/refs/heads/main/custom/graceful%20retry%20actions.png)

---

## **2. Umsetzung in Rule DSL**

```java
// Beispiel: Licht einschalten mit Retry
var Timer retryTimer = null
var int retryCount = 0
var final int MAX_RETRIES = 3
var final int RETRY_INTERVAL = 10 // Sekunden

rule "Graceful Retry LightSwitch"
when
    Item SomeTrigger changed to ON
then
    retryCount = 0
    
    // Funktion zum Senden des Commands mit Retry
    retryTimer = createTimer(now.plusSeconds(0), [ |
        try {
            // Versuche das Command zu senden
            LightSwitch.sendCommand(ON)
            logInfo("Retry", "Command erfolgreich gesendet!")
            if(retryTimer !== null) {
                retryTimer.cancel()
            }
        } catch(Throwable t) {
            retryCount = retryCount + 1
            logWarn("Retry", "Fehler beim Senden, Versuch #" + retryCount)
            
            if(retryCount < MAX_RETRIES) {
                // Timer erneut planen
                retryTimer.reschedule(now.plusSeconds(RETRY_INTERVAL))
            } else {
                logError("Retry", "Maximale Anzahl an Versuchen erreicht!")
                // Alternative Aktion, z. B. Notification
                sendNotification("admin@example.com", "LightSwitch konnte nicht eingeschaltet werden")
            }
        }
    ])
end
```

**Erklärung DSL:**

* `createTimer` erzeugt einen Timer, der die Retry-Funktion ausführt.
* `try/catch` fängt Fehler beim Senden des Commands ab.
* `reschedule` plant den Timer neu, falls weitere Versuche erlaubt sind.
* `MAX_RETRIES` und `RETRY_INTERVAL` sind konfigurierbar.
* Nach Erreichen der Maximalversuche wird eine alternative Aktion ausgeführt.

---

## **3. Umsetzung in Python (Jython)**

```python
from core.rules import rule
from core.triggers import when
from core.log import logging
from core.actions import PersistenceExtensions

import threading

log = logging.getLogger("GracefulRetry")

MAX_RETRIES = 3
RETRY_INTERVAL = 10  # seconds

def send_command_with_retry(item_name, command, retries=0):
    try:
        events.sendCommand(item_name, command)
        log.info(f"Command '{command}' sent to {item_name} successfully!")
    except Exception as e:
        retries += 1
        log.warn(f"Failed to send command to {item_name}, attempt #{retries}")
        if retries < MAX_RETRIES:
            # Timer für erneuten Versuch
            threading.Timer(RETRY_INTERVAL, send_command_with_retry, [item_name, command, retries]).start()
        else:
            log.error(f"Max retries reached for {item_name}. Performing alternative action.")
            # Alternative Aktion, z.B. Notification
            events.sendCommand("NotificationItem", f"{item_name} could not be switched ON!")

@rule("Graceful Retry Action Python")
@when("Item SomeTrigger changed to ON")
def graceful_retry_action(event):
    send_command_with_retry("LightSwitch", "ON")
```

**Erklärung Python:**

* Rekursive Funktion `send_command_with_retry` sendet das Command und retryt bei Fehlern.
* `threading.Timer` plant einen erneuten Versuch nach `RETRY_INTERVAL` Sekunden.
* Maximalversuche werden mit `MAX_RETRIES` begrenzt.
* Alternative Aktionen können z. B. Notifications oder Logging sein.

---

### **Vorteile dieser Umsetzung**

* Robuste Handhabung von instabilen Geräten oder temporären Netzwerkproblemen.
* Regeln blockieren nicht den normalen Ablauf.
* Flexible Anpassung von Interval, Anzahl der Versuche und alternativen Aktionen.

---

# **Graceful Retry-Pattern mit exponentiellem Backoff**

Dann bauen wir auf dem bisherigen **Graceful Retry**-Pattern auf und fügen **exponentielles Backoff** hinzu. Das bedeutet: Nach jedem Fehlversuch wird das Intervall bis zum nächsten Versuch verdoppelt (oder mit einem Faktor multipliziert), bis ein Maximum erreicht ist.

---

## **1. Rule DSL mit Exponential Backoff**

```java
var Timer retryTimer = null
var int retryCount = 0
var final int MAX_RETRIES = 5
var int retryInterval = 5 // Startintervall in Sekunden
var final int MAX_INTERVAL = 60 // Max Intervall in Sekunden

rule "Graceful Retry LightSwitch with Backoff"
when
    Item SomeTrigger changed to ON
then
    retryCount = 0
    retryInterval = 5
    
    retryTimer = createTimer(now.plusSeconds(0), [ |
        try {
            LightSwitch.sendCommand(ON)
            logInfo("RetryBackoff", "Command successfully sent!")
            if(retryTimer !== null) retryTimer.cancel()
        } catch(Throwable t) {
            retryCount = retryCount + 1
            logWarn("RetryBackoff", "Failed attempt #" + retryCount)

            if(retryCount < MAX_RETRIES) {
                // Exponential Backoff
                retryInterval = Math.min(retryInterval * 2, MAX_INTERVAL)
                logInfo("RetryBackoff", "Next attempt in " + retryInterval + " seconds")
                retryTimer.reschedule(now.plusSeconds(retryInterval))
            } else {
                logError("RetryBackoff", "Max retries reached!")
                sendNotification("admin@example.com", "LightSwitch could not be switched ON!")
            }
        }
    ])
end
```

**Erläuterung:**

* Startintervall wird bei jedem Fehler verdoppelt (`retryInterval * 2`) bis `MAX_INTERVAL`.
* `MAX_RETRIES` begrenzt die Anzahl der Versuche.
* Reschedule verwendet das aktuelle Intervall.

---

## **2. Python (Jython) mit Exponential Backoff**

```python
from core.rules import rule
from core.triggers import when
from core.log import logging
import threading

log = logging.getLogger("GracefulRetryBackoff")

MAX_RETRIES = 5
INITIAL_INTERVAL = 5  # seconds
MAX_INTERVAL = 60     # seconds

def send_command_with_backoff(item_name, command, retries=0, interval=INITIAL_INTERVAL):
    try:
        events.sendCommand(item_name, command)
        log.info(f"Command '{command}' sent to {item_name} successfully!")
    except Exception as e:
        retries += 1
        log.warn(f"Attempt #{retries} failed for {item_name}: {e}")
        
        if retries < MAX_RETRIES:
            # Exponential Backoff
            next_interval = min(interval * 2, MAX_INTERVAL)
            log.info(f"Next retry in {next_interval} seconds")
            threading.Timer(next_interval, send_command_with_backoff, [item_name, command, retries, next_interval]).start()
        else:
            log.error(f"Max retries reached for {item_name}. Triggering alternative action.")
            # Alternative action, e.g., Notification
            events.sendCommand("NotificationItem", f"{item_name} could not be switched ON!")

@rule("Graceful Retry Action Python with Backoff")
@when("Item SomeTrigger changed to ON")
def graceful_retry_backoff(event):
    send_command_with_backoff("LightSwitch", "ON")
```

**Erläuterung:**

* Startintervall: `INITIAL_INTERVAL`.
* Nach jedem Fehlversuch wird das Intervall verdoppelt, aber nicht größer als `MAX_INTERVAL`.
* Rekursive Timer-Funktion retryt automatisch.
* Alternative Aktion wird ausgelöst, wenn `MAX_RETRIES` erreicht wird.

---

### **Vorteile des Exponential Backoff**

* Weniger Belastung des Netzwerks bei wiederholten Fehlversuchen.
* Geräte bekommen mehr Zeit zum Wiederverbinden.
* Fehlerbehandlung bleibt elegant und entkoppelt vom normalen Ablauf.
* Parameter wie `INITIAL_INTERVAL`, `MAX_INTERVAL` und `MAX_RETRIES` lassen sich leicht anpassen.

---

# **Generische Multi-Device Retry-Funktion**

Dann bauen wir eine **generische, Multi-Device Retry-Funktion** mit Exponential Backoff für openHAB. Sowohl **Rule DSL** als auch **Python (Jython)**. Damit kann man beliebig viele Aktoren gleichzeitig behandeln, ohne für jedes Gerät eigenen Code zu schreiben.

---

## **1. Rule DSL: Multi-Device Retry mit Backoff**

```java
var Map<String, Timer> retryTimers = newHashMap
var Map<String, Integer> retryCounts = newHashMap
var final int MAX_RETRIES = 5
var final int INITIAL_INTERVAL = 5 // Sekunden
var final int MAX_INTERVAL = 60    // Sekunden

rule "Graceful Retry Multiple Devices"
when
    Item RetryTrigger changed to ON
then
    val devices = newArrayList("LightSwitch", "HeaterSwitch", "FanSwitch")

    devices.forEach [ device |
        retryCounts.put(device, 0)
        
        val int initialInterval = INITIAL_INTERVAL

        val Timer t = createTimer(now.plusSeconds(0), [ |
            try {
                events.sendCommand(device, ON)
                logInfo("MultiRetry", device + " command sent successfully!")
                retryTimers.get(device)?.cancel()
            } catch(Throwable t) {
                val int count = retryCounts.get(device) + 1
                retryCounts.put(device, count)
                logWarn("MultiRetry", device + " attempt #" + count + " failed.")

                if(count < MAX_RETRIES) {
                    val int nextInterval = Math.min(initialInterval * (2^count), MAX_INTERVAL)
                    logInfo("MultiRetry", "Next attempt for " + device + " in " + nextInterval + " seconds")
                    retryTimers.get(device)?.reschedule(now.plusSeconds(nextInterval))
                } else {
                    logError("MultiRetry", device + " max retries reached!")
                    sendNotification("admin@example.com", device + " could not be switched ON!")
                }
            }
        ])
        
        retryTimers.put(device, t)
    ]
end
```

**Erläuterung:**

* `devices` enthält die Liste aller Aktoren, die retryt werden sollen.
* Jeder Aktor bekommt eigenen Timer und Retry-Count (`retryTimers`, `retryCounts`).
* Exponentielles Backoff: `nextInterval = initialInterval * (2^count)` bis `MAX_INTERVAL`.
* Alternative Aktion (Notification) bei Max-Retries.

---

## **2. Python (Jython): Multi-Device Retry mit Backoff**

```python
from core.rules import rule
from core.triggers import when
from core.log import logging
import threading

log = logging.getLogger("MultiDeviceRetryBackoff")

MAX_RETRIES = 5
INITIAL_INTERVAL = 5  # seconds
MAX_INTERVAL = 60     # seconds

def send_command_with_backoff(item_name, command, retries=0, interval=INITIAL_INTERVAL):
    try:
        events.sendCommand(item_name, command)
        log.info(f"Command '{command}' sent to {item_name} successfully!")
    except Exception as e:
        retries += 1
        log.warn(f"Attempt #{retries} failed for {item_name}: {e}")
        
        if retries < MAX_RETRIES:
            next_interval = min(interval * 2, MAX_INTERVAL)
            log.info(f"Next retry for {item_name} in {next_interval} seconds")
            threading.Timer(next_interval, send_command_with_backoff, [item_name, command, retries, next_interval]).start()
        else:
            log.error(f"Max retries reached for {item_name}. Performing alternative action.")
            events.sendCommand("NotificationItem", f"{item_name} could not be switched ON!")

@rule("Graceful Retry Multiple Devices Python")
@when("Item RetryTrigger changed to ON")
def multi_device_retry(event):
    devices = ["LightSwitch", "HeaterSwitch", "FanSwitch"]
    for device in devices:
        send_command_with_backoff(device, "ON")
```

**Erläuterung:**

* `devices` enthält alle Aktoren, die retryt werden sollen.
* Die rekursive Funktion `send_command_with_backoff` behandelt jeden Aktor individuell.
* Exponentielles Backoff wird automatisch angewendet: `interval` verdoppelt nach jedem Fehlversuch.
* Alternative Aktionen (Notification) werden bei Erreichen der maximalen Versuche ausgelöst.

---

### **Vorteile der Multi-Device-Version**

* Skalierbar für beliebig viele Geräte.
* Fehlerbehandlung bleibt entkoppelt vom normalen Ablauf.
* Exponentielles Backoff reduziert Netzwerklast bei wiederholten Fehlversuchen.
* Einfach konfigurierbar: `MAX_RETRIES`, `INITIAL_INTERVAL`, `MAX_INTERVAL`.
* Optional kann die Retry-Funktion leicht angepasst werden, z. B. unterschiedliche Intervalle oder alternative Aktionen pro Gerät.

---

# **Multi-Device Retry-Pattern mit Online-Check**

Dann erweitern wir das Multi-Device Retry-Pattern um einen **Online-Check**. Die Idee: Bevor ein Retry-Command gesendet wird, prüft die Regel, ob das Gerät erreichbar oder online ist (z. B. über `getState()`, Ping, oder einen Status-Item). So werden unnötige Versuche vermieden.

---

## **1. Rule DSL: Multi-Device Retry mit Online-Check**

```java
var Map<String, Timer> retryTimers = newHashMap
var Map<String, Integer> retryCounts = newHashMap
var final int MAX_RETRIES = 5
var final int INITIAL_INTERVAL = 5 // Sekunden
var final int MAX_INTERVAL = 60    // Sekunden

rule "Graceful Retry Multiple Devices with Online Check"
when
    Item RetryTrigger changed to ON
then
    val devices = newArrayList("LightSwitch", "HeaterSwitch", "FanSwitch")

    devices.forEach [ device |
        retryCounts.put(device, 0)
        
        val int initialInterval = INITIAL_INTERVAL

        val Timer t = createTimer(now.plusSeconds(0), [ |
            try {
                // Online-Check: Device ist nur erreichbar, wenn es einen gültigen Zustand liefert
                if(items.get(device) !== NULL) {
                    events.sendCommand(device, ON)
                    logInfo("MultiRetryOnline", device + " command sent successfully!")
                    retryTimers.get(device)?.cancel()
                } else {
                    throw new IllegalStateException("Device " + device + " is offline")
                }
            } catch(Throwable t) {
                val int count = retryCounts.get(device) + 1
                retryCounts.put(device, count)
                logWarn("MultiRetryOnline", device + " attempt #" + count + " failed: " + t.message)

                if(count < MAX_RETRIES) {
                    // Exponentielles Backoff
                    val int nextInterval = Math.min(initialInterval * (2^count), MAX_INTERVAL)
                    logInfo("MultiRetryOnline", "Next attempt for " + device + " in " + nextInterval + " seconds")
                    retryTimers.get(device)?.reschedule(now.plusSeconds(nextInterval))
                } else {
                    logError("MultiRetryOnline", device + " max retries reached!")
                    sendNotification("admin@example.com", device + " could not be switched ON!")
                }
            }
        ])
        
        retryTimers.put(device, t)
    ]
end
```

**Erläuterung DSL:**

* `items.get(device) !== NULL` prüft, ob der Aktor online ist (alternativ kann man z. B. `DeviceStatus` Items prüfen).
* Fehler beim Offline-Status lösen einen Retry aus.
* Exponentielles Backoff wie vorher.

---

## **2. Python (Jython): Multi-Device Retry mit Online-Check**

```python
from core.rules import rule
from core.triggers import when
from core.log import logging
import threading

log = logging.getLogger("MultiDeviceRetryOnline")

MAX_RETRIES = 5
INITIAL_INTERVAL = 5  # seconds
MAX_INTERVAL = 60     # seconds

def send_command_with_online_check(item_name, command, retries=0, interval=INITIAL_INTERVAL):
    try:
        # Online-Check: Prüfen, ob das Item einen gültigen Status hat
        if str(items[item_name]) != "NULL":
            events.sendCommand(item_name, command)
            log.info(f"Command '{command}' sent to {item_name} successfully!")
        else:
            raise Exception(f"{item_name} is offline")
    except Exception as e:
        retries += 1
        log.warn(f"Attempt #{retries} failed for {item_name}: {e}")
        
        if retries < MAX_RETRIES:
            next_interval = min(interval * 2, MAX_INTERVAL)
            log.info(f"Next retry for {item_name} in {next_interval} seconds")
            threading.Timer(next_interval, send_command_with_online_check, [item_name, command, retries, next_interval]).start()
        else:
            log.error(f"Max retries reached for {item_name}. Performing alternative action.")
            events.sendCommand("NotificationItem", f"{item_name} could not be switched ON!")

@rule("Graceful Retry Multiple Devices Python with Online Check")
@when("Item RetryTrigger changed to ON")
def multi_device_retry_online(event):
    devices = ["LightSwitch", "HeaterSwitch", "FanSwitch"]
    for device in devices:
        send_command_with_online_check(device, "ON")
```

**Erläuterung Python:**

* `str(items[item_name]) != "NULL"` prüft, ob der Aktor einen gültigen Status liefert.
* Wenn das Gerät offline ist, wird der Retry-Mechanismus ausgelöst.
* Exponentielles Backoff reduziert unnötige Versuche.
* Max-Retries lösen eine alternative Aktion aus (z. B. Notification).

---

### **Vorteile der Online-Check-Version**

* Vermeidet unnötige Retry-Versuche bei offline Geräten.
* Spart Netzwerktraffic und reduziert Belastung der Aktoren.
* Funktioniert für beliebig viele Geräte gleichzeitig.
* Exponentielles Backoff sorgt für robuste und flexible Wiederholungen.
* Alternative Aktionen informieren zuverlässig, wenn Geräte dauerhaft offline bleiben.

---

# **Dynamische Multi-Device Retry-Version**

Dann erstellen wir jetzt eine **dynamische Multi-Device Retry-Version** mit Online-Check und exponentiellem Backoff für openHAB.

Der Vorteil: Du musst nicht mehr manuell eine Liste von Geräten pflegen – die Regel ermittelt die Aktoren automatisch aus einer **Gruppe** oder einer konfigurierten Item-Liste.

---

## **1. Rule DSL: Dynamische Geräte mit Retry, Backoff und Online-Check**

```java
var Map<String, Timer> retryTimers = newHashMap
var Map<String, Integer> retryCounts = newHashMap
var final int MAX_RETRIES = 5
var final int INITIAL_INTERVAL = 5 // Sekunden
var final int MAX_INTERVAL = 60    // Sekunden

rule "Dynamic Multi-Device Retry with Backoff and Online Check"
when
    Item RetryTrigger changed to ON
then
    // Dynamisch alle Geräte aus einer Gruppe holen
    val devices = GroupDevices.members.filter[i | i !== NULL].map[i | i.name]

    devices.forEach [ device |
        retryCounts.put(device, 0)
        
        val int initialInterval = INITIAL_INTERVAL

        val Timer t = createTimer(now.plusSeconds(0), [ |
            try {
                // Online-Check: Item muss einen gültigen Status haben
                if(items.get(device) !== NULL) {
                    events.sendCommand(device, ON)
                    logInfo("DynamicRetry", device + " command sent successfully!")
                    retryTimers.get(device)?.cancel()
                } else {
                    throw new IllegalStateException("Device " + device + " is offline")
                }
            } catch(Throwable t) {
                val int count = retryCounts.get(device) + 1
                retryCounts.put(device, count)
                logWarn("DynamicRetry", device + " attempt #" + count + " failed: " + t.message)

                if(count < MAX_RETRIES) {
                    val int nextInterval = Math.min(initialInterval * (2^count), MAX_INTERVAL)
                    logInfo("DynamicRetry", "Next attempt for " + device + " in " + nextInterval + " seconds")
                    retryTimers.get(device)?.reschedule(now.plusSeconds(nextInterval))
                } else {
                    logError("DynamicRetry", device + " max retries reached!")
                    sendNotification("admin@example.com", device + " could not be switched ON!")
                }
            }
        ])
        
        retryTimers.put(device, t)
    ]
end
```

**Erklärung DSL:**

* `GroupDevices` ist eine offene Gruppe in openHAB, die alle retry-fähigen Aktoren enthält.
* `members.filter[i | i !== NULL].map[i | i.name]` erzeugt die Liste der aktiven Geräte.
* Exponentielles Backoff wird pro Gerät individuell berechnet.
* Alternative Aktion (Notification) bei Max-Retries.

---

## **2. Python (Jython): Dynamische Geräte mit Retry, Backoff und Online-Check**

```python
from core.rules import rule
from core.triggers import when
from core.log import logging
import threading

log = logging.getLogger("DynamicMultiDeviceRetry")

MAX_RETRIES = 5
INITIAL_INTERVAL = 5  # seconds
MAX_INTERVAL = 60     # seconds

def send_command_with_online_check(item_name, command, retries=0, interval=INITIAL_INTERVAL):
    try:
        if str(items[item_name]) != "NULL":
            events.sendCommand(item_name, command)
            log.info(f"Command '{command}' sent to {item_name} successfully!")
        else:
            raise Exception(f"{item_name} is offline")
    except Exception as e:
        retries += 1
        log.warn(f"Attempt #{retries} failed for {item_name}: {e}")
        if retries < MAX_RETRIES:
            next_interval = min(interval * 2, MAX_INTERVAL)
            log.info(f"Next retry for {item_name} in {next_interval} seconds")
            threading.Timer(next_interval, send_command_with_online_check,
                            [item_name, command, retries, next_interval]).start()
        else:
            log.error(f"Max retries reached for {item_name}. Performing alternative action.")
            events.sendCommand("NotificationItem", f"{item_name} could not be switched ON!")

@rule("Dynamic Multi-Device Retry Python")
@when("Item RetryTrigger changed to ON")
def dynamic_multi_device_retry(event):
    # Alle Geräte aus einer openHAB-Gruppe dynamisch holen
    devices = [i.name for i in items["GroupDevices"].members if i is not None]
    
    for device in devices:
        send_command_with_online_check(device, "ON")
```

**Erklärung Python:**

* `items["GroupDevices"].members` liefert dynamisch alle Mitglieder der Gruppe.
* Jeder Aktor wird individuell behandelt mit Online-Check und exponentiellem Backoff.
* Max-Retries lösen die alternative Aktion aus (Notification).
* Kein manuelles Pflegen von Geräte-Listen mehr nötig.

---

### **Vorteile der dynamischen Multi-Device-Version**

* Automatische Geräteerkennung über Gruppen.
* Exponentielles Backoff reduziert unnötige Versuche.
* Online-Check verhindert sinnlose Retries bei offline Geräten.
* Skalierbar für beliebig viele Aktoren.
* Alternative Aktionen informieren zuverlässig bei dauerhaft offline Geräten.

---

# **Vollständig konfigurierbare Template-Version** in Jython

* Alle Parameter steuerbar über Items: `MAX_RETRIES`, `INITIAL_INTERVAL`, `MAX_INTERVAL`, Gruppe der Geräte, alternative Aktion.
* Unterstützt Multi-Device, Online-Check und exponentielles Backoff.
* Keine Codeänderungen nötig, alles über Items konfigurierbar.

---

## **1. Python (Jython) Template-Version**

```python
from core.rules import rule
from core.triggers import when
from core.log import logging
import threading

log = logging.getLogger("ConfigurableMultiDeviceRetry")

# Konfigurations-Items in openHAB
# Number RetryMaxAttempts "Max Retries"
# Number RetryInitialInterval "Initial Interval"
# Number RetryMaxInterval "Max Interval"
# Group RetryDevices
# String RetryAlternativeAction

def send_command_with_online_check(item_name, command, retries=0, interval=None):
    if interval is None:
        interval = int(items["RetryInitialInterval"])
    max_retries = int(items["RetryMaxAttempts"])
    max_interval = int(items["RetryMaxInterval"])
    
    try:
        if str(items[item_name]) != "NULL":
            events.sendCommand(item_name, command)
            log.info(f"Command '{command}' sent to {item_name} successfully!")
        else:
            raise Exception(f"{item_name} is offline")
    except Exception as e:
        retries += 1
        log.warn(f"Attempt #{retries} failed for {item_name}: {e}")
        
        if retries < max_retries:
            next_interval = min(interval * 2, max_interval)
            log.info(f"Next retry for {item_name} in {next_interval} seconds")
            threading.Timer(next_interval, send_command_with_online_check,
                            [item_name, command, retries, next_interval]).start()
        else:
            log.error(f"Max retries reached for {item_name}. Executing alternative action.")
            alt_action = str(items["RetryAlternativeAction"])
            if alt_action:
                events.sendCommand("NotificationItem", f"{item_name}: {alt_action}")
            else:
                events.sendCommand("NotificationItem", f"{item_name} could not be switched ON!")

@rule("Configurable Multi-Device Retry")
@when("Item RetryTrigger changed to ON")
def configurable_multi_device_retry(event):
    devices = [i.name for i in items["RetryDevices"].members if i is not None]
    for device in devices:
        send_command_with_online_check(device, "ON")
```

---

### **2. Erklärungen**

* **`RetryDevices` (Group)**: Enthält alle Aktoren, die retryt werden sollen. Dynamisch auswertbar.
* **`RetryMaxAttempts` (Number)**: Maximale Wiederholungen (z. B. 5).
* **`RetryInitialInterval` (Number)**: Startintervall in Sekunden (z. B. 5).
* **`RetryMaxInterval` (Number)**: Maximal erlaubtes Intervall (z. B. 60).
* **`RetryAlternativeAction` (String)**: Nachricht oder Aktion, die bei Max-Retries ausgeführt wird.
* **Exponentielles Backoff**: Intervall verdoppelt sich nach jedem Fehlversuch, bis `RetryMaxInterval`.
* **Online-Check**: Prüft, ob das Item einen gültigen Status hat, bevor das Command gesendet wird.
* **Multi-Device**: Alle Geräte in der Gruppe werden gleichzeitig verarbeitet.

---

### **3. Vorteile**

* Vollständig konfigurierbar ohne Codeänderung.
* Unterstützt jede Anzahl von Geräten dynamisch.
* Robuster Umgang mit instabilen Geräten oder Netzwerkproblemen.
* Exponentielles Backoff reduziert unnötige Versuche.
* Alternative Aktionen informieren zuverlässig über fehlgeschlagene Commands.

---

# **Rule DSL Version als voll konfigurierbares Template**

---

## **1. Rule DSL: Configurable Multi-Device Retry Template**

```java
var Map<String, Timer> retryTimers = newHashMap
var Map<String, Integer> retryCounts = newHashMap

rule "Configurable Multi-Device Retry"
when
    Item RetryTrigger changed to ON
then
    // Lese Konfiguration aus Items
    val int MAX_RETRIES = RetryMaxAttempts.state as Number
    val int INITIAL_INTERVAL = RetryInitialInterval.state as Number
    val int MAX_INTERVAL = RetryMaxInterval.state as Number
    val String ALT_ACTION = RetryAlternativeAction.state.toString

    // Dynamisch alle Geräte aus Gruppe holen
    val devices = RetryDevices.members.filter[i | i !== NULL].map[i | i.name]

    devices.forEach [ device |
        retryCounts.put(device, 0)
        
        val int startInterval = INITIAL_INTERVAL

        val Timer t = createTimer(now.plusSeconds(0), [ |
            try {
                // Online-Check: Item muss einen gültigen Status haben
                if(items.get(device) !== NULL) {
                    events.sendCommand(device, ON)
                    logInfo("DynamicRetryDSL", device + " command sent successfully!")
                    retryTimers.get(device)?.cancel()
                } else {
                    throw new IllegalStateException("Device " + device + " is offline")
                }
            } catch(Throwable ex) {
                val int count = retryCounts.get(device) + 1
                retryCounts.put(device, count)
                logWarn("DynamicRetryDSL", device + " attempt #" + count + " failed: " + ex.message)

                if(count < MAX_RETRIES) {
                    // Exponentielles Backoff
                    val int nextInterval = Math.min(startInterval * (2^count), MAX_INTERVAL)
                    logInfo("DynamicRetryDSL", "Next attempt for " + device + " in " + nextInterval + " seconds")
                    retryTimers.get(device)?.reschedule(now.plusSeconds(nextInterval))
                } else {
                    logError("DynamicRetryDSL", device + " max retries reached!")
                    if(ALT_ACTION != "") {
                        sendNotification("admin@example.com", device + ": " + ALT_ACTION)
                    } else {
                        sendNotification("admin@example.com", device + " could not be switched ON!")
                    }
                }
            }
        ])
        
        retryTimers.put(device, t)
    ]
end
```

---

### **2. Konfigurations-Items**

| Item                     | Typ    | Zweck                                                 |
| ------------------------ | ------ | ----------------------------------------------------- |
| `RetryTrigger`           | Switch | Trigger für das Retry-Event                           |
| `RetryDevices`           | Group  | Enthält alle Aktoren, die retryt werden sollen        |
| `RetryMaxAttempts`       | Number | Maximale Anzahl an Wiederholungen                     |
| `RetryInitialInterval`   | Number | Startintervall für Retry in Sekunden                  |
| `RetryMaxInterval`       | Number | Maximal zulässiges Intervall in Sekunden              |
| `RetryAlternativeAction` | String | Nachricht/Aktion bei Max-Retries (z. B. Notification) |

---

### **3. Vorteile**

* Vollständig konfigurierbar über Items, keine Codeänderung nötig.
* Unterstützt beliebig viele Geräte in einer Gruppe.
* Exponentielles Backoff reduziert unnötige Versuche.
* Online-Check verhindert sinnlose Retries bei offline Geräten.
* Alternative Aktionen informieren zuverlässig, wenn Geräte dauerhaft offline bleiben.

---

# **Step-by-Step Guide: Configurable Multi-Device Retry in openHAB**

## **1. Erstelle die benötigten Items**

Du benötigst Items für:

* Trigger
* Gruppe der Aktoren
* Retry-Parameter
* Alternative Aktion

### **1.1 Trigger-Item**

```text
Switch RetryTrigger "Retry Trigger"
```

* Wird auf `ON` geschaltet, um den Retry-Prozess zu starten.

---

### **1.2 Geräte-Gruppe**

```text
Group RetryDevices "Devices to Retry"
```

* Alle Aktoren, die retryt werden sollen, in dieser Gruppe sammeln.
* Beispiel:

```text
Switch LightSwitch "Living Room Light" (RetryDevices)
Switch HeaterSwitch "Heater" (RetryDevices)
Switch FanSwitch "Fan" (RetryDevices)
```

---

### **1.3 Retry-Parameter**

```text
Number RetryMaxAttempts "Max Retries" <number>
Number RetryInitialInterval "Initial Interval (s)" <time>
Number RetryMaxInterval "Max Interval (s)" <time>
String RetryAlternativeAction "Alternative Action"
```

* `RetryMaxAttempts` z. B. 5
* `RetryInitialInterval` z. B. 5 Sekunden
* `RetryMaxInterval` z. B. 60 Sekunden
* `RetryAlternativeAction` z. B. `"Notify admin"`

---

## **2. Python (Jython) Version**

1. Speichere den Python-Code (Template-Version) in `conf/automation/jsr223/python/`.
2. Beispiel-Dateiname: `multi_device_retry.py`
3. Inhalt siehe **Python Template**, den wir vorher erstellt haben.

**Test:**

* Schalte `RetryTrigger` auf `ON`
* Beobachte Logs (`openhab.log`) und Notifications (`NotificationItem`)

---

## **3. Rule DSL Version**

1. Speichere den DSL-Code (Template-Version) in `conf/automation/rules/`.
2. Beispiel-Dateiname: `multi_device_retry.rules`
3. Inhalt siehe **DSL Template**, das wir vorher erstellt haben.

**Test:**

* Schalte `RetryTrigger` auf `ON`
* Überprüfe Logs und Notifications

---

## **4. Parameter anpassen**

| Item                     | Zweck                                              |
| ------------------------ | -------------------------------------------------- |
| `RetryMaxAttempts`       | Anzahl der Wiederholungen pro Gerät                |
| `RetryInitialInterval`   | Startintervall für Retry in Sekunden               |
| `RetryMaxInterval`       | Maximalintervall für Backoff                       |
| `RetryAlternativeAction` | Nachricht bei Max-Retries (z. B. `"Notify admin"`) |

* Anpassung erfolgt direkt über Items – keine Codeänderung nötig.

---

## **5. Erweiterungsmöglichkeiten**

* **Gerätespezifische Intervalle**: Erstelle weitere Items oder Maps, um unterschiedliche Retry-Parameter pro Gerät zu definieren.
* **Bedingte Retry**: Prüfe vorher Status-Items, um nur bestimmte Geräte retryen zu lassen.
* **Logging & Monitoring**: Nutze ein Dashboard oder Notifications, um fehlgeschlagene Commands sofort zu erkennen.

---

## **6. Vorteile**

* Vollständig dynamisch und konfigurierbar.
* Unterstützt beliebig viele Geräte gleichzeitig.
* Online-Check verhindert unnötige Retries.
* Exponentielles Backoff reduziert Netzwerklast.
* Alternative Aktionen informieren zuverlässig über dauerhaft offline Geräte.

---

