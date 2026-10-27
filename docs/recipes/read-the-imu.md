---
eyebrow: 'imu-qmi8658 · Recipe'
lede:    'Read acceleration, rotation and temperature on demand in an init-loop project, and turn the gravity vector into a tilt angle.'
see_also:
  - { href: '../usage/php-api.md', meta: 'PHP API' }
  - { href: './background-sampling.md', meta: 'Recipe' }
prev: { label: 'Qmi8658', href: '../usage/php-api.md' }
next: { label: 'Sample in the background', href: './background-sampling.md' }
---

# Read the IMU

The QMI8658 of a Waveshare ESP32-S3-Touch-LCD-1.85B, on the on-board bus (SDA 11 / SCL 10). Other wiring: a
QMI8658 module on any two free pins, with pull-ups on SDA and SCL, and `SA0` deciding `0x6B` or `0x6A`.

## The project

<!-- @code-block language="json" label="composer.json" -->
```json
{
  "name": "acme/read-imu",
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
      "sdk": "esp32@^2.0",
      "board": "waveshare-esp32-s3-touch-lcd-1.85b",
      "runtime": "init-loop",
      "storage": "embedded"
    }
  }
}
```
<!-- @endcode-block -->

## The script

<!-- @code-block language="php" label="project-src/index.php" -->
```php
<?php
use Baremetal\I2c\Bus;
use Baremetal\I2c\Driver\Qmi8658;

$bus = new Bus(sda: 11, scl: 10);
$imu = new Qmi8658($bus);

foreach ($bus->scan() as $address => $info) {
    printf("0x%02X %s\n", $address, $info['driver'] ?? '(no driver)');
}

function loop(): void
{
    global $imu;

    [$ax, $ay, $az] = $imu->accel();
    [$gx, $gy, $gz] = $imu->gyro();

    $pitch = rad2deg(atan2(-$ax, sqrt($ay * $ay + $az * $az)));
    $roll  = rad2deg(atan2($ay, $az));

    printf("pitch % 6.1f  roll % 6.1f  |  w [% 6.1f % 6.1f % 6.1f] dps  %.1f C\n",
        $pitch, $roll, $gx, $gy, $gz, $imu->temp());
    usleep(200000);
}
```
<!-- @endcode-block -->

## How it works

- `new Qmi8658($bus)` checks the identity and configures the chip once; the scan then lists `0x6B qmi8658`.
- Each reading is one transaction on the PHP task, under the bus lock. Three calls per `loop()` are three
  transactions.
- At rest the accelerometer reads about 1 g on the axis that points up; pitch and roll come from that vector and
  are only meaningful while the board is not accelerating.
- For fixed-rate data without timing jitter from the script, let the executor sample instead:
  [Sample in the background](./background-sampling.md).
