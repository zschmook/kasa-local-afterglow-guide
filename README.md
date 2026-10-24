# Kasa bulbs in Afterglow without a TP-Link account

An investigation and working local setup for one Kasa KL110 bulb, recorded on October 24, 2026.

**Status:** Both Reading and Living Room were factory-reset, provisioned directly onto home Wi-Fi without a TP-Link account, given separate internet-block rules, and registered in Afterglow. Both answered local identity and state queries. The owner confirmed Reading's controls work; successful control requests were also observed in the server log.

This is an observed result on two devices with the same hardware and firmware versions, not a promise that every Kasa bulb behaves the same way.

## What we tested

| Component | Observed version or behavior |
| --- | --- |
| Reading bulb | KL110(US), hardware 4.6 |
| Reading firmware | 1.1.1 Build 260710 Rel.082646 |
| LAN discovery | UDP 20002, `IOT.SMARTBULB`, KLAP, login version 2 |
| Local transport that worked after reset | `KlapTransportV2` |
| Python library | `python-kasa==0.10.2` |
| Router | NETGEAR R7800, stock web interface |
| Dashboard | Afterglow, native Python server on port 10000 |

The network addresses, full device MAC addresses, router credentials, Wi-Fi password, TP-Link account details, and authentication keys are intentionally excluded.

## Why the lights did not appear

The original Afterglow implementation queried the older Kasa LAN protocol on UDP port 9999. The two new bulbs answered discovery on UDP 20002 and advertised KLAP instead. Legacy UDP discovery did not identify them, and password-free legacy TCP connections to their home-network addresses were refused.

Afterglow already contained an unrelated, stale entry named "Living Room Light." That was an old discarded bulb, not the new Living Room device. Keep stable device identities separate from human-readable names when diagnosing replacements.

The investigation also established that the inspected Afterglow server had its own home-device implementation and data file. Reading was registered without restarting Afterglow.

## What did not work

1. Refreshing known lights did not discover new devices.
2. Scanning only legacy Kasa UDP 9999 did not find the two new bulbs.
3. Using blank/default authentication while the new bulbs remained paired did not provide local access.
4. A newly added TP-Link sign-in card did not successfully connect the owner's account to these bulbs. The first implementation also hid distinct failures behind one generic error.
5. Supporting both KLAP handshake variants and improving errors did not establish successful account-based access on the real bulbs. Passing mocked tests was not evidence that the owner's real sign-in worked.

We did not establish the cause of the account-authentication rejection. It must not be represented as a confirmed incorrect password.

