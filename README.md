# Introduction
This article is based upon [Debugging-Unity-Games](https://github.com/dnSpy/dnSpy/wiki/Debugging-Unity-Games), and [Debugging Unity Games](https://bepinex.github.io/bepinex_docs/master/articles/advanced/debug/plugins_vs.html).

## Step 1: Enabling the game's debug mode:

### Converting Valheim into a Development Build
To be able to debug Valheim, the first step is to turn it into a Development Build. This enables IDEs to attach to it and follow code execution.

* Follow the instructions required to apply [BepInEx to Valheim](https://valheim.thunderstore.io/package/denikson/BepInExPack_Valheim/).
* Download [Unity 2022.3.17](https://unity.com/releases/editor/whats-new/2022.3.17) from the unity archive and install it.
* Make a backup of your game. (or trust steam "Verify Integrity" option)
* Navigate to `<unity-install-dir>\2022.3.17f1\Editor\Data\PlaybackEngines\windowsstandalonesupport\Variations\win64_player_development_mono\Data`
  * Make ensure that your path contains the correct version number.
* Copy both `Managed` and `Resources` directories, placing them into `<valheim-install-dir>\valheim_Data` and overwriting 32 files.
* Copy `WindowsPlayer.exe` and paste it into `<valheim-install-dir>`, renaming it as `valheim.exe` and overwriting the original.
* Copy `UnityPlayer.dll` and paste it into `<valheim-install-dir>`, overwriting the original.
* Copy `WinPixEventRuntime.dll` and paste it into `<valheim-install-dir>`
* Open `<valheim-install-dir>\valheim_Data\boot.config` and append the line: `player-connection-debug=1` and save.
* Launch the game

If you have followed all steps correctly, you should be able to observe a text at the bottom right-hand corner of the screen that reads **Development Build**:

![Image Valheim Development Build](https://github.com/Valheim-Modding/Wiki/blob/main/Debugging.Plugins/Development.Build.png)

## Step 2: Compiling symbols
The second step is to compile Symbols for your plugins. This allows IDEs to recognize your code when the Development Build executes it. **IF you are using the Jotunn ModStub, you may skip this step, as it has been automated upon your behalf.**

* Download [pdb2mdb](https://gist.github.com/jbevain/ba23149da8369e4a966f#file-pdb2mdb-exe) and place it in the root of your solution.
  * If you are using [this Rider template](https://github.com/Valheim-Modding/Wiki/blob/main/Debugging.Plugins/ValheimPluginTemplate.zip), then the PostBuild should already be setup, and you can skip the following step.
  * If not, the code below is meant to automate the debug and release process. If you structure your binary/build directories such that the various metadata required for upload are present and in the correct location for archiving, the build events below should be able to be tailored to your needs.
* Add a property group that contains the following PostBuild:
```xml
  <PropertyGroup>
    <PostBuildEvent>
      if $(ConfigurationName) == Release (
      powershell Compress-Archive -Path '$(ProjectDir)Package\*' -DestinationPath '$(SolutionDir)PublishOutput\$(ProjectName).zip' -Force)
      if $(ConfigurationName) == Debug del "$(TargetPath).mdb"
      if $(ConfigurationName) == Debug "$(SolutionDir)pdb2mdb.exe" "$(TargetPath)"
      xcopy "$(TargetDir)" "$(GameDir)\Bepinex\plugins\$(ProjectName)\" /q /s /y /i
      xcopy "$(ProjectDir)README.md" "$(ProjectDir)Package\" /q /y /i
    </PostBuildEvent>
  </PropertyGroup>
```
* Go to your project debug configuration (click advanced for VS users) and ensure that Debug symbols are enabled and set to produce `.pdb`.
![Image showing Rider Debug configuration has generation of debug symbols enabled and set to PDB only](https://github.com/Valheim-Modding/Wiki/blob/main/Debugging.Plugins/PDB.Only.png)
* Build the project, and observe the `.mdb` file sitting alongside your plugin in the plugins directory.
  * If you didn't automate this step, drop `YourPlugin.dll` on top of `pdb2mdb.exe` while `YourPlugin.pbd` is in the same folder as `YourPlugin.dll`. This will create a file called `YourPlugin.mdb`.

## Step 3: Extra Step Required for Debugging with Rider
Rider isn't able to resolve types declared in `UnityEngine.CoreModule.dll` (which includes MonoBehaviour, so this is a big problem) without some help.  
This is due to the way that BepInEx is modifying this dll and then loading the modified version instead.  
The fix is to have BepInEx dump the modified version to disk as another dll, so that Rider can find it when debugging:
1. In `BepInEx\config\BepInEx.cfg` you need to set `DumpAssemblies = true` and `LoadDumpedAssemblies = true`. 
2. Run the game like normal, then quit again once you reach the main menu.
3. **Only do this if it doesn't already work after step 2**: replace your reference to `UnityEngine.CoreModule.dll` with the one now found in `BepInEx\DumpedAssemblies`.

MonoBehaviour-derived types should now resolve correctly in the debugger.

## Step 4: Debugging your plugin

* Create a breakpoint
* Launch the game
* Attach the process
  * (VS2022) Click Debug > Attach Unity Debugger and select the only process available.
  * (Rider) Click the Unity symbol at the top: ![Rider's "attach to Unity process button"](https://github.com/paddywaan/Wiki/blob/main/Debugging.Plugins/Rider.AttachToUnityProcess.png), and attach to Valheim:![](https://github.com/Valheim-Modding/Wiki/blob/main/Debugging.Plugins/Rider.AttachDebug.png)
* When your breakpoint is hit, the game will freeze and you will be able to proceed with debugging your plugin.

## Known Issues
* ### VS2022 "Attach Unity Debugger" button not appearing

Some users (paddy) have experienced extreme difficulty attempting to get visual studio to debug BepInEx plugins, and in some cases VS2019 simply refuses to attach to a unity process. Supposedly you just have to download "Visual Studio Tools for Unity", as shown here:

![Image showing Visual studio installer, showing unity tools is installed](https://github.com/Valheim-Modding/Wiki/blob/main/Debugging.Plugins/VSInstaller.PNG)

But "Attach Unity Debugger" button won't appear in Debug menu:

![Image showing visual studio's debug menu, absent the "Attach to Unity Debug" option](https://github.com/Valheim-Modding/Wiki/blob/main/Debugging.Plugins/VSAttachUnity.png)

**RESOLVED!:** The above issue has been resolved by purging visual studio from the system and re-installing it.

* ### Failed to load Mono

  * **Solution 1.** If you were trying to convert Valheim into a Development Build try to copy `<unity-install-dir>\Editor\Data\PlaybackEngines\windowsstandalonesupport\Variations\win64_development_mono\MonoBleedingEdge` and paste it into `<valheim-install-dir>`, overwriting everything.

  * **Solution 2.** Uninstall **Unity Hub** and/or **Unity 2022.3.17**. Install a fresh copy of Valheim or revert back to your backup. Reinstall [Unity Hub](https://unity3d.com/get-unity/download). Remember to install **Unity 2022.3.17** through **Unity Hub**.

* ### Breakpoints are not being hit
This may be caused by attaching the process too early. Try to attach it after Main Menu finish loading.
