# OPNsense HA Failover Script — Debugging Summary - Created by Claude

Context: single public WAN IPv4 (Vodafone cable), two OPNsense nodes in
CARP HA. Since there's only one real WAN IP, WAN itself can't use CARP
like the LAN-side VLANs do — a script (forked from msz345/lavacanao)
reacts to CARP master/backup transitions and manually enables/disables
the WAN interface and gateways on whichever node is active, to avoid
both nodes presenting the same spoofed MAC to the modem at once.

This document summarizes every bug found and fixed while getting the
script working on OPNsense 26.7.2_2, and the reasoning behind each fix.

## 1. `write_config()` incompatible with `Config::fromArray()`

**Symptom:** The `enable` flag never made it into `config.xml` after
`handleMasterTransition()` ran, with no visible error.

**Cause:** The script combined two different OPNsense config APIs: the
modern `OPNsense\Core\Config` class (`toArray()`/`fromArray()`) to build
the new interface state, then the legacy procedural `write_config()`
function to persist it. In the current core, calling `fromArray()` and
then `write_config()` triggers `write_config()`'s own internal call to
`fromArray()` — passing a `Config` object where an array is expected,
raising an uncaught `TypeError`. Because the surrounding `catch` only
caught `\Exception`, this `\Error` slipped through silently.

**Fix:** Drop `write_config()` entirely. After `fromArray()`, call the
`Config` object's own `save(['description' => $description])` method,
which handles locking, backup, and the write itself.

## 2. Gateway state (`disabled` flag) lives in a separate MVC model

**Symptom:** Wanted to flip a named gateway's enabled/disabled state
from the script . `Config::toArray()`
returned `null` for the `Gateways` section entirely.

**Cause:** Gateways are managed through OPNsense's newer MVC model
system (`OPNsense\Routing\Gateways`), not the legacy raw-XML section
that `Config::toArray()`/`fromArray()` operates on.

**Fix:** Use the model class directly:
```php
$mdl = new \OPNsense\Routing\Gateways();
foreach ($mdl->gateway_item->iterateItems() as $node) {
    if ((string)$node->name === $gatewayName) {
        $node->disabled = $disabled ? '1' : '0';
    }
}
$mdl->serializeToConfig();   // stages in-memory only, no disk write
Config::getInstance()->save(...);   // actual write happens here
```

**Ordering gotcha:** `serializeToConfig()` doesn't write anything by
itself — a later `fromArray()` call elsewhere in the transition (built
from an earlier `toArray()` snapshot that doesn't know about the
Gateways section at all) can end up being followed by a `save()` that
effectively clobbers the staged gateway change, depending on ordering.
Safe pattern found empirically: do the gateway switch either (a) with
its own dedicated `lock()`/`save()`/`unlock()` sandwiched around a
`forceReload()` before the next `toArray()` snapshot is taken, or (b)
strictly after the interface's own `applyConfigurationWithRetry()` call
has completed and written to disk. Doing the gateway switch *before* a
stale in-memory `$configArray` gets `fromArray()`'d back onto the
config is the unsafe order.

## 3. `dhclient` supervisor process survives `SIGTERM` / interface stays administratively up

**Symptom:** the biggest and most stubborn one. After a backup
transition, `configdRun("interface all reconfigure")` would remove
`enable` from the interface config, but on the wire the interface kept
its `UP`/`RUNNING` flags and its real public IP — with an active
`dhclient` still running. Since both nodes spoof the same WAN MAC, this
meant two "live" clients from the modem's perspective: the peer
(supposed new master) then reliably failed to get a DHCP lease
(`invalid_dhcp_lease`, empty IP) because the old master hadn't actually
let go.

**Root cause, found in stages:**
1. `killbypid("/var/run/dhclient.{$if}.pid")` (the pattern already used
   for `dpinger`) didn't work — FreeBSD's `dhclient` here runs under a
   `daemon(8)` supervisor process. `killbypid()` sends `SIGTERM`,
   which the supervisor evidently tolerates/ignores and immediately
   respawns the child.
2. Even `SIGKILL`-ing the supervisor PID (confirmed working manually)
   left three orphaned child processes running independently (main,
   privileged helper, syslog helper) — the supervisor doesn't own them
   in a process group that dies with it.
3. Even with *all* dhclient-related processes fully dead, the interface
   itself stayed administratively `UP` with the stale IP still bound —
   killing the DHCP client only stops lease renewal, it does not bring
   the interface down. That's a separate, explicit step.

**Fix, three steps, all needed:**
```php
// 1. Kill the daemon(8) supervisor via its PID file, with SIGKILL
//    (SIGTERM via killbypid() does not reliably stop it).
$pid = (int)trim(file_get_contents("/var/run/dhclient.{$if}.pid"));
mwexecfm('/bin/kill -9 %s', [(string)$pid]);

// 2. Orphaned children don't die with the supervisor — sweep them too.
mwexecfm('/bin/pkill -9 -f %s', ['dhclient']);

// 3. The actual fix for the MAC conflict: bring the interface down
//    explicitly. configdRun("interface all reconfigure") does not
//    reliably do this on its own.
mwexecfm('/sbin/ifconfig %s down', [$realWanIf]);
```
Symmetrically, `handleMasterTransition()` needed an explicit
`ifconfig up` after applying the config — once the backup path
reliably brings the interface fully down, the master path can no
longer assume "the interface is already up" as it apparently could
before (when the interface was never really going down in the first
place).

**Debugging note:** `posix_kill()` was tried first as a cleaner
alternative to shelling out, but the `posix` PHP extension isn't loaded
on OPNsense by default — an uncaught `Error: Call to undefined function
posix_kill()`, again silently swallowed because the top-level catch
only handled `\Exception`. Switched to `mwexecfm('/bin/kill -9 %s', ...)`
instead, consistent with every other process-management call already in
the script.

## 4. General debugging lesson learned this project

Every "the script does nothing and there's no error" symptom in this
project turned out to be either (a) a shebang/encoding issue preventing
execution entirely, (b) an uncaught `\Error` (not `\Exception`) from a
missing function, both invisible to the script's own `try { } catch
(\Exception $e)` blocks. Tightening the top-level catch in
`createAndRun()` to `catch (\Throwable $e)` was added specifically so
future issues of this class get logged instead of vanishing.

## 5. `dpinger` not auto-starting for a freshly re-enabled gateway

**Symptom:** WAN dashboard showed the newly active gateway as red/dead
even though it worked, `ps aux | grep dpinger` showed nothing for it
after the transition.

**Fix:** `mwexecfm('/usr/local/sbin/pluginctl -c monitor')` after the
interface reconfigure step, restarts gateway monitoring. Logged as a
warning (not a hard failure) since it's cosmetic/dashboard-only — the
actual failover logic doesn't depend on `dpinger` running (see #5).

## Net result

15 consecutive real CARP master/backup transitions completed with no
manual intervention: WAN interface state, DHCP lease, gateway
enable/disable, and monitoring all flip correctly in both directions.
Remaining real-world impact during a transition: 1–3 dropped pings from
a LAN client; a concurrent video stream was unaffected.