TP-Link documents a **Third-Party Compatibility** setting in Kasa under **Me → Settings**, and in Tapo under **Me → Third-Party Services**. That was researched as a possible account-authentication remedy, but was not the verified solution in this session. [TP-Link documentation](https://www.tp-link.com/uk/support/faq/4416/)

## The local setup that worked for Reading

### 1. Reset only the test bulb

Reading was power-cycled ten times, ending powered on. It flashed and advertised its own open setup Wi-Fi network. A later query identified its hardware as version 4.6.

Reset sequences vary by hardware revision. TP-Link lists five cycles for factory reset on KL110 v1, and ten for the newer KL110 revisions covered by its guide, with approximately one second between switches. Three cycles is a soft reset on some older bulbs. Do not assume the same sequence across all models. [Official reset guide](https://www.tp-link.com/us/support/faq/2660/)

Do not add the bulb back through the Kasa app during this test: the goal is to provision Wi-Fi without binding it to a TP-Link account.

### 2. Join the bulb's setup network

The setup SSID followed the pattern `TP-LINK_Smart Bulb_<MAC suffix>`.

On this device:

- The bulb used `192.168.0.1` on its setup network.
- The PC received `192.168.0.100` through DHCP.
- The setup network was open and used 2.4 GHz Wi-Fi.
- UDP 20002 discovery returned the expected model and MAC address.

The Windows Wi-Fi network list initially showed only the connected network. An explicit WLAN scan revealed the bulb's setup network. One empty or stale scan is not proof that reset failed.

Validate the bulb's MAC address before sending configuration. A generic setup SSID or matching model alone is insufficient.

This read-only probe asks the setup network which protocol the bulb speaks. It prints the model and encryption scheme, not credentials or full device identifiers:

```python
import json
import socket

probe = bytes.fromhex("020000010000000000000000463cb5d3")
with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as client:
    client.settimeout(3)
    client.sendto(probe, ("192.168.0.1", 20002))
    packet, sender = client.recvfrom(8192)
    if sender != ("192.168.0.1", 20002):
        raise RuntimeError("Unexpected discovery response source")
    result = json.loads(packet[16:])["result"]
    print(result["device_model"], result["mgt_encrypt_schm"])
```

### 3. Check local authentication before changing Wi-Fi

The helper tried a bounded set of local transports and read `system.get_sysinfo`. Early attempts timed out. A subsequent setup-network discovery and KLAP v2 query succeeded without owner account credentials.

Discovery changed from `ANS: true` while paired on the home LAN to `ANS: false` after reset in setup mode. These are recorded observations, not a claim that changing that field itself bypasses authentication.

The successful query supplied Reading's identity, firmware, and hardware version. A Wi-Fi scan through the bulb then found the home's 2.4 GHz network.

Some Kasa system-information responses use `mic_mac` instead of `mac`; identity validation must handle both.

The following example uses the same KLAP v2 transport that worked in this session. It reads identity information and always closes the connection. Run it while connected to the reset bulb's setup network, with `python-kasa==0.10.2` installed:

```python
import asyncio
from contextlib import asynccontextmanager

from kasa import DeviceConfig
from kasa.iot import IotBulb
from kasa.protocols.iotprotocol import IotProtocol
from kasa.transports.klaptransport import KlapTransportV2


@asynccontextmanager
async def local_bulb(host):
    protocol = IotProtocol(
        transport=KlapTransportV2(config=DeviceConfig(host=host, timeout=4))
    )
    try:
        reply = await protocol.query({"system": {"get_sysinfo": {}}})
        info = reply["system"]["get_sysinfo"]
        bulb = IotBulb(host, protocol=protocol)
        bulb.update_from_discover_info(reply)
        yield bulb, info
    finally:
        await protocol.close()


async def inspect():
    async with local_bulb("192.168.0.1") as (bulb, info):
        print(info["model"], info["hw_ver"], info["sw_ver"])


asyncio.run(inspect())
```

No owner credentials are supplied here. A paired device may reject this query; the example does not bypass a paired device's authentication.

### 4. Block the bulb's internet traffic before reconnecting it

On the R7800, the working configuration was:

1. Reserve one LAN IP address for Reading's exact MAC address.
2. Add one **Block Services** rule for **only that reserved IP**.
3. Select **TCP/UDP**, ports **1–65535**.
4. Set service blocking to **Always**.
5. Read the router settings back and verify both the reservation and the specific rule.

This uses the router's internet-service filter rather than a broad Wi-Fi denial. Do not accidentally select **All IP Addresses**. Other devices and existing rules must be preserved. [NETGEAR Block Services documentation](https://kb.netgear.com/24054/How-do-I-block-web-services-on-my-NETGEAR-router-from-the-router-web-interface)

The rule was confirmed enabled in the router, and local bulb access was confirmed afterward. We did not perform a packet capture or independently prove every possible egress path, including IPv6. Treat this as the verified TCP/UDP IPv4 configuration, not an exhaustive network-isolation audit.

### 5. Provision Wi-Fi directly

Using the verified local connection:

1. Set the bulb alias to `Reading`.
2. Call the library's `wifi_join` with the home's **2.4 GHz** SSID and Wi-Fi key.
3. Keep the key in memory; do not print it, put it in command arguments, or save it in the guide.
4. Restore the PC's original Wi-Fi connection in a `finally` block, including after a timeout or other failure.

Reading acknowledged the Wi-Fi join and returned at its reserved LAN IP address. No cloud-binding command was sent.

The library documents direct Wi-Fi provisioning without its vendor app. Its documentation also distinguishes cloud-paired devices requiring credentials from devices that have not been connected to the cloud. This remains firmware-dependent. [Provisioning documentation](https://python-kasa.readthedocs.io/en/stable/cli.html#provisioning)

Password-free local access here still used the KLAP encrypted transport. It did not mean that the device accepted unencrypted arbitrary commands.

After enabling the router rule, reuse `local_bulb` above and replace the `inspect()` entry point with this provisioning step. It checks the expected bulb identity and prompts for the Wi-Fi key without echoing it:

```python
from getpass import getpass


def normalize_mac(value):
    return value.replace(":", "").replace("-", "").upper()


async def provision():
    expected = normalize_mac(input("Expected bulb MAC address: ").strip())
    if len(expected) != 12 or any(c not in "0123456789ABCDEF" for c in expected):
        raise ValueError("Enter the complete expected MAC address")

    async with local_bulb("192.168.0.1") as (bulb, info):
        actual = normalize_mac(info.get("mac") or info.get("mic_mac") or "")
        if actual != expected or info.get("model") != "KL110(US)":
            raise RuntimeError("Wrong bulb; Wi-Fi settings were not changed")

        ssid = input("Home 2.4 GHz Wi-Fi name: ").strip()
        key = getpass("Wi-Fi password: ")
        if not ssid or not key:
            raise ValueError("Network name and Wi-Fi key are required")
        try:
            await bulb.set_alias("Reading")
            await asyncio.wait_for(bulb.wifi_join(ssid, key, keytype="3"), timeout=12)
            print("Wi-Fi join acknowledged")
        except asyncio.TimeoutError:
            print("Check the home LAN: joining may have interrupted the reply")
        finally:
            key = None


asyncio.run(provision())
```

These snippets demonstrate the device protocol; they do not configure the router or switch the PC's Wi-Fi automatically. Restore the PC's home connection after provisioning. The session's own Windows helper did that in a `finally` block.

### 6. Register and verify in Afterglow

Dashboard source and setup: [Afterglow on GitHub](https://github.com/zschmook/afterglow).

The discovered identity was merged into Afterglow's existing saved lights by stable device ID. Other saved devices were preserved.

Verified results:

- Local identity: `Reading`, `KL110(US)`.
- `is_factory`: `false`.
- `dev_state`: `normal`.
- Afterglow's device list contained Reading.
- Its live state endpoint returned HTTP 200, power on, brightness 100.
- Router readback still showed the reserved address and enabled internet rule.
- The PC had rejoined its original home Wi-Fi.
- No additional Afterglow restart was needed for this registration.

The owner reported physical blinking during setup. Both devices subsequently reported normal state. Reading's ordinary controls were confirmed working afterward; a long-running stability check and an owner-confirmed Living Room control change remain unverified. Some checks temporarily failed, so the initial successful setup must not be mistaken for a completed reliability test.

### Replication on Living Room

The same process succeeded on the second KL110(US), also hardware 4.6 with firmware 1.1.1 Build 260710 Rel.082646. Its setup network was matched to its own MAC, a separate address reservation and internet-block rule were added, and its alias was restored to `Living Room` before directly provisioning Wi-Fi.

The first local connection attempt failed. A subsequent attempt explicitly queried setup discovery on UDP 20002, validated the response identity, and then successfully connected using KLAP v2 without owner credentials. This records the successful sequence; it does not establish why the first attempt failed.

After provisioning, the bulb reported `dev_state: normal`, `is_factory: false`, power on, and brightness 100 through Afterglow. Both internet rules remained enabled. The obsolete discarded-bulb entry was removed from Afterglow with a local recovery copy retained, leaving only the replacement Living Room card.

## Operational lessons

- Diagnose the protocol before adding an account-login workflow.
- Preserve distinct error categories: authentication rejection, connection failure, timeout, and a successful connection with an unexpected response.
- Save authentication material only when appropriate, keep it private, and never return it through device-list APIs. The account-based implementation retained derived keys rather than passwords; the successful Reading setup did not require an owner account key.
- Treat mocked handshake tests, live identity/state queries, and live control changes as different levels of evidence.
- A PC with one Wi-Fi adapter loses its home-network connection when joining a bulb's setup AP. This can interrupt streams served by the PC even if no process is restarted. Check playback, obtain permission, and prepare automatic reconnection first.
- Obtain fresh approval before any interrupting service restart. Earlier approval does not carry forward to a new deployment.
- Avoid flashing custom firmware or sending destructive reset commands as an unexplained experiment. This session used a physical factory reset authorized by the owner and direct local provisioning.

## Remaining work

- Check both bulbs remain usable over time with their configured internet restrictions.
- Confirm an ordinary control change on Living Room with the owner.
- Improve the sign-in card's appearance and behavior separately if retaining that optional path.
- Resolve the broader controller-location question in a separate change.

## Publication scope

This repository contains the investigation and procedure, not a portable installer or a complete Afterglow release. The setup helpers used in the session remained in the local Afterglow checkout; some contained environment-specific device checks and are not presented as reusable public tools.
