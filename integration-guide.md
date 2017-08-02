FASTBuild is an open-source distributed build system, which could be a free alternative to Incredibuild. Unreal Engine 4 (UE4) does not support FASTBuild natively, however it's not hard to integrate it manually.

## Integrate FASTBuild with UE4

### Install FASTBuild
We assume you already have the full UE4 source code. First you'll need to grab the latest FASTBuild tools from [here](http://www.fastbuild.org/docs/home.html). We use v0.93 Windows x64 version in this tutorial. Download it and extract all the files. Here you have several choices:
- Place the files under any folder which could be found with your system's `PATH` environment variable. To see where these folders are, run the `PATH` command in a command prompt window;
- Place the files under the `Engine\Binaries\ThirdParty\FASTBuild` folder of your engine. This is the recommended place;
- Place the files anywhere you like. This is not recommended because you'll have to hard-code the path later.

### ActionExecutor
Then you'll need an `ActionExecutor`, which acts as a bridge between Unreal Build Tool (UBT) and FASTBuild. There is already one written by [liamkf](https://github.com/liamkf/Unreal_FASTBuild), I also forked it and made my own version, which contains bug fixes and improvements, as well as 4.16 support. In this tutorial, we will use this variant. You can get it [here](https://github.com/hillin/Unreal_FASTBuild).

The `ActionExeuctor` is simply a C# class. Download the file `FASTBuild.cs` and place it under the `Engine\Source\Programs\UnrealBuildTool\System` folder of your engine.

If you have placed your FASTBuild files in a random location, you can change the default value of `FBuildExePathOverride` in `FASTBuild.cs`.

### Modify UBT
The next step is to teach Unreal Build Tool how to use FASTBuild. This involves several modifications to the engine code. Add the lines with the plus sign before them to your engine's source code.

#### BuildConfiguration.cs
Location: *Engine\Source\Programs\UnrealBuildTool\Configuration\BuildConfiguration.cs*
```csharp
        [XmlConfig]
        public static bool bUseUHTMakefiles;

+		[XmlConfig]
+		public static bool bAllowFastbuild;

		/// <summary>
		/// Whether DMUCS/Distcc may be used.
		/// </summary>
		[XmlConfig]
		public static bool bAllowDistcc;
```

#### UEBuildPlatform.cs
Location: *Engine\Source\Programs\UnrealBuildTool\Configuration\UEBuildPlatform.cs*
```csharp
		public virtual bool CanUseDistcc()
		{
			return false;
		}

+		public virtual bool CanUseFastbuild()
+		{
+			return false;
+		}

		/// <summary>
		/// If this platform can be compiled with SN-DBS
		/// </summary>
		public virtual bool CanUseSNDBS()
		{
			return false;
		}
```
#### ActionGraph.cs
Location: *Engine\Source\Programs\UnrealBuildTool\System\ActionGraph.cs*
```csharp
				if ((XGE.IsAvailable() && BuildConfiguration.bAllowXGE) || BuildConfiguration.bXGEExport)
				{
					Executor = new XGE();
				}
+				else if (FASTBuild.IsAvailable() && BuildConfiguration.bAllowFastbuild)
+				{
+					Executor = new FASTBuild();
+				}
				else if(BuildConfiguration.bAllowDistcc)
				{
					Executor = new Distcc();
				}
```
#### UEBuildWindows.cs
Location: *Engine\Source\Programs\UnrealBuildTool\Windows\UEBuildWindows.cs*
```csharp
		[XmlConfig]
		public static bool bLogDetailedCompilerTimingInfo = false;

+		public override bool CanUseFastbuild()
+		{
+			return true;
+		}

		/// True if we should use Clang/LLVM instead of MSVC to compile code on Windows platform
		public static readonly bool bCompileWithClang = false;
```
That should be all. Your UE4 source code compilation will be handled by FASTBuild from now on.

## Unleash the Power of FASTBuild

### Setting up Network Distribution

