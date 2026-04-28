# BE WAVE (SATEL) + Home Assistant via Node-RED

> 🇵🇱 [Polski](#-polski) | 🇬🇧 [English](#-english)

---

## 🇵🇱 Polski

Integracja czujek ruchu BE WAVE (SATEL Smart HUB) z Home Assistant za pomocą Node-RED.

### Problem

BE WAVE Smart HUB wysyła zdarzenia jako **surowe stringi TCP** — nie HTTP, nie MQTT, nie REST.  
Home Assistant nie obsługuje tego formatu natywnie, dlatego potrzebny jest pośrednik.

### Architektura

```
Czujka APD-200
      │
      ▼
BE WAVE Smart HUB
      │  (surowy string TCP, np. "ruch-w-kuchni")
      ▼
Node-RED (TCP in → parse → ha-fire-event)
      │
      ▼
Home Assistant (event → automatyzacja)
```

### Wymagania

**Sprzęt:**

- SATEL BE WAVE Smart HUB
- Czujka ruchu APD-200 lub inna czujka BE WAVE

**Oprogramowanie:**

- Home Assistant (HassOS lub inny wariant)
- Node-RED (add-on HA lub osobna instancja)
- Węzeł `node-red-contrib-home-assistant-websocket`

### Konfiguracja sieciowa

BE WAVE i HA mogą być w tej samej sieci lub w osobnych VLANach.  
W tej konfiguracji BE WAVE jest w oddzielnej sieci (np. VLAN dla kamer), a HA wystawia dodatkowy interfejs w tej samej sieci:

| Urządzenie                               | Przykładowy IP |
|------------------------------------------|----------------|
| BE WAVE Smart HUB                        | `192.168.2.13` |
| Home Assistant (interfejs sieci BE WAVE) | `192.168.2.10` |
| Port TCP (Node-RED)                      | `8124`         |

Reguła firewall: zezwól na TCP z IP BE WAVE do HA na porcie 8124.

> Jeśli HA i BE WAVE są w tej samej sieci, dodatkowy interfejs nie jest potrzebny — użyj głównego IP HA.

### Konfiguracja BE WAVE — IP Output

W aplikacji BE WAVE Smart HUB skonfiguruj **IP Output** dla każdej czujki:

- **IP:** adres HA osiągalny ze Smart HUBa
- **Port:** `8124`
- **HTTP output ON request:** wpisz unikalny, losowy string, np. `ruch-w-kuchni-a3f9k2`

> **Uwaga:** Pole nazywa się „HTTP output", ale BE WAVE wysyła **surowy string TCP** — nie żądanie HTTP.  
> Wpisany tekst trafia do Node-RED dokładnie tak jak jest.

> **Bezpieczeństwo:** Protokół TCP nie weryfikuje tożsamości nadawcy — każdy, kto ma dostęp sieciowy do portu, może wysłać dowolny string. Node-RED reaguje wyłącznie na stringi zdefiniowane w mapowaniu, ale sam string pełni rolę jedynego „hasła". Używaj unikalnych, trudnych do odgadnięcia wartości zamiast czytelnych nazw. Najlepiej wygeneruj losowy sufiks, np. `ruch-w-kuchni-a3f9k2`. Ograniczenie dostępu do portu przez firewall (tylko z IP Smart HUBa) jest dodatkową warstwą ochrony.

### Node-RED — flow

Flow składa się z trzech węzłów:

1. `tcp in` — słucha na `0.0.0.0:8124`, tryb `stream`
2. `function` — parsuje string i mapuje go na nazwę eventu HA
3. `ha-fire-event` — odpala event w Home Assistant

**Przykładowy węzeł** `function`**:**

```javascript
const raw = msg.payload.toString().trim();

const map = {
    "ruch-w-kuchni": "bewave_kuchnia_ruch",
};

const eventType = map[raw];
if (eventType) {
    msg.payload = { event_type: eventType };
    return msg;
}
return null;
```

**Mapowanie stringów → eventy:**

| String TCP (BE WAVE) | Event Home Assistant  |
|----------------------|-----------------------|
| `ruch-w-kuchni`      | `bewave_kuchnia_ruch` |

### Automatyzacje Home Assistant

**Przykład: nocne światło w kuchni**

- **Trigger:** event `bewave_kuchnia_ruch`
- **Warunek:** pora nocna (po zachodzie, przed wschodem słońca)
- **Akcja:** włącz `light.kuchnia_nocny_led`, uruchom `timer.kuchnia_nocny_led_timer` (30 min)
- **Drugie wyzwolenie:** `timer.finished` → wyłącz światło

Każde kolejne wykrycie ruchu resetuje timer — światło zostaje włączone przez 30 minut od ostatniego ruchu.

---

## 🇬🇧 English

Integration of BE WAVE (SATEL Smart HUB) motion sensors with Home Assistant using Node-RED.

### The problem

The BE WAVE Smart HUB sends events as **raw TCP strings** — not HTTP, not MQTT, not REST.  
Home Assistant has no native support for this, so a bridge is needed.

### Architecture

```
APD-200 motion sensor
      │
      ▼
BE WAVE Smart HUB
      │  (raw TCP string, e.g. "motion-kitchen")
      ▼
Node-RED (TCP in → parse → ha-fire-event)
      │
      ▼
Home Assistant (event → automation)
```

### Requirements

**Hardware:**

- SATEL BE WAVE Smart HUB
- APD-200 motion sensor or any BE WAVE compatible sensor

**Software:**

- Home Assistant (HassOS or any other variant)
- Node-RED (HA add-on or standalone)
- `node-red-contrib-home-assistant-websocket` node

### Network setup

BE WAVE and HA can be on the same network or on separate VLANs.  
In this setup, BE WAVE is on a separate network (e.g. a camera VLAN) and HA exposes an additional interface on that same network:

| Device                                     | Example IP     |
|--------------------------------------------|----------------|
| BE WAVE Smart HUB                          | `192.168.2.13` |
| Home Assistant (BE WAVE network interface) | `192.168.2.10` |
| TCP port (Node-RED listener)               | `8124`         |

Firewall rule: allow TCP from the BE WAVE IP to HA on port 8124.

> If HA and BE WAVE are on the same network, no additional interface is needed — use the main HA IP.

### BE WAVE configuration — IP Output

In the BE WAVE Smart HUB app, configure **IP Output** for each sensor:

- **IP:** HA address reachable from the Smart HUB
- **Port:** `8124`
- **HTTP output ON request:** enter a unique, random-looking string, e.g. `motion-kitchen-a3f9k2`

> **Note:** Despite the label "HTTP output", BE WAVE sends a **raw TCP string** — not an HTTP request.  
> The text you enter arrives at Node-RED exactly as-is.

> **Security:** TCP has no sender authentication — anyone with network access to the port can send any string. Node-RED only reacts to strings defined in the mapping, but the string itself is the only "password". Use unique, hard-to-guess values rather than readable names. Generate a random suffix, e.g. `motion-kitchen-a3f9k2`. Restricting port access via firewall (only from the Smart HUB IP) adds an extra layer of protection.

### Node-RED flow

The flow consists of three nodes:

1. `tcp in` — listens on `0.0.0.0:8124`, mode `stream`
2. `function` — parses the string and maps it to an HA event name
3. `ha-fire-event` — fires the event in Home Assistant

**Example** `function` **node:**

```javascript
const raw = msg.payload.toString().trim();

const map = {
    "motion-kitchen": "bewave_kitchen_motion",
};

const eventType = map[raw];
if (eventType) {
    msg.payload = { event_type: eventType };
    return msg;
}
return null;
```

**String → event mapping:**

| TCP string (BE WAVE) | Home Assistant event    |
|----------------------|-------------------------|
| `motion-kitchen`     | `bewave_kitchen_motion` |

### Home Assistant automations

**Example: kitchen night light**

- **Trigger:** event `bewave_kitchen_motion`
- **Condition:** nighttime (after sunset, before sunrise)
- **Action:** turn on `light.kitchen_night_led`, start `timer.kitchen_night_led_timer` (30 min)
- **Second trigger:** `timer.finished` → turn off the light

Each subsequent motion detection resets the timer — the light stays on for 30 minutes since the last detected motion.

