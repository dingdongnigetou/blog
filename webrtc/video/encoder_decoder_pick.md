# Webrtc视频编解码器的选择

## Demo
下面是官方demo创建视频编解码器的例子, 
```cpp
//examples/androidapp/src/org/appspot/apprtc/PeerConnectionClient.java
434     if (peerConnectionParameters.videoCodecHwAcceleration) {
435       encoderFactory = new DefaultVideoEncoderFactory(
436           rootEglBase.getEglBaseContext(), true /* enableIntelVp8Encoder */, enableH264HighProfile);
437       decoderFactory = new DefaultVideoDecoderFactory(rootEglBase.getEglBaseContext());
438     } else {
439       encoderFactory = new SoftwareVideoEncoderFactory();
440       decoderFactory = new SoftwareVideoDecoderFactory();
441     }
```
当我们开启硬件加速时便创建DefaultVideoXFactory，否则直接使用软件编解码器。这里的DefaultVideoXFactory内部又做了一些filter。

这里只分析encoder逻辑，至于decoder大同小异。

## DefaultVideoEncoderFactory
```cpp
//sdk/android/api/org/webrtc/DefaultVideoEncoderFactory.java
36   public VideoEncoder createEncoder(VideoCodecInfo info) {
37     final VideoEncoder softwareEncoder = softwareVideoEncoderFactory.createEncoder(info);
38     final VideoEncoder hardwareEncoder = hardwareVideoEncoderFactory.createEncoder(info);
39     if (hardwareEncoder != null && softwareEncoder != null) {
40       // Both hardware and software supported, wrap it in a software fallback
41       return new VideoEncoderFallback(
42           /* fallback= */ softwareEncoder, /* primary= */ hardwareEncoder);
43     }
44
45     return hardwareEncoder != null ? hardwareEncoder : softwareEncoder;
46   }
```
这里如果软硬都支持那么直接交由底层cpp的`fallback encoder`处理，我们先来看java层面的选择

## HardwareVideoEncoderFactory
```cpp
//sdk/android/api/org/webrtc/HardwareVideoEncoderFactory.java
182   private boolean isSupportedCodec(MediaCodecInfo info, VideoCodecMimeType type) {
183     if (!MediaCodecUtils.codecSupportsType(info, type)) {
184       return false;
185     }
186     // Check for a supported color format.
187     if (MediaCodecUtils.selectColorFormat(
188             MediaCodecUtils.ENCODER_COLOR_FORMATS, info.getCapabilitiesForType(type.mimeType()))
189         == null) {
190       return false;
191     }
192     return isHardwareSupportedInCurrentSdk(info, type) && isMediaCodecAllowed(info);
193   }
```
首先调用android的底层接口MediaCodec判断是否支持对应的硬件编码器，否则再继续过滤:
```cpp
197   private boolean isHardwareSupportedInCurrentSdk(MediaCodecInfo info, VideoCodecMimeType type) {
198     switch (type) {
199       case VP8:
200         return isHardwareSupportedInCurrentSdkVp8(info);
201       case VP9:
202         return isHardwareSupportedInCurrentSdkVp9(info);
203       case H264:
204         return isHardwareSupportedInCurrentSdkH264(info);
205     }
206     return false;
207   }
208
209   private boolean isHardwareSupportedInCurrentSdkVp8(MediaCodecInfo info) {
210     String name = info.getName();
211     // QCOM Vp8 encoder is supported in KITKAT or later.
212     return (name.startsWith(QCOM_PREFIX) && Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT)
213         // Exynos VP8 encoder is supported in M or later.
214         || (name.startsWith(EXYNOS_PREFIX) && Build.VERSION.SDK_INT >= Build.VERSION_CODES.M)
215         // Intel Vp8 encoder is supported in LOLLIPOP or later, with the intel encoder enabled.
216         || (name.startsWith(INTEL_PREFIX) && Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP
217                && enableIntelVp8Encoder);
218   }
```
这里截取了VP8的过滤，采用的是白名单的方式，只支持高通、三星以及因特尔的VP8编码器。

这里我们做过测试，如果不是白名单内的编码器，会有兼容性问题，如直接丢帧，当然这里我们暴露参数给到上层去过滤，而不是写死在SDK中。

## SoftwareVideoEncoderFactory
```cpp
21   public VideoEncoder createEncoder(VideoCodecInfo info) {
22     if (info.name.equalsIgnoreCase("VP8")) {
23       return new LibvpxVp8Encoder();
24     }
25     if (info.name.equalsIgnoreCase("VP9") && LibvpxVp9Encoder.nativeIsSupported()) {
26       return new LibvpxVp9Encoder();
27     }
28
29     return null;
30   }
```
这里只支持google的VPX软件编码器，至于H264需要自己添加JNI接口才行，我们后面看看。

要想兼容性最佳就统一使用软编，只是有些性能低的机型CPU/GPU会跑不动，这个需要覆盖测试了，最好还是能用硬编就用硬编，性能至上。

## TODO
* 关于cpp层的fallback处理
* H264/H265的软编解码器支持


