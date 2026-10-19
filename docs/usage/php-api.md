---
eyebrow: 'imu-qmi8658 · PHP API'
lede:    'Baremetal\I2c\Driver\Qmi8658: the constructor, the three readings with their units, and the 12-byte raw sample that poll() collects and decode() converts.'
see_also:
  - { href: '../recipes/background-sampling.md', meta: 'Recipe' }
  - { href: 'package:php-baremetal/ext-i2c#docs/usage/device.md', meta: 'ext-i2c', label: 'Device' }
  - { href: 'package:php-baremetal/ext-i2c#docs/usage/polling-and-events.md', meta: 'ext-i2c', label: 'Background sampling and events' }
prev: { label: 'Overview', href: '../overview.md' }
next: { label: 'Read the IMU', href: '../recipes/read-the-imu.md' }
---

# Qmi8658

`Baremetal\I2c\Driver\Qmi8658` extends `Baremetal\I2c\Device` and implements `Baremetal\Sensor\Imu`. All the
`Device` methods work on it too; this page covers what the driver adds.

## Constructor

<!-- @method name="new Qmi8658(Bus $bus, int $address = 0x6B, int $hz = 400000)" visibility="public" -->

<!-- @params -->
<!-- @param name="$bus" type="Bus" required -->
The bus the chip is wired to.
<!-- @endparam -->
<!-- @param name="$address" type="int" default="0x6B" -->
7-bit address. Pass `0x6A` for a chip strapped to its other address.
<!-- @endparam -->
<!-- @param name="$hz" type="int" default="400000" -->
SCL clock for this device. A value of 0 or less uses 400 kHz.
<!-- @endparam -->
<!-- @endparams -->

The constructor attaches the device, reads the identity register and configures the chip: address
auto-increment, accelerometer ±4 g, gyroscope ±512 dps, both enabled, then a 10 ms settle. It throws
`\Exception`:

| Message | Cause |
| --- | --- |
| `cannot init qmi8658 at 0x..: ESP_ERR_NOT_FOUND` | Something answered, but its identity is not `0x05`. |
| `cannot init qmi8658 at 0x..: ...` | The identity read failed, for example `ESP_ERR_TIMEOUT` with no chip. |
| `address 0x.. on this bus is already used by another driver` | A raw device or another driver holds the address. |

Constructing it again with the same bus and address returns a handle to the same device; the chip is not
reconfigured.

## Readings

<!-- @method name="Qmi8658::accel(): array" returns="array" visibility="public" -->

`[x, y, z]` acceleration in g (8192 counts per g). Throws `QMI8658 accel read failed`.

<!-- @method name="Qmi8658::gyro(): array" returns="array" visibility="public" -->

`[x, y, z]` angular rate in degrees per second (64 counts per dps). Throws `QMI8658 gyro read failed`.

<!-- @method name="Qmi8658::temp(): float" returns="float" visibility="public" -->

Chip temperature in °C (raw value / 256). Throws `QMI8658 temp read failed`.

Each call is one bus transaction (6 bytes for `accel()` and `gyro()`, 2 for `temp()`).

## Background samples

<!-- @method name="Qmi8658::decode(string $raw): array" returns="array" visibility="public" -->

Converts one raw sample from `sample()` or `drain()` into
`['accel' => [x, y, z], 'gyro' => [x, y, z]]`, in g and degrees per second. Throws `\ValueError`
(`a QMI8658 sample is 12 bytes`) for a shorter string.

The driver declares a sampler, so `poll()`, `sample()` and `drain()` work on this class. Each sample is one
12-byte read starting at register `0x35`:

| Bytes | Content |
| --- | --- |
| 0 to 5 | accelerometer x, y, z, signed 16-bit little-endian |
| 6 to 11 | gyroscope x, y, z, signed 16-bit little-endian |

Temperature is not part of the sample. The descriptor states a 1000 Hz maximum rate; ext-i2c does not enforce
it, and `poll()` cannot go faster than one sample per millisecond anyway.

<!-- @code-block language="php" label="drain and decode" -->
```php
<?php
use Baremetal\I2c\Bus;
use Baremetal\I2c\Driver\Qmi8658;

$imu = new Qmi8658(new Bus(sda: 11, scl: 10));
$imu->poll(hz: 50, depth: 64);

function loop(): void
{
    global $imu;

    foreach ($imu->drain() as $raw) {
        ['accel' => [, , $az]] = $imu->decode($raw);
        printf("az % .3f g\n", $az);
    }
    usleep(250000);
}
```
<!-- @endcode-block -->

## Capability

The class implements `Baremetal\Sensor\Imu`, a marker interface ext-i2c always defines. Type a parameter or an
`instanceof` check on it to accept any IMU driver rather than this chip; the interface declares no methods.
