⚠️ This environment is still in beta, crashes and future changes are to be expected ⚠️
# Examples

General:
1. [Natives](#natives)
2. [Coroutines replace Tasks](#coroutines-replace-tasks)
3. [Events callable from anywhere](#events-callable-from-anywhere)
4. [Export invocation](export-invocation)
5. [Register Events and restrict their accessibility](#register-events-and-restrict-their-accessibility)
6. [SourceAttribute](#sourceattribute)
7. [ExportAttribute](#exportattribute)
8. [Disable & Enable BaseScripts](#disable--enable-basescripts)
9. [EnableOnLoadAttribute](#enableonloadattribute)
10. [CString](#cstring)

Other:
1. [Implicit parameter conversion with Events, Exports, and callbacks](MsgPack.md)


## Natives
As before, but now they are in the **Natives** class, found in the *CitizenFX.FiveM.Native*, *CitizenFX.RedM.Native*, *CitizenFX.Server.Native*, or *CitizenFX.Shared.Native* namespace
```csharp
using CitizenFX.FiveM.Native; // client FiveM
//using CitizenFX.RedM.Native; // client RedM
using CitizenFX.Server.Native; // server
using CitizenFX.Shared.Native; // shared (for shared libraries)
...
int health = Natives.GetEntityHealth(ped);
int health = Natives.Call<int>(Hash.GET_ENTITY_HEALTH, ped);
```

## Coroutines replace Tasks
Where you used `Task`/`Task<T>` before, you now use `Coroutine`/`Coroutine<T>`, it's as simple as replacing Task with Coroutine. Below you can see that scheduling them as a repeating coroutine is still the same, if you want to schedule a coroutine/action as a non-repeating action then use `Scheduler.Schedule(Action [, int])`
```csharp
public Constructor()
{
	Tick += AsyncFunction; // creates and activates a repeating coroutine
}

async Coroutine AsyncFunction()
{
	// do some stuff here
	await WaitUntilNextFrame();

	// do some more stuff here
	await Yield(); // same as WaitUntilNextFrame

	// let's do stuff, then 1 second sleepy sleepy
	await Wait(1000);

	// done, though it'll run again next frame as it's registered as a repeating tick
}
```
\* *As a side note; although its scheduler is no longer restricted, we do not offer any support for using `Task` in v2 and may be prone to deactivation if required.*

## Events callable from anywhere
Events callable from anywhere
```csharp
Events.TriggerEvent("EventName", ...)
Events.TriggerServerEvent("EventName", ...)
Events.TriggerClientEvent("EventName", Player, ...)
```

## Export invocation
Export's calling syntax has changed, v1 used dynamic intermediate code that caused seconds of JIT lag, with the following syntax we circumvent that:
```csharp
Exports["ResourceName", "ExportName"](...); // in BaseScript inherited classes
Exports.Local["ResourceName", "ExportName"](...); // call from anywhere
```

## Register Events and restrict their accessibility
EventHandlers can now use the `Binding` overload to determine who can call this handler/export. The options are:
1. **Local** (default): client only accepts client events, server only server events
2. **Remote**: client only accepts server events, server only accepts client events
3. **All**: accept events from both client and server

Mind you, unlike the v1 runtime; v2's `EventHandlers["EventName"] += ` is an indirect call to `EventHandlers["EventName"].Add(DynFunc, Binding.Local)`, local binding is the default for security reasons. You should call `EventHandlers["EventName"].Add(DynFunc, <your desired binding>)` to override and/or make your intent clear on who may call the given event listener.

```csharp
[EventHandler("EventName", Binding.Local)]
private string EventFunction()
{
	return "Remote can't touch this, ...";
}
```

### Extra ways to (un)register:
Events no longer pass `List<object>` objects, they are now pure arrays `object[]`.
```csharp
Constructor()
{
	// Add as Binding.Local
	EventHandlers["EventName"] += Func.Create<int, object[]>(EventFunction1);
	EventHandlers["EventName"].Add(Func.Create<float>(EventFunction2));
	EventHandlers["EventName"] += Func.Create<float, int>(EventFunction3WithReturn);
	
	// Remove
	EventHandlers["EventName"] -= Func.Create<int, object[]>(EventFunction1);
	EventHandlers["EventName"].Remove(Func.Create<float>(EventFunction2));
	EventHandlers["EventName"] -= Func.Create<float, int>(EventFunction3WithReturn);
	
	// Add as Binding.Remote or Binding.All
	EventHandlers["EventName"].Add(Func.Create<float>(EventFunction2), Binding.Remote);
	EventHandlers["EventName"].Add(Func.Create<float>(EventFunction2), Binding.All);

	// Register from anywhere
	Event.RegisterEventHandler("EventName", Func.Create<int, object[]>(EventFunction1), Binding.Local);
	
	// Unregister from anywhere
	Event.UnregisterEventHandler("EventName", Func.Create<int, object[]>(EventFunction1));
}

void EventFunction1(int i, object[] values) { }
void EventFunction2(float f) { }
int EventFunction3WithReturn(float f) { }
```

## SourceAttribute
Previously `[FromSource]`, now simplified to `[Source]`. It also allows clients to know if the event was called from the server or not. Also note that these parameters don't influence the parameter order when invoked as an event, so if an exception like a `CastException` occurs solve them as if any `[Source]` parameter is absent.
```csharp
// Server
[EventHandler("EventName", Binding.All)]
private string EventFunction([Source] Player source) { }

// Client
[EventHandler("EventName", Binding.All)]
private string EventFunction([Source] bool isServer) { }

// Shared
[EventHandler("EventName", Binding.All)]
private string EventFunction([Source] Remote remote) { }

[EventHandler("EventName", Binding.All)]
private string EventFunction([Source] bool isRemote) { }
```


## ExportAttribute
Previously non-existent, can be used as follows:
```csharp
[Export("ExportName")]
private string ExportE2()
{
	return "yeehaa";
}
```

## Disable & Enable BaseScripts
BaseScript's ticks, events, exports, and commands can all be disabled and enabled with ease, allowing you to create and cache something like game modes or anything else you'd like to use it for
```csharp
class MyScript
{
	[Tick]
	private async Coroutine MyTick()
	{
		// tick that'll not be rescheduled when disabled
	}
	
	[Tick]
	private async Coroutine MyTickwhileLooped()
	{
		while (IsEnabled) // run as long as this script is enabled
		{
			// my things
		}
		
		// only when we get here will this tick be disabled
	}
}

BaseScript script = new MyScript();

script.Disable();
// script is inactive, ticks will come to a halt when they exit their scope

script.Enable();
// script is activated, all events, exports, and commands are enabled, ticks are rescheduled or continued
```

## EnableOnLoadAttribute
Want to make use of the BaseScript functionalities but without activating it on startup? Add `[EnableOnLoad(false)]` to the class and it won't be automatically activated.
```csharp
[EnableOnLoad(false)]
class MyScript : BaseScript
{ }

...

MyScript myScript = new MyScript();

void SomeFunctionToActivateMyScript()
{
	BaseScript.RegisterScript(myScript);
}
```


## CString
A UTF8 encoded string dedicated for interop. Do not use this as a replacement to C#'s `string`, if at all. However it is an interesting storage type for repeated interop usage of data that remains constant.
```csharp
CString value = "This string is converted to a UTF8 string and doesn't need reconversion on interop!";
```
