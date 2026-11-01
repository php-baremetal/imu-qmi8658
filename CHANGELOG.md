# Changelog

## 1.0.2

- Fixed: the constructor ignored failed configuration writes, so a half-configured chip passed as
  initialised. The first failed write now makes it throw `cannot init qmi8658 at 0x..: ...`.
- Changed: the descriptor no longer lists `0x6A` as a default address, since only the first default was ever
  used. A chip strapped to `0x6A` is constructed with `new Qmi8658($bus, 0x6A)`, as before.
- Changed: a negative `$hz` throws `\ValueError` (with ext-i2c 1.0.2); it used 400 kHz. `0` still does.

## 1.0.1

- Changed: the author's contact email.
- Docs: projects use `"sdk": "esp32@^2.0"`.

## 1.0.0

Initial release.
