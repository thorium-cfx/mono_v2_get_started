⚠️ This environment is still in beta, crashes and future changes are to be expected ⚠️
# Set up your C# solution
This guide expects that the reader already knows how to work with Visual Studio and how to [set up and distribute FiveM resources](https://docs.fivem.net/docs/scripting-manual/introduction/).

1. Create the solution and projects
	* CLI, automatic:  
		```cmd
		dotnet new sln -o MySolution
		chdir MySolution
		dotnet new classlib --target-framework-override net452 -o Client
		dotnet new classlib --target-framework-override net452 -o Server
		dotnet sln add Client Server
		xcopy %localappdata%\FiveM\FiveM.app\citizen\clr2\lib\mono\4.5\v2\Native\CitizenFX.FiveM.Native.dll Client\bin\ /Y /I
		:: for RedM: remove or comment out the above line and uncomment (remove ::) from below line
		::xcopy %localappdata%\FiveM\FiveM.app\citizen\clr2\lib\mono\4.5\v2\Native\CitizenFX.RedM.Native.dll Client\bin\ /Y /I
		pause
		```
		\* Higher Framework versions are possible, but Visual Studio may complain on some systems.
	* Manual:  
		Create a new C# **Class Library (.NET Framework)** and add another project in the solution for the missing server or client, also copy **Native/CitizenFX.FiveM.Native.dll** with your client project.
2. Go into your \<MySolution\> directory and open the solution
3. In Visual Studio you right click the project and *Add -> Add Assembly Reference...*
	* Client FiveM:
		**CitizenFX.FiveM.dll** and **CitizenFX.Core.dll** from `%localappdata%\FiveM\FiveM.app\citizen\clr2\lib\mono\4.5\v2\`,
		**CitizenFX.FiveM.Native.dll** from `bin\` or where your copied it to.
	* Client RedM:
		**CitizenFX.RedM.dll** and **CitizenFX.Core.dll** from `%localappdata%\RedM\RedM.app\citizen\clr2\lib\mono\4.5\v2\`,
		**CitizenFX.RedM.Native.dll** from `bin\` or where your copied it to.
	* Server:  
		**CitizenFX.Server.dll** and **CitizenFX.Core.dll** from your server files `<your server files>\citizen\clr2\lib\mono\4.5\v2\`.
4. Replace the contents of both **Class.cs** files and preferably rename them as well
	```csharp
	using CitizenFX.Core;
	//using CitizenFX.FiveM; // FiveM game related types (client only)
	//using CitizenFX.FiveM.Native; // FiveM natives (client only)
	//using CitizenFX.RedM; // RedM game related types (client only)
	//using CitizenFX.RedM.Native; // RedM natives (client only)
	//using CitizenFX.Server.Native; // Server natives (server only)
	//using CitizenFX.Shared.Native; // Shared natives (there for shared libraries)
	
	namespace MyLibrary.Client // probably update your default namespace as well
	{
		public class Class1 : BaseScript
		{
			
		}
	}
	```
5. Now you should have a solution with 2 projects, targeting .NET Framework 4.5.2 (or higher), and have all references correctly set up.
6. Edit your **fxmanifest.lua** file and add the following to it:  
	`mono_rt2 'Prerelease expiring 2023-06-30. See https://aka.cfx.re/mono-rt2-preview for info.'`  
	\* *rt2 stands for runtime 2, as to not confuse it with a potential v2 of mono itself.*
	\* *this flag is considered temporary and will be removed at some point.*
7. Add your copy of **CitizenFX.FiveM.Native.dll** or **CitizenFX.RedM.Native.dll** (depending on your game) to the resource and also add it in your **fxmanifest.lua** as well, just like any other dll dependency

[Continue by looking into examples](Examples.md)
