# gst-plugin-pylon

> The official GStreamer source plug-in for Basler cameras powered by Basler pylon Camera Software Suite


This plugin allows to use any Basler 2D camera (supported by Basler pylon Camera Software Suite) as source element in a GStreamer pipeline.

All camera features are available in the plugin by dynamic runtime mapping to gstreamer properties.

**Please Note:**
This project is offered with no technical support by Basler AG.
You are welcome to post any questions or issues on [GitHub](https://github.com/basler/gst-plugin-pylon).

[![CI](https://github.com/basler/gst-plugin-pylon/actions/workflows/ci.yml/badge.svg)](https://github.com/basler/gst-plugin-pylon/actions/workflows/ci.yml)

The next chapters describe how to [use](#getting-started) and [build](#Building) the *pylonsrc* plugin.

# Getting started 

To display the video stream of a single Basler camera is as simple as:

`gst-launch-1.0 pylonsrc ! videoconvert ! autovideosink`

The following sections describe how to select and configure the camera.

## Camera selection
If only a single camera is connected to the system, `pylonsrc` will use this camera without any further actions required.

If more than one camera is connected, you have to select the camera.
Cameras can be selected via index, serial-number or device-user-name

If no selection is given `pylonsrc` will show an error message which will list the available cameras

```bash
gst-launch-1.0 pylonsrc ! fakesink
Setting pipeline to PAUSED ...
ERROR: Pipeline doesn't want to pause.
ERROR: from element /GstPipeline:pipeline0/GstPylonSrc:pylonsrc0: Failed to start camera.
Additional debug info:
../ext/pylon/gstpylonsrc.c(524): gst_pylon_src_start (): /GstPipeline:pipeline0/GstPylonSrc:pylonsrc0:
At least 4 devices match the specified criteria, use "device-index" to select one from the following list:
[0]:  21656705
[1]:  0815-0000
[2]:  0815-0001
[3]:  0815-0002
```
### examples
select second camera from list:

` gst-launch-1.0 pylonsrc device-index=1 ! videoconvert ! autovideosink`

select camera with serial number `21656705`

` gst-launch-1.0 pylonsrc device-serial-number='21656705' ! videoconvert ! autovideosink`

select camera with user name `top-left`

` gst-launch-1.0 pylonsrc device-user-name='top-left' ! videoconvert ! autovideosink`

## Configuring the camera

The configuration of the camera is defined by
* gstreamer pipeline capabilities
* the active UserSet or supplied PFS file
* feature properties of the plugin 

### Capabilities

The pixel format, image width, image height and acquisition framerate ( FPS ) are set during capability negotiation.


#### example

configuring to 640x480 @ 10fps in Mono8 format:

`gst-launch-1.0 pylonsrc ! "video/x-raw,width=640,height=480,framerate=10/1,format=GRAY8"  ! videoconvert ! autovideosink`

#### Pixel format definitions

For the pixel format check the format supported for your camera on https://docs.baslerweb.com/pixel-format

The mapping of the camera pixel format names to the gstreamer format names is:

```
|Pylon              | GSTREAMER  |
|-------------------|------------|
| Mono8             |  GRAY8     |
| RGB8Packed        |  RGB       |
| RGB8              |  RGB       |
| BGR8Packed        |  BGR       |
| BGR8              |  BGR       |
| YCbCr422_8        |  YUY2      |
| YUV422_8          |  YUY2      |
| YUV422_YUYV_Packed|  YUY2 
| YUV422_8_UYVY     |  UYVY      |
| YUV422Packed      |  UYVY      |


```

### Fixation 

If two pipeline elements don't specify which capabilities to choose, a fixation step gets applied.

This could result in an unexpected pixel format or width/height.

It is recommended to set a caps-filter to explicitly set the wanted capabilities.

### UserSet handling

`pylonsrc` always loads a UserSet of the camera before applying any further properties. 

This feature is controlled by the enumeration property `user-set`.

If this property is not set, or set to the value `Auto`, the power-on UserSet gets loaded. How to select another power-on default is documented for [UserSetDefault](https://docs.baslerweb.com/user-sets#choosing-a-startup-set) feature.

To select dynamically another UserSet the `user-set` property accepts any other implemented UserSet of the camera.

e.g to activate the `HighGain` UserSet on a Basler ace camera:

```
gst-launch-1.0 pylonsrc user-set=HighGain ! videoconvert ! autovideosink
```

Overall UserSets topics for the camera are documented in [Chapter UserSets](https://docs.baslerweb.com/user-sets) in the Basler product documentation.

### PFS file

`pylonsrc` can load a custom configuration from a PFS file. A PFS file is a file with previously saved settings of camera features.

This feature is controlled by the property `pfs-location`.

To use a PFS file, specify the filepath using the `pfs-location` property.

e.g to use a PFS file with name `example-file-name.pfs` on a Basler ace camera:

```
gst-launch-1.0 pylonsrc pfs-location=example-file-name.pfs ! videoconvert ! autovideosink
```

**Important:** Using this property will result in overriding the camera features set by the `user-set` property if also specified.

An example on how to generate PFS files using pylon Viewer is documented in [Chapter Overview of the pylon Viewer](https://docs.baslerweb.com/overview-of-the-pylon-viewer#camera-menu) in the Basler product documentation.

### Features

After applying the UserSet and the gstreamer properties any other camera feature gets applied.

The `pylonsrc` plugin dynamically exposes all writable features of the camera as  gstreamer [child properties](https://gstreamer.freedesktop.org/documentation/gstreamer/gstchildproxy.html?gi-language=c) with the prefix `cam::`.

Pylon features like `ExposureTime` or `Gain` are mapped to the gstreamer properties `cam::ExposureTime` and `cam::Gain`.

Examples to set Exposuretime to 2000µs and Gain to 10.3dB

`gst-launch-1.0 pylonsrc cam::ExposureTime=2000 cam::Gain=10.3 ! videoconvert ! autovideosink`

All available features can be listed by by calling

`gst-inspect-1.0 pylonsrc`

### Selected Features

Some of the camera features are not directly available but have to be selected first.

As an example the feature `TriggerSource` has to be selected by `TriggerSelector` first. ( see [TriggerSource](https://docs.baslerweb.com/trigger-source) in the Basler product documentation)

These two steps ( select and access ) are mapped to `pylonsrc` properties as a single step with this pattern

`cam::<featurename>-<selectorvalue>`

For the above  `TriggerSource` example if  the `TriggerSource` of the trigger `FrameStart` has to be configured the property is:

`cam::TriggerSource-FrameStart`

Example:
Configure a hardware trigger on Line1 for the trigger FrameStart:

`gst-launch-1.0 pylonsrc cam::TriggerSource-FrameStart=Line1 cam::TriggerMode-FrameStart=On ! videoconvert ! autovideosink`



# Building

This plugin is build using the [meson](https://mesonbuild.com/) build system. The meson version has to be >= 0.61.

As a first step install Basler pylon according to your platform. Downloads are available at: [Basler software downloads](https://www.baslerweb.com/en/downloads/software-downloads/#type=pylonsoftware;language=all;version=all)

The supported pylon versions on the different platforms are:


|                 | 7.1  | 6.2  |
|-----------------|:----:|:----:|
| Windows x86_64  |   x  |      |
| Linux x86_64    |   x  |      |
| Linux aarch64   |   x  |   x  |
| macOS x86_64    |   -  |      |


> macOS build not available for now due to current meson/cmake interaction issues

Installing Basler pylon SDK will also install the Basler pylon viewer. You should use this tool to verify, that the cameras work properly in your system and to learn about the their features.

The differences in the build steps for [Linux](#Linux), [Windows](#Windows) and [macos](#macOS) are described in the next sections.

## Linux
Make sure the dependencies are properly installed. In Debian-based
systems you can run the following commands:

```bash
# Meson build system.
# Remove older meson from APT and install newer PIP version
sudo apt remove meson
sudo -H python3 -m pip install meson

# GStreamer
sudo apt install libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev cmake
```

The build process relies on `PYLON_ROOT` pointing to the Basler pylon install directory.

```bash
# for pylon in default location
export PYLON_ROOT=/opt/pylon
```

Then proceed to configure the project. Check `meson_options.txt` for a
list of configuration options. On Debian-based systems, make sure you
configure the project as:

```bash
meson builddir --prefix /usr/
```

Build, test and install the project:

```bash
# Build
ninja -C builddir

# Test
ninja -C builddir test

# Install
sudo ninja -C builddir install
```

Finally, test for proper installation:

```bash
gst-inspect-1.0 pylonsrc
```

### Maintainer configuration

If you are a maintainer or plan to extend the plug-in, we recommend
the following configuration:

```
meson builddir --prefix /usr/ --werror --buildtype=debug -Dgobject-cast-checks=enabled -Dglib-asserts=enabled -Dglib-checks=enabled
```

### Cross compilation for linux targets
#### NVIDIA Jetson Jetpack
TBD

#### YOCTO recipe
TBD
A yocto recipe will be provided in  [meta-basler-tools](https://github.com/basler/meta-basler-tools) layer on github.

## Windows
The following commands should be run from a *Visual Studio command prompt*.

Install the gstreamer runtime **and** development packages ( using the files for MSVC 64-bit (VS 2019, Release CRT )

[https://gstreamer.freedesktop.org/download/](https://gstreamer.freedesktop.org/download/)

Specify the path to pkgconfig configuration files for GStreamer and the pkg-config binary ( shipped as part of gstreamer development )

```bash
set PKG_CONFIG_PATH=%GSTREAMER_1_0_ROOT_MSVC_X86_64%lib\pkgconfig
set PATH=%PATH%;%GSTREAMER_1_0_ROOT_MSVC_X86_64%\bin
```

The build process relies on CMAKE_PREFIX_PATH pointing to Basler pylon cmake support files. This is normally set by the Basler pylon installer.
```bash
set CMAKE_PREFIX_PATH=C:\Program Files\Basler\pylon 7\Development\CMake\pylon\
```


Then the plugin can be compiled and installed using Ninja:

```
meson build --prefix=%GSTREAMER_1_0_ROOT_MSVC_X86_64%
ninja -C build
```

Or made available as a Visual Studio project:

```
meson build --prefix=%GSTREAMER_1_0_ROOT_MSVC_X86_64% --backend=vs2022
```

This will generate a Visual Studio solution in the build directory

To test without install:

```
set GST_PLUGIN_PATH=build/ext/pylon  

%GSTREAMER_1_0_ROOT_MSVC_X86_64%\bin\gst-launch-1.0.exe 

pylonsrc ! videoconvert ! autovideosink  
```

To install into main gstreamer directory

```
ninja -C build install
```

>Workaround for running from normal cmdshell in a non US locale: 
>As visual studio currently ships a broken vswhere.exe tool, that meson relies on.  
Install a current vswhere.exe from [Releases · microsoft/vswhere · GitHub](https://github.com/microsoft/vswhere/releases) that fixes ( [Initialize console after parsing arguments by heaths · Pull Request #263 · microsoft/vswhere · GitHub](https://github.com/microsoft/vswhere/pull/263) )


Finally, test for proper installation:

```bash
gst-inspect-1.0 pylonsrc
```

## macOS
Installation on macOS is currently not supported due to conflicts between meson and underlying cmake in the configuration phase.

This target will be integrated after a Basler pylon 7.x release for macOS

# Known issues
* typos and unsupported feature names are silently ignored in this version


 
