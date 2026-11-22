# 01: "Device responded with incorrect information"

> [!WARNING]
> **LLM Disclosure**
>
> This post was authored by claude-opus-5-5 on behalf of @JPHutchins. It was prompted by a calibre ↔ KOReader wireless connection error. The request was to diagnose the error and record the findings durably, citing pinned upstream sources.

**Status (2026-11-21):** the likely cause is identified but not confirmed on the device. See [§5](#5-how-to-confirm).

## Summary

- calibre's message means that KOReader **closed the TCP connection** while calibre was waiting for the reply to `GET_DEVICE_INFORMATION`. calibre didn't receive bad data; it received zero bytes.
- KOReader's `connect()` gives calibre **1 second** between the first and second handshake messages. If that window expires, KOReader disconnects with "handshake timeout".
- calibre sends its second message only after a round trip through its Qt GUI thread. A busy GUI thread, or slow Wi-Fi round trips, can push that past 1 s.
- One alternative is not ruled out: a KOReader crash produces the same calibre traceback.

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
3. **Block on the GUI thread.** `connected_slot` is `FunctionDispatcher(self.device_detected)` ([L966][cal-device-966]). Called from a non-GUI thread, `FunctionDispatcher.__call__` emits a queued Qt signal and blocks on `self.q.get()` until the GUI thread has run the function ([`gui2/__init__.py` L910–L915][cal-gui-910]).
4. **Queue the request.** The GUI thread runs `device_detected` ([L1163][cal-device-1163]) whenever its event loop gets to it. That queues the device-information job ([L1176][cal-device-1176], then `create_job_step` [L507–L516][cal-device-507]).
5. **Send it.** The device thread unblocks, takes the job from its queue ([`next_job` L412][cal-device-412], loop [L481–L498][cal-device-481]), and sends `GET_DEVICE_INFORMATION` ([L1233][cal-driver-1233]).

The 1 s budget therefore has to cover:

- the Wi-Fi round trip (KOReader's reply to calibre, then calibre's request back),
- the rest of `open()`,
- and **however long calibre's Qt GUI thread takes to handle a queued signal**.

If the GUI thread is busy for more than about a second, KOReader hangs up.

**Ruled out:** the device thread's 2 s poll sleep ([L498][cal-device-481]) is *not* on this path. Step 3 blocks until the job is queued, so the device thread finds it on its next `next_job()` call without sleeping. An earlier hypothesis in this investigation got that wrong.

## 4. Other explanations not ruled out

- **KOReader crashed while connecting.** That produces the same calibre traceback. A KOReader maintainer calls the calibre-side error "the usual red herring" in [#13859][issue-13859]. Check KOReader's `crash.log`.
- **Slow e-reader Wi-Fi** (for example power saving) using up the window. This is a reporter's hypothesis in [#16261][issue-16261]; it hasn't been verified.
- **calibre's "Enable the no-activity timeout" setting.** In [#16261][issue-16261], one user reports that turning it off fixed connecting. In the source, that option only disconnects after 30 minutes of inactivity ([L235][cal-driver-235], [L975][cal-driver-975]). I found no way it could affect the handshake.

## 5. How to confirm

1. **Read KOReader's on-screen message** after a failed attempt. "(handshake timeout)" confirms §2.
2. **Time the handshake in calibre's debug log.**
   - Turn on "Print extra debug information" in the SmartDevice App Interface plugin settings.
   - Start calibre in debug mode (`calibre-debug -g`).
   - Compare the timestamps of `_call_client GET_INITIALIZATION_INFO` and `_call_client GET_DEVICE_INFORMATION`. A gap near or above 1 s confirms §3.
3. **Check KOReader's `crash.log`** for a crash at the same moment, which would point to §4.

## 6. Mitigations

- **Retry while calibre is idle:** no jobs running and no dialogs open.
- **Restart calibre, not only KOReader.** Restarting the reader resolved the same traceback in [#15169][issue-15169]. It did not help here, which points to calibre or the network rather than the reader's state.
- **Be careful with repeated retries.** calibre counts connection attempts per IP. After 5 attempts in a row that don't complete `open()`, it ignores that IP until its wireless server is restarted ([L232][cal-driver-232], [L998][cal-driver-998]; also described in [#13704][pr-13704]). The failure here happens after `open()` completes and the counter resets ([L1201][cal-driver-1201]). Failures earlier in the handshake do count.
- **Not effective, according to the source:** updating KOReader, since `connect()` is identical (§2), and the no-activity timeout (§4).

## 7. Implications for a replacement server

A server that sends `GET_DEVICE_INFORMATION` as soon as it receives the initialization reply never hits this race. There is no cross-thread hop in between. See [03-wire-protocol.md](03-wire-protocol.md).

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
[cal-device-966]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/gui2/device.py#L966
[cal-device-1163]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/gui2/device.py#L1163
[cal-device-1176]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/gui2/device.py#L1176
[cal-gui-910]: https://github.com/kovidgoyal/calibre/blob/d022762b4635598ce76cae248b68a27b091d3805/src/calibre/gui2/__init__.py#L910-L915
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
