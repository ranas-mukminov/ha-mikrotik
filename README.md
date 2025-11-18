# ha-mikrotik

High availability code for MikroTik routers.

This repository is a fork of [svlsResearch/ha-mikrotik](https://github.com/svlsResearch/ha-mikrotik), maintained by [Ranas Mukminov](https://github.com/ranas-mukminov).

## Status

**Status date:** December 10th 2019 (upstream baseline)

**Tested and stable on:**
- RouterOS **6.44.6** (CCR1009-8g-1s-1s+)
- Tested across 6 different pairs of CCR1009s for over a year
- Multiple administrative failover events and hardware failovers confirmed working

**Compatibility warning:**
- RouterOS versions newer than 6.44.6 are **untested and not recommended** until extensively validated
- MikroTik has broken features that ha-mikrotik relies on in various RouterOS releases
- Do not upgrade RouterOS expecting it to work without thorough lab testing first
- Ensure you use compatible hardware, RouterOS version, and ha-mikrotik release versions

## ⚠️ Critical Safety Warning

**DO NOT test this on production routers.**

This code is **dangerous by design** and can:
- **Completely wipe all configuration and files** from your devices if misconfigured
- **Cause network outages** if improperly deployed
- **Result in race conditions** during failover that may disrupt traffic

**Required before testing:**
- ✅ Lab environment with **out-of-band serial console access** to both routers
- ✅ Valid backups of all configurations stored externally
- ✅ Understanding of VRRP, MAC address cloning, and RouterOS scripting
- ✅ Ability to recover from complete configuration loss

**This is a proof of concept.** You will need to read and understand the code before deployment.

## Known Issues

**Race condition on standby startup:**
The #1 issue is a race condition during startup of the standby router after receiving updated configuration. The standby must quickly disable all interfaces to avoid taking traffic (MACs are cloned) from the active router. 

If you use spanning tree on your switches, this will likely happen fast enough. Without spanning tree or with fast port forwarding, Layer2/3 may come up and cause strange routing/bridging issues. **Test this very carefully in your lab environment.**

## Concept

Using a dedicated interface, VRRP, scripts, and backups, we can make a pair of MikroTik routers highly available:

- **Dedicated sync interface** (default: `ether8`) connects both routers directly
- **VRRP** provides heartbeat and automatic failover detection
- **Configuration and files** are actively synchronized from active to standby router
- **Standby router** remains ready to take over when VRRP heartbeat fails

### Core Functions

The following global functions are available after importing `HA_init.rsc`:

- `$HAInstall` - Prepare both routers for HA, configure the dedicated link
- `$HASyncStandby` - Synchronize configuration and files to standby router
- `$HAPushStandby` - Push updated HA code to standby router
- `$HASwitchRole` - Swap active/standby roles in a controlled manner

## Hardware Originally Developed For

- **Pair of CCR1009-8g-1s-1s+**
- RouterOS v6.33.5 (upgraded and tested on v6.44.6)
- RouterBOARD firmware 3.27
- Both routers bootstrapped from completely erased state, then HA installed

## Video Tutorial

[MikroTik - REAL HA Configuration](https://www.youtube.com/watch?v=GEef9P8wwxs) by The Network Berg provides a walkthrough of ha-mikrotik setup and operation.

## Installing

**Prerequisites:**
1. Source a **pair of matching routers**, ideally CCR1009-8g-1s-1s+
2. Install RouterOS **v6.44.6** and ensure RouterBOARD firmware is up to date
3. Ensure you have **serial console connections** to both devices
4. Prepare a random password or SHA-1 hex hash for HA authentication

**Installation steps:**

1. **Reset both routers** (⚠️ destroys all configuration):
   ```
   /file remove [find];
   /system reset-configuration keep-users=no no-defaults=yes skip-backup=yes
   ```

2. **Connect the HA sync link:**
   - Connect an Ethernet cable between `ether8` on both routers
   - This is the default HA sync interface

3. **Configure basic network on router A:**
   - On router A only, configure a minimal IP address on `ether1` so you can copy files to it

4. **Upload and import HA_init.rsc on router A:**
   ```
   /import HA_init.rsc
   ```

5. **Install HA on router A** (replace MAC addresses and password):
   ```
   $HAInstall interface="ether8" macA="[MAC_OF_A_ETHER8]" macB="[MAC_OF_B_ETHER8]" password="[RANDOM_PASSWORD_OR_SHA1_HEX]"
   ```
   
   **Note:** Use the `orig-mac-address` of each router's `ether8` interface, not the current MAC.

6. **Bootstrap router B:**
   - Follow the on-screen instructions from `$HAInstall` to bootstrap the secondary router
   - MAC telnet (as described at the top of HA_init.rsc) is the recommended method
   - Serial console can also be used

7. **Wait for router B to bootstrap and reboot:**
   - Router B will reboot into a basic networking mode after initial setup

8. **Push configuration from A to B:**
   ```
   $HASyncStandby
   ```
   
   **Note:** This will reboot router B with the synchronized configuration.

9. **Verify HA operation:**
   - Router B should return in standby mode
   - Router A should remain active
   - Test failover by rebooting router A - router B should take over

## Upgrading ha-mikrotik

To upgrade to a newer ha-mikrotik release:

1. **Download the new release** of ha-mikrotik

2. **Upload HA_init.rsc to the active router and import:**
   ```
   /import HA_init.rsc
   ```

3. **Push the new code to standby** (⚠️ reboots standby):
   ```
   $HAPushStandby
   ```

4. **Wait for standby to return and verify:**
   - Login to standby after reboot
   - Check logs: `/log print`

5. **Synchronize configuration to standby:**
   ```
   $HASyncStandby
   ```
   
   Note: There should be no changes unless something else changed on the active between steps.

6. **Switch roles** (⚠️ reboots active router):
   ```
   $HASwitchRole
   ```

7. **Verify upgrade:**
   - After both routers reboot, the previous standby is now active
   - Both routers should be running the upgraded HA version

## Rebuilding a Hardware-Failed Standby

If the standby hardware fails and needs replacement:

**Prerequisites:**
- Install a compatible version of RouterOS on the replacement hardware
- Factory reset the configuration
- Connect the new hardware exactly as the old standby was connected (same ether8 link)

**If router A is active, run from A:**

1. ```
   $HAInstall interface=$haInterface macA=$haMacMe macB="[NEW_MAC_OF_B_ETHER8]" password=$haPassword
   ```

2. Follow on-screen instructions just like the original installation

3. Once the standby returns:
   ```
   $HASyncStandby
   ```

**If router B is active, run from B:**

1. ```
   $HAInstall interface=$haInterface macB=$haMacMe macA="[NEW_MAC_OF_A_ETHER8]" password=$haPassword
   ```

2. Follow on-screen instructions just like the original installation

3. Once the standby returns:
   ```
   $HASyncStandby
   ```

## RouterOS 7.x Compatibility Status

⚠️ **RouterOS 7.x is NOT officially supported by this repository.**

- This code was developed and tested on RouterOS 6.x (specifically 6.44.6)
- RouterOS 7.x has significant changes to scripting, backup/restore, and networking behavior
- Some users report issues with `$HASyncStandby` and other operations on RouterOS 7.1.x+
- The `ha_switchrole.script` includes experimental RouterOS 7.x detection code, but it is **untested**

**If you wish to experiment with RouterOS 7.x:**
- Use **isolated lab devices only**
- Expect scripts to fail or behave unexpectedly
- Maintain external backups
- Document your findings and report issues with:
  - Exact RouterOS version
  - Hardware model
  - Complete logs and error outputs

**Do NOT assume production readiness for RouterOS 7.x.**

## Notes and Best Practices

- Always maintain **current backups external to the HA mechanism**
- Monitor logs on both routers after any HA operation:
  - `/log print where topics~"error"`
  - `/log print where topics~"warning"`
- Key operations that require monitoring:
  - `$HASyncStandby` - Configuration synchronization
  - `$HAPushStandby` - HA code updates
  - `$HASwitchRole` - Role switching
- If you see repeated startup failures, check for:
  - Hardware initialization timing issues
  - Interface naming mismatches
  - Spanning tree / port forwarding delays on upstream switches

## Troubleshooting

**Common issues:**

1. **Race condition on standby startup:** Standby may briefly forward traffic during boot. Enable spanning tree on upstream switches or adjust startup timing.

2. **Hardware initialization delays:** CCR devices may have slow interface initialization. Scripts include retry logic with delays.

3. **"Unable to find interface" errors:** Interface may be in transient state during boot. Scripts retry up to 120 times with 1-second delays.

4. **MAC address mismatch:** If routers report "UNKNOWN - WRONG MAC", there may be interface position swapping on boot. Router will automatically reboot to recover.

## License

This fork maintains the same license as the original [svlsResearch/ha-mikrotik](https://github.com/svlsResearch/ha-mikrotik) project.
