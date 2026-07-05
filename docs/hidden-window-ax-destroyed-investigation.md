# Why the hidden-window tiling bug doesn't reproduce on every machine

Investigation notes, 2026-07-05. Companion to [PR #2](https://github.com/mmv08/Amethyst/pull/2)
("Fix tiling after reopening apps that keep windows alive", commit `3e78920`) and
[ianyh/Amethyst#1335](https://github.com/ianyh/Amethyst/issues/1335).

## TL;DR

The bug's preconditions exist on **every** machine tested — including the one that
"can't reproduce". What differs is a single undocumented macOS behavior: whether the OS
delivers `kAXUIElementDestroyedNotification` when an app **hides** a window (the
Telegram/Slack/Claude close-button pattern, `orderOut` without destroying the window).

- **Machine that gets the notification** → Amethyst untracks the window the moment it
  hides (and reflows), so the revived window is adopted as a brand-new window and tiles
  normally. The bug cannot manifest. The PR's "known limitation" (no reflow on hide)
  doesn't exist here either.
- **Machine that doesn't get the notification** → the stale, invalidated element stays
  tracked under the same `(pid, CGWindowID)` identity, the revived window's live element
  is rejected as "already tracked", and the window stops tiling permanently. This is the
  bug PR #2 fixes.

Delivery of this notification is not documented by Apple and is known in the
window-manager community to vary per machine (macOS version and/or other accessibility
clients running). Rebooting does not change it.

## The immune machine

| Fact | Value |
| --- | --- |
| macOS | 26.5.1 (25F80), on 26.5.x since 2026-05-22 |
| Amethyst | Production 0.24.3 (129), Developer ID–signed by Ian Ynda-Hummel, **without** the PR #2 fix |
| Repro apps | Telegram 12.8 (native, non-MAS), Slack 4.50.143, Claude 1.18286.0 |
| Other AX-ish tools running | Raycast Beta only (does not interfere) |
| Amethyst config | Defaults, `new-windows-to-main: false`, empty float list |

## Experiment

A minimal AppKit probe app that mimics the hide-on-close apps exactly:

- `windowShouldClose` → `orderOut` (hide, keep the `NSWindow` alive), revive the same
  window later with `makeKeyAndOrderFront`.
- Self-reports its `windowNumber`/CGWindowID, its frame (so Amethyst's tiling of it is
  observable), and the validity of a **saved** AX element for its own window —
  self-inspection requires no accessibility permission.
- Registers an `AXObserver` on itself for `kAXUIElementDestroyedNotification` on the
  window element — the same per-window registration Amethyst uses
  (`ApplicationObservation.windowNotificationsForWindow`).
- Dumps all onscreen layer-0 windows via `CGWindowListCopyWindowInfo` so the whole
  layout's reaction is visible.

Sequence driven by signals: launch → hide → revive → open a second window (forces a
reflow: can Amethyst still *move* the revived window?) → close it.

### Results on the immune machine (macOS 26.5.1, production Amethyst 0.24.3)

All of the bug's preconditions reproduced:

```
post-launch:  A tiled by Amethyst at {{3200,0},{640,1590}} (column layout, cgID 5733)
hide:         saved AX element reads -> invalidUIElement   (element invalidated, as on all machines)
              CG window survives offscreen with the SAME id 5733
revive:       fresh AX fetch -> DIFFERENT element ref mapping to the SAME cgID 5733
```

…but the trap never springs, because of this, 7 ms after the hide:

```
[10:45:05.172] >>> SIGUSR1: hiding windowA (orderOut)
[10:45:05.179] !!! AX-NOTIFICATION: AXUIElementDestroyed
[10:45:07.285] onscreen: Chrome w=640->960, Claude w=640->960   <- Amethyst reflowed on hide
[10:45:09.186] !!! AX-NOTIFICATION: AXWindowCreated cgID=5733   <- revival seen as a NEW window
```

Discriminating test — after hide/revive, opening window B forced a reflow and Amethyst
**moved the revived window A** (x=3200 w=640 → x=2880 w=480, and back after B closed).
The revived window is fully managed; nothing is orphaned.

Conclusion: on this machine the destroyed notification arrives at hide time, Amethyst
untracks the window, and `isWindowTracked` is never true for the revived element. On the
affected machines (per the debugging that produced PR #2) the notification never arrives,
which is the only missing link in the chain.

## Why delivery differs between machines

Apple documents
[`kAXUIElementDestroyedNotification`](https://developer.apple.com/documentation/applicationservices/kaxuielementdestroyednotification)
with one line ("An accessibility object was disposed of") — nothing about hidden
windows. The macOS 26.3–26.5 release notes contain no accessibility changes. The
observable behavior is community-documented:

- [koekeishiya/yabai#2431](https://github.com/koekeishiya/yabai/issues/2431) — since
  macOS Sequoia, AX destroyed events can silently stop being delivered **machine-wide
  while certain other AX-client apps are running**; users pinned Contexts, Phoenix, and
  Amazon Q; quitting the offender restored delivery. yabai 7.1.4
  ([commit `6f9006d`](https://github.com/koekeishiya/yabai/commit/6f9006dd957100ec13096d187a8865e85a164a9b))
  stopped relying on the AX notification entirely and switched to private SkyLight APIs
  (`SLSRegisterConnectionNotifyProc` + `SLSRequestNotificationsForWindows`).
- [tmandry/glide#10](https://github.com/tmandry/glide/issues/10) — on 15.1, destroyed
  events missing for windows of apps launched **before** the window manager started.
- [kasper/phoenix#366](https://github.com/kasper/phoenix/issues/366) — AX window events
  not firing at all on macOS 26.0.

So the two candidates for why other machines reproduce the bug:

1. **Different macOS version** (the immune machine has been on 26.5.x since 2026-05-22).
2. **An AX-client app running there that suppresses delivery** (Contexts, Phoenix,
   Amazon Q, or similar launchers/window/clipboard tools).

Either way PR #2 is the right fix: it repairs the tracked entry when the live element
shows up and does not depend on the destroyed notification being delivered at all.

## Checklist for an affected machine

1. Note the macOS version and compare with the immune machine.
2. Build and run the probe (source below):

   ```sh
   swiftc -O -o HideProbe2 HideProbe2.swift
   ./HideProbe2 > probe2.log 2>&1 &
   PID=$!
   sleep 6;  kill -USR1 $PID   # hide  — does "AXUIElementDestroyed" appear in the log?
   sleep 4;  kill -USR2 $PID   # revive
   sleep 5;  kill -HUP  $PID   # open window B — does Amethyst MOVE window A?
   sleep 5;  kill -INT  $PID   # close window B
   sleep 5;  kill -TERM $PID
   ```

3. Read `probe2.log`:
   - No `AXUIElementDestroyed` line after the hide → delta confirmed; this machine's OS
     isn't delivering the notification.
   - Window A's frame frozen while B tiles around it → the orphan bug, live.
4. If the notification is missing on the *same* macOS version as the immune machine:
   quit AX-using tools one at a time (Contexts, Phoenix, BetterTouchTool, Amazon Q,
   launchers, clipboard/window utilities) and re-run the probe after each.

Extract the probe with:

```sh
awk '/^```swift HideProbe2.swift$/{f=1;next} f&&/^```$/{exit} f' \
  docs/hidden-window-ax-destroyed-investigation.md > HideProbe2.swift
```

## Probe source

```swift HideProbe2.swift
import AppKit
import ApplicationServices

@_silgen_name("_AXUIElementGetWindow")
func _AXUIElementGetWindow(_ element: AXUIElement, _ windowID: UnsafeMutablePointer<CGWindowID>) -> AXError

func now() -> String {
    let f = DateFormatter()
    f.dateFormat = "HH:mm:ss.SSS"
    return f.string(from: Date())
}

func out(_ s: String) {
    print("[\(now())] \(s)")
    fflush(stdout)
}

func axErrName(_ e: AXError) -> String {
    switch e {
    case .success: return "success"
    case .invalidUIElement: return "invalidUIElement"
    case .cannotComplete: return "cannotComplete"
    case .apiDisabled: return "apiDisabled"
    case .noValue: return "noValue"
    case .illegalArgument: return "illegalArgument"
    case .notificationUnsupported: return "notificationUnsupported"
    case .notificationAlreadyRegistered: return "notificationAlreadyRegistered"
    default: return "err(\(e.rawValue))"
    }
}

func axNotificationCallback(observer: AXObserver, element: AXUIElement, notification: CFString, refcon: UnsafeMutableRawPointer?) {
    var wid: CGWindowID = 0
    let widErr = _AXUIElementGetWindow(element, &wid)
    out("!!! AX-NOTIFICATION: \(notification) element-cgID=\(wid) (\(axErrName(widErr)))")
}

final class Delegate: NSObject, NSApplicationDelegate, NSWindowDelegate {
    var windowA: NSWindow!
    var windowB: NSWindow?
    var savedElement: AXUIElement?
    var observer: AXObserver?
    var appElement: AXUIElement!

    func applicationDidFinishLaunching(_ notification: Notification) {
        windowA = makeWindow(title: "HideProbeA", x: 137)
        out("LAUNCHED pid=\(getpid()) windowA number=\(windowA.windowNumber)")

        appElement = AXUIElementCreateApplication(getpid())
        var obs: AXObserver?
        let cerr = AXObserverCreate(getpid(), axNotificationCallback, &obs)
        if let obs = obs, cerr == .success {
            observer = obs
            CFRunLoopAddSource(CFRunLoopGetMain(), AXObserverGetRunLoopSource(obs), .defaultMode)
            let e1 = AXObserverAddNotification(obs, appElement, kAXWindowCreatedNotification as CFString, nil)
            out("self-observer created; windowCreated-on-app=\(axErrName(e1))")
        } else {
            out("self-observer creation FAILED: \(axErrName(cerr))")
        }

        installSignal(SIGUSR1) { [weak self] in self?.hideA() }
        installSignal(SIGUSR2) { [weak self] in self?.reviveA() }
        installSignal(SIGHUP) { [weak self] in self?.openB() }
        installSignal(SIGINT) { [weak self] in self?.closeB() }
        installSignal(SIGTERM) { NSApp.terminate(nil) }

        DispatchQueue.main.asyncAfter(deadline: .now() + 3.0) {
            self.captureAX()
            self.report("post-launch")
        }
    }

    func makeWindow(title: String, x: CGFloat) -> NSWindow {
        let w = NSWindow(
            contentRect: NSRect(x: x, y: 153, width: 800, height: 600),
            styleMask: [.titled, .closable, .miniaturizable, .resizable],
            backing: .buffered,
            defer: false
        )
        w.title = title
        w.isReleasedWhenClosed = false
        w.delegate = self
        w.makeKeyAndOrderFront(nil)
        NSApp.activate(ignoringOtherApps: true)
        return w
    }

    var sources: [DispatchSourceSignal] = []
    func installSignal(_ sig: Int32, _ handler: @escaping () -> Void) {
        signal(sig, SIG_IGN)
        let src = DispatchSource.makeSignalSource(signal: sig, queue: .main)
        src.setEventHandler(handler: handler)
        src.resume()
        sources.append(src)
    }

    func windowShouldClose(_ sender: NSWindow) -> Bool {
        if sender == windowA {
            out("windowShouldClose(A) -> hiding instead")
            windowA.orderOut(nil)
            return false
        }
        return true
    }

    func hideA() {
        out(">>> SIGUSR1: hiding windowA (orderOut)")
        windowA.orderOut(nil)
        DispatchQueue.main.asyncAfter(deadline: .now() + 2.0) { self.report("A hidden +2s") }
    }

    func reviveA() {
        out(">>> SIGUSR2: reviving windowA (same NSWindow)")
        windowA.makeKeyAndOrderFront(nil)
        NSApp.activate(ignoringOtherApps: true)
        DispatchQueue.main.asyncAfter(deadline: .now() + 2.5) {
            self.report("A revived +2.5s")
            self.reobserveDestroyOnFreshElement()
        }
    }

    func openB() {
        out(">>> SIGHUP: opening windowB (forces Amethyst reflow — can it move A?)")
        windowB = makeWindow(title: "HideProbeB", x: 400)
        DispatchQueue.main.asyncAfter(deadline: .now() + 3.0) { self.report("B open +3s") }
    }

    func closeB() {
        out(">>> SIGINT: destroying windowB (forces reflow back)")
        windowB?.delegate = nil
        windowB?.close()
        windowB = nil
        DispatchQueue.main.asyncAfter(deadline: .now() + 3.0) { self.report("B closed +3s") }
    }

    func captureAX() {
        var value: CFTypeRef?
        let err = AXUIElementCopyAttributeValue(appElement, kAXWindowsAttribute as CFString, &value)
        guard err == .success, let arr = value as? [AXUIElement], let first = arr.first else {
            out("AX self-capture FAILED: \(axErrName(err))")
            return
        }
        savedElement = first
        var wid: CGWindowID = 0
        _ = _AXUIElementGetWindow(first, &wid)
        if let obs = observer {
            let derr = AXObserverAddNotification(obs, first, kAXUIElementDestroyedNotification as CFString, nil)
            out("AX self-capture OK cgID=\(wid); destroyed-observer-on-window=\(axErrName(derr))")
        }
    }

    func reobserveDestroyOnFreshElement() {
        guard let obs = observer else { return }
        var value: CFTypeRef?
        let err = AXUIElementCopyAttributeValue(appElement, kAXWindowsAttribute as CFString, &value)
        guard err == .success, let arr = value as? [AXUIElement] else { return }
        for el in arr {
            var wid: CGWindowID = 0
            _ = _AXUIElementGetWindow(el, &wid)
            if wid == CGWindowID(windowA.windowNumber) {
                let derr = AXObserverAddNotification(obs, el, kAXUIElementDestroyedNotification as CFString, nil)
                out("re-observed destroyed on FRESH element cgID=\(wid): \(axErrName(derr))")
            }
        }
    }

    func report(_ phase: String) {
        out("---- REPORT: \(phase) ----")
        out("A: number=\(windowA.windowNumber) visible=\(windowA.isVisible) frame=\(NSStringFromRect(windowA.frame))")
        if let b = windowB {
            out("B: number=\(b.windowNumber) visible=\(b.isVisible) frame=\(NSStringFromRect(b.frame))")
        }

        if let el = savedElement {
            var title: CFTypeRef?
            let terr = AXUIElementCopyAttributeValue(el, kAXTitleAttribute as CFString, &title)
            out("saved-original-element read: \(axErrName(terr))")
        }

        let apps = NSWorkspace.shared.runningApplications
        func appName(_ pid: Int32) -> String {
            if pid == getpid() { return "HideProbe" }
            return apps.first { $0.processIdentifier == pid }?.localizedName ?? "pid\(pid)"
        }
        if let list = CGWindowListCopyWindowInfo([.optionOnScreenOnly], kCGNullWindowID) as? [[String: Any]] {
            let interesting = list.filter {
                (($0[kCGWindowLayer as String] as? Int) ?? -1) == 0
            }.compactMap { w -> (Int, String, CGRect)? in
                guard let id = w[kCGWindowNumber as String] as? Int,
                      let pid = w[kCGWindowOwnerPID as String] as? Int32,
                      let b = w[kCGWindowBounds as String] as? [String: CGFloat] else { return nil }
                let rect = CGRect(x: b["X"] ?? 0, y: b["Y"] ?? 0, width: b["Width"] ?? 0, height: b["Height"] ?? 0)
                guard rect.width >= 200 && rect.height >= 200 else { return nil }
                return (id, appName(pid), rect)
            }.sorted { $0.2.minX < $1.2.minX }
            for (id, name, rect) in interesting {
                out("  onscreen: x=\(Int(rect.minX)) w=\(Int(rect.width)) h=\(Int(rect.height)) id=\(id) \(name)")
            }
        }
        out("---- END REPORT ----")
    }
}

let app = NSApplication.shared
app.setActivationPolicy(.regular)
let delegate = Delegate()
app.delegate = delegate
app.run()
```

## Open questions

- Which factor differs on the affected machines: macOS version, or a notification-
  suppressing AX client? (Run the checklist above to find out.)
- Did the bug reproduce on the immune machine before its 26.5 update on 2026-05-22? If
  yes, that points squarely at the OS version.
- Whether Apple considers destroyed-on-hide delivery intended behavior at all; nothing
  is documented, so it may change again in either direction.
