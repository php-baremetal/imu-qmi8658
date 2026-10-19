# imu-qmi8658

A driver for the QMI8658 6-axis IMU (accelerometer, gyroscope, temperature) on I2C. It registers into
ext-i2c's `i2c_drivers` registry, so a firmware built with it has the class `Baremetal\I2c\Driver\Qmi8658`: it
extends `Baremetal\I2c\Device`, implements `Baremetal\Sensor\Imu`, reads acceleration in g and rotation in
degrees per second, and can sample the chip in the background.

The package requires [ext-i2c](package:php-baremetal/ext-i2c), so requiring the driver alone is enough;
conductor adds the I2C HAL for your SDK platform.

## Install

In a project created with `phpbm init`:

```bash
composer require php-baremetal/imu-qmi8658
```

## Example

```php
<?php
use Baremetal\I2c\Bus;
use Baremetal\I2c\Driver\Qmi8658;

$imu = new Qmi8658(new Bus(sda: 11, scl: 10)); // 0x6B, 400 kHz

[$ax, $ay, $az] = $imu->accel();               // g
[$gx, $gy, $gz] = $imu->gyro();                // degrees per second
printf("accel [% .2f % .2f % .2f] g, %.1f C\n", $ax, $ay, $az, $imu->temp());
```

## Documentation

- [Overview](docs/overview.md): requirements, install, a first project, notes.
- [PHP API](docs/usage/php-api.md): constructor, methods, units, the raw sample and `decode()`.
- [Read the IMU](docs/recipes/read-the-imu.md) and [Sample in the background](docs/recipes/background-sampling.md)

## License

MIT.
