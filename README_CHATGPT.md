Repro for dotnet/maui#38612 (related: discussion #35580)
Summary

An iOS app crashes with SIGABRT during UIKit layout while MAUI traverses the native view hierarchy.

Exception:

Failed to marshal the Objective-C object ... (type: Microsoft_Maui_Platform_WrapperView / LayoutView) ... does not have a constructor that takes one NativeHandle argument

Stack:

MauiView.LayoutSubviews -> ValidateSafeArea -> ... FindParent -> UIView.get_Superview

Suspected cause (unconfirmed): UIKit still holds a native view in its hierarchy, but its managed peer is no longer available. When MAUI traverses Superview during layout, the .NET for iOS runtime cannot recreate the managed instance.

The repro combines forced compacting GC, rapid UI updates, collection replacement, template rebuilding, scrolling, and native-backed views to help isolate the trigger.

Environment
.NET 10 SDK
Microsoft.Maui.Controls 10.0.110 (referenced in the project file)
Version 10.0.51 has reportedly worked; a controlled comparison is still needed to confirm a regression.
Physical iPhone: crash reproduces.
iOS simulator: crash has not reproduced.
Both Debug and Release/TestFlight builds can reproduce the crash.
Android: the same Chaos scenario has not reproduced the crash.
Steps to reproduce
Deploy the app to a physical iPhone.
Chaos starts automatically 3 seconds after launch (AutoStartChaos in MainPage.xaml.cs), or tap Chaos START.
Wait up to a minute.
If the app does not crash, terminate it and launch it again. The first launch tends to be fine.
What Chaos does (ChaosEngine.cs)
A background thread repeatedly forces full, blocking, compacting collections using GC.Collect(2, Forced, blocking, compacting) and calls WaitForPendingFinalizers.
(ProcessorCount - 1) threads burn CPU.
A background thread posts 30 MainThread.BeginInvokeOnMainThread updates per millisecond to simulate a high volume of MQTT callbacks.
A UI timer runs every 10 ms and:
Flips Right between null and a value on random rows.
Replaces Rows with a new collection.
Toggles the list's IsVisible.
Calls ScrollTo on random rows with animation disabled.
Scrolled starts unthrottled asynchronous polls to unreachable 192.168.1.1:80 without a timeout, then calls InvalidateSurface() and InvalidateMeasure() on the cell's SKCanvasView.

The Chaos tab provides independent switches for these operations and a duration setting, allowing individual components to be disabled during bisection.

MQTT-shaped simulation (MqttSimulator.cs)

FakeMqttClient follows the general shape of MQTTnet, with ConnectedAsync, DisconnectedAsync, and MessageReceivedAsync delegates of type Func<..., Task>, plus a TryPingAsync / ConnectAsync reconnect loop.

MQTT bad (MqttMode.BadMainThreadLoop)
The reconnect loop runs directly on the main thread, without Task.Run or Task.Delay.
Because the awaited operations complete synchronously, the loop does not return to the normal UI run loop for approximately 20 seconds.
The UI becomes completely unresponsive during this period.
Handlers execute inline and update the UI, including rebuilding Rows, flipping Right, toggling IsVisible, calling ScrollTo, and forcing LayoutIfNeeded.

With this mode enabled, the app remains frozen during the blocking period and the crash does not occur during that period.

MQTT good (MqttMode.GoodBackgroundLoop)
The reconnect loop runs in Task.Run and uses Task.Delay.
Handlers dispatch work through MainThread.BeginInvokeOnMainThread.
UI updates are coalesced every 200 ms.

With this mode, the main thread remains responsive, normal UI layout and event processing continue, and the crash reproduces.

AutoMqttMode in MainPage.xaml.cs selects the mode started automatically 5 seconds after launch. The default is bad.

Observed difference

The important distinction is not simply CPU load. In MQTT bad mode, the main thread is blocked and the normal UI event/layout cycle cannot proceed. In MQTT good mode, UI operations continue while Chaos is active, and the crash can occur.

This suggests that the crash depends on UI execution or its timing. It does not yet establish which operation is necessary for reproduction.

UI structure

The structure mirrors the relevant part of the real application:

Shell with flyout
→ RefreshView
→ CollectionView (MeasureAllItems, DataTemplateSelector)
→ row template with two TemplateHost controls
→ Border with Shadow and/or SKCanvasView

TemplateHost is a ContentView that synchronously rebuilds its child using a DataTemplateSelector whenever HostContent changes. Right can be null.

The Border shadow creates a platform WrapperView on the iOS path.

Expected behavior

The app should remain running under the stress scenario without a native/managed interop exception.

Actual behavior

The app terminates with SIGABRT and the ObjCRuntime.RuntimeException described above.

The stack points to MAUI layout and safe-area processing while traversing the UIKit superview hierarchy.

Crash logging

Without a debugger, CrashLogger hooks:

AppDomain.UnhandledException
TaskScheduler.UnobservedTaskException
ObjCRuntime.Runtime.MarshalManagedException
ObjCRuntime.Runtime.MarshalObjectiveCException

The logger appends entries to crash.log in AppDataDirectory.

After a crash, relaunch the app and open the Log tab. Use Refresh, Clear, or Share to inspect or export the log.

The log records the active Chaos configuration at Chaos START, whether a debugger was attached, runtime information, and the app's startup configuration. If a crash occurs after a Chaos START entry without a corresponding Chaos STOP, the crash happened during that run.

Bisection controls

The Chaos tab provides switches for:

GC hammer
CPU burn
UI-thread flood
Right flip
Rows replacement
List IsVisible toggle
Automatic ScrollTo
Camera/network polling triggered by Scrolled
Run duration

The tab also provides experiments for:

KeepAlive: retains strong references to elements, handlers, platform views, and container/WrapperView instances.
Card shadow: disables the card shadow.
Rounded shape: disables rounded shapes.

These experiments are intended to help distinguish a managed-object lifetime issue from a problem involving platform wrappers or other layout behavior. Their results should be treated as evidence, not proof, of a particular cause.

Hot Reload finding

In a Visual Studio debug session, the crash does not reproduce while Hot Reload is enabled, but it does reproduce when Hot Reload is disabled. Changing the XAML designer setting alone has no effect.

The issue therefore is not limited to running without a debugger.

Hot Reload or associated tooling changes the conditions required to reproduce the crash. Possible explanations include additional object references, altered object lifetimes, or timing differences. The exact mechanism has not been established.

Current working hypothesis

The failure may involve an inconsistency between the lifetime of a managed MAUI view and the lifetime or hierarchy of its native UIKit counterpart. Rapid template rebuilding, collection replacement, scrolling, layout invalidation, and forced compacting GC may increase the likelihood of the failure.

However, the exception alone does not prove that GC is the root cause. Handler disconnection/disposal, native view lifetime management, or another interaction in the MAUI iOS or .NET for iOS implementation may also be involved.

Questions for investigation
Can this be reproduced with a minimal combination of Chaos switches?
Does retaining strong references to managed views and their platform views prevent the crash?
Does disabling Border.Shadow or rounded shapes change the result?
Is there a regression between MAUI 10.0.51 and 10.0.110 under otherwise identical conditions?
Is the failure caused by MAUI's iOS view lifecycle, .NET for iOS Objective-C interop, or an interaction between them?
