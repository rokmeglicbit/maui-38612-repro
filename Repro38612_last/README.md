# Repro for dotnet/maui#38612 (related: discussion #35580)

iOS crash (SIGABRT):
`Failed to marshal the Objective-C object ... (type: Microsoft_Maui_Platform_WrapperView / LayoutView) ... does not have a constructor that takes one NativeHandle argument`

Stack: `MauiView.LayoutSubviews -> ValidateSafeArea -> ... FindParent -> UIView.get_Superview`

Cause (suspected): the managed peer of a native view is garbage collected while iOS still holds the native view
in the hierarchy; during layout MAUI walks `Superview` and the runtime cannot re-create the managed instance.

## Environment
- .NET 10 SDK, Microsoft.Maui.Controls 10.0.110 (see the `Microsoft.Maui.Controls` PackageReference in csproj; reportedly OK on 10.0.51)
- PHYSICAL iPhone (does not reproduce on simulator), Debug and Release/TestFlight

## Steps
1. Deploy to a physical device. Chaos starts automatically 3 s after launch (`AutoStartChaos` in MainPage.xaml.cs),
   or tap the **Chaos START** button.
2. Wait up to a minute. If it doesn't crash: kill the app and launch it again (the first launch tends to be fine).

## What "Chaos" does (ChaosEngine.cs)
- background thread forces `GC.Collect(2, Forced, blocking, compacting)` + `WaitForPendingFinalizers` in a loop
- (ProcessorCount - 1) threads burn CPU
- background thread posts 30 `MainThread.BeginInvokeOnMainThread` updates per ms (simulates MQTT callbacks)
- UI timer (10 ms): flips `Right` null <-> value on random rows, replaces `Rows` with a new collection,
  hides/shows the list (`IsVisible`), calls `ScrollTo` on random rows with animate=false
- `Scrolled` starts un-throttled async polls to unreachable `192.168.1.1:80` (no timeout) and then calls
  `InvalidateSurface()` + `InvalidateMeasure()` on the cell's `SKCanvasView`

## MQTT-shaped simulation (MqttSimulator.cs)
`FakeMqttClient` has the same shape as MQTTnet (`ConnectedAsync`, `DisconnectedAsync`, `MessageReceivedAsync` as `Func<..., Task>`,
`TryPingAsync` + `ConnectAsync` reconnect loop).

- **MQTT bad** (`MqttMode.BadMainThreadLoop`): the reconnect loop runs ON the main thread (no `Task.Run`, no `Task.Delay`).
  Because every await completes synchronously the loop never returns to the run loop (20 s); the app only "works" because
  the handlers run inline and update the UI (rebuild Rows, flip Right, toggle IsVisible, ScrollTo, forced `LayoutIfNeeded`).
- **MQTT good** (`MqttMode.GoodBackgroundLoop`): loop in `Task.Run` with `Task.Delay`, handlers only call
  `BeginInvokeOnMainThread` and UI updates are coalesced every 200 ms.

Run both with **Chaos** on (background GC / CPU burn / flood) and compare. `AutoMqttMode` in MainPage.xaml.cs selects what
starts automatically 5 s after launch (default: bad).

## UI structure (same as the real app)
Shell with flyout > RefreshView > CollectionView (MeasureAllItems, DataTemplateSelector)
> row template with 2x `TemplateHost` (ContentView that synchronously rebuilds its child from a DataTemplateSelector
when `HostContent` changes; `Right` can be null) > Border with Shadow (creates WrapperView) / SKCanvasView.

## Expected
No crash. 

## Actual
App terminates with the ObjCRuntime.RuntimeException above.

## Crash log without a debugger
`CrashLogger` hooks AppDomain.UnhandledException, TaskScheduler.UnobservedTaskException,
ObjCRuntime.Runtime.MarshalManagedException and MarshalObjectiveCException and appends to `crash.log` in AppDataDirectory.
After a crash, relaunch and open the **Log** tab (Refresh / Clear / Share).

## Bisecting
The **Chaos** tab has a switch for every part (GC hammer, CPU burn, UI flood, Right flip, Rows replace, IsVisible toggle,
auto ScrollTo, camera polls) and a duration. The active options and whether a debugger was attached are written to `crash.log`
("Chaos START ..." / "App start ... DebuggerAttached=..."), so after a crash the log shows which combination was running.

## Hot Reload finding
In a debug session the crash does NOT happen while Hot Reload is enabled, but DOES happen with Hot Reload disabled
(XAML designer on/off has no effect). Hot Reload / Live Visual Tree probably keeps strong references to elements and hides a lifetime problem.
The Chaos tab has experiments: **KeepAlive** (keeps strong references to all elements, handlers, platform views and container/WrapperViews),
**Card shadow** (WrapperView) and **Rounded shape**.
