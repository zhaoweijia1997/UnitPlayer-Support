# FFmpeg in UnitPlayer for Android

UnitPlayer 1.8 and later include a small FFmpeg decoding library (`libffmpegJNI.so`). It is used only for audio formats the phone can't decode itself; since 1.9 those are ALAC (Apple Lossless), APE (Monkey's Audio), WavPack, TTA and TAK. FFmpeg is licensed under the GNU Lesser General Public License 2.1; this folder holds the exact source code used and how it was built.

## Source

- `ffmpeg-7.1-98581d5-source.tar.xz`: FFmpeg 7.1, branch `release/7.1`, commit `98581d546ff453ce1f29c4af8448996d6ac992f7`, unmodified.
  SHA-256 `064708ac1462d202292f2513feeaa9293eb2d51ce139aec4f376a67d43b7984c`
- The same code upstream: https://github.com/FFmpeg/FFmpeg/tree/release/7.1

## Versions

- AndroidX Media3 `1.11.1` (commit `8c6678b657ede1e7883fc164ef73ed483c7796c3`), its `libraries/decoder_ffmpeg` module (Apache 2.0) for the Java side and the JNI glue, with the changes listed below
- FFmpeg 7.1 as above. Media3's notes mention FFmpeg 6.0, but its JNI code only needs APIs available since 5.1 (`ch_layout`, `swr_alloc_set_opts2`), so 7.1 builds as is
- Android NDK r29 (`29.0.14206865`), CMake 4.1.2
- Configure: `--disable-everything --enable-decoder=alac --enable-decoder=ape --enable-decoder=wavpack --enable-decoder=tta --enable-decoder=tak --enable-swresample`. No GPL or non-free parts are enabled
- Four CPU types: arm64-v8a, armeabi-v7a, x86_64, x86. The 64-bit libraries are aligned to 16 KB pages

## Changes to the JNI glue (1.9)

FFmpeg itself is unmodified. `jni/` holds the files we build with: `ffmpeg_jni.cc` differs from Media3's in two ways (this is Media3's Apache 2.0 code, not FFmpeg's), and `CMakeLists.txt` and `build_ffmpeg.sh` are as used for this build.

- `ffmpegInitialize` takes one more argument, `bitsPerSample`. The APE and TAK decoders read the stream's sample rate, channels and bit depth from the codec context, so these are filled in before the decoder opens.
- In `decodePacket`, after the output buffer grows, writing carries on after what was already written. Upstream started over, so a packet bigger than the buffer lost its first samples.

## How to rebuild

The library is a separate shared object inside the app and can be replaced with your own build.

```sh
# Paths: MEDIA = a checkout of androidx/media at 1.11.1, NDK = Android NDK r29, CMAKE = CMake 4.1.2 bin directory
M=$MEDIA/libraries/decoder_ffmpeg/src/main

# 0. Unpack the source here and link it where Media3 expects it
tar -xJf ffmpeg-7.1-98581d5-source.tar.xz
ln -s "$PWD/ffmpeg-7.1" "$M/jni/ffmpeg"

# 1. Our JNI glue over Media3's, then the static FFmpeg libraries
cp jni/ffmpeg_jni.cc "$M/jni/ffmpeg_jni.cc"
cd "$M/jni" && ./build_ffmpeg.sh "$M" "$NDK" darwin-x86_64 21 alac ape wavpack tta tak

# 2. libffmpegJNI.so for each CPU type
for ABI in arm64-v8a armeabi-v7a x86_64 x86; do
  "$CMAKE/cmake" -S "$M/jni" -B "build-$ABI" -G Ninja -DCMAKE_MAKE_PROGRAM="$CMAKE/ninja" \
    -DCMAKE_TOOLCHAIN_FILE="$NDK/build/cmake/android.toolchain.cmake" \
    -DANDROID_ABI=$ABI -DANDROID_PLATFORM=android-29 -DCMAKE_BUILD_TYPE=Release
  "$CMAKE/cmake" --build "build-$ABI"
done

# 3. Strip
T=$NDK/toolchains/llvm/prebuilt/darwin-x86_64/bin
for ABI in arm64-v8a armeabi-v7a x86_64 x86; do
  mkdir -p "out/$ABI"
  "$T/llvm-strip" --strip-unneeded -o "out/$ABI/libffmpegJNI.so" "build-$ABI/libffmpegJNI.so"
done
```

On Linux, use `linux-x86_64` instead of `darwin-x86_64`. The native methods are registered by class name in `JNI_OnLoad`, so the Java package `androidx.media3.decoder.ffmpeg` must stay as it is, and a replacement library needs the `ffmpegInitialize` signature above.

## Contact

techinfor404@gmail.com
