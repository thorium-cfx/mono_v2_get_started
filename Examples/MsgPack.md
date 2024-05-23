⚠️ This environment is still in beta, crashes and future changes are to be expected ⚠️
# Serialization and Deserialization with MsgPack data

⚠️ Note: these features will be enabled once [PR 2546](https://github.com/citizenfx/fivem/pull/2546) has been merged.

## Implicit parameter conversion
Enjoy automatic and implicit conversion of all incoming data to the parameters that you request, no more need to create a `Player` object from an `int` yourself. This is done by converting the internal MsgPack byte data directly to the requested type, unlike mono v1 who constructs an intermediate object first, packs it in an `object[]` array, who in turn are converted again.

This approach is at least 3 times faster than the one used in mono v1, that does mean that creating multiple listeners is not advised as the conversion cost will now be per listener, best keep it to 1 or at least 3 or less.

Example:
```csharp
[EventHandler("myEvent")] public static async Coroutine GimmeAll(int a)
	=> Debug.WriteLine($"GimmeAll1 {a}");

[EventHandler("myEvent")] public static async Coroutine GimmeAll(string a, int b)
	=> Debug.WriteLine($"GimmeAll2 {a} {b}");

[EventHandler("myEvent")] public static async Coroutine GimmeAll(string a, int b, string c = "Hey")
	=> Debug.WriteLine($"GimmeAll3 {a} {b} {c}");

[EventHandler("myEvent")] public static async Coroutine GimmeAll(int a, string b, string c = "Oh", int d = 678)
	=> Debug.WriteLine($"GimmeAll4 {a} {b} {c} {d}");

[EventHandler("myEvent")] public static async Coroutine GimmeAll(int a, Player b, string c = "Oh", int d = 678)
	=> Debug.WriteLine($"GimmeAll5 {a} {b} {c} {d}");

// Trigger it!
[Command("Gimme")] public async Coroutine Gimme(uint source, object[] objects, string raw)
	=> Events.TriggerServerEvent("myEvent", 1234, "5678");
```
The above will result in the following output:
```
[    script:csharp_v2] GimmeAll1 1234
[    script:csharp_v2] GimmeAll2 1234 5678
[    script:csharp_v2] GimmeAll3 1234 5678 Hey
[    script:csharp_v2] GimmeAll4 1234 5678 Oh 678
[    script:csharp_v2] GimmeAll5 1234 Player(5678) Oh 678
```


## Serializable classes/structs
Want to automate your custom type's (de)serialization, then include `﻿using CitizenFX.MsgPack;` and apply the `MsgPackSerializable(Layout)` attribute to your type.

#### Best practices
Before we jump into how this is done, here's a list of best practices to make your resource more performant and as a result lets your servers run more smoothly.
1. Reduce packet size:
	1. Only send data that is necessary, e.g.: send update packets with player ids instead of sending the player's name and/or other data you don't need with each request.
	2. Use `Layout.Indexed` over any mapped/keyed based solution, removing unnecessary key strings in your packets.
	3. Send integers (incl. enums) instead of strings.
	4. Don't send strings, unless it's required, mind you: each character takes in 1 byte and we need to store the length.
	5. Send strings only once, e.g.: sending a players name per each name change.
	6. Send `float`s instead of `double`s, this'll reduce byte usage by half: 4B vs 8B.
	7. Check the [MsgPack spec](https://github.com/msgpack/msgpack/blob/master/spec.md) for type sizes. We already store integers in the smallest possible container that can hold the given value.
	8. While networking: stay below or aim for 1KB packets or more wisely: read into and consider maximum packet sizes of MTU and UDP, also consider headers and our network layer overhead, including event names.
	9. Locally: best to keep everything within 1 assembly and limit your use of events, exports, and callbacks (NUI, refFuncs), although size is less of an issue, the serialization + internal redirection logic + deserialization will slow down your resources unnecessarily.
3. Send packets as little as possible:
	1. Better to send a packet that triggers an event to happen every second than sending an event every second, the more you send the more packet loss can occur.
	2. The less you send the more players you can be of service, both computational and network overhead.

#### Layouts
You can choose between 3 types of layouts for your types: `Indexed`, `Keyed`, and `Default`, mentioned in the order you should generally consider them. Read below for more information about them.

Note: classes and structs that aren't marked as `MsgPackSerializable` will default to the `Layout.Default` layout. This may result in errors for some types, generally these are considered to be a user-error, meaning you are responsible for fixing these in your code.

##### Layout.Indexed
(De)serializes all fields and properties marked with the `Index` attribute to/from an array. Great to reduce data and increase networking performance.
```csharp
[MsgPackSerializable(Layout.Indexed)]
public class MyClass
{
	[Index(0)] public uint m_id = 0;
	public string m_name = "My very long name" // Let's not send this to reduce packet size, client/server should already know who this is with the given id
	[Index(1)] private float m_health = 300.0;
}
```
TThe above will result in the following serialized form (visualized): `[ 0, 300.0f ]` // 6 bytes

\* Concerns: there's a lack of context per field, meaning other scripts who don't know about the array's layout/order may not be able to use them dynamically.

##### Layout.Keyed
Maps all fields and properties marked with the `Key` attribute
```csharp
[MsgPackSerializable(Layout.Keyed)]
public class MyClass
{
	[Key("id")] public uint m_id = 0;
	[Key("name")] public string m_name = "My very long name";
	
	private double m_health = 300.0;
	[Key("health")] public double Health
	{
		get => m_health;
		set => m_health = value;
	}
}
```
The above will result in the following serialized form (visualized): `{ "id": 0, "name": "My very long name", "health": 300.0 }` // 43 bytes


\* Concerns: the strings and double values take most room in our packet, consider going for `Layout.Indexed`, remove `name`, and/or (de)serializing floats instead of doubles.

##### Layout.Default
Maps all public fields and properties, except those marked with the `Ignore` attribute
```csharp
[MsgPackSerializable(Layout.Default)]
public class MyClass
{
	public uint m_id = 0;
	[Ignore] public string m_name = "Let's not send this to reduce packet size, client/server should already know who this is with the given id";
	
	private double m_health = 300.0;
	public double Health
	{
		get => m_health;
		set => m_health = value;
	}
}
```
The above will result in the following serialized form (visualized): `{ "m_id": 0, "Health": 300.0 }` // 22 bytes

\* Concerns: the strings and double values take most room in our packet, consider going for `Layout.Indexed` and/or (de)serializing floats instead of doubles.

#### Constructors
Don't want to open up all fields/properties for write permission? Use a constructor instead. Make sure you match the fields/properties and constructor's parameter types, including their order.

Note: only incoming arrays and types marked `Layout.Indexed` are considered for this fast form of construction.

```csharp
[MsgPackSerializable(Layout.Indexed)]
public class MyClass
{
	// Mind the types and order of these fields
	[Index(0)] readonly public uint m_id = 0;
	[Index(1)] public float m_health = 100.0;
	
	private float m_armor = 100.0;
	[Index(2)] public float Armor => m_armor;
	
	// Follow above fiels/properties their types and order
	public MyClass(uint id, float health, float armor)
	{
		m_id = id;
		m_health = health;
		m_armor = armor;
	}
}
```