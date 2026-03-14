# ARM64 (aarch64) Linux 下编译 Flutter Android APK 完整指南

> 在 ARM64 嵌入式 Linux（如 Debian/Ubuntu aarch64）上从零搭建 Flutter Android 编译环境。
> 经验证环境：Debian 11 bullseye / aarch64 / OpenJDK 17 / Flutter 3.41.x

---

## 1. 前置条件

### 1.1 硬件要求

- ARM64 (aarch64) 处理器（全志 A733 / RK3588 等均可）
- 至少 2GB RAM + 2GB swap（低内存设备必须配置 swap，详见 1.3）
- 推荐 4GB+ RAM 以获得流畅编译体验
- 至少 10GB 可用磁盘空间（SDK + Gradle 缓存 + 编译产物）

### 1.2 系统依赖

```bash
sudo apt update
sudo apt install -y \
  git curl unzip xz-utils zip \
  openjdk-17-jdk \
  clang cmake ninja-build pkg-config \
  libgtk-3-dev liblzma-dev libstdc++-10-dev \
  qemu-user-static
```

> **关键：为什么需要 qemu-user-static？**
>
> Android SDK build-tools（`aapt2`、`adb`、`zipalign` 等）**只提供 x86-64 预编译二进制**，没有原生 ARM64 版本。
> `qemu-user-static` 通过 binfmt_misc 内核机制，让系统透明地用 QEMU 用户态模拟运行这些 x86-64 工具，
> 无需手动配置，安装后即生效。
>
> 验证：
> ```bash
> file ~/Android/build-tools/*/aapt2
> # 应显示: ELF 64-bit LSB ... x86-64
>
> ~/Android/build-tools/36.0.0/aapt2 version
> # 如果能输出版本号，说明 QEMU 模拟正常工作
> ```
>
> **注意**：QEMU 模拟会带来约 3-5 倍的性能损耗，这也是 ARM64 上编译比 x86 慢的主要原因之一。

### 1.3 配置 swap（低内存设备必须）

RAM 低于 4GB 的设备，Gradle 编译时极易 OOM。**强烈建议配置 swap**：

```bash
# 创建 2GB swap 文件
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# 验证
free -h
# 应看到 Swap 行有 2.0Gi

# 写入 fstab 使其开机自动挂载
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

> 如果磁盘是 eMMC/SD 卡，swap 会加速磨损，编译完成后可 `sudo swapoff /swapfile` 关闭。

验证 Java：

```bash
java -version
# 应显示: openjdk version "17.x.x"
```

---

## 2. 安装 Flutter SDK

### 2.1 下载

Flutter 官方提供 linux-arm64 版本：

```bash
cd ~
# 从官方下载 stable 最新版（以 3.41.4 为例）
curl -LO https://storage.googleapis.com/flutter_infra_release/releases/stable/linux/flutter_linux_arm64_3.41.4-stable.tar.xz

# 或直接 clone
git clone https://github.com/flutter/flutter.git -b stable ~/flutter-sdk
```

### 2.2 解压与配置 PATH

```bash
# 如果是 tar.xz
tar xf flutter_linux_arm64_*.tar.xz -C ~/
mv ~/flutter ~/flutter-sdk  # 可选，统一路径

# 配置环境变量（写入 ~/.bashrc 或 ~/.profile）
export FLUTTER_HOME="$HOME/flutter-sdk"
export PATH="$FLUTTER_HOME/bin:$PATH"
```

### 2.3 验证

```bash
flutter --version
# Flutter 3.41.4 • channel stable
# Tools • Dart 3.11.1

flutter doctor
```

---

## 3. 安装 Android SDK（无 Android Studio）

ARM64 Linux 没有 Android Studio，需要手动安装 command-line tools。

### 3.1 下载 Command-Line Tools

```bash
mkdir -p ~/Android/cmdline-tools
cd ~/Android/cmdline-tools

# 下载最新版 commandlinetools-linux（通用，支持 aarch64）
curl -LO https://dl.google.com/android/repository/commandlinetools-linux-11076708_latest.zip
unzip commandlinetools-linux-*.zip
mv cmdline-tools latest  # 必须改名为 latest
rm commandlinetools-linux-*.zip
```

### 3.2 配置环境变量

```bash
# 写入 ~/.bashrc
export ANDROID_HOME="$HOME/Android"
export ANDROID_SDK_ROOT="$ANDROID_HOME"
export PATH="$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/platform-tools:$PATH"
```

```bash
source ~/.bashrc
```

### 3.3 安装 SDK 组件

```bash
# 接受许可证
yes | sdkmanager --licenses

