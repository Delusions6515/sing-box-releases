# patches

Patch sets for the [lurixo/sing-box-releases](https://github.com/lurixo/sing-box-releases) custom build. This is a fork that builds sing-box with additional features and fixes not yet available upstream.

All patches are applied automatically by the `build-all.yml` workflow during CI.

---

## sing-box/

Applied to [SagerNet/sing-box](https://github.com/SagerNet/sing-box) testing branch.

### tcp_keep_alive_count.patch

Adds `TCPKeepAliveCount` option to both dialer and listener, allowing configuration of the number of unacknowledged TCP keepalive probes before a connection is dropped.

### urltest_unified_delay.patch

From [reF1nd/sing-box](https://github.com/reF1nd/sing-box/tree/reF1nd-testing-next). Adds `urltest_unified_delay` experimental option. When enabled, URL tests send a second request after the initial one and measure latency from the second request, excluding connection setup and TLS handshake overhead from the result.

### reject-unknown-sni.patch

From [reF1nd/sing-box](https://github.com/reF1nd/sing-box/tree/reF1nd-testing-next). Adds `server_names` (list) field and `reject_unknown_sni` option to the inbound TLS config. When `reject_unknown_sni` is enabled, connections with an SNI not matching any configured server name are rejected. `server_name` and `server_names` cannot be used together.

### dns_group_transport.patch

From [reF1nd/sing-box](https://github.com/reF1nd/sing-box/tree/reF1nd-testing-next). Adds a `group` DNS transport type. Wraps multiple DNS servers under a single tag, queries all of them concurrently, and returns the fastest successful response. Groups cannot be nested and cannot contain fakeip servers.

### libbox_http_timeout.patch

Adds dial timeout and response header timeout to the libbox HTTP client, preventing hangs on unresponsive servers during update checks and other HTTP requests.

### anytls_interface_updated.patch

Implements `InterfaceUpdateListener` on the AnyTLS outbound. When the network interface changes, calls `client.Reset()` to close stale sessions. Depends on `session-resilience.patch`.

---

## sing-anytls/

Applied to [anytls/sing-anytls](https://github.com/anytls/sing-anytls). Apply in order listed.

> **⚠️ Breaking change:** The `record-shaper.patch` replaces the upstream padding system entirely. A patched server will reject connections from unpatched clients with `cmdAlert` (`client does not support record shaper, please upgrade`). Both the server and all clients must run the patched build.

### session-resilience.patch

Session pool resilience on connection failures and network changes.

If `OpenStream` fails on a reused idle session (e.g. the underlying connection was reset while pooled), catches the error and falls back to creating a fresh session instead of propagating the failure.

Adds `Reset()` to the client, which closes all sessions (idle and active) without cancelling the client context, so new sessions can still be created afterward. Called by `anytls_interface_updated.patch` on network changes.

Adds `IsClosed()` checks to `getIdleSession` and `idleCleanupExpTime`. Without these checks, a session whose TCP connection has died silently (NAT timeout, carrier drop) stays in the idle pool until TCP keepalive detects the failure. The next request picks it up, writes succeed (kernel-buffered), then blocks until the per-stream SYNACK timeout fires after 5 seconds. By then the caller's context deadline is nearly exhausted, leaving insufficient time for `createSession` to resolve the server domain and complete TCP/TLS handshakes. With `min_idle_session >= 1`, `idleCleanupExpTime` resets `idleSince` on the last remaining session every cycle regardless of state, so a dead session is never cleaned up by timeout alone. `getIdleSession` now loops past closed sessions instead of returning them, and `idleCleanupExpTime` removes closed sessions before evaluating timeout and `min_idle_session` logic.

### sing-anytls-write-deadline.patch

Moves `SetWriteDeadline` inside `writeConn` under `connLock` protection, fixing a race where concurrent data and control frame writes interfere with each other's deadlines. Also adds `s.Close()` to the `writeDataFrame` error path.

### unknown-cmd-skip-data.patch

The `default` branch in `recvLoop` does not read `hdr.Length()` bytes for unknown commands. If a newer peer or an active prober sends a frame with an unknown command carrying data, subsequent frame parsing desynchronizes.

Reads and discards `hdr.Length()` bytes for unknown commands.

### uint16-overflow-protect.patch

`writeDataFrame` and `writeControlFrame` cast data length to `uint16` without bounds checking. Payloads exceeding 65535 bytes silently truncate, desynchronizing frame parsing on the receiver.

`writeDataFrame` now chunks oversized payloads into multiple PSH frames. `writeControlFrame` returns an error if data exceeds the limit.

### synack-per-stream-timeout.patch

The session-level SYNACK timer fires a single 3-second deadline that kills the entire session on timeout, taking all concurrent streams down together.

Replaces with per-stream timers (5 seconds). On timeout only the individual stream is closed. If no SYNACK has been received on the session since the stream was opened, the session is also closed.

### record-shaper.patch

Replaces the upstream frame-level padding system with TLS record-level traffic shaping. A `RecordShaper` sits between the session frame layer and the TLS connection, handling two things:

**Control frame padding.** Bare control frames (cmdSYN, cmdFIN, cmdHeartbeat, cmdSYNACK without data) are exactly 7 bytes — a size that never appears in real HTTP/2 traffic. The shaper pads frames ≤ 21 bytes by appending a `cmdWaste` trailer to reach a target size sampled from the `pad_targets` distribution. The default targets are `17, 30-50, 30-50, 80-150`, producing variable-size output that overlaps with h2 PING, small SETTINGS, and HEADERS frames. `pad_targets` is independent of `idle_sizes`, so control-frame padding can cover a wider range without affecting idle injection.

**Idle keepalive injection.** A background goroutine writes small waste TLS records (9/13/17 bytes, matching h2 SETTINGS ACK, WINDOW_UPDATE, and PING sizes) during idle periods, preventing traffic-analysis detection of connection inactivity. The injection interval defaults to 3–8 seconds.

Data frames pass through to `conn.Write` without reshaping. Go's `crypto/tls` implements dynamic record sizing natively — small records (~1208 bytes) for the first 128 KB, then 16 KB records for bulk transfer, shrinking back after 1 second idle. This produces a TLS record size distribution identical to any real Go HTTPS server or client without additional overhead.

Auth packet padding uses weighted bucket sampling (`SamplePaddingLen`) with `crypto/rand` selection and `math/rand/v2` ChaCha8 PRNG fill, replacing the dead `0=30-30` upstream scheme entry and zero-fill.

The patched client advertises `rs=1` in `cmdSettings`. If the server does not see this flag, it sends `cmdAlert` and closes the session. Server-side buffering of `cmdUpdatePaddingScheme` + `cmdServerSettings` is conditional on actually needing to send both, so they merge into a single `writeConn` call.

All shaper parameters are configurable via the PaddingScheme and propagate to live sessions through the existing `cmdUpdatePaddingScheme` mechanism. The shaper lazily reloads its configuration whenever the scheme MD5 changes, so parameter updates take effect without reconnecting.

| Key | Format | Default | Description |
|-----|--------|---------|-------------|
| `pad_targets` | `17,30-50,80-150` | `17,30-50,30-50,80-150` | Target sizes for control frame padding (fixed or range) |
| `idle_sizes` | `9,13,17` | `9,9,13,13,13,17` | Record sizes for idle injection (weighted by repetition) |
| `idle_interval` | `3000-8000` | `3000-8000` | Injection interval range in ms |
| `idle_threshold` | `500` | `500` | Silence before idle detection in ms |
| `pad_dist` | `30-60:20,100-300:35,...` | multi-modal | Weighted bucket distribution for auth packet padding |

#### padding_scheme configuration

The `padding_scheme` field is a string array in the sing-box **inbound** config. Each element is one line of the scheme. The server pushes the full scheme to clients via `cmdUpdatePaddingScheme` on first connect.

```json
{
  "type": "anytls",
  "padding_scheme": [
    "pad_dist=30-60:20,100-300:35,500-1200:30,1200-4000:15",
    "pad_targets=17,30-50,30-50,80-150",
    "idle_sizes=9,9,13,13,13,17",
    "idle_interval=3000-8000",
    "idle_threshold=500"
  ]
}
```

When `padding_scheme` is omitted from the inbound config, the built-in default is used. All keys are optional — only include the ones you want to override.

---

## android/

Applied to [SagerNet/sing-box-for-android](https://github.com/SagerNet/sing-box-for-android) dev branch.

### update_checker_versioncode.patch

Changes update detection to compare both `versionName` and `versionCode`, so builds with the same version name but different build codes are correctly identified as updates.

### fix_all_anr.patch

Attempts to fix freezes when returning to SFA, when SFA fails to stop, or when SFA fails to start, by removing `runBlocking` / `withContext(Dispatchers.IO)` calls on the main thread.

### fix_update_dialog_darkmode.patch

Fixes the update dialog in dark mode where rendering issues cause parts of the text to be unreadable.

### remove_fdroid_features.patch

Removes F-Droid update source logic and UI. This fork distributes exclusively through GitHub Releases.

### fix_ime_swipe_delete.patch

Fixes an issue where swiping up on certain IMEs cannot fully clear the editor content.

### fix_doze_wake_reset_network.patch

Calls `resetNetwork()` when the device exits Doze idle mode. Attempts to fix Telegram being stuck on the loading screen for an extended time during cold start. May slightly increase power consumption.

### screen_on_reset_network.patch

Registers an `ACTION_SCREEN_ON` receiver and calls `resetNetwork()` on screen wake. Attempts to fix Telegram being stuck on the loading screen for an extended time during cold start. May slightly increase power consumption.