[FASTBuild Documentation](http://www.fastbuild.org/docs/features/distribution.html) has fully covered this topic. Essentially, you'll need to:
- Have a shared folder which everyone in the intra-network can read and write
- Have the build machine (master) and all worker machines (slaves) set their `FASTBUILD_BROKERAGE_PATH` environment variable to the address of this shared folder. You can use the [`setx`](https://technet.microsoft.com/en-us/library/cc755104(v=ws.11).aspx) command or  the [Advanced System Settings window](https://www.computerhope.com/issues/ch000549.htm) to do this. 
- Run `FBuildWorker.exe` on all worker machines

I prefer to write a batch file so all team members can set this up in a double-click:
```bat
setx FASTBUILD_BROKERAGE_PATH "\\smellyriver\FASTBuild"	:: set environment variable
reg add HKCU\Software\Microsoft\Windows\CurrentVersion\Run /v FASTBuildWorker /d FBuildWorker.exe	:: start with windows
start FBuildWorker.exe	:: run build worker
```

You can check the shared folder to see which worker machines are available.

### Enable Cached Mode

FASTBuild can cache built objects. As long as the corresponding source is not changed, FASTBuild can reuse the cached object, rather than compile it again. This brings a tremendous boost to compile speed.
You can either place the cache folder locally or in a network sharing place. Local cache has better performance (due to heavy IO), but network cache could be shared with your team, so team members could take advantage of others' build result.

To enable cache mode, you can either:
- set the `FASTBUILD_CACHE_PATH` environment variable to the path of your cache folder
- or, modify *FASTBuild.cs*:
```C#
private bool bEnableCaching = true;	// enable caching
private string CachePath = @"\\YourCacheFolderPath";   // set cache folder path
```

## Troubleshooting

### Incredibuild is used over FASTBuild
If you have Incredibuild installed, it will have a higher priority over FASTBuild. You can turn off UE4's Incredibuild support by modifying `BuildConfiguration.xml`:
#### BuildConfiguration.xml
Location: *Engine\Saved\UnrealBuildTool\BuildConfiguration.xml*
```xml
<?xml version="1.0" encoding="utf-8" ?>
<Configuration xmlns="https://www.unrealengine.com/BuildConfiguration">
	<BuildConfiguration>
+		<bAllowXGE>false</bAllowXGE>
	</BuildConfiguration>
</Configuration>

```

### PUSH_MACRO and POP_MACRO issue
If you encounters a compilation issue at a `PUSH_MACRO` (and `POP_MACRO`) macro complaining a syntax error of `')'`, try to replace them with `#pragma push_macro(...)` and `#pragma pop_macro(...)` as a workaround. This is a FASTBuild bug, you can find more details [here](https://github.com/fastbuild/fastbuild/issues/256).

### Issue with Windows 10 SDK
If you are using Windows 10 SDK, there could be some issues with the XInput library. Modify `Core.Build.cs` to fix this:
#### Core.Build.cs
Location: *Engine\Source\Runtime\Core\Core.Build.cs*
```csharp
-			AddEngineThirdPartyPrivateStaticDependencies(Target,
-				"IntelTBB",
-				"XInput");

+			AddEngineThirdPartyPrivateStaticDependencies(Target,
+				"IntelTBB");

+			if (!WindowsPlatform.bUseWindowsSDK10)
+			{
+				AddEngineThirdPartyPrivateStaticDependencies(Target, "XInput");
+			}
+			else
+			{
+				PublicAdditionalLibraries.Add("XInput.lib"); //Included in Win10 SDK
+			}
```
### Warning C4628
The following warning could be raised during compilation:
```
warning C4628: digraphs not supported with -Ze. Character sequence '<:' not interpreted as alternate token for '['
```
You can suppress this warning by modifying `WindowsPlatformCompilerSetup.h`:
#### WindowsPlatformCompilerSetup.h
Location: *Engine\Source\Runtime\Core\Public\Windows\WindowsPlatformCompilerSetup*
```cpp
-		#pragma warning(default : 4628) // digraphs not supported with -Ze. Character sequence 'digraph' not interpreted as alternate token for 'char'
+		#pragma warning(disable : 4628) // digraphs not supported with -Ze. Character sequence 'digraph' not interpreted as alternate token for 'char'
```
(i.e. change `default` to `disable`.)

## Further Reading
[FASTBuild Documentation](http://www.fastbuild.org/docs/documentation.html)

[Unreal_FASTBuild](https://github.com/liamkf/Unreal_FASTBuild)
