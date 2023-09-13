# armhf-swift-toolchain
Swift Toolchian for arm-linux-gnueabihf

# How to install in macOS host

Firstly you need to install official swift version `5.8.1`, download it from (https://swift.org)[https://swift.org]

Create `Destinations` folder in `/Library/Developer` if there is not exist.

Create `Runtimes` folder in `/Library/Developer` if there is not exist

`mkdir -p /Library/Developer/Destinations`
`mkdir -p /Library/Developer/Runtimes`

Copy destinations file
`cp Destinations/macos/arm-none-linux-gnueabihf-5.8.1.json /Library/Developer/Destinations`

Decompression file `swift-5.8.1-runtime-arm-none-linux-gnueabihf` into `/Library/Developer/Runtimes` 

```
mkdir -p /Library/Developer/Runtimes/swift-5.8.1-runtime-arm-none-linux-gnueabihf
tar xf swift-5.8.1-runtime-arm-none-linux-gnueabihf.tar.xz -C /Library/Developer/Runtimes/swift-5.8.1-runtime-arm-none-linux-gnueabihf
```

# How to install runtime in linux target

You need to upload runtime libraries into you armv7 linux device.
```
scp -r /Library/Developer/Runtimes/swift-5.8.1-runtime-arm-none-linux-gnueabihf/usr/lib/swift/linux <username>@<target_host>:<install_path>

```

Login to your target device, add `<install_path>` to you ldconfig search path, than use `ldconfig` to update


# How to build

Create an example project

```
swift package init --type executable
swift build --destination /Library/Developer/Destinations/arm-none-linux-gnueabihf-5.8.1.json
```

After building success, you can upload binary to you target device, and run it.


--------------
`Have fun and play with Swift!`
