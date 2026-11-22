# 01: "Device responded with incorrect information"

> [!WARNING]
> **LLM Disclosure**
>
> This post was authored by claude-opus-5-5 on behalf of @JPHutchins. It was prompted by a calibre ↔ KOReader wireless connection error. The request was to diagnose the error and record the findings durably, citing pinned upstream sources.

**Status (2026-11-21):** the mechanism is identified, but the reason it keeps happening is not confirmed. See [§5](#5-how-to-confirm).

## Summary

- calibre's message means that KOReader **closed the TCP connection** while calibre was waiting for the reply to `GET_DEVICE_INFORMATION`. calibre didn't receive bad data; it received zero bytes.
- KOReader's `connect()` gives calibre **1 second** between the first and second handshake messages. If that window expires, KOReader disconnects with "handshake timeout".
- calibre sends the second message only after its Qt GUI thread has run the whole `device_detected` function. That function includes every slot connected to the public `device_connection_changed` signal, which calibre GUI plugins can hook.
- calibre's own work on that path doesn't obviously scale with library size. Since the failure here **repeats every time**, the leading candidates are a calibre plugin, a consistently slow Wi-Fi round trip, or a KOReader crash.

## Environment and observations

- calibre 9.6.0 on Windows (the traceback shows Windows paths). Evidence: [`../evidence/2026-11-21-calibre-traceback.txt`](../evidence/2026-11-21-calibre-traceback.txt).
- Every line number in the traceback matches calibre tag `v9.6.0` (commit [`d022762`][cal]) exactly. The table in §1 maps each frame.
- KOReader version: not recorded. The reader supports KOReader's SSH plugin.
- The connection had worked minutes before it started failing.
- Restarting KOReader did **not** fix it; the failure repeats the same way.

## 1. What calibre's error means

| Traceback frame | Source at v9.6.0 |
|---|---|
| `gui2/device.py:110 run` | [`DeviceJob` runs the queued function][cal-device-110] |
| `gui2/device.py:541 _get_device_information` | [calls the driver's `get_device_information`][cal-device-541] |
| `driver.py:1233 get_device_information` | [`self._call_client('GET_DEVICE_INFORMATION', {})`][cal-driver-1233] |
| `driver.py:648 _call_client` | [sends the request, then `return self._receive_from_client(...)`][cal-driver-648] |
| `driver.py:690 _receive_from_client` | [`raise ControlError(desc='Device responded with incorrect information')`][cal-driver-690] |

`_receive_from_client` reaches line 690 only when `_read_string_from_net` returns `b''`. That function returns `b''` only when `recv` returns zero bytes ([L588][cal-driver-588], [L596][cal-driver-596]), and its comment says so: "documentation says the socket is broken permanently". calibre then logs `protocol error -- empty json string` ([L676][cal-driver-676]) and falls through to line 690.

Other failures produce different messages. A timeout raises `TimeoutError`, and an `OSError` such as a connection reset raises "Device closed the network connection" ([L684][cal-driver-684]).

**Conclusion:** KOReader closed the connection cleanly (TCP FIN) around the time calibre sent `GET_DEVICE_INFORMATION`, without replying.

## 2. Why KOReader closes the connection there

KOReader `plugins/calibre.koplugin/wireless.lua`, [L326–L335][ko-330]:

```lua
        if pcall(self.initCalibreMQ, self, host, port) then
            CalibreMetadata:init(inbox_dir)
            -- And wait for initial requests: GET_INITIALIZATION_INFO, followed by
            -- GET_DEVICE_INFORMATION (or a specific DISPLAY_MESSAGE on password error).
            for _, timeout in ipairs{5, 1} do
                ok = resume_in(timeout)
                if not ok then
                    break
                end
            end
```

- Every message KOReader receives resumes the waiting coroutine ([L479][ko-479]). `resume_in` returns `nil` when its timeout expires ([L311][ko-311]).
- When the timeout expires, KOReader sets the error to `"handshake timeout"` ([L350][ko-350]) and calls `self:disconnect(...)` ([L361][ko-361]), which closes the socket. The reader then shows "Cannot connect to calibre server at … (handshake timeout)".
- So KOReader waits **5 s for `GET_INITIALIZATION_INFO`** and then only **1 s for `GET_DEVICE_INFORMATION`**.
- This behaviour came from KOReader PR [#13704 "calibre: rework connection setup"][pr-13704] (merged 2025-05-05, commit `8fef5c1766`).
- `connect()` is identical in the latest stable release `v2026.07.1` ([L299][kos-299]) and in master at [`1294bb3`][ko]. Updating KOReader does not change it.

## 3. What calibre does inside that 1 s

1. **Finish `open()`.** calibre receives KOReader's `OK` reply to `GET_INITIALIZATION_INFO` inside `open()` ([L1053][cal-driver-1053]) and runs the rest of it. That includes writing its `format_map` config ([L1156][cal-driver-1156]) and resetting the per-IP connection-attempt counter ([L1201][cal-driver-1201]).
2. **Signal the GUI.** On the device thread, `do_connect` calls `after_device_connect` ([L264][cal-device-264], [L268][cal-device-268]), which calls `self.connected_slot(True, device_kind)` ([L287][cal-device-287]).
3. **Block on the GUI thread.** `connected_slot` is `FunctionDispatcher(self.device_detected)` ([L966][cal-device-966]).
   - Called from a non-GUI thread, `FunctionDispatcher.__call__` emits a queued Qt signal and blocks on `self.q.get()` ([`gui2/__init__.py` L910–L916][cal-gui-910]).
   - `dispatch` calls `q.put` only after the function **returns** ([L918–L923][cal-gui-918]).
   - So the device thread waits for the whole of `device_detected`.
4. **Run `device_detected` on the GUI thread** ([L1163–L1205][cal-device-1163]), whenever its event loop gets to it:
   - It queues the device-information job ([L1176][cal-device-1176], via `create_job_step` [L507–L516][cal-device-507]).
   - It sets the default thumbnail, shows a status-bar message and makes the "On device" column visible ([`views.py` L852–L862][cal-views-852]).
   - It calls `refresh_ondevice(reset_only=True)`, which only resets a cache inside `preserve_state()` ([L1262][cal-device-1262]).
   - Finally it emits **`device_signals.device_connection_changed`** ([L1205][cal-device-1205]). This is a public signal "emitted once when the device is detected" ([L945–L950][cal-device-945]), meant for plugins. Slots on the GUI thread run synchronously inside this emit.
5. **Send it.** The device thread unblocks, takes the job from its queue ([`next_job` L412][cal-device-412], loop [L481–L498][cal-device-481]), and sends `GET_DEVICE_INFORMATION` ([L1233][cal-driver-1233]).

The 1 s budget therefore has to cover:

- the Wi-Fi round trip (KOReader's reply to calibre, then calibre's request back),
- the rest of `open()`,
- the wait until the GUI thread picks up the queued signal,
- all of `device_detected`, **including every plugin slot connected to `device_connection_changed`**.

**Assessment:**

- None of the built-in work in step 4 obviously scales with library size. A momentarily busy GUI thread explains an *occasional* miss.
- A miss that *repeats every time* needs a delay that's always there. The candidates are in §4.

**Ruled out:**

- **The device thread's 2 s poll sleep** ([L498][cal-device-481]). Step 3 blocks until the job is queued, so the device thread finds it on its next `next_job()` call without sleeping. An earlier hypothesis in this investigation got that wrong.
- **An "allow this device?" dialog.** `ASK_TO_ALLOW_CONNECT` defaults to `False` ([`interface.py` L142][cal-iface-142]), and the wireless driver doesn't override it.

## 4. Candidates for a failure that keeps repeating

- **A calibre GUI plugin connected to `device_connection_changed`.** Its slot runs inside the 1 s window on every connection (§3 step 4). Installing or updating a plugin would also explain "it worked 5 minutes ago".
- **Consistently slow e-reader Wi-Fi**, for example power saving, which adds the same delay to every round trip. A reporter suggested this in [#16261][issue-16261]; it hasn't been verified.
- **KOReader crashing while connecting.** That produces the same calibre error. A KOReader maintainer calls the calibre-side error "the usual red herring" in [#13859][issue-13859].
- **Not a candidate: calibre's "Enable the no-activity timeout" setting.** In [#16261][issue-16261], one user reports that turning it off fixed connecting. In the source, that option only disconnects after 30 minutes of inactivity ([L235][cal-driver-235], [L975][cal-driver-975]). I found no way it could affect the handshake.

The tracebacks in related reports fail at a *different stage*, during `open()` at `GET_INITIALIZATION_INFO` ([#15169][issue-15169]) and in the log posted in [#16261][issue-16261]. Here the failure is at `GET_DEVICE_INFORMATION`. The error message is the same; the stack is not.

## 5. How to confirm

1. **Read KOReader's on-screen message** after a failed attempt. "(handshake timeout)" confirms §2.
2. **Start calibre with custom plugins disabled** (`calibre --ignore-plugins`, [`gui2/main.py` L59–L61][cal-main-59]) and retry. If it now connects, a plugin is the cause.
3. **Time the handshake in calibre's debug log.**
   - Turn on "Print extra debug information" in the SmartDevice App Interface plugin settings.
   - Start calibre in debug mode (`calibre-debug -g`).
   - Compare the timestamps of `_call_client GET_INITIALIZATION_INFO` and `_call_client GET_DEVICE_INFORMATION`. A gap near or above 1 s confirms §3.
4. **Check KOReader's `crash.log`** for a crash at the same moment.

## 6. Mitigations

- **Restart calibre's wireless connection, or calibre itself.** KOReader has already been restarted without effect; calibre hasn't.
- **Reboot the reader** (a full device reboot, not just a KOReader restart). That resolved a related failure at the earlier `open()` stage in [#15169][issue-15169]. It hasn't been tried here.
- **Retry while calibre is idle:** no jobs running and no dialogs open.
- **Be careful with repeated retries.** calibre counts connection attempts per IP. After 5 attempts in a row that don't complete `open()`, it ignores that IP until its wireless server is restarted ([L232][cal-driver-232], [L998][cal-driver-998]; also described in [#13704][pr-13704]). The failure here happens after `open()` completes and the counter resets ([L1201][cal-driver-1201]). Failures earlier in the handshake do count.
- **Not effective, according to the source:** updating KOReader, since `connect()` is identical (§2), and the no-activity timeout (§4).

## 7. Implications for a replacement server

A server that sends `GET_DEVICE_INFORMATION` as soon as it receives the initialization reply never hits this race. There is no GUI thread or plugin hook in between. See [03-wire-protocol.md](03-wire-protocol.md).

[cal]: https://github.com/kovidgoyal/calibre/tree/d022762b4635598ce76cae248b68a27b091d3805
[ko]: https://github.com/koreader/koreader/tree/1294bb3d377c51fdd0b7ef250620aee2ca08927c
[cal-device-110]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/gui2/device.py#L110
[cal-device-264]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/gui2/device.py#L264
[cal-device-268]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/gui2/device.py#L268
[cal-device-287]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/gui2/device.py#L287
[cal-device-412]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/gui2/device.py#L412
[cal-device-481]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/gui2/device.py#L481-L498
[cal-device-507]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/gui2/device.py#L507-L516
[cal-device-541]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/gui2/device.py#L541
[cal-device-945]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/gui2/device.py#L945-L950
[cal-device-966]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/gui2/device.py#L966
[cal-device-1163]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/gui2/device.py#L1163-L1205
[cal-device-1176]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/gui2/device.py#L1176
[cal-device-1205]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/gui2/device.py#L1205
[cal-device-1262]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/gui2/device.py#L1262
[cal-views-852]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/gui2/library/views.py#L852-L862
[cal-gui-910]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/gui2/__init__.py#L910-L916
[cal-gui-918]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/gui2/__init__.py#L918-L923
[cal-iface-142]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/devices/interface.py#L142
[cal-main-59]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/gui2/main.py#L59-L61
[cal-driver-232]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/devices/smart_device_app/driver.py#L232
[cal-driver-235]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/devices/smart_device_app/driver.py#L235
[cal-driver-588]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/devices/smart_device_app/driver.py#L588
[cal-driver-596]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/devices/smart_device_app/driver.py#L596
[cal-driver-648]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/devices/smart_device_app/driver.py#L648
[cal-driver-676]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/devices/smart_device_app/driver.py#L676
[cal-driver-684]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/devices/smart_device_app/driver.py#L684
[cal-driver-690]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/devices/smart_device_app/driver.py#L690
[cal-driver-975]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/devices/smart_device_app/driver.py#L975
[cal-driver-998]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/devices/smart_device_app/driver.py#L998
[cal-driver-1053]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/devices/smart_device_app/driver.py#L1053
[cal-driver-1156]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/devices/smart_device_app/driver.py#L1156
[cal-driver-1201]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/devices/smart_device_app/driver.py#L1201
[cal-driver-1233]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/devices/smart_device_app/driver.py#L1233
[ko-311]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/plugins/calibre.koplugin/wireless.lua#L311
[ko-330]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/plugins/calibre.koplugin/wireless.lua#L326-L335
[ko-350]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/plugins/calibre.koplugin/wireless.lua#L350
[ko-361]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/plugins/calibre.koplugin/wireless.lua#L361
[ko-479]: https://github.com/koreader/koreader/blob/1294bb3d377c51fdd0b7ef250620aee2ca08927c/plugins/calibre.koplugin/wireless.lua#L479
[kos-299]: https://github.com/koreader/koreader/blob/9192014d8bd82a91dc1012473be0f238dedfdb54/plugins/calibre.koplugin/wireless.lua#L299
[pr-13704]: https://github.com/koreader/koreader/pull/13704
[issue-13859]: https://github.com/koreader/koreader/issues/13859
[issue-15169]: https://github.com/koreader/koreader/issues/15169
[issue-16261]: https://github.com/koreader/koreader/issues/16261
