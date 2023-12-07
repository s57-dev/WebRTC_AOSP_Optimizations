## Setting WebView/Chromium command line flags

[Various command line flags](https://peter.sh/experiments/chromium-command-line-switches/) can be configured using the **/data/local/tmp/webview-command-line** file on the system. Changes in the command line flags require restarting of the WebView process.



### WebRTC CPU allocation

Notice: the **--webrtc-max-cpu-consumption-percentage** CLI flag was [removed](https://chromium.googlesource.com/chromium/src/+/24bc3bcdf90fa46398b2e2887537c2de0872c693) from chromium in v115 release. As of the time of writing this document, the maximum CPU consumption is set to [50%](https://chat.openai.com/URL), and this setting no longer applies to Android devices.



## Background task management

CPU heavy operations like periodic log collections and upload tasks should always be placed on the AOSP background CPU task set. 

Shell scripts should place their PID into the background group:

```bash
#!/system/bin/sh
echo $$ > /dev/cpuset/background/tasks
# All child processes will inherit the background cpu set assigment
```


For C/C++ based programs use the CPU set Kernel API.



## VP9 performance/quality configuration

On slower ARM devices, it can be considered lowering the VP9 speed configuration and disabling denoising .  Default config variables of flags.settings_by_resolution can be changed in libvpx_vp9_encoder.cc under the third_party/webrtc  project.

Example:

```c++
LibvpxVp9Encoder::GetDefaultPerformanceFlags() {
  PerformanceFlags flags;
  flags.use_per_layer_speed = true;
#if defined(WEBRTC_ARCH_ARM) || defined(WEBRTC_ARCH_ARM64) || defined(ANDROID)
  // Speed 8 on all layers for all resolutions.
  flags.settings_by_resolution[0] = {.base_layer_speed = 9,
                                     .high_layer_speed = 9,
                                     .deblock_mode = 0,
                                     .allow_denoising = false};
#else
  ...
```



Checking for software decoder fallback issues

## Benchmarking performance improvements

Depending on the application configuration, WebRTC may employ different strategies dealing with resource shortages. This includes lack of CPU resources to perform encoding/decoding and network bandwidth limitation issues. If encode/decode performance fails below a target frame rate, WebRTC may decrease frame rate or resolution of the stream. To benchmark at a fixed resolution/frame rate configuration for arbitrary WebRTC enabled application, it may be desired to temporarily disable the frame rate/resolution step down functionalities.

For example, in order to disable the resolution down step function for MAINTAIN_FRAMRATE strategy,   the *GetAdadaptationDownStep* function can be modified to return *Adaptation::Status::kAdaptationDisabled*  for degradation preference set to MAINTAIN_FRAMERATE.



## Running standalone vp9 performance test on Android



Get the libvpx9 project

```
$ mkdir ~/vpx
$ cd ~/vpx
$ git clone https://github.com/webmproject/libvpx
```



Ensure that the `ndk-build` binary from the Android NDK is in the PATH environment variable and then build the performance tests

```bash
$ ./libvpx/configure --target=arm64-android-gcc --enable-external-build \
                                   --enable-postproc --disable-install-srcs --enable-multi-res-encoding \
                                   --disable-temporal-denoising --disable-runtime-cpu-detect --enable-encode_perf_tests
$ NDK_PROJECT_PATH=. ndk-build \
        APP_BUILD_SCRIPT=./libvpx/test/android/Android.mk \
        APP_ABI=arm64-v8a APP_OPTIM=release \
        APP_STL=c++_static -j12 V=1 \
        APP_CFLAGS="-O3 -Rpass-missed=loop-vectorize  -Rpass-analysis=loop-vectorize"
```


Flags  **-Rpass-missed=loop-vectorize  -Rpass-analysis=loop-vectorize** will provide additional logs on compiler loop vectorization issues. For generating a runtime profile, you may consider adding **-fprofile-generate**.

Using adb, copy the libvpx performance tests to the target device:

```bash
adb shell mdkir /data/test/
adb push desktop_640_360_30.yuv /data/test/
adb push kirland_640_480_30.yuv /data/test/
adb push macmarcomoving_640_480_30.yuv /data/test/
adb push macmarcostationary_640_480_30.yuv /data/test/
adb push niklas_1280_720_30.yuv /data/test/
adb push niklas_640_480_30.yuv /data/test/
adb push tacomanarrows_640_480_30.yuv /data/test/
adb push tacomasmallcameramovement_640_480_30.yuv /data/test/
adb push thaloundeskmtg_640_480_30.yuv /data/test/
```

After pushing the test binary to the device, execute the test suite

```bash
adb push libs/arm64-v8a/vpx_test /data/test/
adb shell
(adb) $ cd /data/test
(adb) $ nice -n -19 ./vpx_test --gtest_filter=VP9/VP9EncodePerfTest*
```



## Validating HW Encoder and Decoder support



In case of any issues with the hardware codec, the *VideoDecoderSoftwareFallbackWrapper* in WebRTC will trigger a fallback to software encoder/decoder. Since the capabilities of ARM devices heavily depend on the media codecs provided by the SoC vendors, it may be required to apply SoC specific modifications to Chromium. For instance, a typical scenario is the VP9 decoder falling back to software encoding if VP9 spatial layer support of the HW codec is not correctly configured and exposed to Chromium (See **Vp9HwSupportForSpatialLayers()*  in `rtc_video_decoder_adapter.cc`).



## Using custom SoC-specific toolchains

Some SoCs, like Qualcomm, offer custom LLVM toolchains that include additional optimizations for their implementation of the ARM architecture.  To utilize these toolchains, the Chromium gn arguments should contain:

```
clang_base_path = "/opt/qcom/Qualcomm_Snapdragon_LLVM_ARM_Toolchain_OEM/17.0.0.0/"
clang_version = "17"
clang_use_chrome_plugins = false
```

Since building Chromium for ARM64  depends on building a set of host **x64** binaries (64bit Linux Host), the target toolchain must supports the **x86_64-linux-gnu** target and **msse3** compilation flags.  Even a toolchain could not be used to fully build the Chromium WebView, building specific performance tests could be beneficial in identifying hot areas for potential optimizations.

