#!/bin/bash
set -e

# 🧰 Environment Variables
FFMPEG_VERSION=6.1.1
NDK_VERSION=28.2.13676358
ANDROID_API=29
ARCH=arm64-v8a
CPU=aarch64
PLATFORM=android
LIBS_DIR=$PWD/prebuilt
BUILD_DIR=$PWD/build
INSTALL_DIR=$BUILD_DIR/install
FFMPEG_DIR=$BUILD_DIR/ffmpeg

# 📁 Prepare dirs
mkdir -p $LIBS_DIR $BUILD_DIR $INSTALL_DIR

# 🧭 Toolchain setup
export ANDROID_NDK_HOME=$HOME/android-ndk
export PATH=$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin:$PATH
export CROSS_PREFIX=$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/bin
export CC="${CROSS_PREFIX}/aarch64-linux-android$ANDROID_API-clang"
export CXX="${CROSS_PREFIX}/aarch64-linux-android$ANDROID_API-clang++"
export AR="${CROSS_PREFIX}/llvm-ar"
export LD="${CROSS_PREFIX}/ld"
export STRIP="${CROSS_PREFIX}/llvm-strip"

# 🛠 Download and build library from URL
build_external_lib() {
  name=$1
  url=$2
  config_cmd=$3

  echo -e "\n📦 Building $name..."
  cd $LIBS_DIR
  rm -rf $name && git clone --depth=1 $url $name
  cd $name

  mkdir -p build-android && cd build-android
  $config_cmd
  make -j$(nproc)
  make install
}

# 🧩 External libraries (audio/video/image/subtitles)
build_all_external_libs() {
  export PKG_CONFIG_PATH=$INSTALL_DIR/lib/pkgconfig
  export CFLAGS="-fPIC -I$INSTALL_DIR/include"
  export LDFLAGS="-L$INSTALL_DIR/lib"

  build_external_lib x264 https://code.videolan.org/videolan/x264.git \
    "../configure --host=aarch64-linux --cross-prefix=${CROSS_PREFIX}/aarch64-linux-android- --prefix=$INSTALL_DIR --enable-static --disable-opencl --disable-cli --enable-pic --disable-shared --disable-asm --enable-strip"

  build_external_lib x265 https://github.com/videolan/x265.git \
    "cmake -DCMAKE_SYSTEM_NAME=Android -DCMAKE_SYSTEM_VERSION=$ANDROID_API -DCMAKE_ANDROID_ARCH_ABI=$ARCH -DCMAKE_ANDROID_NDK=$ANDROID_NDK_HOME -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK_HOME/build/cmake/android.toolchain.cmake -DENABLE_SHARED=OFF -DENABLE_CLI=OFF -DCMAKE_INSTALL_PREFIX=$INSTALL_DIR ../source"

  build_external_lib libvpx https://chromium.googlesource.com/webm/libvpx.git \
    "../configure --target=arm64-linux-gcc --disable-examples --disable-tools --disable-docs --disable-unit-tests --enable-pic --prefix=$INSTALL_DIR && make -j$(nproc) && make install"

  build_external_lib aom https://aomedia.googlesource.com/aom \
    "cmake -DCMAKE_SYSTEM_NAME=Android -DCMAKE_SYSTEM_VERSION=$ANDROID_API -DCMAKE_ANDROID_ARCH_ABI=$ARCH -DCMAKE_ANDROID_NDK=$ANDROID_NDK_HOME -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK_HOME/build/cmake/android.toolchain.cmake -DENABLE_SHARED=OFF -DCMAKE_INSTALL_PREFIX=$INSTALL_DIR .."

  build_external_lib dav1d https://code.videolan.org/videolan/dav1d.git \
    "meson setup --cross-file=$PWD/../../meson-android.ini --prefix=$INSTALL_DIR .. && ninja && ninja install"

  build_external_lib libass https://github.com/libass/libass.git \
    "../configure --host=aarch64-linux-android --prefix=$INSTALL_DIR --enable-static --disable-shared"

  build_external_lib freetype https://github.com/freetype/freetype.git \
    "cmake -DCMAKE_INSTALL_PREFIX=$INSTALL_DIR -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK_HOME/build/cmake/android.toolchain.cmake -DANDROID_ABI=$ARCH -DANDROID_PLATFORM=android-$ANDROID_API .."

  build_external_lib fribidi https://github.com/fribidi/fribidi.git \
    "../configure --host=aarch64-linux-android --prefix=$INSTALL_DIR --enable-static --disable-shared"

  build_external_lib libwebp https://chromium.googlesource.com/webm/libwebp \
    "../configure --host=aarch64-linux-android --prefix=$INSTALL_DIR --enable-static --disable-shared"

  build_external_lib lame https://sourceforge.net/p/lame/code/ci/master/tree/ \
    "../configure --host=aarch64-linux-android --prefix=$INSTALL_DIR --disable-shared --enable-static"

  build_external_lib opus https://github.com/xiph/opus.git \
    "./autogen.sh && ./configure --host=aarch64-linux-android --prefix=$INSTALL_DIR --disable-shared --enable-static"

  build_external_lib libvorbis https://github.com/xiph/vorbis.git \
    "./autogen.sh && ./configure --host=aarch64-linux-android --prefix=$INSTALL_DIR --disable-shared --enable-static"

  build_external_lib libiconv https://github.com/apple-oss-distributions/libiconv.git \
    "./configure --host=aarch64-linux-android --prefix=$INSTALL_DIR --disable-shared --enable-static"
}

# 📦 Download & extract FFmpeg
prepare_ffmpeg() {
  cd $BUILD_DIR
  rm -rf ffmpeg
  curl -L https://ffmpeg.org/releases/ffmpeg-$FFMPEG_VERSION.tar.bz2 | tar xj
  mv ffmpeg-$FFMPEG_VERSION ffmpeg
}

# 🏗 Build FFmpeg
build_ffmpeg() {
  echo -e "\n🏗 Configuring FFmpeg..."
  cd $FFMPEG_DIR

  PKG_CONFIG_PATH="$INSTALL_DIR/lib/pkgconfig" ./configure \
    --prefix=$INSTALL_DIR \
    --target-os=android \
    --arch=aarch64 \
    --cpu=armv8-a \
    --enable-cross-compile \
    --cross-prefix=$CROSS_PREFIX/aarch64-linux-android- \
    --cc=$CC \
    --nm="${CROSS_PREFIX}/llvm-nm" \
    --sysroot=$ANDROID_NDK_HOME/toolchains/llvm/prebuilt/linux-x86_64/sysroot \
    --pkg-config=pkg-config \
    --enable-gpl --enable-version3 \
    --enable-static --disable-shared \
    --disable-doc --disable-programs \
    --enable-libx264 --enable-libx265 --enable-libvpx --enable-libaom --enable-libwebp \
    --enable-libopus --enable-libvorbis --enable-libmp3lame \
    --enable-libass --enable-libfreetype --enable-libfribidi \
    --enable-libdav1d \
    --enable-zlib \
    --enable-libiconv \
    --extra-cflags="-I$INSTALL_DIR/include" \
    --extra-ldflags="-L$INSTALL_DIR/lib"

  echo -e "\n🔨 Building FFmpeg..."
  make -j$(nproc)
  make install
}

# 🔁 Build Process
build_all_external_libs
prepare_ffmpeg
build_ffmpeg

echo -e "\n✅ Build complete. libffmpeg.a is in $INSTALL_DIR/lib"
