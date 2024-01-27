# armhf-swift-toolchain
Swift Toolchian for arm-linux-gnueabihf

# Current support Swift version

* `5.9` (recommend)

# How to install in macOS host

Firstly you need to install official swift version `5.9`, download it from https://swift.org

Create `Destinations` folder in `/Library/Developer` if there is not exist.

Create `Runtimes` folder in `/Library/Developer` if there is not exist

`mkdir -p /Library/Developer/Destinations`
`mkdir -p /Library/Developer/Runtimes`

Copy destinations file
`cp Destinations/macos/arm-none-linux-gnueabihf-*.json /Library/Developer/Destinations`

Decompression file `swift-5.9-runtime-arm-none-linux-gnueabihf` into `/Library/Developer/Runtimes` 

```
mkdir -p /Library/Developer/Runtimes/swift-5.9-runtime-arm-none-linux-gnueabihf
tar xf swift-5.9-runtime-arm-none-linux-gnueabihf.tar.gz -C /Library/Developer/Runtimes/swift-5.9-runtime-arm-none-linux-gnueabihf
```

# How to install runtime in linux target

You need to upload runtime libraries into you armv7 linux device.
```
scp -r /Library/Developer/Runtimes/swift-5.9-runtime-arm-none-linux-gnueabihf/usr/lib/swift/linux <username>@<target_host>:<install_path>

```

Login to your target device, add `<install_path>` to you ldconfig search path, than use `ldconfig` to update


# How to build

Create an example project

```
swift package init --type executable
swift build --destination /Library/Developer/Destinations/arm-none-linux-gnueabihf-5.9.json
```

Build Static
```
swift package init --type executable
swift build --destination /Library/Developer/Destinations/arm-none-linux-gnueabihf-5.9-static.json --static-swift-stdlib
```

After building success, you can upload binary to you target device, and run it.


# Cxx Support

**Cxx Support is disabled**

--------------
# How to build swift for linux-armhf

## Dependence

Build system: `Ubuntu 22.04`

### Install packages for compile

