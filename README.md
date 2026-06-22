# Gobbledegook

Gobbledegook is a proof of concept (PoC) hotplug attack payload resistant to post-incident disk-forensics targeting Linux systems.

**Lab Environment:**
- Ubuntu Server 24.04
- Ubuntu Desktop 24.04
- [Hak5 USB Rubber Ducky](https://shop.hak5.org/products/usb-rubber-ducky).

## Abstract
Gobbledegook utilizes a two step encryption method implemented via `openssl` to encrypt an attacker's payload as well as encrypt the hot plug downloader payload performed by the hot plug device to increase resistance to post-incident disk-forensic. The payload is encrypted and `base64` wrapped via `openssl` using symmetric environmental keying derived from "architectual parameters" gathered via `lsusb -v` from the hot plug device itself. This means payload decryption key derivation is performed in memory at runtime of the attack; not stored on the hot plug device or fetched from external sources. This allows an attacker to host payloads (semi-secure-ish-ly) on publicly viewable servers such as [Github](https://github.com) being utilized as a payload delivery server. The values gathered from "architectual parameters" are not typicaly logged to disk and are then strung together to create a key via `lsusb -v -d 0x046D:0xC05A 2>/dev/null | grep -E "bmAttri|MaxPower|wMaxPacket" | awk '"'"'{print $2}'"'"' | paste -sd "-" -` (key example: `0xc0-100mA-3-0x0008`). Hot plug vendor and product ID are spoofed (`VID_0x046D PID_0xC05A MAN_Logitech SERIAL_0 PROD_M100`) so that post-incident log analysis identifies the hot plug device as an innocuous Logitech mouse. The hot plug downloader payload (as previously mentioned) is also encrypted. This means that commands injected are not decrypted until runtime, thus, any logging of terminal command input will only provide cipher text; `set +o history && unset HISTFILE` are also utilized as additional methods to reduce the logging surface. Once injected, decryption key derivation is perfomed and passed to open `openssl` to decrypt the downloader payload. The payload is then piped to `Bash` to be executed in memory, forgoing writing any file to disk.


, this allows an attacker to host payloads (semi-securely-ish) on publicly viewable servers such as [Github](https://github.com).
