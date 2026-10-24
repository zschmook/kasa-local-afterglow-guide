# Kasa bulbs in Afterglow without a TP-Link account

An investigation and working local setup for one Kasa KL110 bulb, recorded on October 24, 2026.

**Status:** Reading was factory-reset, provisioned directly onto home Wi-Fi without a TP-Link account, isolated from the internet, and registered in Afterglow. Its local identity and state queries succeeded. The second new bulb, Living Room, has not undergone this process yet.

This is an observed result on one device and firmware version, not a promise that every Kasa bulb behaves the same way.

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

The investigation also established that the inspected Afterglow server had its own home-device implementation and data file. It did not need an M3U Picker restart to register Reading. The separate issue of where the wider home-lighting controller belongs was deferred.

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

### 3. Check local authentication before changing Wi-Fi

The helper tried a bounded set of local transports and read `system.get_sysinfo`. Early attempts timed out. A subsequent setup-network discovery and KLAP v2 query succeeded without owner account credentials.

Discovery changed from `ANS: true` while paired on the home LAN to `ANS: false` after reset in setup mode. These are recorded observations, not a claim that changing that field itself bypasses authentication.

The successful query supplied Reading's identity, firmware, and hardware version. A Wi-Fi scan through the bulb then found the home's 2.4 GHz network.

Some Kasa system-information responses use `mic_mac` instead of `mac`; identity validation must handle both.

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

### 6. Register and verify in Afterglow

The discovered identity was merged into Afterglow's existing saved lights by stable device ID. Other saved devices were preserved.

Verified results:

- Local identity: `Reading`, `KL110(US)`.
- `is_factory`: `false`.
- `dev_state`: `normal`.
- Afterglow's device list contained Reading.
- Its live state endpoint returned HTTP 200, power on, brightness 100.
- Router readback still showed the reserved address and enabled internet rule.
- The PC had rejoined its original home Wi-Fi.
- No additional Afterglow or M3U service restart was needed for this registration.

The owner reported physical blinking during setup. The device subsequently reported normal state, but the owner had not yet confirmed whether the visible blinking had stopped. A real brightness/power change after provisioning, a long-running stability check, and the second-bulb setup also remain unverified.

## Operational lessons

- Diagnose the protocol before adding an account-login workflow.
- Preserve distinct error categories: authentication rejection, connection failure, timeout, and a successful connection with an unexpected response.
- Save authentication material only when appropriate, keep it private, and never return it through device-list APIs. The account-based implementation retained derived keys rather than passwords; the successful Reading setup did not require an owner account key.
- Treat mocked handshake tests, live identity/state queries, and live control changes as different levels of evidence.
- A PC with one Wi-Fi adapter loses its home-network connection when joining a bulb's setup AP. This can interrupt streams served by the PC even if no process is restarted. Check playback, obtain permission, and prepare automatic reconnection first.
- Obtain fresh approval before any interrupting service restart. Earlier approval does not carry forward to a new deployment.
- Avoid flashing custom firmware or sending destructive reset commands as an unexplained experiment. This session used a physical factory reset authorized by the owner and direct local provisioning.

## Remaining work

- Confirm Reading's visible blinking has stopped and test an ordinary control change.
- Check Reading remains usable over time with the configured internet restriction.
- Repeat the verified setup with the new Living Room bulb, after the owner resets that specific bulb.
- Remove the stale discarded-bulb entry if requested.
- Improve the sign-in card's appearance and behavior separately if retaining that optional path.
- Resolve the broader controller-location question in a separate change.

## Publication scope

This repository contains the investigation and procedure, not a portable installer or a complete Afterglow release. The setup helpers used in the session remained in the local Afterglow checkout; some contained environment-specific device checks and are not presented as reusable public tools.
