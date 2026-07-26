# ejeas-reveng
Reverse engineering of protocol to talk with EJEAS (X10) motorcycle intercom system.

This goes together with a blog post over at https://spaq.in/blog/ejeas-reveng

The work here has been LLM assisted. I'll mark these parts in more detail.

Links here are from the official site (yes they use Google Drive), or pulled from the API.

# Hardware
![the device](device.jpg)

On the outside, few buttons, a scroll wheel, USB-C for charging and DFU. Requires a special cable with additional pins, bundled with the unit. On the bottom, pins for audio output (stereo) and microphone.

Wireless capabilities include:
Bluetooth (LE) for audio,
FM radio,
MESH (intercom).

As it's rain resistant, I am not opening it.

Inside, a Qualcomm QCC512X according to firmware (below), and an unknown JiaLi chip (mentioned in the app). The exact setup who is responsible for what, not sure (voice bits?). QC does not support FM radio, and the device that pops up when you connect a cable seems to be the JiaLi.

# [Firmware](https://drive.google.com/drive/folders/1yATB-rH9VwsFFR_HuOvPrw2ZLpg9rqqa)

That's only for the BLE module, MESH is not available at all.

Separate firmware files for different voice bits (e.g. poweron, shutdown, mode change) available from the manufacturer's website. They differ in size. Only for the "international" hardware version. That one's officially stuck with Putonghwa. 

Firmware if opened in hex editor starts with ``APPUPHDR``, chip info and proceeds to encrypted ``PARTDATA``. Ends with ``APPUPFTR`` (also looks encrypted).

Firmware zip comes with a DFU tool. Only English and Spanish firmwares are available online.

OTA updates are available as well, through the app.

## OTA Updates

Only through the app. App asks the API of the manufacturer, sending the current versions and language, gets a response that either says it's already the newest version, or a link to another binary.

In the app, the Chinese X10 Phantom is marked as ID "45", "model": "X10 PhantomB".

Unfortunately, the fw versions binaries seem to be randomly generated; no guesswork.

[v1.9.0 BLE Chinese fw](https://app-obs-cn.obs.cn-east-3.myhuaweicloud.com/device_firmware/20260520/46c59757-3d1e-4747-a33a-92babf725167.bin)

[v3.12.37 MESH fw](app-obs-cn.obs.cn-east-3.myhuaweicloud.com/device_firmware/20260526/98411812-4f32-457c-83d1-9ee181a0f5ab.zip) (ufw file in a zip)

If you play with the language, you can find others:

[v1.9.0 BLE English fw (language = 3)](https://app-obs-cn.obs.cn-east-3.myhuaweicloud.com/device_firmware/20260520/b6404c83-b6e9-4fe5-9007-1b80e6895027.bin)

[v1.9.0 BLE language 4 (?)](https://app-obs-cn.obs.cn-east-3.myhuaweicloud.com/device_firmware/20260520/6230109b-7232-4583-81b7-8b23c4639417.bin)

[v1.9.0 BLE language 5 (?)](https://app-obs-cn.obs.cn-east-3.myhuaweicloud.com/device_firmware/20260520/c0781656-847b-4a03-8e26-8586fab49911.bin)

language 6 does not exist

[v1.9.0 BLE language 7 (?)](https://app-obs-cn.obs.cn-east-3.myhuaweicloud.com/device_firmware/20260520/475d308d-d56d-4aef-b29c-cc046b7d142a.bin)

[v1.9.0 BLE language 8 (?)](https://app-obs-cn.obs.cn-east-3.myhuaweicloud.com/device_firmware/20260520/56b14dda-55dd-4ebe-9ce0-4305e7b2f09b.bin)

[v1.9.0 BLE language 9 (?)](https://app-obs-cn.obs.cn-east-3.myhuaweicloud.com/device_firmware/20260520/0938f7fe-aac7-4a9c-bd3d-dd231e78040a.bin)

[v1.9.0 BLE language 10 (?)](https://app-obs-cn.obs.cn-east-3.myhuaweicloud.com/device_firmware/20260520/a4f2b390-6a7a-4097-a298-637b51bc1298.bin)

language 11 does not exist

[v1.9.0 BLE language 12 (?)](https://app-obs-cn.obs.cn-east-3.myhuaweicloud.com/device_firmware/20260520/9461a862-2fed-4455-a313-85e93aad388a.bin)

13 and 14 don't exist either, haven't gone further.

These binaries might be flashable with the DFU tool (the ones available online on the official app are not, probably because they're designed for a different model number ("40", "X10 Phantom" (no "B"))).

# DFU flasher tool

Written in ``.net``, barebones wrapper for a DFU library. Easy to open in ILSpy. No particular checks for the firmware - these are done on the hardware, the flasher just reports an error as it comes.

# [Android app](https://drive.google.com/file/d/16MjbAWICSdgqtGi2wdei8YBRLDZbsfXS/view)

This is the part that I used an LLM for.

It uses Bluetooth LE to communicate with the unit for settings. It's also responsible for voice recognition for voice commands. Deals with OTA.

OTA is done with Qualcomm's library, GAIA protocol. The settings seem to be done with a proprietary in-house protocol, where the app sends packets starting with 0xAA, and responses come with 0xBB first.

It's also a Flutter app, so the most of the business logic is in ``libapp.so`` that needs further decompilation. Uses SSL pinning, so use Frida to get around that if you want to listen to API calls.

That also means that if you want to listen into the API calls, you need to use Frida, and a flutter SSL bypass codeshare script.

The findings of the protocol can be found in [protocol-1.md](./protocol-1.md) and [protocol-2.md](./protocol-2.md) - these are LLM generated files, based on different logs, included in the ble_logs directory, logs generated by the app (can be found in the app data)

# Next steps

A free open source app that would perform the app's functions without bloat. Maybe with OTA (choose your binary).

# Thanks

mom and dad - love you

EJEAS - the guy who decided to region lock the device, hope he chokes on a bag of dicks, but without him I wouldn't even bother looking into it