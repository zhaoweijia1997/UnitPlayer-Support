# FFmpeg in UnitPlayer for Android

UnitPlayer 1.8 and later include a small FFmpeg decoding library (`libffmpegJNI.so`). It is used only for audio formats the phone can't decode itself; today that is ALAC (Apple Lossless). FFmpeg is licensed under the GNU Lesser General Public License 2.1; this folder holds the exact source code used and how it was built.

## Source

- `ffmpeg-7.1-98581d5-source.tar.xz`: FFmpeg 7.1, branch `release/7.1`, commit `98581d546ff453ce1f29c4af8448996d6ac992f7`, unmodified.
  SHA-256 `064708ac1462d202292f2513feeaa9293eb2d51ce139aec4f376a67d43b7984c`
- The same code upstream: https://github.com/FFmpeg/FFmpeg/tree/release/7.1

## Versions

- AndroidX Media3 `1.11.1` (commit `8c6678b657ede1e7883fc164ef73ed483c7796c3`), its `libraries/decoder_ffmpeg` module (Apache 2.0) for the Java side and the JNI glue
- FFmpeg 7.1 as above. Media3's notes mention FFmpeg 6.0, but its JNI code only needs APIs available since 5.1 (`ch_layout`, `swr_alloc_set_opts2`), so 7.1 builds as is
- Android NDK r29 (`29.0.14206865`), CMake 4.1.2
- Configure: `--disable-everything --enable-decoder=alac --enable-swresample`. No GPL or non-free parts are enabled
- Four CPU types: arm64-v8a, armeabi-v7a, x86_64, x86. The 64-bit libraries are aligned to 16 KB pages

## How to rebuild

The library is a separate shared object inside the app and can be replaced with your own build.

```sh
# Paths: MEDIA = a checkout of androidx/media at 1.11.1, NDK = Android NDK r29, CMAKE = CMake 4.1.2 bin directory
M=$MEDIA/libraries/decoder_ffmpeg/src/main

# 0. Unpack the source here and link it where Media3 expects it
tar -xJf ffmpeg-7.1-98581d5-source.tar.xz
ln -s "$PWD/ffmpeg-7.1" "$M/jni/ffmpeg"

# 1. Static FFmpeg libraries (add more decoder names after "alac" if needed)
cd "$M/jni" && ./build_ffmpeg.sh "$M" "$NDK" darwin-x86_64 21 alac

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

On Linux, use `linux-x86_64` instead of `darwin-x86_64`. The native methods are registered by class name in `JNI_OnLoad`, so the Java package `androidx.media3.decoder.ffmpeg` must stay as it is.

## Contact

techinfor404@gmail.com