# 安装必要组件
sdkmanager \
  "platforms;android-34" \
  "build-tools;34.0.0" \
  "platform-tools"

# 如果项目需要更高版本（如 compileSdk 36）
sdkmanager "platforms;android-36" "build-tools;36.0.0"
```

### 3.4 配置 Flutter 指向 Android SDK

```bash
flutter config --android-sdk ~/Android
flutter doctor --android-licenses
```

---

## 4. 项目配置

### 4.1 local.properties

在 `android/local.properties` 中确保路径正确：

```properties
sdk.dir=/home/<你的用户名>/Android
flutter.sdk=/home/<你的用户名>/flutter-sdk
flutter.buildMode=release
```

### 4.2 Gradle 配置

**android/gradle/wrapper/gradle-wrapper.properties**:

```properties
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-8.4-all.zip
```

> Gradle 8.4 经验证可用。更高版本需确认 AGP 兼容性。

**android/gradle.properties**（内存优化，ARM64 设备通常内存有限）:

```properties
# 4GB+ RAM 设备
org.gradle.jvmargs=-Xmx1536M -XX:MaxMetaspaceSize=512M -XX:+HeapDumpOnOutOfMemoryError

# 4GB 以下 RAM 设备，使用更保守的参数：
# org.gradle.jvmargs=-Xmx1024M -XX:MaxMetaspaceSize=256M -XX:+HeapDumpOnOutOfMemoryError

android.useAndroidX=true
android.enableJetifier=true
```

### 4.3 settings.gradle（含国内镜像加速）

ARM64 设备通常网络受限，配置阿里云镜像可大幅加速依赖下载：

```groovy
pluginManagement {
    def flutterSdkPath = {
        def properties = new Properties()
        file('local.properties').withInputStream { properties.load(it) }
        def flutterSdkPath = properties.getProperty('flutter.sdk')
        assert flutterSdkPath != null, 'flutter.sdk not set in local.properties'
        return flutterSdkPath
    }()

    includeBuild("$flutterSdkPath/packages/flutter_tools/gradle")

    repositories {
        maven { url 'https://maven.aliyun.com/repository/gradle-plugin' }
        maven { url 'https://maven.aliyun.com/repository/google' }
        maven { url 'https://maven.aliyun.com/repository/central' }
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}

plugins {
    id 'dev.flutter.flutter-plugin-loader' version '1.0.0'
    id 'com.android.application' version '8.3.2' apply false
    id 'org.jetbrains.kotlin.android' version '1.9.22' apply false
}

dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.PREFER_SETTINGS)
    repositories {
        maven { url 'https://maven.aliyun.com/repository/google' }
        maven { url 'https://maven.aliyun.com/repository/central' }
        maven { url 'https://storage.googleapis.com/download.flutter.io' }
        google()
        mavenCentral()
    }
}

include ':app'
```

### 4.4 app/build.gradle 关键配置

```groovy
android {
    namespace = "com.example.myapp"
    compileSdk = 34                    // 或 36
    ndkVersion = flutter.ndkVersion

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }

    kotlinOptions {
        jvmTarget = JavaVersion.VERSION_17
    }

    defaultConfig {
        applicationId = "com.example.myapp"
        minSdk = 24                    // Android 7.0+
        targetSdk = 34                 // 或 36
        versionCode = flutter.versionCode
        versionName = flutter.versionName
    }

    buildTypes {
        release {
            signingConfig = signingConfigs.debug  // 测试用；生产需配置正式签名
        }
    }
}
```

---

## 5. 编译 APK

### 5.1 标准编译

```bash
cd /path/to/your/flutter/project

# 获取依赖
flutter pub get

# 静态分析（可选但推荐）
flutter analyze

# 运行测试（可选）
flutter test

# 编译 release APK
flutter build apk --release
```

产物路径：`build/app/outputs/flutter-apk/app-release.apk`

### 5.2 跳过构建依赖校验

如果遇到 Gradle/Kotlin/AGP 版本 warning：

```bash
flutter build apk --release --android-skip-build-dependency-validation
```

### 5.3 指定 target 架构（可选）

```bash
# 只编译 arm64（减小 APK 体积）
flutter build apk --release --target-platform android-arm64

