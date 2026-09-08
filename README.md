# TWAI Daemon (`twai_daemon`)

Modernized ESP-IDF Two-Wire Automotive Interface (TWAI / CAN 2.0B) driver daemon built on the object-oriented `esp_driver_twai` architecture (`esp_twai.h`, `esp_twai_onchip.h`).

## Features

- **Zero-Copy Direct TX**: Telemetry tasks transmit frames via `twai_transmit_msg()` or `twai_transmit_frame()` through a static descriptor pool without intermediate FreeRTOS TX queues.
- **Event-Driven RX Dispatch**: Hardware ISR callbacks (`on_rx_done`) capture incoming frames directly into `g_twai_rx_queue`.
- **Automatic Bus-Off Recovery**: Hardware ISR detects `TWAI_ERROR_BUS_OFF` and triggers recovery via `twai_node_recover()`.
- **Direct Queue Access**: Draining and direct inspection via `twai_receive_queued_frame()` and `twai_clear_rx_queue()`.

## Target Support & Requirements

- **ESP-IDF**: `>= 5.5.0`
- **Supported Targets**: `esp32`, `esp32s2`, `esp32s3`, `esp32c3`, `esp32c5`, `esp32c6`, `esp32h2`, `esp32p4` (any target with on-chip TWAI peripheral).
- **Physical Bus**: Requires an external 3.3V CAN transceiver (e.g. SN65HVD230, TJA1050). Default bitrate: 500 kbps (Classic CAN 2.0B).

## Quick Start

### 1. Add Component Dependency

In your project or calling component's `idf_component.yml`:

```yaml
dependencies:
  twai_daemon:
    version: "^0.1.0"
```

Or clone directly into your project's `components/` directory.

### 2. Configuration (`Kconfig`)

Configure via `idf.py menuconfig` under **TWAI Daemon**:
- `CONFIG_CAN_TX`: TX GPIO pin (default: `39`, header fallback: `2`)
- `CONFIG_CAN_RX`: RX GPIO pin (default: `38`, header fallback: `3`)
- `CONFIG_CAN_CORE_AFFINITY`: FreeRTOS core pinned for RX worker (default: `1`)
- `CONFIG_CAN_RX_TIMEOUT_MS`: RX worker timeout (default: `3000` ms)
- `CONFIG_CAN_ENABLE_LOOPBACK`: Hardware self-reception for diagnostics (default: `n`)

### 3. Usage Example

```cpp
#include "twai_daemon.h"

// Optional: Custom frame dispatcher callback
static esp_err_t handle_can_frame(const twai_frame_t *frame) {
    printf("RX Frame ID: 0x%03lX, DLC: %d\n", frame->header.id, frame->header.dlc);
    return ESP_OK;
}

void app_main(void) {
    // Initialize CAN peripheral and start RX worker task
    ESP_ERROR_CHECK(initCAN(handle_can_frame));

    // Transmit a CAN frame (standard 11-bit ID, non-RTR, 10ms timeout)
    uint8_t payload[4] = {0x11, 0x22, 0x33, 0x44};
    esp_err_t err = twai_transmit_msg(0x120, payload, sizeof(payload), false, 10);
    if (err != ESP_OK) {
        printf("TX failed: %s\n", esp_err_to_name(err));
    }
}
```

For in-depth architectural details and sequence diagrams, refer to [TOO.MD](TOO.MD).

## License

[MIT](LICENSE) © 2026 Martin ROGER
