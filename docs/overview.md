---
eyebrow: 'Package · Driver'
lede:    'The QMI8658 6-axis IMU as a PHP class: acceleration in g, rotation in degrees per second, chip temperature, and 12-byte background samples. Registers into ext-i2c.'
see_also:
  - { href: './usage/php-api.md', meta: 'PHP API' }
  - { href: './recipes/read-the-imu.md', meta: 'Recipe' }
  - { href: 'package:php-baremetal/ext-i2c', meta: 'Extension', label: 'ext-i2c' }
  - { href: 'package:php-baremetal/ext-i2c#docs/drivers/registry.md', meta: 'Registry', label: 'The i2c_drivers registry' }
prev: { label: 'No previous page', href: '#' }
next: { label: 'Qmi8658', href: './usage/php-api.md' }
---

# QMI8658 IMU

imu-qmi8658 adds the class `Baremetal\I2c\Driver\Qmi8658` to the PHP on your board. Constructing it checks
that a QMI8658 answers on the bus and configures it; then `accel()`, `gyro()` and `temp()` read the chip, and
`poll()` hands the reading to the executor task, which fills a ring buffer that PHP drains and decodes.

| Property | Value |
| --- | --- |
| Class | `Baremetal\I2c\Driver\Qmi8658`, extends `Baremetal\I2c\Device` |
| Capability | `Baremetal\Sensor\Imu` |
| Default address, clock | `0x6B`, 400 kHz |
| Ranges | accelerometer ±4 g, gyroscope ±512 dps, fixed |
| Units | `accel()` g, `gyro()` degrees per second, `temp()` °C |
| Background sample | 12 bytes (accelerometer then gyroscope) |

## Requirements

| Requires | Why |
| --- | --- |
| `php-baremetal/ext-i2c` | The `Bus` and `Device` classes and the `i2c_drivers` registry this driver registers into. |

ext-i2c in turn requires the virtual `php-baremetal/i2c-hal`, `php-baremetal/ext-events` and
`php-baremetal/bm-common`. You never require the HAL yourself: conductor adds
[esp32-i2c-hal](package:php-baremetal/esp32-i2c-hal) for an `esp32` SDK. The driver itself is C code with no
platform calls; it only uses the transport functions of ext-i2c.

## Install

<!-- @steps -->
- **Install conductor once per machine**

  `composer global require php-baremetal/conductor`

- **Create the project**

  `phpbm init my-imu` writes `composer.json` with conductor, the package registry, and the SDK, board, runtime
  and storage you pick.

- **Require the driver**

  `composer require php-baremetal/imu-qmi8658` (ext-i2c comes with it).

- **Build and flash**

  `phpbm flash`, then `phpbm monitor` for the console.
<!-- @endsteps -->

The resulting `composer.json` for a Waveshare ESP32-S3-Touch-LCD-1.85B, which has a QMI8658 on its on-board
bus (SDA 11 / SCL 10):

<!-- @code-block language="json" label="composer.json" -->
```json
{
  "name": "acme/my-imu",
  "type": "project",
  "repositories": [
    { "type": "composer", "url": "https://repository.php-baremetal.com", "canonical": true }
  ],
  "require": {
    "php-baremetal/conductor": "^1.0",
    "php-baremetal/imu-qmi8658": "^1.0"
  },
  "config": {
    "allow-plugins": { "php-baremetal/conductor": true },
    "platform": { "php": "8.5.11" }
  },
  "extra": {
    "baremetal": {
      "sdk": "esp32@^1.0",
      "board": "waveshare-esp32-s3-touch-lcd-1.85b",
      "runtime": "init-loop",
      "storage": "embedded"
    }
  }
}
```
<!-- @endcode-block -->

## A first script

<!-- @code-block language="php" label="project-src/index.php" -->
```php
<?php
use Baremetal\I2c\Bus;
use Baremetal\I2c\Driver\Qmi8658;

$imu = new Qmi8658(new Bus(sda: 11, scl: 10));

function loop(): void
{
    global $imu;

    [$ax, $ay, $az] = $imu->accel();
    [$gx, $gy, $gz] = $imu->gyro();
    printf("a [% .2f % .2f % .2f] g  w [% 6.1f % 6.1f % 6.1f] dps  %.1f C\n",
        $ax, $ay, $az, $gx, $gy, $gz, $imu->temp());
    usleep(500000);
}
```
<!-- @endcode-block -->

## PHP API at a glance

| Method | Returns | Meaning |
| --- | --- | --- |
| `new Qmi8658(Bus $bus, int $address = 0x6B, int $hz = 400000)` | | Attach, check the chip identity, configure the ranges. |
| `accel()` | `array` | `[x, y, z]` in g, read now. |
| `gyro()` | `array` | `[x, y, z]` in degrees per second, read now. |
| `temp()` | `float` | Chip temperature in °C. |
| `decode(string $raw)` | `array` | `['accel' => [x, y, z], 'gyro' => [x, y, z]]` from one 12-byte sample. |

The class also has every `Device` method (`probe()`, `readReg()`, `poll()`, `drain()`, …). Details in
[Qmi8658](./usage/php-api.md).

## Registration

The package's `composer.json` declares `"register": { "i2c_drivers": ["qmi8658_driver_desc"] }`. At build time
the descriptor `qmi8658_driver_desc` goes into ext-i2c's driver table; at startup ext-i2c registers the class
from it. Nothing to configure: requiring the package is enough, and `Bus::scan()` reports `qmi8658` for a
device opened through this class. How the registry works:
[The i2c_drivers registry](package:php-baremetal/ext-i2c#docs/drivers/registry.md).

## Notes

- The constructor reads the identity register (`0x00`) and expects `0x05`. Any other value, or no answer,
  throws `cannot init qmi8658 at 0x6B: ESP_ERR_NOT_FOUND` (or the transfer error, such as `ESP_ERR_TIMEOUT`).
- The chip is configured at construction: address auto-increment, ±4 g, ±512 dps, both sensors on. The ranges
  cannot be changed through the class; use `writeReg()` only if you then convert the raw values yourself.
- The driver's descriptor also lists `0x6A` (the chip's other strap address), but the constructor only
  defaults to `0x6B`. For a chip strapped to `0x6A`, pass it: `new Qmi8658($bus, 0x6A)`.
- `poll()` runs on the executor task. With the `power-save` option on, sampling still works on the default
  `Bus::SYNC` bus, but each sample wakes the chip. See
  [Bus ownership and power-save](package:php-baremetal/ext-i2c#docs/reference/ownership-and-power.md).
- Failed reads throw `\Exception` with `QMI8658 accel read failed`, `QMI8658 gyro read failed` or
  `QMI8658 temp read failed`.
