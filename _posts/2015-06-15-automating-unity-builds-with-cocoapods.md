---
layout:     post
title:      Automating Unity iOS builds
date:       2015-06-15 15:20:00
summary:    Using CocoaPods, Xcode Manipulation API and shell to automate the Unity iOS build process with third party frameworks.
categories: unity cocoapods iOS
---

One of the things I’ve always found most annoying in Unity3D was the tricky and error-prone build process for the iOS platform, specially if you need to integrate third-party frameworks and localized resources onto your Xcode project.Hopefully, a more streamlined process can be achieved by using CocoaPods, some bash scripts and C# helpers. In this blog post I’ll try to explain the steps I performed in a Unity3D project I’m working on. It’s worth mentioning that these steps were tested with [Unity 4.6.6p2](http://unity3d.com/unity/qa/patch-releases/4.6.6p2) and may not work depending on the Unity3D version you’re using because [Unity3D patches](http://unity3d.com/unity/qa/patch-releases) and upgrades are [a lot buggy](http://forum.unity3d.com/threads/unity-5-parse-ios-nsurlerrordomain-error-1012.308569/).### 1. Install required software ###-	[Unity 4.6.6p2](http://unity3d.com/unity/qa/patch-releases/4.6.6p2)-	[CocoaPods](http://guides.cocoapods.org/using/getting-started.html)-	[Xcode 6.3.2](https://itunes.apple.com/br/app/xcode/id497799835?mt=12)

### 2. Download external resources ###

- [XcodeAPI.zip](/posts-files/2015-06-15/XcodeAPI.zip): extract it and move all files into Unity\`s `/Assets/Editor/` folder. This contains the [Xcode Manipulation API](https://bitbucket.org/Unity-Technologies/xcodeapi/overview) and my `XcodePostprocess.cs` script, which will be explained later.
- [XCodeFiles.zip](/posts-files/2015-06-15/XCodeFiles.zip): extract it and move the `XCodeFiles` folder into the Unity project root folder. The final folder structure must look like this:

        ...
        ProjectSettings
        Assets
        XCodeFiles/
        ├── AppController
        │   └── UnityAppController.mm
        ├── Pod
        │   ├── Podfile
        │   ├── open_pods.command
        │   └── pods.command
        └── Strings (OPTIONAL)
            ├── en.lproj
            │   └── InfoPlist.strings
            ├── es.lproj
            │   └── InfoPlist.strings
            └── pt.lproj
                └── InfoPlist.strings
        ...### 3. PostProcessBuild inside Unity3D ###The first part of automating iOS builds is performed right inside Unity3D, let's take a deeper look at the contents of the `XcodePostprocess.cs` file:

#### Setting Xcode Build Settings properties ####

In order to the generated Xcode project gracefully work with CocoaPods we’ll need to set some Xcode Build Settings on the target project. This will avoid linker errors on the final project.

```C#
// 1. CocoaPods support.
proj.AddBuildProperty (target, "HEADER_SEARCH_PATHS", "$(inherited)");
proj.AddBuildProperty (target, "FRAMEWORK_SEARCH_PATHS", "$(inherited)");
proj.AddBuildProperty (target, "OTHER_CFLAGS", "$(inherited)");
proj.AddBuildProperty (target, "OTHER_LDFLAGS", "$(inherited)");

```
#### Optional 64-bit support ####
Handling optional 64-bit support is important because:

- [Apple required 64-bit support starting from June 1, 2015](https://developer.apple.com/news/?id=04082015a).
- It comes in handy to use debug/develop builds without 64-bit support because they are a lot faster.

```C#
// 2. Optional 64-bit support
var arch = (iPhoneArchitecture) PlayerSettings.GetPropertyInt ("Architecture", BuildTargetGroup.iPhone);
if (arch == iPhoneArchitecture.ARM64)
    proj.SetBuildProperty (target, "ARCHS", "$(ARCHS_STANDARD)");
