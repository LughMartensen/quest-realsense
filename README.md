# Meta Quest 3 - Intel RealSense Integration
Integration of Intel RealSense depth cameras with the Meta Quest 3 standalone VR headset using Unity.


## Foreword

<br/>

This project consolidates the knowledge gained while attempting to power & operate an arbitrary RealSense camera directly from a Meta Quest 3 VR headset.

The solution is based on an [existing repository](https://github.com/GeorgeAdamon/quest-realsense) that makes an Intel Realsense D435i compatible with an Oculus Quest (1).

Some adjustments had to made to the process to get everything to work, due to the newer operating system and Unity version.

Aim of this repository is to describe the steps necessary to achieve the Quest-RealSense integration inside Unity 2022.3.x, as well as provide a sample Unity project, showcasing the method.

<br/>

## Introduction

The project was tested in the following environments.

**Unity environment:**

| Label | Info  |
|----------------------------|-----------------|
| Unity Version              | 2022.3.42f1     |
| Graphics API               | OpenGLES3       |
| Minimum API Level          | 32 (Android 12) |
| Target API Level           | Automatic       |
| Graphics API               | OpenGLES3       |
| Scripting Backend          | Mono            |
| API Compatability Level    | .NET Framework  |
| Target Architecture Level  | ARMv7           |

<br/>

**RealSense Depth Camera Information:**

| Label  |  Info  |
|----------------------------|-----------------|
| Camera Model               | D405            |
| SDK Version                | 2.55.1          | 

<br/>

**Meta Quest Information:**

| Label  |  Info  |
|----------------------------|-----------------|
| Headset Model              | Quest3          |
| Headset Version            | 256550.6170.5 + | 
| Unity Package Version      | 1.36 +          | 

<br/>

**Compiler Information:**

| Label  |  Info  |
|----------------------------|-----------------|
| Gradle Version             | ?               |
| JRE Version                | ?               |

## Step 1: Project Setup (Windows)
### Prerequisites #1.1: Meta VR
This guide assumes that the necessary steps for setting-up the Unity development environment for the Meta Quest 3 have been completed, as described in the official Meta Quest documentation: [[1]](https://developers.meta.com/horizon/documentation/unity/unity-development-overview/) [[2]](https://unity.com/blog/engine-platform/get-started-developing-for-quest-3-with-unity) [[3]](https://medium.com/antaeus-ar/creating-a-mixed-reality-app-for-meta-quest-3-using-unity-b8cbc87a24e7).

Before proceeding to the next steps, the project should be able to produce a working Android .apk build, which runs on the Quest without issues.

### Prerequisites #1.2: Android App

Alternatively, you can create a simple 2D Android app with Unity that can run on your phone [[4]](https://docs.unity3d.com/Manual/android-getting-started.html). The generated .apk can be loaded to your Quest using the software \emph{sidequest} [[5]](https://side.quest/) [[6]](https://www.youtube.com/watch?v=ee_ltiOtHzo).

The advantage of this is that you do not need any external XR libraries that could cause shader problems or incompatibilites with certain android versions. This eliminates error sources and makes it easier to debug potential errors in the RealSense SDK. Additionaly, we can simply open the RealSense sample scenes without needing to add an XR Rig.

Before proceeding to the next steps, the project should be able to produce a working Android .apk build, which runs on the Quest without issues.

### Prerequisites #2: Intel RealSense wrappers
This guide assumes that the Intel RealSense Unity wrappers [**v2.55.1**](https://github.com/IntelRealSense/librealsense/releases/tag/v2.55.1) are imported succesfully in Unity 2022.3.x without errors. The reason for choosing this version is that at the time of writing (March 2025), this is the newest version which provides a unitypackage. However, it should also be possible to generate a unitypackage for a newer version.

To setup the project, download the Intel.RealSense.unitypackage file [**here**](https://github.com/IntelRealSense/librealsense/releases/tag/v2.55.1). In Unity, go to _**Assets > Import Package > Custom package**_ and select the file.

Before proceeding to the next steps, the project should be able to run one of Intel's provided example Scenes in Unity's Play Mode, provided a RealSense device is connected to an appropriate USB port of a Windows machine. More information in the official [realsense repository](https://github.com/IntelRealSense/librealsense/tree/master/wrappers/unity).

### Scripting Backend : Mono
Navigate to Unity's _**Project Settings > Player > Other Settings**_ and ensure that the Scripting Backend is set to **Mono**.
According to [this](https://github.com/IntelRealSense/librealsense/issues/4155#issuecomment-499363798) reply the RealSense library does not support Unity's IL2CPP library, at least at the time of writing this.

![](https://github.com/GeorgeAdamon/quest-realsense/blob/master/resources/img-scripting-backend.png)

## Step 2: Building the librealsense.aar Android library
### Build Process
In general, in order to allow a Unity project to access the RealSense cameras when targeting a platform other than Windows, the appropriate wrappers for this platform need to be built as Native Plugins first, and included in the Unity project.

In this case, because we are targeting Android (the OS of Oculus Quest) we will have to build the **librealsense.aar** plugin from the provided Android Java source code, based on the [official guidelines](https://github.com/IntelRealSense/librealsense/tree/master/wrappers/android). 

In my experience, building from the Windows Command Prompt as an Administrator, using the ```gradlew assembleRelease``` [command](https://github.com/IntelRealSense/librealsense/tree/master/wrappers/android#build-with-gradle) proved to be the most straightforward, less error-prone, way:

![](https://github.com/GeorgeAdamon/quest-realsense/blob/master/resources/img-gradle-build.png)

A succesful build process should take around 10 minutes on a decent machine, and look like this:
![](https://github.com/GeorgeAdamon/quest-realsense/blob/master/resources/img-gradle-build-02.png)
![](https://github.com/GeorgeAdamon/quest-realsense/blob/master/resources/img-gradle-build-04.png)

If the build is succesful, the generated .aar file will be located in 
```<librealsense_root_dir>/wrappers/android/librealsense/build/outputs/aar```.

### Unity Side
The generated **librealsense.aar** file should be placed inside your Unity project, in the _**Assets / RealSenseSDK2.0 / Plugins**_ directory, alongside the Intel.RealSense.dll and librealsense2.dll. A succesful setup should look like this:

![](https://github.com/GeorgeAdamon/quest-realsense/blob/master/resources/img-unity-plugins.png)

## Step 3: Initializing the RsContext Java class from Unity
> Note: A big shout-out to [**ogoshen**](https://github.com/ogoshen) for generously providing the solution of this next step!

Now that all the libraries are in place, before actually being able to access the RealSense camera, we need a C# script that performs two crucial jobs: 
* Initializes a new instance of the Java class [**RsContext**](https://github.com/IntelRealSense/librealsense/blob/master/wrappers/android/librealsense/src/main/java/com/intel/realsense/librealsense/RsContext.java)
* Makes sure that Android Camera Permissions are explicitly requested from the user, if not provided already.

Attaching the following script to any GameObject in your Scene, would ensure that those two operations are executed in the beginning of your application:

```c#
using UnityEngine;

public class AndroidPermissions : MonoBehaviour
{
#if UNITY_ANDROID && !UNITY_EDITOR
    void Awake()
    {
        if (!UnityEngine.Android.Permission.HasUserAuthorizedPermission(UnityEngine.Android.Permission.Camera))
        {
            UnityEngine.Android.Permission.RequestUserPermission(UnityEngine.Android.Permission.Camera);

        }

        using (var javaUnityPlayer = new AndroidJavaClass("com.unity3d.player.UnityPlayer"))
        using (var currentActivity = javaUnityPlayer.GetStatic<AndroidJavaObject>("currentActivity"))
        using (var rsContext = new AndroidJavaClass("com.intel.realsense.librealsense.RsContext"))
        {
            Debug.Log(rsContext);
            rsContext.CallStatic("init", currentActivity);
        }
    }
#endif
}
```


## Step 4: Using Quest-friendly shaders
As stated [in the original discussion](https://github.com/IntelRealSense/librealsense/issues/4155#issuecomment-522884739), if you are using any other XR mode apart from **Multi-Pass Stereo**, Geometry Shaders will not work on the Quest.

This means that if you try to load an example Unity project, such as the [PointCloudDepthAndColor](https://github.com/IntelRealSense/librealsense/blob/master/wrappers/unity/Assets/RealSenseSDK2.0/Scenes/Samples/PointCloudDepthAndColor.unity) scene from the Unity [samples](https://github.com/IntelRealSense/librealsense/tree/master/wrappers/unity/Assets/RealSenseSDK2.0/Scenes/Samples), where the _PointCloudMat_ material assigned to the _PointCloudRenderer_ component is using by default the [Custom/PointCloudGeom](https://github.com/IntelRealSense/librealsense/blob/master/wrappers/unity/Assets/RealSenseSDK2.0/Shaders/PointCloudGeom.shader) shader ( a geometry shader ), you will get an  
`OPENGL NATIVE PLUG-IN ERROR: GL_INVALID_OPERATION: Operation illegal in current state` error.

Switching the shader of this material to the simple [Custom/PointCloud](https://github.com/IntelRealSense/librealsense/blob/master/wrappers/unity/Assets/RealSenseSDK2.0/Shaders/PointCloud.shader) shader should work like a charm!  

Alternatively, you can switch your XR mode to Multi-Pass stereo.
