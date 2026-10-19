---
eyebrow: 'imu-qmi8658 · Recipe'
lede:    'Let the executor read the QMI8658 at a fixed rate into a ring buffer, then drain and decode the samples from loop() without touching the bus.'
see_also:
  - { href: '../usage/php-api.md', meta: 'PHP API' }
  - { href: 'package:php-baremetal/ext-i2c#docs/usage/polling-and-events.md', meta: 'ext-i2c', label: 'Background sampling and events' }
  - { href: 'package:php-baremetal/ext-i2c#docs/recipes/imu-events.md', meta: 'ext-i2c', label: 'Stream IMU samples as events' }
prev: { label: 'Read the IMU', href: './read-the-imu.md' }
next: { label: 'No next page', href: '#' }
---

# Sample in the background

100 samples a second from the QMI8658 of a Waveshare ESP32-S3-Touch-LCD-1.85B (SDA 11 / SCL 10). The executor
task takes the samples; `loop()` wakes five times a second, drains what has accumulated and prints the average.

## The project

The same `composer.json` as [Read the IMU](./read-the-imu.md): `imu-qmi8658` required, runtime `init-loop`.

## The script

<!-- @code-block language="php" label="project-src/index.php" -->
```php
<?php
use Baremetal\I2c\Bus;
use Baremetal\I2c\Driver\Qmi8658;

$imu = new Qmi8658(new Bus(sda: 11, scl: 10));
$imu->poll(hz: 100, depth: 128);

function loop(): void
{
    global $imu;

    $samples = $imu->drain();
    if ($samples === []) {
        usleep(200000);
        return;
    }

    $sum = [0.0, 0.0, 0.0];
    foreach ($samples as $raw) {
        ['accel' => $a] = $imu->decode($raw);
        foreach ($a as $i => $v) {
            $sum[$i] += $v;
        }
    }
    $n = count($samples);
    printf("%3d samples, mean accel [% .3f % .3f % .3f] g\n",
        $n, $sum[0] / $n, $sum[1] / $n, $sum[2] / $n);
    usleep(200000);
}
```
<!-- @endcode-block -->

## How it works

- `poll(hz: 100, depth: 128)` starts a sampler on the executor task: every 10 ms one 12-byte read into a ring of
  128 samples. In 200 ms about 20 samples accumulate, well below the ring size; if the script stalls longer than
  1.28 s, the oldest samples are overwritten.
- `drain()` returns the unread samples oldest first and never touches the bus, so it is cheap to call often.
- `decode()` turns each raw string into g and degrees per second; it does not touch the bus either.
- To react to each sample instead of polling the ring, pass `event:` to `poll()` and use the `event-driven`
  runtime: see [Stream IMU samples as events](package:php-baremetal/ext-i2c#docs/recipes/imu-events.md).

<!-- @callout variant="note" title="One sampler per device" -->
Calling `poll()` again on the same object changes its rate. The executor runs at most eight samplers for the
whole firmware, shared with every other polled device.
<!-- @endcallout -->
