# Gobbledegook

Gobbledegook is a proof of concept (PoC) hotplug attack payload resistant to post-incident disk-forensics targeting Linux systems.

**Lab Environment:**
- Ubuntu Server 24.04
- Ubuntu Desktop 24.04
- [Hak5 USB Rubber Ducky](https://shop.hak5.org/products/usb-rubber-ducky).

## Abstract
Gobbledegook utilizes a two step payload encryption method implemented via `openssl` to encrypt an attacker's payload as well as encrypt the hot plug downloader payload to increase resistance to post-incident disk-forensics. The payload is encrypted and `base64` wrapped via `openssl` using symmetric environmental keying derived from "architectual parameters" gathered via `lsusb -v` from the hot plug device itself. This means payload decryption key derivation is performed in memory at runtime of the attack; not stored on the hot plug device, fetched from external sources, or saved to disk. This allows an attacker to easily host payloads (semi-secure-ish-ly) on publicly viewable platforms such as [Github](https://github.com) being utilized as a payload delivery server. The values gathered from "architectual parameters" are not typicaly logged to disk and are strung together to create a key via `lsusb -v -d 0x046D:0xC05A 2>/dev/null | grep -E "bmAttri|MaxPower|wMaxPacket" | awk '"'"'{print $2}'"'"' | paste -sd "-" -` (key example: `0xc0-100mA-3-0x0008`). Hot plug vendor and product ID are spoofed (`VID_0x046D PID_0xC05A MAN_Logitech SERIAL_0 PROD_M100`) so that post-incident log analysis identifies the device as an innocuous Logitech mouse. The hot plug downloader payload (as previously mentioned) is also encrypted. This means that commands injected are not decrypted until runtime, thus, any logging of terminal command input will only provide cipher text; `set +o history && unset HISTFILE` is also utilized as additional methods to reduce the logging surface. Once injected decryption key derivation is perfomed and passed to `openssl` to decrypt the downloader payload. The payload is then piped to `Bash` to be executed in memory, forgoing writing any file to disk. The primary payload is then fetched via `curl`, passed to `openssl` for decryption, and then piped to `Bash` for execution; the process is performed entirely in memory. Terminal `exit` is then used post-execution.

## Use Case
Disk-forensic operational security considerations when attempting to compromise Linux machines via hot plug attack.

## Considerations
While steps in this PoC carry an emphasis on anti disk-forensics, it should be noted that *it is **NOT** all encompassing* regarding total forensic circumvention. Custom logging, EDR systems, network monitoring systems, RAM/process monitoring, recovering data from non re-allocated memory space, etc may circumvent anti-forensic efforts.
