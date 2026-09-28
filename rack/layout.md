# Rack Layout

## GEEEKPi DeskPi RackMate T1

- **Rack:** GEEEKPi DeskPi RackMate T1
- **Capacity:** 8U
- **Width:** 10-inch
- **Depth:** 7.87 inches

## Physical Equipment Order

The current physical arrangement is documented from **top to bottom**. The Minisforum X1 Lite-255 AI server was added between the network switch and the Beelink Kali box.

| Order | Equipment | Role |
|---|---|---|
| **1** | GL.iNet GL-BE9300 Flint 3 | Wi-Fi 7 access point |
| **2** | AC Infinity MULTIFAN S3 | Exhaust fan |
| **3** | TP-Link TL-SG108E | 8-port managed Gigabit switch |
| **4** | MINISFORUM X1 Lite-255 | AI server / 24/7 LLM |
| **5** | Raspberry Pi 5 | Pi-hole ad blocker |
| **6** | Beelink EQ Mini PC / Intel N150 | Kali box / Lab zone |
| **7** | Intel J1900 mini PC | pfSense router/firewall |
| **8** | ElecVoztile PDU | Rack power distribution |
| **9** | Noctua NF-F12 5V | Intake fan |

> **Rack capacity note:** The rack is rated at 8U, while the current physical equipment list contains 9 pieces of equipment. Several items (including the fans, PDU, and mini PCs) do not necessarily occupy a full conventional 1U rack space. The table therefore records the physical order rather than claiming that each item occupies exactly one rack unit.

## Airflow

The rack uses a dedicated intake and exhaust fan arrangement:

```text
             Rack top
                |
                v
        +-------------------+
        | Flint 3           |
        | EXHAUST FAN       | ---> Exhaust airflow
        | Network switch    |
        | AI Server         |
        | Raspberry Pi 5    |
        | Beelink / N150    |
        | pfSense J1900     |
        | PDU               |
        | INTAKE FAN        | <--- Intake airflow
        +-------------------+
                ^
                |
             Rack bottom
```

## KVM Switch

The lab also uses a **KCEVE 8-port HDMI KVM switch** for centralized local console access.

- **Manufacturer:** KCEVE
- **Type:** HDMI KVM switch
- **Ports:** 8 computers / 1 monitor
- **USB:** USB 3.0
- **Video:** Supports up to 4K @ 60 Hz
- **USB peripherals:** Shared keyboard and mouse
- **Switching:** Hotkey switching supported
- **Role:** Centralized keyboard, mouse, and monitor access for multiple computers
- **Physical position:** External rack accessory

### KVM Port Assignments

| KVM Port | Connected Computer | Role |
|---|---|---|
| **Port 1** | Intel J1900 mini PC | pfSense router/firewall |
| **Port 2** | Beelink EQ Mini PC / N150 | Kali box / Lab zone |
| **Port 3** | Raspberry Pi 5 | Pi-hole ad blocker |
| **Port 4** | Unassigned | Available |
| **Port 5** | Unassigned | Available |
| **Port 6** | Unassigned | Available |
| **Port 7** | Unassigned | Available |
| **Port 8** | Unassigned | Available |
