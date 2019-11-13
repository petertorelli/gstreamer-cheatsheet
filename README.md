# Introduction

A list of GStreamer commands I've found useful.

# macOS

OS Version: 10.14.6

~~~
% gcc --version
Configured with: --prefix=/Applications/Xcode.app/Contents/Developer/usr --with-gxx-include-dir=/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX10.14.sdk/usr/include/c++/4.2.1
Apple LLVM version 10.0.1 (clang-1001.0.46.4)
Target: x86_64-apple-darwin18.7.0
Thread model: posix
InstalledDir: /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin
% uname -a
Darwin aqualung 18.7.0 Darwin Kernel Version 18.7.0: Thu Jun 20 18:42:21 PDT 2019; root:xnu-4903.270.47~4/RELEASE_X86_64 x86_64 i386 MacBookPro11,2 Darwin
~~~

## Compiling and installing

Compiling was unsuccessful for some very strange reasons. Even installation via MacPorts and Brew provided a mixed-back of 

Attempt #1 : Runtime frameworks from the website. THese are just frameworks and do not have the `gst-*` command-line functions.

Attempt #2 : Compiling from the source repo's that use GNU Autotools. All of the gst-plugins-base were blacklisted. Upon inspection the gst-inspect-1.0 ximagesink reported "not a GStreamer plugin".

Attempt #3 : Compile the Git gst-build meson/ninja release. Similar issues, except the error indicated that ximagesink was not compiled for this architecture.

Attempt #4 : MacPorts. This worked except it was missing the `applemedia` AVF video tools which are in gst-plugins-bad. `autovideo{src,sink}` are unreleable and yielded "error(4)".

Attempt #5 : Brew. Brew provided the AVF bad plugins, but no X plugins (ximagesink). Fortunately it has `gtksink` which is much more stable than `glimagesink` although neither can be resized by caps.

## Pipelines

A lot of trial and error here. First wave of debug was determining if the plugins were present. Second wave was determining which worked if they were present (even `base` plugins had issues on different distros). Last debug involved cap weirdness, e.g., one MPEG codec expected height and width propeties as strings, not integers. Moral of the story: when in doubt, use `-v` and copy the entire caps string.

### Use a webcam locally

Since `autovideosrc` didn't work under any circumstance, I had to find a distribution that included the `applemedia` plugins, specifically `avfvideosrc`. This source allows camera selection by integer. Still not sure how to do a query of all available cameras (not sure you can access `readable` caps with `gst-inspect`). Again, `glimagesink` could be used, but it is not resizable. Too bad `ximagesink` isn't present in Brew.

~~~
% gst-launch-1.0 -v avfvideosrc device-index=1 ! gtksink
~~~

### Mix multiple webcams into a mosaic

Here I combine three webcams and an MPEG4 playback (loop didn't work), into one mosiac. I needed to use `sync=false` 
to prevent the "[you're droppping a lot of frames or your computer is too slow]". Not sure the `queue` elements are necessary. The size caps going into `gtksink` have no impact on the final window size, but they are necessary to properly place the incoming images. Also, each source image caps must match the camera aspect ratio or it fails to negotiate.

~~~
% gst-launch-1.0 \
    avfvideosrc device-index=0 ! video/x-raw,width=320,height=240 ! queue ! m.sink_0 \
    avfvideosrc device-index=1 ! video/x-raw,width=320,height=240 ! queue ! m.sink_1 \
    avfvideosrc device-index=2 ! video/x-raw,width=320,height=240 ! queue ! m.sink_2 \
    multifilesrc location=file_example_MP4_1920_18MG.mp4 loop=true ! decodebin ! videoconvert ! videoscale ! video/x-raw,width=320,height=240 ! queue ! m.sink_3 \
    videomixer name=m \
        sink_0::xpos=320 \
        sink_3::ypos=240 \
        sink_2::ypos=240 \
        sink_2::xpos=320 \
    ! video/x-raw,width=640,height=480 ! gtksink sync=false
~~~

### Send a webcam over UDP and decode on the same host

Hoo boy this took a long time to figure out. Ultimately I had to copy the giant cap string for all the elements and remove pieces until I understood (kinda) what was going on.

First the server. This is the easiest part. There's no `h264` or `x264` codecs you'll see in online forums in the latest Brew or MacPorts GStreamer distros (1.16.1, btw). So instead I used `gst-libav`. Payloading for UDP was new to me, too.

We need the `videoconvert` since `avenc_mpeg4` expects `video/x-raw,format=I420` and `avfvideosrc` only outputs UYVY, NV12, YUY2 and BGRA.

~~~
% gst-launch-1.0 -v \
    avfvideosrc device-index=1 ! \
    queue ! \
    videoconvert ! \
    avenc_mpeg4 ! \
    rtpmp4vpay ! \
    udpsink host=127.0.0.1 port=5000
~~~

The client took a while to figure out since a lot of the caps are not automatically transmitted in the binary stream, which seems odd. I thought MPEG I-frames indicated the native resolution. /shrugs/ It is important to use `-v` and verify the frameates are both the same. Originally I had copied the caps from the server command to every element of the client command, and then started pulling them out until it failed.

~~~
% gst-launch-1.0 -v \
    udpsrc address=127.0.0.1 port=5000 caps='application/x-rtp' ! \
    rtpmp4vdepay ! \
    "video/mpeg, width=(int)800, height=(int)600,framerate=20/1" ! \
    avdec_mpeg4 ! \
    videoconvert ! \
    gtksink
~~~
