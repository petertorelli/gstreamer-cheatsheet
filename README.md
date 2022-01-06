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

Compiling was unsuccessful for some very strange reasons. Even installation via MacPorts and Brew provided a mixed-bag of issues. 

Attempt #1 : Runtime frameworks from the website. THese are just frameworks and do not have the `gst-*` command-line functions.

Attempt #2 : Compiling from the source repo's that use GNU Autotools. All of the gst-plugins-base were blacklisted. Upon inspection the gst-inspect-1.0 ximagesink reported "not a GStreamer plugin".

Attempt #3 : Compile the Git gst-build meson/ninja release. Similar issues, except the error indicated that ximagesink was not compiled for this architecture.

**NOTE** (2019-12-01) I rebuilt 1.17.0 of everything and it is now working fine on macOS. Just remember to NOT INSTALL ANYTHING!!! Build to a local `--prefix` and setup PATH and PKG_CONFIG_PATH becuase you'll probably find something else breaks and need to build again. Which means I am no longer using macports or brew, I'm using my own local version.

Attempt #4 : MacPorts. This worked except it was missing the `applemedia` AVF video tools which are in gst-plugins-bad. `autovideo{src,sink}` are unreleable and yielded "error(4)".

Attempt #5 : Brew. Brew provided the AVF bad plugins, but no X plugins (ximagesink). Fortunately it has `gtksink` which is much more stable than `glimagesink` although neither can be resized by caps.

TODO: Go back and include the actual error messages for the sake of people web-searching.

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

### Send a webcam over UDP and decode

Hoo boy this took a long time to figure out. Ultimately I had to copy the giant cap string for all the elements and remove pieces until I understood (kinda) what was going on.

First, the server. This is the easiest part. There's no `h264` or `x264` codecs you'll see in online forums in the latest Brew or MacPorts GStreamer distros (1.16.1, btw). So instead I used `gst-libav`. Payloading for UDP was new to me, too.

Next, we need the `videoconvert` since `avenc_mpeg4` expects `video/x-raw,format=I420` and `avfvideosrc` only outputs UYVY, NV12, YUY2 and BGRA. The `queue` should be limited to avoid lag, hence the max setting.

Lastly, keep in mind that `host` is where the packets are being *sent*. This isn't TCP, it's a UDP push to the host, which is the, er, client. If you have a firewall you'll need to open port 5000. Like the good old days of internet gaming. I did an experiment using the `tcpserver{sink,src}` and it had up to a 3-second delay, which may have been related to not payloading the data. I've read UDP sacrifices quality for low-latency since the packets have a maximum size that is much smaller than TCP. Need moar dat-uh.

~~~
% gst-launch-1.0 -v \
    avfvideosrc device-index=1 ! \
    queue max-size-buffers=1 ! \
    videoconvert ! \
    videoscale ! \
    video/mpeg,width=640,height=480,framerate=20/1 ! \
    avenc_mpeg4 ! \
    rtpmp4vpay ! \
    udpsink host=127.0.0.1 port=5000
~~~

The client took a while to figure out since a lot of the caps are not automatically transmitted in the binary stream, which seems odd. I thought MPEG I-frames indicated the native resolution. /shrugs/ It is important to use `-v` and verify the frame rates are both the same. Originally I had copied the caps from the server command to every element of the client command, and then started pulling them out until it failed.

~~~
% gst-launch-1.0 -v \
    udpsrc port=5000 caps='application/x-rtp' ! \
    rtpmp4vdepay ! \
    video/mpeg,width=640,height=480,framerate=20/1 ! \
    avdec_mpeg4 ! \
    videoconvert ! \
    gtksink
~~~

If teeing to a file, don't forget `async=false`:

~~~
gst-launch-1.0 -v -e \
  v4l2src ! \
  videoconvert ! \
  videoscale ! \
  video/x-raw,width=640,height=480,framerate=20/1 ! \
  tee name=t ! \
    queue ! \
    avenc_mpeg4 ! \
    rtpmp4vpay ! \
    udpsink host=127.0.0.1 port=5000 \
  t. ! \
    queue ! \
    x264enc ! \
    mp4mux ! \
    filesink location=video.mp4 async=false
~~~

## RTSP to File for CCTV Pro cameras

`h265parse` is required so that `mp4mux` knows video format.

~~~
gst-launch-1.0 -e -v \
    rtspsrc location="rtsp://USER:PASSWORD@HOST:PORT/cam/realmonitor?channel=1&subtype=0" ! \
    rtph265depay ! \
    h265parse ! \
    mp4mux ! \
    filesink location=~/video.mp4
~~~

# NVIDIA Jetpack Headless RTSP to macOS XQuartz

NVIDIA Xavier doesn't like running with a remote X host. I tried VNC and xhosting to my mac, and I constantly run into:

~~~
nvbuf_utils: Could not get EGL display connection
~~~

To which there is no solution on internet (just lots people arguing about what to set DISPLAY to .. sigh). Meanwhile, everything runs fine when I connect a display to the Xavier. My X-debug-fu is pretty rusty ... I haven't programmed in X since 1993.

For now I'm logging in with an X tunnel via ssh, `ssh -X`, after adding this to `/etc/ssh/ssh_config`:

~~~
    XAuthLocation /usr/X11/bin/xauth
    ServerAliveInterval 60
    ForwardX11Timeout 596h
~~~

My GStreamer hacky workaround is to set up the CCTV cameras and the Xavier on their own subnet, and then process the RTSP streams on the Xavier and dump them to a UDP port. This can be enabled or disabled by a cloud connection to the Xavier for the cloud video server. But here is the pipieline for future self-reference:

Source (standard RTSP port 554, UDP 5000)
~~~
gst-launch-1.0 -e rtspsrc 'location=rtsp://USERNAME:PASSWORD@CAMERAIPADDRESS:554' latency=300 ! udpsink host=TARGET port=5000
~~~

Sink:
~~~
gst-launch-1.0 -v udpsrc port=5000 caps='application/x-rtp' ! rtph264depay ! avdec_h264 ! videoconvert ! gtksink
~~~

I use a separate router for a subnet so I'm not trashing my work network performance.

Now I can use NVIDIA inference gstreamer plugins on the Xavier as a smart closed-circuit CCTV server.


## Thoughts

Do you ever get the feeling that no one really understands GStreamer and we're just throwing around a bunch of ! videoconvert ! and ! video/x-raw ! elements until stuff works? Just me? Ok then...