Reference: [https://github.com/apple/swift-docker/blob/main/swift-ci/master/ubuntu/22.04/Dockerfile]()

```bash
apt-get -y update && apt-get -y install \
  build-essential       \
  cmake                 \
  git                   \
  icu-devtools          \
  libcurl4-openssl-dev  \
  libedit-dev           \
  libicu-dev            \
  libncurses5-dev       \
  libpython3-dev        \
  libsqlite3-dev        \
  libxml2-dev           \
  ninja-build           \
  pkg-config            \
  python2               \
  python-six            \
  python2-dev           \
  python3-six           \
  python3-distutils     \
  python3-pkg-resources \
  python3-psutil        \
  rsync                 \
  swig                  \
  systemtap-sdt-dev     \
  tzdata                \
  uuid-dev              \
  zip
```

### Prepare compile workspace
Create an empty folder, which is used as our compile workspace.
```bash
mkdir swift-armhf-workspace
cd swift-armhf-workspace
export WORKSPACE_SWIFT_ARMHF=`pwd`
```

### Download GCC Toolchain
Be careful to the version of gcc toolchain, we need `12.2 Rel1`
```bash
cd $WORKSPACE_SWIFT_ARMHF
mkdir tool
cd tool
wget https://developer.arm.com/-/media/Files/downloads/gnu/12.2.rel1/binrel/arm-gnu-toolchain-12.2.rel1-x86_64-arm-none-linux-gnueabihf.tar.xz
tar xf arm-gnu-toolchain-12.2.rel1-x86_64-arm-none-linux-gnueabihf.tar.xz
```

### Prepare Ubuntu-base-22.04-armhf rootfs
Firstly, you need to install `qemu-user-static` for chroot.
```bash
sudo apt install qemu-user-static
```

Then download `ubuntu-base`
```bash
cd $WORKSPACE_SWIFT_ARMHF
cd tool
mkdir ubuntu-base
wget https://cdimage.ubuntu.com/ubuntu-base/releases/22.04.3/release/ubuntu-base-22.04.3-base-armhf.tar.gz
tar xf ubuntu-base-22.04.3-base-armhf.tar.gz -C ubuntu-base
```

Install dependence in `ubuntu-base`
```bash
cd $WORKSPACE_SWIFT_ARMHF/tool
sudo mount --bind /tmp ubuntu-base/tmp
sudo mount --bind /dev ubuntu-base/dev

sudo chroot ubuntu-base

# Chroot to ubuntu-base
echo "nameserver 8.8.8.8" >> /etc/resolv.conf
apt update
apt install -y libxml2-dev libicu-dev libcurl4-openssl-dev
exit
```


### Download the same version of swift toolchain
For example, if we are compiling swift-5.9, we need to download offical version of swift-5.9.
```bash
cd $WORKSPACE_SWIFT_ARMHF/tool
mkdir swift
wget https://download.swift.org/swift-5.9-release/ubuntu2204/swift-5.9-RELEASE/swift-5.9-RELEASE-ubuntu22.04.tar.gz
tar xzf swift-5.9-RELEASE-ubuntu22.04.tar.gz -C swift
```


### Prepare workspace link
```bash
cd $WORKSPACE_SWIFT_ARMHF
ln -sf tool/arm-gnu-toolchain-12.2.rel1-x86_64-arm-none-linux-gnueabihf toolchain
ln -sf tool/arm-gnu-toolchain-12.2.rel1-x86_64-arm-none-linux-gnueabihf/arm-none-linux-gnueabihf/libc sysroot
ln -sf tool/ubuntu-base deproot
ln -sf tool/swift/swift-5.9-RELEASE-ubuntu22.04 swift
```

### Download swift source code
```bash
cd $WORKSPACE_SWIFT_ARMHF
mkdir swift-project
cd swift-project
git clone https://github.com/apple/swift.git swift
cd swift
utils/update-checkout --clone --tag swift-5.9-RELEASE
```

## Patch source code

### Add build presets
Open file `$WORKSPACE_SWIFT_ARMHF/swift-project/swift/utils/build-presets.ini`, and append at the end of this file.

```ini
[preset: buildbot_linux_armv7_runtime_common]
build-subdir=buildbot_linux

release
no-assertions
build-ninja

cross-compile-hosts=linux-armv7
cross-compile-deps-path=%(sysroot)s
cross-compile-append-host-target-to-destdir=False

# Then set the paths to our native tools. If compiling against a toolchain,
# these should all be the ./usr/bin directory.
native-swift-tools-path=%(toolchain_path)s
host-cc=/usr/lib/llvm-14/bin/clang
host-cxx=/usr/lib/llvm-14/bin/clang++

build-swift-tools=0
skip-early-swift-driver
skip-early-swiftsyntax

skip-build-cmark

build-swift-static-stdlib
build-swift-static-sdk-overlay

swift-threading-package=c11
swift-stdlib-supports-backtrace-reporting=0

install-prefix=/usr
install-swift
install-swiftsyntax

skip-test-linux
skip-build-benchmarks
skip-test-swift
skip-test-swiftpm
skip-test-swift-driver
skip-test-llbuild
skip-test-lldb
skip-test-cmark
skip-test-playgroundsupport
skip-test-swiftsyntax
skip-test-swiftformat
skip-test-skstresstester
skip-test-swiftevolve
skip-test-swiftdocc

libdispatch-cmake-options=-DCMAKE_Swift_NUM_THREADS=1
foundation-cmake-options=-DCMAKE_Swift_NUM_THREADS=1

# Disable experimental differentiable programming, otherwise the compiler will crash
enable-experimental-differentiable-programming=0

# Disable Cxx interop, otherwise build break.I don't known how to fix it now.
enable-experimental-cxx-interop=0
enable-cxx-interop-swift-bridging-header=0

reconfigure


[preset: buildbot_linux_armv7_llvm]
mixin-preset=
    buildbot_linux_armv7_runtime_common
    
skip-build-swift

extra-cmake-options=
    -DCOMPILER_RT_BUILD_LIBFUZZER=OFF


[preset: buildbot_linux_armv7_swift]
mixin-preset=
    buildbot_linux_armv7_runtime_common
build-subdir=buildbot_linux

release
reconfigure

install-destdir=%(install_destdir)s
installable-package=%(installable_package)s
build-toolchain-only=1
build-swift-static-stdlib
skip-local-build

swift-enable-backtracing=0

extra-cmake-options=
    -DCOMPILER_RT_BUILD_LIBFUZZER=OFF
    -DPython3_EXECUTABLE="/usr/bin/python3.10"
    -DCMAKE_C_FLAGS="-target arm-none-linux-gnueabihf --ld-path=%(armhf_toolchain_bin)s/arm-none-linux-gnueabihf-ld -fPIC -Wno-error=unused-command-line-argument --sysroot=%(sysroot)s --gcc-toolchain=%(armhf_toolchain_bin)s/../ --sysroot=%(sysroot)s"
    -DCMAKE_CXX_FLAGS="-target arm-none-linux-gnueabihf --ld-path=%(armhf_toolchain_bin)s/arm-none-linux-gnueabihf-ld -Wno-error=unused-command-line-argument -fPIC --gcc-toolchain=%(armhf_toolchain_bin)s/.. --sysroot=%(sysroot)s -isystem %(armhf_toolchain_bin)s/../arm-none-linux-gnueabihf/include/c++/12.2.1 -isystem %(armhf_toolchain_bin)s/../arm-none-linux-gnueabihf/include/c++/12.2.1/arm-none-linux-gnueabihf -isystem %(armhf_toolchain_bin)s/../arm-none-linux-gnueabihf/include/c++/12.2.1/backward -isystem %(armhf_toolchain_bin)s/../arm-none-linux-gnueabihf/include -isystem %(sysroot)s/usr/include -B%(armhf_toolchain_bin)s/../lib/gcc/arm-none-linux-gnueabihf/12.2.1 -L%(armhf_toolchain_bin)s/../lib/gcc/arm-none-linux-gnueabihf/12.2.1"
    -DCMAKE_SYSROOT="%(sysroot)s"
    -DCMAKE_C_COMPILER_TARGET="arm-none-linux-gnueabihf"
    -DCMAKE_CXX_COMPILER_TARGET="arm-none-linux-gnueabihf"
    -DCMAKE_Swift_FLAGS="\"-target arm-none-linux-gnueabihf -Xcc --sysroot=%(sysroot)s\ -Xcc --gcc-toolchain=%(armhf_toolchain_bin)s/..\""

extra-swift-args=(Cxx|Glibc|SwiftPrivate|Backtracing|_Differentiation|Distributed|StdlibUnittest|DifferentiationUnittest|StdlibUnicodeUnittest|StdlibCollectionUnittest);-Xcc;(Cxx|Glibc|SwiftPrivate|Backtracing|_Differentiation|Distributed|StdlibUnittest|DifferentiationUnittest|StdlibUnicodeUnittest|StdlibCollectionUnittest);--sysroot=%(sysroot)s;(Cxx|Glibc|SwiftPrivate|Backtracing|_Differentiation|Distributed|StdlibUnittest|DifferentiationUnittest|StdlibUnicodeUnittest|StdlibCollectionUnittest);-Xcc;(Cxx|Glibc|SwiftPrivate|Backtracing|_Differentiation|Distributed|StdlibUnittest|DifferentiationUnittest|StdlibUnicodeUnittest|StdlibCollectionUnittest);--gcc-toolchain=%(armhf_toolchain_bin)s/..;(Cxx|Glibc|SwiftPrivate|Backtracing|_Differentiation|Distributed|StdlibUnittest|DifferentiationUnittest|StdlibUnicodeUnittest|StdlibCollectionUnittest);-Xcc;(Cxx|Glibc|SwiftPrivate|Backtracing|_Differentiation|Distributed|StdlibUnittest|DifferentiationUnittest|StdlibUnicodeUnittest|StdlibCollectionUnittest);-isystem;(Cxx|Glibc|SwiftPrivate|Backtracing|_Differentiation|Distributed|StdlibUnittest|DifferentiationUnittest|StdlibUnicodeUnittest|StdlibCollectionUnittest);-Xcc;(Cxx|Glibc|SwiftPrivate|Backtracing|_Differentiation|Distributed|StdlibUnittest|DifferentiationUnittest|StdlibUnicodeUnittest|StdlibCollectionUnittest);%(armhf_toolchain_bin)s/../arm-none-linux-gnueabihf/include/c++/12.2.1;(Cxx|Glibc|SwiftPrivate|Backtracing|_Differentiation|Distributed|StdlibUnittest|DifferentiationUnittest|StdlibUnicodeUnittest|StdlibCollectionUnittest);-Xcc;(Cxx|Glibc|SwiftPrivate|Backtracing|_Differentiation|Distributed|StdlibUnittest|DifferentiationUnittest|StdlibUnicodeUnittest|StdlibCollectionUnittest);-isystem;(Cxx|Glibc|SwiftPrivate|Backtracing|_Differentiation|Distributed|StdlibUnittest|DifferentiationUnittest|StdlibUnicodeUnittest|StdlibCollectionUnittest);-Xcc;(Cxx|Glibc|SwiftPrivate|Backtracing|_Differentiation|Distributed|StdlibUnittest|DifferentiationUnittest|StdlibUnicodeUnittest|StdlibCollectionUnittest);%(armhf_toolchain_bin)s/../arm-none-linux-gnueabihf/include/c++/12.2.1/arm-none-linux-gnueabihf;(Cxx|Glibc|SwiftPrivate|Backtracing|_Differentiation|Distributed|StdlibUnittest|DifferentiationUnittest|StdlibUnicodeUnittest|StdlibCollectionUnittest);-Xcc;(Cxx|Glibc|SwiftPrivate|Backtracing|_Differentiation|Distributed|StdlibUnittest|DifferentiationUnittest|StdlibUnicodeUnittest|StdlibCollectionUnittest);-isystem;(Cxx|Glibc|SwiftPrivate|Backtracing|_Differentiation|Distributed|StdlibUnittest|DifferentiationUnittest|StdlibUnicodeUnittest|StdlibCollectionUnittest);-Xcc;(Cxx|Glibc|SwiftPrivate|Backtracing|_Differentiation|Distributed|StdlibUnittest|DifferentiationUnittest|StdlibUnicodeUnittest|StdlibCollectionUnittest);%(armhf_toolchain_bin)s/../arm-none-linux-gnueabihf/include/c++/12.2.1/backward;(Cxx|Glibc|SwiftPrivate|Backtracing|_Differentiation|Distributed|StdlibUnittest|DifferentiationUnittest|StdlibUnicodeUnittest|StdlibCollectionUnittest);-Xcc;(Cxx|Glibc|SwiftPrivate|Backtracing|_Differentiation|Distributed|StdlibUnittest|DifferentiationUnittest|StdlibUnicodeUnittest|StdlibCollectionUnittest);-isystem;(Cxx|Glibc|SwiftPrivate|Backtracing|_Differentiation|Distributed|StdlibUnittest|DifferentiationUnittest|StdlibUnicodeUnittest|StdlibCollectionUnittest);-Xcc;(Cxx|Glibc|SwiftPrivate|Backtracing|_Differentiation|Distributed|StdlibUnittest|DifferentiationUnittest|StdlibUnicodeUnittest|StdlibCollectionUnittest);%(armhf_toolchain_bin)s/../arm-none-linux-gnueabihf/include;(Cxx|Glibc|SwiftPrivate|Backtracing|_Differentiation|Distributed|StdlibUnittest|DifferentiationUnittest|StdlibUnicodeUnittest|StdlibCollectionUnittest);-Xcc;(Cxx|Glibc|SwiftPrivate|Backtracing|_Differentiation|Distributed|StdlibUnittest|DifferentiationUnittest|StdlibUnicodeUnittest|StdlibCollectionUnittest);-isystem;(Cxx|Glibc|SwiftPrivate|Backtracing|_Differentiation|Distributed|StdlibUnittest|DifferentiationUnittest|StdlibUnicodeUnittest|StdlibCollectionUnittest);-Xcc;(Cxx|Glibc|SwiftPrivate|Backtracing|_Differentiation|Distributed|StdlibUnittest|DifferentiationUnittest|StdlibUnicodeUnittest|StdlibCollectionUnittest);%(sysroot)s/usr/include;(Cxx|Glibc|SwiftPrivate|Backtracing|_Differentiation|Distributed|StdlibUnittest|DifferentiationUnittest|StdlibUnicodeUnittest|StdlibCollectionUnittest);-Xcc;(Cxx|Glibc|SwiftPrivate|Backtracing|_Differentiation|Distributed|StdlibUnittest|DifferentiationUnittest|StdlibUnicodeUnittest|StdlibCollectionUnittest);-fmodule-map-file=%(workspace_dir)s/swift-project/build/buildbot_linux/swift-linux-armv7/lib/swift/linux/armv7/glibc.modulemap
```

### Patch components build arguments
Open file `$WORKSPACE_SWIFT_ARMHF/swift-project/swift/utils/swift_build_support/swift_build_support/products/product.py` and modify the implmentation of function `generate_linux_toolchain_file`.

The keypoint of the modification is below the line `target = self.get_linux_target(platform, arch)`, where we will tell cmake the build arguments for different targets.


**YOU MUST NEED TO MODIFY THE LINE** `workspace_dir = ""`

Set the `$WORKSPACE_SWIFT_ARMHF` value to `workspace_dir`.
```python
  def generate_linux_toolchain_file(self, platform, arch):
        """
        Generates a new CMake tolchain file that specifies Linux as a target
        plaftorm.

            Returns: path on the filesystem to the newly generated toolchain file.
        """

        shell.makedirs(self.build_dir)
        toolchain_file = os.path.join(self.build_dir, 'BuildScriptToolchain.cmake')

        toolchain_args = {}

        toolchain_args['CMAKE_SYSTEM_NAME'] = 'Linux'
        toolchain_args['CMAKE_SYSTEM_PROCESSOR'] = arch

        # We only set the actual sysroot if we are actually cross
        # compiling. This is important since otherwise cmake seems to change the
        # RUNPATH to be a relative rather than an absolute path, breaking
        # certain cmark tests (and maybe others).
        maybe_sysroot = self.get_linux_sysroot(platform, arch)
        if maybe_sysroot is not None:
            toolchain_args['CMAKE_SYSROOT'] = maybe_sysroot

        target = self.get_linux_target(platform, arch)

        # You need to modify this workspace_dir to $WORKSPACE_SWIFT_ARMHF path
        # For Example:
        # workspace_dir = "/home/eki/Project/swift-armhf-workspace/"
        # Tree of /home/eki/Project/swift-armhf-workspace/
        # .
        #├── deproot -> tool/ubuntu-base
        #├── swift -> tool/swift/swift-5.9-RELEASE-ubuntu22.04
        #├── swift-project
        #├── sysroot -> tool/arm-gnu-toolchain-12.2.rel1-x86_64-arm-none-linux-gnueabihf/arm-none-linux-gnueabihf/libc
        #├── tool
        #└── toolchain -> tool/arm-gnu-toolchain-12.2.rel1-x86_64-arm-none-linux-gnueabihf

        workspace_dir = ""

        if target == 'arm-unknown-linux-gnueabihf':
            if self.toolchain.cc.endswith('clang'):
                toolchain_args['CMAKE_C_COMPILER_TARGET'] = target
            if self.toolchain.cxx.endswith('clang++'):
                toolchain_args['CMAKE_CXX_COMPILER_TARGET'] = target
            # Swift always supports cross compiling.
            toolchain_args['CMAKE_Swift_COMPILER_TARGET'] = target
            toolchain_args['CMAKE_FIND_ROOT_PATH_MODE_PROGRAM'] = 'NEVER'
            toolchain_args['CMAKE_FIND_ROOT_PATH_MODE_LIBRARY'] = 'ONLY'
            toolchain_args['CMAKE_FIND_ROOT_PATH_MODE_INCLUDE'] = 'ONLY'
            toolchain_args['CMAKE_FIND_ROOT_PATH_MODE_PACKAGE'] = 'ONLY'
            
            toolchain_args['CMAKE_C_FLAGS'] = f"\"-target arm-none-linux-gnueabihf --ld-path={workspace_dir}/toolchain/bin/arm-none-linux-gnueabihf-ld -Wno-error=unused-command-line-argument -fPIC --gcc-toolchain={workspace_dir}/toolchain --sysroot={workspace_dir}/sysroot/\""

            # 使用 -isystem 优先使用这些头文件
            toolchain_args['CMAKE_CXX_FLAGS'] = f"\"-target arm-none-linux-gnueabihf --ld-path={workspace_dir}/toolchain/bin/arm-none-linux-gnueabihf-ld -Wno-error=unused-command-line-argument -fPIC --gcc-toolchain={workspace_dir}/toolchain --sysroot={workspace_dir}/sysroot/ -isystem {workspace_dir}/toolchain/arm-none-linux-gnueabihf/include/c++/12.2.1 -isystem {workspace_dir}/toolchain/lib/gcc/arm-none-linux-gnueabihf/12.2.1/../../../../arm-none-linux-gnueabihf/include/c++/12.2.1/arm-none-linux-gnueabihf -isystem {workspace_dir}/toolchain/lib/gcc/arm-none-linux-gnueabihf/12.2.1/../../../../arm-none-linux-gnueabihf/include/c++/12.2.1/backward -isystem {workspace_dir}/toolchain/lib/gcc/arm-none-linux-gnueabihf/12.2.1/../../../../arm-none-linux-gnueabihf/include -isystem {workspace_dir}/sysroot/usr/include -B{workspace_dir}/toolchain/lib/gcc/arm-none-linux-gnueabihf/12.2.1 -L{workspace_dir}/toolchain/lib/gcc/arm-none-linux-gnueabihf/12.2.1\""

            toolchain_args['CMAKE_SYSROOT'] = f"{workspace_dir}/sysroot/"

            toolchain_args['CMAKE_C_COMPILER_TARGET'] = "arm-unknown-linux-gnueabihf"
            toolchain_args['CMAKE_CXX_COMPILER_TARGET'] = "arm-unknown-linux-gnueabihf"

        else:
            if self.toolchain.cc.endswith('clang'):
                toolchain_args['CMAKE_C_COMPILER_TARGET'] = target
            if self.toolchain.cxx.endswith('clang++'):
                toolchain_args['CMAKE_CXX_COMPILER_TARGET'] = target
            # Swift always supports cross compiling.

            toolchain_args['CMAKE_CXX_FLAGS'] = "\"-fPIC -I/usr/include/c++/11 -I/usr/include/x86_64-linux-gnu/c++/11 -I/usr/lib/gcc/x86_64-linux-gnu/11 -L/usr/lib/gcc/x86_64-linux-gnu/11\""
            toolchain_args['CMAKE_Swift_COMPILER_TARGET'] = target
            toolchain_args['CMAKE_FIND_ROOT_PATH_MODE_PROGRAM'] = 'NEVER'
            toolchain_args['CMAKE_FIND_ROOT_PATH_MODE_LIBRARY'] = 'ONLY'
            toolchain_args['CMAKE_FIND_ROOT_PATH_MODE_INCLUDE'] = 'ONLY'
            toolchain_args['CMAKE_FIND_ROOT_PATH_MODE_PACKAGE'] = 'ONLY'


        if self.toolchain.cc.endswith('clang'):
            toolchain_args['CMAKE_C_COMPILER_TARGET'] = target
        if self.toolchain.cxx.endswith('clang++'):
            toolchain_args['CMAKE_CXX_COMPILER_TARGET'] = target
        # Swift always supports cross compiling.
        toolchain_args['CMAKE_Swift_COMPILER_TARGET'] = target
        toolchain_args['CMAKE_FIND_ROOT_PATH_MODE_PROGRAM'] = 'NEVER'
        toolchain_args['CMAKE_FIND_ROOT_PATH_MODE_LIBRARY'] = 'ONLY'
        toolchain_args['CMAKE_FIND_ROOT_PATH_MODE_INCLUDE'] = 'ONLY'
        toolchain_args['CMAKE_FIND_ROOT_PATH_MODE_PACKAGE'] = 'ONLY'

        # Sort by the key so that we always produce the same toolchain file
        data = sorted(toolchain_args.items(), key=lambda x: x[0])
        if not self.args.dry_run:
            with open(toolchain_file, 'w') as f:
                f.writelines("set({} {})\n".format(k, v) for k, v in data)
        else:
            print("DRY_RUN! Writing Toolchain file to path: {}".format(toolchain_file))

        return toolchain_file 
```


### Patch Float16 Support at 32-bit platform
Open file `$WORKSPACE_SWIFT_ARMHF/swift-project/swift/stdlib/public/runtime/Float16Support.cpp`

Replace first `#if` marco
```C
#if (defined(__ANDROID__) && defined(__ARM_ARCH_7A__) && defined(__ARM_EABI__)) || \
  ((defined(__i386__) || defined(__i686__) || defined(__x86_64__)) && !defined(__APPLE__))
```

with

```C
#if ((defined(__ANDROID__) || defined(__linux__)) && (defined(__arm__) || (defined(__riscv) && __riscv_xlen == 64) || defined(__powerpc__) || (defined(__powerpc64__) && defined(__LITTLE_ENDIAN__)))) || \
  ((defined(__i386__) || defined(__i686__) || defined(__x86_64__)) && !defined(__APPLE__))
```

### Patch glibc header
Open file `$WORKSPACE_SWIFT_ARMHF/swift-project/swift/stdlib/public/Platform/glibc.modulemap.gyb`

**Comment or delete** this line
```C
textual header "assert.h"
```


### Patch cmake can't found processor
Open file `$WORKSPACE_SWIFT_ARMHF/swift-project/swift-corelibs-foundation/Sources/CMakeLists.txt`.
At the begin of this file, add
```
# Force Build For armv7l
set(CMAKE_SYSTEM_PROCESSOR "armv7l")
```

Open file `$WORKSPACE_SWIFT_ARMHF/swift-project/swift-corelibs-libdispatch/src/swift/CMakeLists.txt`.
At the begin of this file, add
```
# Force Build For armv7l
set(CMAKE_SYSTEM_PROCESSOR "armv7l")
```

### Patch type dismatch at 32-bit platform
Open file `$WORKSPACE_SWIFT_ARMHF/swift-project/swift-corelibs-libdispatch/src/benchmark.c`
Please refere to this patch to replace `long double` to `uint64_t`.
```diff
diff --git a/src/benchmark.c b/src/benchmark.c
index 15e9f55..c977fd2 100644
--- a/src/benchmark.c
+++ b/src/benchmark.c
@@ -44,7 +44,7 @@ _dispatch_benchmark_init(void *context)
 #if DISPATCH_SIZEOF_PTR == 8 && !defined(_WIN32)
        __uint128_t lcost;
 #else
-       long double lcost;
+       uint64_t lcost;
 #endif
 #if HAVE_MACH_ABSOLUTE_TIME
        kern_return_t kr;
@@ -96,7 +96,7 @@ dispatch_benchmark_f(size_t count, register void *ctxt,
 #if DISPATCH_SIZEOF_PTR == 8 && !defined(_WIN32)
        __uint128_t conversion, big_denom;
 #else
-       long double conversion, big_denom;
+       uint64_t conversion, big_denom;
 #endif
        size_t i = 0;
```

After all patch done, this is what we workspace dir looks like:
```bash
        .
        ├── deproot -> tool/ubuntu-base
        ├── swift -> tool/swift/swift-5.9-RELEASE-ubuntu22.04
        ├── swift-project
        ├── sysroot -> tool/arm-gnu-toolchain-12.2.rel1-x86_64-arm-none-linux-gnueabihf/arm-none-linux-gnueabihf/libc
        ├── tool
        └── toolchain -> tool/arm-gnu-toolchain-12.2.rel1-x86_64-arm-none-linux-gnueabihf 
```

## Start build

### Build LLVM
```bash
cd $WORKSPACE_SWIFT_ARMHF/swift-project/swift
utils/build-script --preset=buildbot_linux_armv7_llvm \
    toolchain_path=$WORKSPACE_SWIFT_ARMHF/swift/usr/bin \
    install_destdir=$WORKSPACE_SWIFT_ARMHF/install \
    installable_package=$WORKSPACE_SWIFT_ARMHF/swift-linux-armhf-runtime.tar.gz \
    sysroot=$WORKSPACE_SWIFT_ARMHF/sysroot/ \
    armhf_toolchain_bin=$WORKSPACE_SWIFT_ARMHF/toolchain/bin \
	workspace_dir=$WORKSPACE_SWIFT_ARMHF
```

### Build Swift
```bash
cd $WORKSPACE_SWIFT_ARMHF/swift-project/swift
utils/build-script --preset=buildbot_linux_armv7_swift \
    toolchain_path=$WORKSPACE_SWIFT_ARMHF/swift/usr/bin \
    install_destdir=$WORKSPACE_SWIFT_ARMHF/install \
    installable_package=$WORKSPACE_SWIFT_ARMHF/swift-linux-armhf-runtime.tar.gz \
    sysroot=$WORKSPACE_SWIFT_ARMHF/sysroot/ \
    armhf_toolchain_bin=$WORKSPACE_SWIFT_ARMHF/toolchain/bin \
	workspace_dir=$WORKSPACE_SWIFT_ARMHF  
```

### Build libdispatch
```bash
GCC_TOOLCHAIN_PATH=$WORKSPACE_SWIFT_ARMHF/toolchain
GCC_SYSROOT_PATH=$WORKSPACE_SWIFT_ARMHF/sysroot
GCC_TOOLCHAIN_BIN=${GCC_TOOLCHAIN_PATH}/bin
SWIFT_NATIVE_TOOLCHAIN_PATH=$WORKSPACE_SWIFT_ARMHF/swift/usr/bin
INSTALL_DIR_PATH=$WORKSPACE_SWIFT_ARMHF/install
SWIFT_SOURCE_ROOT=$WORKSPACE_SWIFT_ARMHF/swift-project/
SWIFT_BUILD_PRODUCT_PATH=${SWIFT_SOURCE_ROOT}/build/buildbot_linux

cd $SWIFT_BUILD_PRODUCT_PATH
mkdir libdispatch-linux-armv7
cd libdispatch-linux-armv7


# build static
cmake -G Ninja \
    -DCMAKE_SYSTEM_NAME=Linux \
    -DCMAKE_C_COMPILER=${SWIFT_BUILD_PRODUCT_PATH}/llvm-linux-x86_64/bin/clang \
    -DCMAKE_C_COMPILER_TARGET=arm-none-linux-gnueabihf \
    -DCMAKE_CXX_COMPILER_TARGET=arm-none-linux-gnueabihf \
    -DCMAKE_CXX_COMPILER=${SWIFT_BUILD_PRODUCT_PATH}/llvm-linux-x86_64/bin/clang++ \
    \
    -DCMAKE_SYSROOT=${GCC_SYSROOT_PATH} \
    -DCMAKE_INSTALL_PREFIX=${INSTALL_DIR_PATH}/usr \
    -DCMAKE_BUILD_TYPE=Release \
    -DBUILD_SHARED_LIBS=OFF \
    -DENABLE_SWIFT=ON \
    -DCMAKE_Swift_COMPILER="${SWIFT_NATIVE_TOOLCHAIN_PATH}/swiftc" \
    \
    -DCMAKE_C_FLAGS="-target arm-none-linux-gnueabihf --ld-path=${GCC_TOOLCHAIN_BIN}/arm-none-linux-gnueabihf-ld -Wno-error=unused-command-line-argument -fPIC --sysroot=${GCC_SYSROOT_PATH} --gcc-toolchain=${GCC_TOOLCHAIN_PATH}" \
    \
    -DCMAKE_CXX_FLAGS="-target arm-none-linux-gnueabihf --ld-path=${GCC_TOOLCHAIN_BIN}/arm-none-linux-gnueabihf-ld -Wno-error=unused-command-line-argument -fPIC --sysroot=${GCC_SYSROOT_PATH} --gcc-toolchain=${GCC_TOOLCHAIN_PATH} " \
    \
    -DCMAKE_Swift_FLAGS="-target armv7-unknown-linux-gnueabihf -use-ld=${GCC_TOOLCHAIN_BIN}/arm-none-linux-gnueabihf-ld -sdk ${GCC_SYSROOT_PATH} -tools-directory ${GCC_TOOLCHAIN_PATH}/lib/gcc/arm-none-linux-gnueabihf/12.2.1 -resource-dir ${SWIFT_BUILD_PRODUCT_PATH}/swift-linux-armv7/lib/swift -L${GCC_TOOLCHAIN_PATH}/arm-none-linux-gnueabihf/libc/lib/ -L${GCC_TOOLCHAIN_PATH}/lib/gcc/arm-none-linux-gnueabihf/12.2.1/" \
    ../../../swift-corelibs-libdispatch

ninja -C .
ninja -C . install

# build shared
rm -rf $SWIFT_BUILD_PRODUCT_PATH/libdispatch-linux-armv7
cmake -G Ninja \
    -DCMAKE_SYSTEM_NAME=Linux \
    -DCMAKE_C_COMPILER=${SWIFT_BUILD_PRODUCT_PATH}/llvm-linux-x86_64/bin/clang \
    -DCMAKE_C_COMPILER_TARGET=arm-none-linux-gnueabihf \
    -DCMAKE_CXX_COMPILER_TARGET=arm-none-linux-gnueabihf \
    -DCMAKE_CXX_COMPILER=${SWIFT_BUILD_PRODUCT_PATH}/llvm-linux-x86_64/bin/clang++ \
    \
    -DCMAKE_SYSROOT=${GCC_SYSROOT_PATH} \
    -DCMAKE_INSTALL_PREFIX=${INSTALL_DIR_PATH}/usr \
    -DCMAKE_BUILD_TYPE=Release \
    -DBUILD_SHARED_LIBS=ON \
    -DENABLE_SWIFT=ON \
    -DCMAKE_Swift_COMPILER="${SWIFT_NATIVE_TOOLCHAIN_PATH}/swiftc" \
    \
    -DCMAKE_C_FLAGS="-target arm-none-linux-gnueabihf --ld-path=${GCC_TOOLCHAIN_BIN}/arm-none-linux-gnueabihf-ld -Wno-error=unused-command-line-argument -fPIC --sysroot=${GCC_SYSROOT_PATH} --gcc-toolchain=${GCC_TOOLCHAIN_PATH}" \
    \
    -DCMAKE_CXX_FLAGS="-target arm-none-linux-gnueabihf --ld-path=${GCC_TOOLCHAIN_BIN}/arm-none-linux-gnueabihf-ld -Wno-error=unused-command-line-argument -fPIC --sysroot=${GCC_SYSROOT_PATH} --gcc-toolchain=${GCC_TOOLCHAIN_PATH} " \
    \
    -DCMAKE_Swift_FLAGS="-target armv7-unknown-linux-gnueabihf -use-ld=${GCC_TOOLCHAIN_BIN}/arm-none-linux-gnueabihf-ld -sdk ${GCC_SYSROOT_PATH} -tools-directory ${GCC_TOOLCHAIN_PATH}/lib/gcc/arm-none-linux-gnueabihf/12.2.1 -resource-dir ${SWIFT_BUILD_PRODUCT_PATH}/swift-linux-armv7/lib/swift -L${GCC_TOOLCHAIN_PATH}/arm-none-linux-gnueabihf/libc/lib/ -L${GCC_TOOLCHAIN_PATH}/lib/gcc/arm-none-linux-gnueabihf/12.2.1/" \
    ../../../swift-corelibs-libdispatch

ninja -C .
ninja -C . install  
```

### Build Foundation
```bash
# Shared

GCC_TOOLCHAIN_PATH=$WORKSPACE_SWIFT_ARMHF/toolchain
GCC_SYSROOT_PATH=$WORKSPACE_SWIFT_ARMHF/sysroot/
GCC_TOOLCHAIN_BIN=${GCC_TOOLCHAIN_PATH}/bin
DEP_ROOT_PATH=$WORKSPACE_SWIFT_ARMHF/deproot
SWIFT_NATIVE_TOOLCHAIN_PATH=$WORKSPACE_SWIFT_ARMHF/swift/usr/bin
INSTALL_DIR_PATH=$WORKSPACE_SWIFT_ARMHF/install
SWIFT_SOURCE_ROOT=$WORKSPACE_SWIFT_ARMHF/swift-project/
SWIFT_BUILD_PRODUCT_PATH=${SWIFT_SOURCE_ROOT}/build/buildbot_linux
ICU_ROOT=${SWIFT_BUILD_PRODUCT_PATH}/libicu-linux-armv7/tmp_install
ICU_LIBDIR=${SWIFT_BUILD_PRODUCT_PATH}/libicu-linux-armv7/lib

cd $SWIFT_BUILD_PRODUCT_PATH
mkdir foundation-linux-armv7
cd foundation-linux-armv7

rm -rf ${SWIFT_BUILD_PRODUCT_PATH}/libicu-linux-armv7
mkdir -p ${ICU_ROOT}/include
mkdir -p ${ICU_LIBDIR}

cp -rf `readlink -e ${DEP_ROOT_PATH}/usr/lib/arm-linux-gnueabihf/libicudata.so` ${ICU_LIBDIR}/libicudataswift.so
cp -rf `readlink -e ${DEP_ROOT_PATH}/usr/lib/arm-linux-gnueabihf/libicuuc.so` ${ICU_LIBDIR}/libicuucswift.so
cp -rf `readlink -e ${DEP_ROOT_PATH}/usr/lib/arm-linux-gnueabihf/libicui18n.so` ${ICU_LIBDIR}/libicui18nswift.so

cp -rf `readlink -e ${DEP_ROOT_PATH}/usr/lib/arm-linux-gnueabihf/libicudata.a` ${ICU_LIBDIR}/libicudataswift.a
cp -rf `readlink -e ${DEP_ROOT_PATH}/usr/lib/arm-linux-gnueabihf/libicuuc.a` ${ICU_LIBDIR}/libicuucswift.a
cp -rf `readlink -e ${DEP_ROOT_PATH}/usr/lib/arm-linux-gnueabihf/libicui18n.a` ${ICU_LIBDIR}/libicui18nswift.a

cp -rf ${DEP_ROOT_PATH}/usr/include/unicode ${ICU_ROOT}/include

cmake -G Ninja \
    -DCMAKE_SYSTEM_NAME=Linux \
    -DCMAKE_INSTALL_PREFIX=${INSTALL_DIR_PATH}/usr \
    -DCMAKE_BUILD_TYPE=Release \
    -DBUILD_SHARED_LIBS=ON \
    -Ddispatch_DIR=${SWIFT_BUILD_PRODUCT_PATH}/libdispatch-linux-armv7/cmake/modules \
    \
    -DICU_ROOT:PATH=${ICU_ROOT} \
    -DICU_INCLUDE_DIR:PATH=${ICU_ROOT}/include \
    -DICU_DATA_LIBRARIES:FILEPATH=${ICU_LIBDIR}/libicudataswift.so \
    -DICU_DATA_LIBRARY:FILEPATH=${ICU_LIBDIR}/libicudataswift.so \
    -DICU_DATA_LIBRARY_DEBUG:FILEPATH=${ICU_LIBDIR}/libicudataswift.so \
    -DICU_DATA_LIBRARY_RELEASE:FILEPATH=${ICU_LIBDIR}/libicudataswift.so \
    -DICU_UC_LIBRARIES:FILEPATH=${ICU_LIBDIR}/libicuucswift.so \
    -DICU_UC_LIBRARY:FILEPATH=${ICU_LIBDIR}/libicuucswift.so \
    -DICU_UC_LIBRARY_DEBUG:FILEPATH=${ICU_LIBDIR}/libicuucswift.so \
    -DICU_UC_LIBRARY_RELEASE:FILEPATH=${ICU_LIBDIR}/libicuucswift.so \
    -DICU_I18N_LIBRARIES:FILEPATH=${ICU_LIBDIR}/libicui18nswift.so \
    -DICU_I18N_LIBRARY:FILEPATH=${ICU_LIBDIR}/libicui18nswift.so \
    -DICU_I18N_LIBRARY_DEBUG:FILEPATH=${ICU_LIBDIR}/libicui18nswift.so \
    -DICU_I18N_LIBRARY_RELEASE:FILEPATH=${ICU_LIBDIR}/libicui18nswift.so \
    \
    -DCURL_LIBRARY=${DEP_ROOT_PATH}/usr/lib/arm-linux-gnueabihf/libcurl.a \
    -DCURL_INCLUDE_DIR=${DEP_ROOT_PATH}/usr/include/arm-linux-gnueabihf \
    \
    -DLIBXML2_LIBRARY=${DEP_ROOT_PATH}/usr/lib/arm-linux-gnueabihf/libxml2.a \
    -DLIBXML2_INCLUDE_DIR=${DEP_ROOT_PATH}/usr/include/libxml2 \
    -DLIBXML2_DEFINITIONS="-DLIBXML_STATIC" \
    \
    -DCMAKE_C_COMPILER=${SWIFT_BUILD_PRODUCT_PATH}/llvm-linux-x86_64/bin/clang \
    -DCMAKE_C_COMPILER_TARGET=arm-none-linux-gnueabihf \
    -DCMAKE_CXX_COMPILER_TARGET=arm-none-linux-gnueabihf \
    -DCMAKE_CXX_COMPILER=${SWIFT_BUILD_PRODUCT_PATH}/llvm-linux-x86_64/bin/clang++ \
    -DCMAKE_ASM_COMPILER_TARGET=arm-none-linux-gnueabihf \
    \
    -DCMAKE_HAVE_LIBC_PTHREAD=YES \
    -DCMAKE_SYSROOT=${GCC_SYSROOT_PATH} \
    -DCMAKE_Swift_COMPILER="${SWIFT_NATIVE_TOOLCHAIN_PATH}/swiftc" \
    -DCMAKE_C_FLAGS="-target arm-none-linux-gnueabihf --ld-path=${GCC_TOOLCHAIN_BIN}/arm-none-linux-gnueabihf-ld -Wno-error=unused-command-line-argument -fPIC --sysroot=${GCC_SYSROOT_PATH} --gcc-toolchain=${GCC_TOOLCHAIN_PATH} -I${DEP_ROOT_PATH}/usr/include/" \
    \
    -DCMAKE_CXX_FLAGS="-target arm-none-linux-gnueabihf --ld-path=${GCC_TOOLCHAIN_BIN}/arm-none-linux-gnueabihf-ld -Wno-error=unused-command-line-argument -fPIC --sysroot=${GCC_SYSROOT_PATH} --gcc-toolchain=${GCC_TOOLCHAIN_PATH}" \
    \
    -DCMAKE_Swift_FLAGS="-target armv7-unknown-linux-gnueabihf -use-ld=${GCC_TOOLCHAIN_BIN}/arm-none-linux-gnueabihf-ld -sdk ${GCC_SYSROOT_PATH} -tools-directory ${GCC_TOOLCHAIN_PATH}/lib/gcc/arm-none-linux-gnueabihf/12.2.1 -resource-dir ${SWIFT_BUILD_PRODUCT_PATH}/swift-linux-armv7/lib/swift -L${GCC_TOOLCHAIN_PATH}/arm-none-linux-gnueabihf/libc/lib/ -L${GCC_TOOLCHAIN_PATH}/lib/gcc/arm-none-linux-gnueabihf/12.2.1/ -Xcc -isystem -Xcc ${SWIFT_BUILD_PRODUCT_PATH}/llvm-linux-x86_64/lib/clang/13.0.0/include -L${ICU_LIBDIR} -licui18nswift -licuucswift -licudataswift" \
    ../../../swift-corelibs-foundation

ninja -C . install
cp -rf ${ICU_LIBDIR}/libicu*.so ${INSTALL_DIR_PATH}/usr/lib/swift/linux/

# Static
rm -rf $SWIFT_BUILD_PRODUCT_PATH/foundation-linux-armv7/*
cmake -G Ninja \
    -DCMAKE_SYSTEM_NAME=Linux \
    -DCMAKE_INSTALL_PREFIX=${INSTALL_DIR_PATH}/usr \
    -DCMAKE_BUILD_TYPE=Release \
    -DBUILD_SHARED_LIBS=OFF \
    -Ddispatch_DIR=${SWIFT_BUILD_PRODUCT_PATH}/libdispatch-linux-armv7/cmake/modules \
    \
    -DICU_ROOT:PATH=${ICU_ROOT} \
    -DICU_INCLUDE_DIR:PATH=${ICU_ROOT}/include \
    -DICU_DATA_LIBRARIES:FILEPATH=${ICU_LIBDIR}/libicudataswift.a \
    -DICU_DATA_LIBRARY:FILEPATH=${ICU_LIBDIR}/libicudataswift.a \
    -DICU_DATA_LIBRARY_DEBUG:FILEPATH=${ICU_LIBDIR}/libicudataswift.a \
    -DICU_DATA_LIBRARY_RELEASE:FILEPATH=${ICU_LIBDIR}/libicudataswift.a \
    -DICU_UC_LIBRARIES:FILEPATH=${ICU_LIBDIR}/libicuucswift.a \
    -DICU_UC_LIBRARY:FILEPATH=${ICU_LIBDIR}/libicuucswift.a \
    -DICU_UC_LIBRARY_DEBUG:FILEPATH=${ICU_LIBDIR}/libicuucswift.a \
    -DICU_UC_LIBRARY_RELEASE:FILEPATH=${ICU_LIBDIR}/libicuucswift.a \
    -DICU_I18N_LIBRARIES:FILEPATH=${ICU_LIBDIR}/libicui18nswift.a \
    -DICU_I18N_LIBRARY:FILEPATH=${ICU_LIBDIR}/libicui18nswift.a \
    -DICU_I18N_LIBRARY_DEBUG:FILEPATH=${ICU_LIBDIR}/libicui18nswift.a \
    -DICU_I18N_LIBRARY_RELEASE:FILEPATH=${ICU_LIBDIR}/libicui18nswift.a \
    \
    -DCURL_LIBRARY=${DEP_ROOT_PATH}/usr/lib/arm-linux-gnueabihf/libcurl.a \
    -DCURL_INCLUDE_DIR=${DEP_ROOT_PATH}/usr/include/arm-linux-gnueabihf \
    \
    -DLIBXML2_LIBRARY=${DEP_ROOT_PATH}/usr/lib/arm-linux-gnueabihf/libxml2.a \
    -DLIBXML2_INCLUDE_DIR=${DEP_ROOT_PATH}/usr/include/libxml2 \
    \
    -DCMAKE_C_COMPILER=${SWIFT_BUILD_PRODUCT_PATH}/llvm-linux-x86_64/bin/clang \
    -DCMAKE_C_COMPILER_TARGET=arm-none-linux-gnueabihf \
    -DCMAKE_CXX_COMPILER_TARGET=arm-none-linux-gnueabihf \
    -DCMAKE_CXX_COMPILER=${SWIFT_BUILD_PRODUCT_PATH}/llvm-linux-x86_64/bin/clang++ \
    -DCMAKE_ASM_COMPILER_TARGET=arm-none-linux-gnueabihf \
    \
    -DCMAKE_HAVE_LIBC_PTHREAD=YES \
    -DCMAKE_SYSROOT=${GCC_SYSROOT_PATH} \
    -DCMAKE_Swift_COMPILER="${SWIFT_NATIVE_TOOLCHAIN_PATH}/swiftc" \
    -DCMAKE_C_FLAGS="-target arm-none-linux-gnueabihf --ld-path=${GCC_TOOLCHAIN_BIN}/arm-none-linux-gnueabihf-ld -Wno-error=unused-command-line-argument -fPIC --sysroot=${GCC_SYSROOT_PATH} --gcc-toolchain=${GCC_TOOLCHAIN_PATH} -I${DEP_ROOT_PATH}/usr/include/" \
    \
    -DCMAKE_CXX_FLAGS="-target arm-none-linux-gnueabihf --ld-path=${GCC_TOOLCHAIN_BIN}/arm-none-linux-gnueabihf-ld -Wno-error=unused-command-line-argument -fPIC --sysroot=${GCC_SYSROOT_PATH} --gcc-toolchain=${GCC_TOOLCHAIN_PATH}" \
    \
    -DCMAKE_Swift_FLAGS="-target armv7-unknown-linux-gnueabihf -use-ld=${GCC_TOOLCHAIN_BIN}/arm-none-linux-gnueabihf-ld -sdk ${GCC_SYSROOT_PATH} -tools-directory ${GCC_TOOLCHAIN_PATH}/lib/gcc/arm-none-linux-gnueabihf/12.2.1 -resource-dir ${SWIFT_BUILD_PRODUCT_PATH}/swift-linux-armv7/lib/swift -L${GCC_TOOLCHAIN_PATH}/arm-none-linux-gnueabihf/libc/lib/ -L${GCC_TOOLCHAIN_PATH}/lib/gcc/arm-none-linux-gnueabihf/12.2.1/ -Xcc -isystem -Xcc ${SWIFT_BUILD_PRODUCT_PATH}/llvm-linux-x86_64/lib/clang/13.0.0/include -L${ICU_LIBDIR} -licui18nswift -licuucswift -licudataswift -lstdc++" \
    ../../../swift-corelibs-foundation

ninja -C . install
cp -rf ${ICU_LIBDIR}/libicu*.a ${INSTALL_DIR_PATH}/usr/lib/swift_static/linux/
```

After building success, you can found sdk at `$WORKSPACE_SWIFT_ARMHF/install`

## Pack toolchain
After building success, you can refer to the [swift-5.9-runtime-armhf](https://github.com/CmST0us/armhf-swift-toolchain/blob/main/swift-5.9-runtime-arm-none-linux-gnueabihf.tar.gz) to pack the full runtime sdk.

There are two keypoints:
1. Use ld on macOS, which can link armhf target.
2. For convenience, packing the toolchain in sdk.


--------------
`Have fun and play with Swift everywhere!`
