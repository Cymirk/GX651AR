ASUS GX651AR Armoury Power Limits

Adds ASUS ROG Zephyrus Duo GX651AR support to the Linux asus-armoury power-limit table.

This patch adds a DMI-based entry for the `GX651AR` board in `drivers/platform/x86/asus-armoury.h`. It defines the configurable CPU and NVIDIA GPU power-control ranges exposed by ASUS Armoury.

The patch defines the following ranges for AC power and, where specified, battery power.

| Setting | AC power | Battery power |
| CPU PL1 / SPL | 30–75 W | 30–75 W |
| CPU PL2 / SPPT | 38–80 W | 38–80 W |
| NVIDIA Dynamic Boost | 5–25 W | Not defined by this entry |
| NVIDIA temperature target | 75–87 °C | 75–87 °C |
| NVIDIA TGP | 80–115 W | Not defined by this entry |

These values define the ranges available through the Armoury controls. They do not mean that the CPU or GPU continuously operates at the maximum value.


## Patch sent upstream https://lists.openwall.net/linux-kernel/2026/09/18/2887?