else
    UnityEngine.Debug.LogWarning (String.Format ("Current architecture is '{0}', please use '{1}' for release builds.", arch, iPhoneArchitecture.ARM64));```#### Moving the final UnityAppController.mm file ####In case your `UnityAppController.mm` needs to be modified to initialize third-party frameworks (i.e.: set API keys, callbacks, etc) you can just edit it once and then grab a copy of it on `<Unity project root folder>/XCodeFiles/AppController/UnityAppController.mm`. The following code snippet will copy the edited file onto the final Xcode project, replacing the default one created by Unity3D.
```C#
// 3. Replace the final AppController file.
foreach (string appFilePath in AppControllerFilePaths)
    CopyAndReplaceFile (appFilePath, Path.Combine (Path.Combine (path, "Classes/"), Path.GetFileName (appFilePath)));
```#### Moving Pod files ####

The following code snipped simply copies the contents of `<Unity project root folder>/XCodeFiles/Pod/` folder into the final Xcode project, which will be used by CocoaPods later to install the project's dependencies.

```C#
// 4. Include Podfile into the project root folder.
foreach (string podFilePath in PodFilePaths)
    CopyAndReplaceFile (podFilePath, Path.Combine (path, Path.GetFileName (podFilePath)));
```

#### Moving InfoPlist.strings files ####

Not much to be said here, the following code simply copies the contents of `<Unity project root folder>/XCodeFiles/Strings/` folder into the final Xcode project, which will be used later to localize the app name into different languages.

```C#
// 5. Include localized 'InfoPlist.strings' into the project root folder.
foreach (string locStringFolderName in LocalizedStringsFolderNames)
    CopyDirectory (Path.Combine (StringsFolderPath, locStringFolderName), Path.Combine (path, locStringFolderName));
```

### 4. CocoaPods ###

[CocoaPods](https://cocoapods.org) is a great tool for integrating iOS project dependencies. My project uses a few third-party libraries and the `Podfile` looks like this, make sure to adapt yours according to your project's needs.

```
source 'https://github.com/CocoaPods/Specs.git'

platform :ios, '6.0'
pod 'Google-Mobile-Ads-SDK', '~> 7.0'
pod 'GoogleConversionTracking'
pod 'GoogleAnalytics-iOS-SDK'
pod 'Parse'
```

A good approach is to have a `pods.command` file in the same directory as the `Podfile` specification. This allows you to start the CocoaPods installation simply by double-clicking on `pods.command` in Finder.

```
#!/bin/bash
cd "`dirname "$0"`"
pod install
open "./Unity-iPhone.xcworkspace"
```

And that's all. Once Unity3D finishes building your iOS project you can just jump into the Xcode project root and double-click `pods.command` to start CocoaPods. As soon as CocoaPods completes the integration it'll open Xcode so that you can continue editing your project or submit it to the App Store.

*NOTE: It's possible to invoke `pods.command` programmatically from C# using `System.Diagnostics.Process` API. In order to do this please uncomment `StartPodsProcess()` method call from `XcodePostprocess.cs`. This approach, however, seems to screw up the Xcode project for some reason and that's why I'd rather invoke `pods.command` manually*.

### 5. InfoPlist.strings localization ###

In case you have your project localized to more than one language (in my case  English, Portuguese and Spanish) it's often required that you display your application bundle display name.

The easiest way I have found so far is to drag the localization files that were copied into the Xcode root folder (`en.lproj/InfoPlist.strings`, `es.lproj/InfoPlist.strings` and `pt.lproj/InfoPlist.strings`) from Finder into the Xcode Project navigator - one at a time. Xcode will detect the localization on the target files and add support for the required languages automatically.

## Conclusion ##

Unity3D provides very limited support for automated builds on iOS platform, although it is still a tricky process it can be partially performed with the help of CocoaPods, some bash scripts and C# helpers. The approach described above could still be improved but has proven to save me lot of time and effort when building my Unity3D iOS projects. Nonetheless, CocoaPods makes it easy to have third-party libraries always up-to-date in your project.

If you have any issues or suggestions please feel free to [contact me](mailto://dev@educoelho.com).