# 编译 split APK（按架构分包）
flutter build apk --release --split-per-abi
```

---

## 6. 常见问题与解决方案

### 6.1 Gradle Daemon 崩溃（No space left on device）

ARM64 嵌入式设备磁盘通常很小。

```bash
# 查看磁盘
df -h /

# 清理 Gradle 缓存
rm -rf ~/.gradle/caches/transforms-*
rm -rf ~/.gradle/caches/build-cache-*
rm -rf ~/.gradle/daemon/*/daemon-*.out.log

# 清理 Flutter 编译产物
flutter clean
# 或手动
rm -rf build/

# 清理旧 Gradle daemon
cd android && ./gradlew --stop && cd ..
```

**预防措施**：在 `gradle.properties` 中限制内存：

```properties
org.gradle.jvmargs=-Xmx1024M -XX:MaxMetaspaceSize=256M
```

### 6.2 Gradle Daemon 消失（disappeared unexpectedly）

通常是旧 daemon 进程僵死：

```bash
# 停止所有 Gradle daemon
cd android && ./gradlew --stop && cd ..

# 如果还不行，杀掉所有 Java 进程
pkill -f gradle

# 重新编译
flutter build apk --release
```

### 6.3 flutter 命令找不到

ARM64 系统可能没有将 flutter 加入 PATH：

```bash
# 直接用绝对路径
/home/<用户>/flutter-sdk/bin/flutter build apk --release

# 或创建符号链接
sudo ln -s /home/<用户>/flutter-sdk/bin/flutter /usr/local/bin/flutter
```

### 6.4 sdkmanager 下载超时

国内网络环境下，配置代理或使用镜像：

```bash
# 设置 Flutter 国内镜像
export FLUTTER_STORAGE_BASE_URL=https://storage.flutter-io.cn
export PUB_HOSTED_URL=https://pub.flutter-io.cn

# Gradle 使用阿里云镜像（已在 settings.gradle 中配置）
```

### 6.5 首次编译极慢

ARM64 设备 CPU 性能有限，首次编译需要下载 + 编译所有依赖：
- **高性能 SoC**（RK3588 4xA76+4xA55 / 全志 A733 2xA76+6xA55）：15-30 分钟
- **低端 SoC**（纯 Cortex-A55 等）：30-60 分钟甚至更久

后续增量编译会快很多。

如果 Gradle daemon 空闲太久被回收，可能需要重新初始化：

```bash
# 预热 Gradle
cd android && ./gradlew tasks && cd ..

# 然后编译
flutter build apk --release
```

### 6.6 CupertinoIcons 字体缺失 warning

如果使用了 Cupertino 风格的 Widget，需要在 `pubspec.yaml` 中添加依赖：

```yaml
dependencies:
  flutter:
    sdk: flutter
  cupertino_icons: ^1.0.8
```

如果项目中没有使用任何 `CupertinoIcons`，可以忽略此 warning。

### 6.7 编译时 OOM（Out of Memory）

低内存设备编译大项目时 Gradle 或 Dart compiler 被 OOM killer 杀掉：

```bash
# 检查是否被 OOM kill
dmesg | grep -i "oom\|kill"

# 解决方案：
# 1. 确保已配置 swap（见 1.3）
# 2. 降低 Gradle JVM 内存（见 4.2）
# 3. 关闭其他占内存的进程再编译
# 4. 使用 --no-daemon 避免 daemon 常驻
cd android && ./gradlew assembleRelease --no-daemon && cd ..
```

> QEMU 模拟会额外增加内存开销（每个 x86 进程都需要翻译缓存），OOM 阈值比纯原生环境更低。

### 6.8 QEMU 模拟相关问题

ARM64 上编译 Flutter 依赖 `qemu-user-static` 透明模拟 Android SDK 的 x86-64 工具，
这是**隐性依赖**，容易遗漏导致莫名失败。

**症状：aapt2 / adb 等工具报 "Exec format error"**

```bash
# 检查 binfmt 是否注册了 x86-64
ls /proc/sys/fs/binfmt_misc/qemu-x86_64
# 应存在此文件

# 如果不存在，重新注册
sudo systemctl restart binfmt-support

# 验证 QEMU 能正常模拟
file ~/Android/build-tools/36.0.0/aapt2   # 应显示 x86-64
~/Android/build-tools/36.0.0/aapt2 version  # 应能输出版本号
```

**症状：flutter analyze 被 signal -9 杀掉**

Dart analysis server 在 QEMU 模拟下内存翻倍膨胀，容易触发 OOM killer：

```bash
# 跳过 analyze 直接编译
flutter build apk --release

# 或限制 Dart analyzer 内存
export DART_VM_OPTIONS="--old_gen_heap_size=512"
flutter analyze
```

**性能提示**：QEMU 用户态模拟会让 aapt2 等工具慢 3-5 倍。
如果有条件，可在 x86-64 机器上交叉编译 APK 再传到 ARM64 设备部署，效率更高。

---

保存为 `build_apk.sh`：

```bash
#!/bin/bash
set -e

# 环境配置
export JAVA_HOME="/usr/lib/jvm/java-17-openjdk-arm64"
export ANDROID_HOME="$HOME/Android"
export ANDROID_SDK_ROOT="$ANDROID_HOME"
export FLUTTER_HOME="$HOME/flutter-sdk"
export PATH="$FLUTTER_HOME/bin:$JAVA_HOME/bin:$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/platform-tools:$PATH"

echo "=== 环境检查 ==="
flutter --version
java -version

# 检查 swap（低内存设备）
TOTAL_RAM_MB=$(free -m | awk '/Mem:/{print $2}')
SWAP_MB=$(free -m | awk '/Swap:/{print $2}')
if [ "$TOTAL_RAM_MB" -lt 4096 ] && [ "$SWAP_MB" -lt 1024 ]; then
    echo "WARNING: RAM ${TOTAL_RAM_MB}MB < 4GB 且 swap ${SWAP_MB}MB < 1GB，编译可能 OOM"
    echo "建议配置 swap，参考文档 1.3 节"
fi

echo "=== 获取依赖 ==="
flutter pub get

echo "=== 静态分析 ==="
flutter analyze || echo "WARNING: analyze 发现问题，继续编译"

echo "=== 运行测试 ==="
flutter test || echo "WARNING: 部分测试失败"

echo "=== 编译 Release APK ==="
flutter build apk --release --android-skip-build-dependency-validation

APK_PATH="build/app/outputs/flutter-apk/app-release.apk"
if [ -f "$APK_PATH" ]; then
    SIZE=$(du -h "$APK_PATH" | cut -f1)
    echo "=== 编译成功: $APK_PATH ($SIZE) ==="
else
    echo "=== 编译失败 ==="
    exit 1
fi
```

```bash
chmod +x build_apk.sh
./build_apk.sh
```

---

## 8. 经验证的版本组合

以下版本组合在 Debian 11 aarch64 上经过实际验证：

| 组件 | 版本 | 备注 |
|------|------|------|
| Linux | Debian 11 bullseye | aarch64 |
| SoC | [全志 A733](https://www.allwinnertech.com/index.php?c=product&id=139) (2xA76 + 6xA55, PowerVR BXM-4-64 GPU, 3Tops NPU) | Radxa Cubie A7A |
| JDK | OpenJDK 17.0.18 | arm64 版 |
| Flutter | 3.41.4 stable | linux-arm64 |
| Dart | 3.11.1 | 随 Flutter 自带 |
| Gradle | 8.4 | gradle-wrapper |
| AGP | 8.3.2 | Android Gradle Plugin |
| Kotlin | 1.9.22 | Kotlin Android Plugin |
| Android SDK | platforms 34/36 | build-tools 34/36 |
| compileSdk | 34 或 36 | 均可 |
| minSdk | 24 | Android 7.0+ |

---

## 9. 磁盘空间参考

| 目录 | 大小 | 说明 |
|------|------|------|
| flutter-sdk/ | ~1.5GB | Flutter SDK |
| Android/ | ~2GB | Android SDK + build-tools |
| ~/.gradle/ | ~2-4GB | Gradle 缓存（可清理） |
| build/ | ~0.5-1GB | 编译产物（flutter clean 可清） |
| **合计** | **~6-8GB** | 最小可用磁盘建议 10GB |

> 提示：嵌入式设备磁盘紧张时，编译完成后执行 `flutter clean` 和清理 Gradle 缓存回收空间。

---

*最后更新: 2026-03-14*
*验证平台: Debian 11 aarch64 / [全志 A733](https://www.allwinnertech.com/index.php?c=product&id=139) (2xA76 + 6xA55) / Radxa Cubie A7A*
