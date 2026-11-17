# Formula 适配最佳实践

## 1 前言

本文介绍“在 OpenHarmony 上适配一个 upstream formula”的技术写法。

本文的目标不是教你“让某个包能编过”，而是让适配后的 formula 满足以下三条底线：

1. **可维护**：进行上游同步时冲突尽可能少，后来者能看懂为什么这么改。不引入不必要的运行时依赖（尤其是 gcc 运行时）。
2. **可信**：不掩盖问题。不绕过网络、不吞测试失败、不伪造构建或测试结果。
3. **合规**：依赖和代码来源可追溯。新增依赖必须显式声明，不引入未声明来源的代码，且不违反上游许可证。

本文会随逐轮检视持续更新。当维护者给出的检视意见与本文规范冲突时，以检视意见为准。

## 2 适配可行性判断

在动手之前先判断这个包是否适合适配，避免在无法适配的包上投入时间。

**可以直接放弃的情况：**

- 强依赖 Linux 内核专有接口、systemd、特权操作、GUI 框架（GTK/Qt 等），且短期内无法提供等价能力。
- 依赖闭包中存在无法适配、且无替代方案的软件包。
- 上游已经废弃，或被其他 formula 取代，或上游 formula 已标记 `deprecate!` / `disable!`。
- 上游源码已经无法获取（主页和 tarball 均 404）。

**需要谨慎的情况：**

- 包本身能用，但测试强依赖真实网络（如在线视频、云服务 API）。测试可以通过 `allow_network_access!` 声明，但不得为了“让它过”而吞掉错误。
- 包依赖一个尚未适配的级联依赖。此时应先修级联依赖，而不是在本 formula 里绕过去（见 3.3）。

无法适配的包不要硬塞 workaround。一个带病提交的 formula 比没有这个 formula 更糟。

## 3 依赖处理

### 3.1 不要通过裁剪功能来规避依赖

upstream 在 macOS/Linux 上启用的可选能力，OpenHarmony 上也应尽量保留。常见做法是补齐依赖，而不是加 `--disable-xxx`：

```ruby
depends_on "gettext" if OS.ohos?      # Formula/f/flex.rb、Formula/g/gnutls.rb
depends_on "util-linux" if OS.ohos?   # Formula/m/mosquitto.rb、Formula/d/drogon.rb
depends_on "libomp" if OS.ohos?       # Formula/m/msolve.rb、Formula/m/mmseqs2.rb
```

只有当功能确实依赖 OpenHarmony 不具备的能力（如 GPU、特定内核接口）时，才关闭对应选项，并在注释中写明原因。

### 3.2 依赖库带 libintl 符号时

OpenHarmony 的 lld 默认带 `--no-allow-shlib-undefined`。如果依赖的 `.so` 引用了 gettext 的 `libintl` 符号（例如 `util-linux` 的 `libblkid`），只加 `depends_on "gettext"` 还不够：configure 的 `-lblkid` 探测仍然会因为 `libintl_gettext` 未定义而失败。

需要显式让链接器带上 `-lintl`：

```ruby
depends_on "gettext" if OS.ohos?
```

```ruby
ENV.append "LIBS", "-lintl" if OS.ohos?
system "./configure", *args
```

参考实现：`Formula/l/linux-pam.rb`（显式追加 `-lintl`）。

### 3.3 修级联依赖，而不是在消费者里绕过

如果 `A` 依赖 `B`，而 `B` 在 OpenHarmony 上有缺陷，正确做法是修 `B`，然后重新构建 `A`。

典型场景：

- 解释器（`lua`、`perl`、`ruby`、`python`、`zsh`）的符号可见性问题，应修解释器本身，而不是给每个消费者打补丁（见 5.3）。
- `openjdk` 的 JVM 行为问题，应修 `openjdk`，而不是给 `hbase`、`solr` 之类的消费者加 workaround。
- 某个库的错误行为应由该库的 formula 修复，消费者只做版本升级。

### 3.4 不要为编译器版本问题引入 gcc

`depends_on "gcc"` 会让产物在运行态依赖 GCC 运行时库（`libstdc++`、`libgcc_s`），进而把整个 GCC 拖进依赖闭包。因此不要为了让旧源码通过新标准检查而新增 gcc 依赖。

这条限制只针对新增的 gcc 依赖。如果 upstream formula 本身已经 `depends_on "gcc"`，应保留上游行为，不要在本仓库里移除。

遇到“SDK 的 Clang 版本太旧、编不过新标准”的情况，优先级为：

1. 用补丁让源码兼容当前 Clang（优先，且不引入运行时依赖）。
2. 用仓库内已适配的 `llvm` 编译，并链接它提供的私有 **`libc++_shared.so`**，不要默认把 libc++ 静态塞进每个产物。
3. 暂时放弃该包。

不要用 `depends_on "gcc" => :build` 绕过编译器版本问题。

**用 llvm + 私有 `libc++_shared.so` 解决 C++20/23 需求**

OHOS SDK 的 Clang 15 不认 `-std=c++23`，libc++ 15 也缺 `<expected>`、`<memory_resource>` 等头文件。仓库里的 `llvm` formula（bottled）提供 Clang 23 和完整的 libc++。

默认不要把 libc++ 静态链接进每个可执行文件。实际程序很少是完全静态的单体，插件、语言运行时和共享库都可能加载外部 `.so`。静态链接会在同一进程里塞进多份 libc++，运行时的 RTTI、异常和分配器状态很容易不一致。

仓库里的 `llvm` 现在已经同时安装：

```text
lib/aarch64-linux-ohos/libc++.a          # 静态 linker script
lib/aarch64-linux-ohos/libc++_static.a
lib/aarch64-linux-ohos/libc++abi.a
lib/aarch64-linux-ohos/libunwind.a
lib/aarch64-linux-ohos/libc++_shared.so
```

它**不安装** `libc++.so` / `libc++.so.1`，所以默认 `-lc++` 仍然命中静态 `libc++.a`，现有静态下游不受影响。

需要 shared 的下游统一从 `llvm` 复制私有副本：

```ruby
depends_on "llvm" => :build if OS.ohos?

def install
  if OS.ohos?
    llvm = formula_opt_prefix("llvm")
    libcxx = llvm/"lib/aarch64-linux-ohos/libc++_shared.so"
    odie "llvm is missing libc++_shared.so" unless libcxx.exist?

    libdir = libexec/"libcxx"
    libdir.mkpath
    cp libcxx, libdir/"libc++_shared.so"
    ENV.append "LDFLAGS",
               "-nostdlib++ -L#{libdir} -Wl,-rpath,#{libdir} " \
               "-Wl,--no-as-needed -lc++_shared"
  end
  system "make", ...
end
```

要点：

- 复制到产物自己的 `libexec` / `lib/<name>`，RPATH 指向该目录，不要依赖 `opt/llvm`，因为 llvm 是 `:build` 依赖。
- 链接时用 `-nostdlib++` 抑制 clang 默认的 `-lc++`，再显式 `-lc++_shared`。
- 默认不要直接链接 `/lib64/libc++_shared.so`：系统那份的版本和 ABI 由设备决定，优先使用 llvm 提供的私有副本。
- 例外：如果产物链接 Homebrew 的 `libLLVM.so`，`libLLVM` 本身 NEEDED 系统 libc++，此时必须使用同一份 `/lib64/libc++_shared.so`，否则同一进程会出现两套 libc++，RTTI、异常和分配器状态会不一致。

两个常见坑：

- 只有把 `CXX` 显式设为 llvm keg 的绝对路径（例如 `/opt/llvm/bin/clang++`）时，这个调用才绕过 superenv，依赖的 `-I`/`-L` 路径会丢失，需要用 `CPATH`/`LIBRARY_PATH` 补回来。使用默认的 `cc` / `c++` 时仍然经过 superenv，并通过 `PATH` 选中引入的 llvm。
- libc++ 对头文件更严格，GCC 下能隐式找到的 `std::abs` 之类需要显式 `-include cmath`，或给源码补 `#include <cmath>`。

### 3.5 测试依赖用 `:test`

测试需要系统工具时，补一个测试依赖即可，不要改测试逻辑去规避：

```ruby
depends_on "util-linux" => :test if OS.ohos?
```

### 3.6 toybox 的 `find`/`xargs` 不兼容时，统一引入 GNU findutils

OpenHarmony 的 toybox 提供的 `find`/`xargs` 在 `-printf`、`-regex`、`--no-run-if-empty` 等行为上与 GNU 版本有差异。不要为每个包写 toybox 适配补丁或改测试，统一下沉到 `findutils` 依赖：

```ruby
# 只在构建/安装期用到
depends_on "findutils" => :build if OS.ohos?

# 安装后由随包脚本在运行时调用（例如 gradlew、bin/grails）
depends_on "findutils" if OS.ohos?
```

`findutils` 在 OHOS 上安装的是无前缀的 `find`/`xargs`，且非 keg-only，构建期 PATH 直接生效。构建期依赖在 bottle 安装后会被 `brew autoremove` 清掉，所以运行时也要用的包必须用普通依赖。

## 4 formula 写法

### 4.1 改动上游语句时，用 `if OS.ohos? ... else 原样 end`

OpenHarmony 在编译器视角是 `<arch>-linux-ohos` 的 Linux 发行版。在 Formula 中，`OS.linux?` 和 `OS.ohos?` 都返回 true。所有 OHOS 专属差异必须用 `if OS.ohos?` 或 `unless OS.ohos?` 保护，并提供配套注释，不能影响 Linux 或全局路径。

**错误写法**——把 upstream 的语句拆开改写：

```ruby
args = ["--disable-silent-rules"]
args << "--disable-examples" if OS.ohos?
system "./configure", *args, *std_configure_args
```

**正确写法**——upstream 语句原样保留在 `else` 分支：

```ruby
if OS.ohos?
  system "./configure", "--disable-silent-rules", "--disable-examples", *std_configure_args
else
  system "./configure", "--disable-silent-rules", *std_configure_args
end
```

这样上游同步改到这一行时，`else` 分支仍然能与上游对齐，冲突范围可控。

`brew style` 对分支写法有两条约束，落笔前先想清楚：

- `else` 里如果只有一行赋值，可能触发 `Style/ConditionalAssignment`，此时给变量在 `if` 前补一个默认值，但 `else` 里仍必须是上游原句。
- 不要用 modifier 形式 `... end unless OS.ohos?`：多行 modifier 会被 `Style/MultilineIfModifier` 拒绝。

**整块内容在 OHOS 上不适用时，用 `unless OS.ohos?` 包块**

`fails_with :clang do ... end` 这类整块内容在 OpenHarmony 上会被 SDK 的 clang 15 误触发，直接包块，不要写空分支：

```ruby
unless OS.ohos?
  fails_with :clang do
    ...
  end
end
```

```ruby
# 错误：空分支
if OS.ohos?
  # OHOS 不需要 fails_with
else
  fails_with :clang do
    ...
  end
end
```

注意：`service do ... end` 不要用这种方式屏蔽，保持 upstream 原样，原因见 10.2。

同样的规则适用于 `test do` 块。需要给出不同的断言时用 if/else，两边都有实体内容：

```ruby
if OS.ohos?
  assert_match "no init system", shell_output("#{bin}/foo status")
else
  assert_equal "Permission denied", shell_output(testpath/"test").chomp
end
```

如果某条断言在 OHOS 上根本跑不了、也没有替代断言，用 `unless OS.ohos?` 包住它，不要留空分支。

### 4.2 用三元表达式和插值缩小 diff

能用一行表达式表达的差异，不要拆成两个分支：

```ruby
wasm-bindgen = "#{OS.ohos? ? "=" : ""}#{version}"
```

```ruby
ldflags = %W[
  -X oras.land/oras/internal/version.Version=#{version}
  -X oras.land/oras/internal/version.BuildMetadata=#{OS.ohos? ? "Homebrew" : tap.user}
]
```

**长参数列表只追加少量可选参数时，用条件 splat**

如果上游是一条很长的 `system` 调用，OHOS 只需要追加少量参数，不要为了这几个参数把整条命令拆成 `if/else` 两个分支。可以把可选参数写成条件数组再 splat：

```ruby
system "make", "install", "GNUSTEP_HEADERS=#{include}",
                          "GNUSTEP_LIBRARY=#{share}",
                          "GNUSTEP_LOCAL_DOC_MAN=#{man}",
                          "GNUSTEP_LOCAL_LIBRARIES=#{lib}",
                          *(["GNUSTEP_SKIP_ETC_SERVICES=1"] if OS.ohos?),
                          "GNUSTEP_LOCAL_TOOLS=#{bin}"
```

优点：

- 上游参数列表保持原样，只多一行
- `if OS.ohos?` 为假时表达式是 `nil`，`*nil` 不会追加任何参数
- `brew style` 可以通过。

适用边界：

- 只适合**追加**少量参数
- 如果 OHOS 需要替换、删除或重排多个上游参数，仍然用 `if OS.ohos? ... else 上游原样 end`，不要用 splat 拼装。

参考实现：`Formula/g/gnustep-base.rb` 的 `GNUSTEP_SKIP_ETC_SERVICES=1`。

**分支顺序：OHOS 分支尽量放在最后**

如果上游已经有 `if OS.mac? ... else ... end`，优先追加 `elsif OS.ohos?`，而不是在最前面插一个 `if OS.ohos?` 再 `else` 抄一遍上游：

```ruby
if OS.mac?
  # 上游原文
elsif OS.ohos?
  # OHOS 专用
else
  # 上游原文
end
```

这样 `if OS.mac?` 和 `else` 两行都不用动，diff 更小。

### 4.3 heredoc 里用模板插值

需要生成配置文件时，不要复制一整段几乎相同的 heredoc，用插值：

```ruby
(etc/"app.conf").write <<~EOS
  port = 8080
  #{"locking = posix_sem" unless OS.ohos?}
EOS
```

### 4.4 使用 `formula_opt_*` 辅助方法

引用其他 formula 的路径时，使用仓库提供的辅助方法，不要写 `Formula["x"].opt_bin`：

```ruby
formula_opt_prefix("openssl@4")
formula_opt_bin("openjdk")
formula_opt_lib("llvm")
formula_opt_include("curl")
formula_opt_libexec("openjdk")
```

这些方法定义在 `Library/Homebrew/utils/path.rb`，相比 `Formula[...]` 更轻量，也不会意外触发 formula 加载。

**只在我们新增的代码里使用这些辅助方法。** 上游原本就写 `Formula["x"].opt_lib`、`Formula["x"].version` 的地方，不要顺手改成 `formula_opt_*`——那属于无必要的改写，会让 diff 变大。

### 4.5 注释规范

- 在注释和 Commit Message 中默认禁止使用 `HarmonyOS` 这个词汇。只有两种情况可以使用：1. 你解决的问题是 HarmonyOS 这个发行版特有的（例如鸿蒙内核相关问题）。2. 你需要特指华为鸿蒙 PC（HarmonyOS PC）这个设备。其他情况应使用 `OpenHarmony` 或其简称 `OHOS`。注意：应用沙箱和 hmdfs 并非 HarmonyOS 这个发行版特有，这些机制在 OpenHarmony 中同样有实现。
- 注释只写**长期成立**的原因，例如“OpenHarmony 的 libc 不声明 `getpass()`”。
- 不要写“这里和上游不一样，因为某次 CI 失败……”这类会随时间和版本失效的叙述。
- 不要留下无信息量的注释（如 `# OHOS support`）。

### 4.6 如无特殊需求，不要新增构建期 `odie` 守卫

下游适配默认不要为了“报错更清楚”在 formula 里新增：

```ruby
odie "OHOS sysroot missing" unless ...
odie "target libc++ missing" unless ...
```

这类检查等于替构建系统做前置校验，上游 formula 很少这样做。删除后，后续的 `cp`、`cmake`、`system` 本身就会失败，堆栈也更接近真实问题。

处理原则：

- 默认直接删掉防御性断言
- 需要保证的不变量，优先用动作修正，而不是断言。例如 `libc++.so` 不应存在时，直接 `unlink`
- **上游已有的 `odie` 版本守卫不要删**，例如“升级到某版本后请移除 workaround”。

只有在确有特殊需求时才保留新增 `odie`，例如：

- 检查的是构建系统之外的强制前置条件，失败信息会非常隐蔽
- 错误状态会静默改变 ABI 或产物布局，且无法用修正动作处理
- 维护者在 review 中明确要求保留。

保留时要在注释里说明为什么普通构建失败不够，并把检查范围限制到最小，不要对每个路径、每个资源都加一遍。


### 4.7 不要引入冗余修改

提交前必须将当前 formula 与上游 Homebrew/homebrew-core 的版本做 diff，逐项确认每一处差异的必要性。适配目标是最小差异，不是让包“能编过”。

重点检查以下冗余修改：

- 无意义的构建参数和环境变量。
- 无意义的级联依赖。
- 重复声明 ci-runner 镜像已预装的 make、gawk 等通用构建工具。
- 上游未启用、且不是 OHOS 适配必需的构建选项。
- 只在构建期生效的鸿蒙内核适配项。
- 与 OHOS 适配无关的功能增强。
- 网络规避。默认不改，只有维护者确认 CI 无法访问具体地址时，才按 10.1 的受控特例处理，并注明来源和影响范围。

所有 OHOS 差异都应能说明原因，注释按 4.5 写长期成立的理由。

### 4.8 自带运行时的 RPATH

Rust、Go、Node 的产物如果自带运行时目录（`libexec`），RPATH 要指向 keg 内的私有库，而不是系统路径。

## 5 C/C++ 与平台适配

### 5.1 使用 `__OHOS__` 宏

OHOS 的 clang 在编译 C、C++ 和 Objective-C 源码时会自动定义 `__OHOS__`，可用于源码中的平台判断。

常见用途：

- 系统头文件或 libc 函数差异。
- 编译器版本或语言特性差异。
- 平台特有的业务行为。

例如：

```c
#if defined(__OHOS__)
  /* OHOS-specific implementation */
#endif
```

使用原则：

- 源码里的平台判断用 `__OHOS__`。
- formula 里的依赖、构建选项和文件选择用 `OS.ohos?`。
- 优先复用 upstream 已有的平台条件分支，不要为了一行差异重写整段逻辑。
- 不要把 `__OHOS__` 用在不需要平台差异的代码上，也不要用它伪装其他平台的运行时行为。

### 5.2 `__MUSL__` 与 `sockaddr_storage` 重定义

OHOS sysroot 的 `<sys/socket.h>` 有两条 `sockaddr_storage` 定义路径：

- 定义了 `__MUSL__` 时，它包含 `<linux/socket.h>`，使用 UAPI 里的 `struct sockaddr_storage`，并跳过自己的定义
- 未定义 `__MUSL__` 时，它不包含 `<linux/socket.h>`，而是自己定义 `struct sockaddr_storage`。

因此，同一个编译单元同时包含 `<sys/socket.h>` 和 `<linux/socket.h>` 时：

- 没有 `__MUSL__`：两个头各定义一次 `struct sockaddr_storage`，产生 redefinition
- 有 `__MUSL__`：`<sys/socket.h>` 只使用 UAPI 定义，不再重复定义。

Harmonybrew 在 superenv 中统一注入 `-D__MUSL__`，就是为了让大多数构建走第二条路径，减少这类重定义。但 superenv 只在编译器经过 shim 时生效，以下场景仍可能覆盖不到。

`__MUSL__` 不是 OHOS clang 的预定义宏，而是 Homebrew 在编译器 shim 中额外传入的定义。因此只有经过 shim 的编译才会带上它。以下场景需要显式补：

1. **构建系统绕过 shim**：SCons、部分 autotools/CMake 配置会直接调用 `/usr/bin/clang` 或缓存绝对编译器路径，实际命令行中没有 `-D__MUSL__`。这时应在 formula 中显式补：
   ```ruby
   ENV.append "CPPFLAGS", "-D__MUSL__" if OS.ohos?
   ```
   参考 `gpsd` 的 SCons 构建。

2. **`brew test` 使用标准环境**：`brew test` 不会重新注入 superenv 的编译参数。测试里用 `ENV.cxx` 直接编译示例时，需要自己补：
   ```ruby
   args << "-D__MUSL__" if OS.ohos?
   ```
   参考 `libtorrent-rasterbar` 的 test 块。

3. **软件自带同名 UAPI 头**：例如 strongSwan 自带 `src/include/linux/socket.h`，`src/include` 在系统头之前，遮蔽了 sysroot 的版本。它只定义 `__kernel_sockaddr_storage`，不定义 `sockaddr_storage`，宏无法解决。需要针对自带头的补丁，参考 `Patches/strongswan/0001-ohos-sockaddr-storage.patch`。

4. **`netinet/in.h` 不是自包含的**：OHOS 的 `netinet/in.h` 直接使用 `struct sockaddr_storage`，但自己不包含 `<sys/socket.h>` 或 `<linux/socket.h>`。如果它先于 socket 头被解析，即使加了 `-D__MUSL__`，也只会得到 incomplete type。此时需要调整 include 顺序或补头文件，而不是只加宏。

判断顺序：

- 先看真实编译命令里有没有 `-D__MUSL__`，不要只看 formula 代码
- 再确认 `sockaddr_storage` 是在哪个头、哪一行被定义或使用
- 最后决定是补编译参数、调整 include 顺序，还是给自带头打补丁。

不要机械地为所有 `sockaddr_storage` 冲突写 `#include <net/if.h>` 守卫，也不要机械地认为 superenv 一定已注入宏。

### 5.3 符号命名空间：`-Wl,-z,global`

OpenHarmony 的链接器使用 namespaced symbol resolution：通过 `dlopen` 加载的模块无法反解析主程序中的符号。解释器、插件宿主类软件需要显式导出符号：

```ruby
ENV.append "LDFLAGS", "-Wl,-z,global" if OS.ohos?
```

参考实现：`Formula/z/zsh.rb`、`Formula/p/python@3.15.rb`、`Formula/o/openjdk.rb`。

这类问题**必须修宿主 formula**。例如 Lua 的 C 模块无法解析 Lua 符号，应修 `Formula/l/lua.rb`，而不是给 `luarocks`、`lpeg` 等每个消费者打补丁。

**加之前先确认真的需要。** 只有“通过 `dlopen` 加载的模块要反解析宿主符号”时才需要这个 flag。如果插件只通过宿主注册的结构体/回调通信（`weechat`），或者插件自己链接了定义符号的库（`syslog-ng`），加了就是冗余。判断方法：`readelf -sW <插件>` 看未定义符号里有没有宿主导出的 API，没有就不加。

还要区分符号提供者和消费者。`-Wl,-z,global` 只适用于当前 formula **定义**了这些符号、供插件反解析的情况。如果当前 formula 只是消费者，未定义符号来自另一个库，就不应该给自身加这个 flag，而应该直接链接提供符号的库。

例如 `libgdiplus` 调用了 `pangoft2` 的 `pango_fc_font_map_set_config`，但 configure 只链接了 pangocairo。OpenHarmony 的 lld 不会沿 `DT_NEEDED` 间接解析这个符号，所以消费方应显式链接：

```ruby
ENV.append "LIBS", "-lpangoft2-1.0" if OS.ohos?
```

给 `libgdiplus` 自身加 `-Wl,-z,global` 不会定义它没有的符号。给 `pango` 全量加 `-Wl,-z,global` 则会把 pangoft2 的符号暴露给所有消费者，影响范围过大，也不是对应修复。

判断顺序：

- 符号由当前 formula 定义，供 `dlopen` 模块反解析：给提供符号的宿主加 `-Wl,-z,global`。
- 符号由其他库定义，当前 formula 调用：直接链接定义符号的库，不要加 `-Wl,-z,global`。

### 5.4 SDK 头文件与 C23 的冲突

新版 `ohos-sdk-native` 的 sysroot 头文件里有若干 K&R 风格的声明，例如：

```c
/* <string.h> */
char *basename();
/* <time.h> */
char *asctime (const struct tm *);
```

在 C17 及以前，`()` 表示“参数未指定”，调用时参数个数不受限制。但 **C23 把 `()` 重新定义为“无参数”**。一些项目的 configure 会自动选择 `-std=gnu23`，于是 `basename(path)`、`asctime(&tm)` 这类调用直接编译失败：

```
error: too many arguments to function call, expected 0, have 1
```

处理方式：在 formula 里把编译器标准固定回 C17：

```ruby
ENV.append_to_cflags "-std=gnu17" if OS.ohos?
```

参考实现：`Formula/n/ntfs-3g.rb`、`Formula/n/ntp.rb`。

同类问题如果只出现在个别文件，也可以给源码补一个正确的函数原型。但当 sysroot 里有多处这种声明时，统一切回 C17 更省事。

### 5.5 缺失的 libc 函数

OpenHarmony 的 musl libc 缺少一部分 glibc 接口，常见的有 `getpass()`、`getutxent()`、`versionsort()`、`qsort_r()`、`fts(3)`、`argp`、`obstack`、shadow 密码接口、`error.h` 和 `program_invocation_*`。

处理顺序：

1. **可选功能**：只有功能确实无法在 OHOS 提供时，才关闭并注释原因（例如示例程序依赖 `getpass()`，可以只 `--disable-examples`）。
2. **必需接口**：优先使用仓库里的 `muslext`（见 5.6）。
3. **muslext 没有的接口**：复用 upstream 源码里已有的 musl / Android / BSD 兼容分支（把 `__OHOS__` 加进条件），或写最小补丁，不要引入第三方实现。
4. `pthread_cancel()` / `pthread_setcancelstate()` 不在 muslext 中。它们涉及线程取消语义和清理行为，不能用无操作的空实现替代，应打调用点补丁，或关闭确实依赖它的插件，并在注释中说明。

源码中的平台判断见 5.1。

**运行时已有符号、仅头文件缺失**

这里要区分“muslext 的 A 类接口”和“一次性旧别名”。

OHOS 的 musl 运行时经常导出符号，但构建用头文件或链接 stub 没有暴露。例如：

```text
pthread_setaffinity_np
getpass
getutxent
getspnam
program_invocation_short_name
```

这些符号在运行时存在，`nm -D /lib/ld-musl-aarch64.so.1` 能看到。但直接编译会缺声明或链接信息。这正是 muslext A 类包装的用途，**不要因为“运行时已经导出”就排除 muslext**。

判断顺序：

1. 该接口是否已经是 muslext 的一部分？
   - 是：优先用 muslext。A 类接口由 muslext 统一处理，多个消费者不必各自写声明或 dlsym。
2. 不是 muslext 接口，但只有一个调用点，且只是旧别名/局部差异？
   - 用源码补丁改成标准接口，或局部 `-D` 宏，不要为一个调用点扩展 muslext 的公共 API。
3. 运行时确实没有这个符号？
   - 走 5.6：muslext 有实现就用，没有就复用 upstream 源码里已有的 musl / Android / BSD 兼容分支，或写最小补丁。

`rindex` 属于第 2 类：它是 `strrchr` 的旧别名，muslext 没有这个接口，rpcsvc-proto 也只有一处调用，所以用局部宏或源码补丁更合适。

```ruby
# musl exports rindex(3) but does not declare it. This is a header
# declaration gap rather than a missing runtime interface, so use the
# standard strrchr alias instead of adding a `muslext` dependency for one
# call site.
ENV.append "CFLAGS", "-Drindex=strrchr" if OS.ohos?
```

宏是完整 token 替换，不会影响 `my_rindex`、`rindex_foo` 这类标识符，但如果有同名变量/结构体成员，还是应该用源码补丁。

写 formula 注释时也要区分：

- 运行时没有符号：写“musl libc does not export …”
- 运行时导出、只是头文件/链接 stub 缺失：写“musl libc does not declare …”或“the build headers do not expose …”。

不要把 A 类接口写成“does not export”，否则会误导后来者以为需要自己实现。

### 5.6 接入 muslext

`muslext` 为 OHOS 补齐了一部分 glibc 兼容接口。需要的 formula 加：

```ruby
# muslext supplies getpass(3); OHOS headers do not declare it.
depends_on "muslext" if OS.ohos?
```

superenv 已经提供 `HOMEBREW_PREFIX/include` 和 `HOMEBREW_PREFIX/lib`，**不要再加 `-I` / `-L` 指向 muslext**。

按需选择最小头文件，不要把 umbrella header 全局注入：

```ruby
ENV.append "CFLAGS", "-include muslext/getpass.h"
ENV.append "LDFLAGS", "-Wl,--no-as-needed -lmuslext"
ENV.append "LIBS", "-lmuslext"
```

C++ 源文件还要同时加：

```ruby
ENV.append "CXXFLAGS", "-include muslext/getpass.h"
```

常用头文件：

| 接口 | 头文件 |
| --- | --- |
| `getpass` | `muslext/getpass.h` |
| `utmpx` / `getutxent` 等 | `muslext/utmpx.h` |
| `versionsort` | `muslext/dirent.h` |
| `qsort_r` | `muslext/qsort_r.h` |
| shadow 密码接口 | `muslext/shadow.h` |
| `statx` | `muslext/statx.h` |
| `program_invocation_*` | `muslext/program.h` |
| `pthread_tryjoin_np` / `pthread_timedjoin_np` / `pthread_yield` | `muslext/pthread.h` |
| `pthread_setaffinity_np` / `pthread_getaffinity_np` | `muslext/pthread_affinity.h`，需要 `_GNU_SOURCE` |
| `__sigsetjmp` | `muslext/setjmp.h`，需要 POSIX 或 GNU feature-test 宏 |
| `error.h` / `fts.h` / `argp.h` / `obstack.h` / `sys/tree.h` | 根目录替换头，按安装路径引用 |

注意：

- **不要全局 `-include muslext/muslext.h`。** 它会把所有兼容头一起拖进来，容易和项目自带声明、CMake 探测冲突。
- muslext 头文件不会替调用方定义 `_GNU_SOURCE`。使用 `pthread_affinity.h` 或 `setjmp.h` 时，必须在包含任何系统头文件之前自行定义 `_GNU_SOURCE`（或对应的 POSIX feature-test 宏）。
- 根目录替换头（`fts.h`、`argp.h`、`obstack.h`、`error.h`、`sys/tree.h`）通常只需要链接 `-lmuslext`，不需要 `-include`。
- 强制注入声明会让 CMake / configure 的探测失真，见 5.8。
- `-Wl,--no-as-needed` 用于避免链接器把只通过 `dlsym` 使用的 muslext 当成未使用库丢掉，同时保留 `LIBS`，因为有些构建系统只认 `LIBS`。

### 5.7 独立实现 vs muslext

`muslext` 是**薄兼容层**，不是通用第三方库的替代品。判断标准不是“OHOS 上能不能编过”，而是这个接口的规模和归属。

**应该引入独立 formula：**

- 接口本身是一个完整库/子系统，有自己的 ABI、版本、配置和生命周期，例如：
  - `gettext` 提供 `libintl`
  - `argp-standalone` 提供完整 argp 解析器
  - `libiconv` 提供独立 iconv 实现
  - `util-linux` 提供一组系统库/工具。
- 上游已经有维护良好的独立实现，且 Homebrew core 已有 formula
- 多个消费者需要同一套完整行为，而不是一两个声明
- 许可证、版本和升级节奏与 muslext 不同，混在一起会增加维护负担
- 需要完整功能（argp 的 children/help/错误处理、libintl 的域/复数规则等），而不是“能链接就行”。

**应该用 muslext：**

- 只缺一个很小的 libc 兼容接口
- 运行时已有实现，只是头文件/链接 stub 没暴露（A 类）
- 或者需要一组小而独立的兼容函数（`getpass`、`utmpx`、`qsort_r`、`fts`、`obstack`）
- 没有合适的独立 formula，或者为一个函数引入完整库不划算。

**应该用补丁/局部宏：**

- 只有一个调用点
- 只是旧别名、平台条件或 configure 探测差异
- 不需要新增公共 API，例如 `rindex` → `strrchr`。

典型边界：

- `argp`：保留 `argp-standalone`，muslext 不再实现 argp，避免重复头和符号
- `gettext` / `libintl`：用 `gettext` formula，不在 muslext 里重写
- `libiconv`：用 `libiconv` 或 `uses_from_macos "libiconv"`
- `getpass` / `utmpx` / `qsort_r` / `fts` / `obstack`：用 muslext
- `rindex`：局部处理，不扩展 muslext。

原则：**能用现有独立 formula 解决的完整库，不要塞进 muslext，muslext 只负责 libc 兼容层。**

### 5.8 构建系统探测与汇编

强制头文件和 `-lmuslext` 会改变 configure / CMake 的探测结果，必须显式校正：

- **C++ 项目**：强制注入头文件时，除 `CFLAGS` 外还要同时设置 `CXXFLAGS`，CMake 项目必要时显式传 `-DCMAKE_CXX_FLAGS=...`。
- **CMake `check_function_exists`**：`-include` 进来的声明可能和探测代码冲突，导致探测失败。应显式设置 `-DHAVE_XXX=ON/OFF`，不要依赖自动探测。
- **`-lmuslext` 让链接探测成功**：如果头文件并没有声明该函数（例如 OHOS 的 `stdlib.h` 不声明 `reallocarray`），configure 可能误判为可用，随后编译报隐式声明。用 `ac_cv_func_reallocarray=no` 之类的缓存变量把结果固定下来。
- **Go cgo**：`CGO_CFLAGS` 会传给 `runtime/cgo` 的 `.S` 汇编文件，C 头文件会直接让汇编器报错。若依赖是预编译静态库，只需要 `CGO_LDFLAGS` 链接 muslext，不要为了强制头文件去改 `CGO_CFLAGS`。
- **汇编文件**：不要给 `.S` 传 `-include` C 头文件，把注入限制在 C/C++ 编译单元。

### 5.9 系统路径：证书与字体

以下路径是 OpenHarmony 系统自带的，直接用，不要为了它们引入 formula：

| 资源 | 路径 | 说明 |
| --- | --- | --- |
| CA 证书 | `/etc/ssl/certs/cacert.pem` | 仓库 `Formula/c/ca-certificates.rb` 中注明的 OpenHarmony 系统路径，`ejabberd`、`cmake` 已按此写 |
| 系统字体 | `/system/fonts` | 构建容器里**不存在**，只有运行时才有 |

因此：

- 需要 `cacerts_path` / `SSL_CERT_FILE` / `CURL_CA_BUNDLE` 时指向 `/etc/ssl/certs/cacert.pem`，不要 `depends_on "ca-certificates"`。
- 需要字体时在运行时加载 `/system/fonts`（加载前判断目录是否存在，构建期不存在也能安全跳过），不要把字体文件打进 formula，默认字体候选列表里记得加上 `HarmonyOS Sans`，否则系统字体匹配不上默认 family。

参考实现：`Formula/e/ejabberd.rb`（证书路径）。

## 6 Rust 与平台适配

### 6.1 先检查依赖

先看 `Cargo.lock`，不要一上来就改 crate：

- `libc`、`nix`、`socket2` 等依赖是否认识 `target_env = "ohos"`
- 旧版本不认识 OHOS 时，优先升级依赖版本，lock 只由 `cargo update` 生成
- 上游 lock 不能动，或升级后仍有问题，才 patch crate 源码。

### 6.2 aws-lc-sys

大量 Rust 软件通过 `rustls` 间接依赖 `aws-lc-sys`。该 crate 在 OpenHarmony 上必须走 CMake 构建路径，且 superenv 的 `-O2` 会破坏 CPU Jitter RNG 所需的 `-O0`。

标准处理只有两步：

```ruby
depends_on "cmake" => :build if OS.ohos?
```

```ruby
def install
  # Work around superenv breaking aws-lc-sys `-O0` needed to build CPU Jitter RNG
  ENV["AWS_LC_SYS_NO_JITTER_ENTROPY"] = "1" if OS.ohos?

  system "cargo", "install", *std_cargo_args
end
```

参考实现：`Formula/m/monolith.rb`、`Formula/m/mlc.rb`、`Formula/d/drill.rb`。

### 6.3 zstd-sys

`zstd-sys` 的 bundled C 源码在 `dictBuilder/cover.c` 里调用 glibc 的 `qsort_r`，OHOS 上会编译失败。统一使用 `zstd-sys` 官方支持的 pkg-config 路径：

```ruby
depends_on "pkgconf" => :build if OS.ohos?
depends_on "zstd" if OS.ohos?

def install
  ENV["ZSTD_SYS_USE_PKG_CONFIG"] = "1" if OS.ohos?
  system "cargo", "install", *std_cargo_args
end
```

要点：

- `ZSTD_SYS_USE_PKG_CONFIG` 是 zstd-sys 官方支持的 build knob，不再编译 bundled C
- zstd formula 已经处理了 OHOS 的 `qsort_r`
- 不要给 `cover.c` 单独打 muslext 补丁，也不要用全局 `CFLAGS=-include muslext/qsort_r.h`，后者会污染其他 crate 的 C++/汇编编译
- 不需要逐版本维护 zstd-sys 的 resource/patch，升级 zstd-sys 后仍然适用。

如果只做构建期依赖，先检查最终产物：

```sh
readelf -d <binary> | grep zstd
```

没有 `NEEDED libzstd.so` 时可以把 `depends_on "zstd"` 改成 `=> :build`，出现运行时依赖时保留普通依赖。

参考实现：`Formula/o/onefetch.rb`、`Formula/c/codex.rb`、`Formula/c/cc-switch-cli.rb`、`Formula/t/turso.rb`。

### 6.4 使用 resource 修补 crate

需要改 1–2 个 crate 时，用 resource 声明 crates.io 源码，在 `fetch` 阶段解到 buildpath 的 `vendor/`，再用 path source 挂载。

```ruby
resource "socket2" do
  url "https://static.crates.io/crates/socket2/socket2-0.4.10.crate"
  sha256 "..."
end

def fetch
  if OS.ohos?
    resource("socket2").stage { (buildpath/"vendor/socket2").install Dir["*"] }
    File.write(buildpath/"Cargo.toml", <<~EOS, mode: "a")
      [patch.crates-io]
      socket2 = { path = "vendor/socket2" }
    EOS
    system "cargo", "update", "--package", "socket2@0.4.10", "--precise", "0.4.10"
  end

  system "cargo", "fetch", *std_cargo_fetch_args
end
```

补丁文件放在 `Patches/<formula>/`，在 fetch 阶段对 `vendor/<crate>` 应用，或使用 resource 的 `patch do` 块：

```ruby
resource "socket2" do
  url "https://static.crates.io/crates/socket2/socket2-0.4.10.crate"
  sha256 "..."
  patch do
    file "Patches/xxx/0001-ohos-socket2.patch"
  end
end
```

要求：

- crate 必须在依赖图里，也就是 `Cargo.lock` 中存在，直接依赖或传递依赖都可以，**不要求**在 `Cargo.toml` 里直接声明
- resource 的版本必须满足该 crate 在依赖图中的 semver 约束
- 如果 resource 版本和原 `Cargo.lock` 相同：用 path patch 并去掉 lock 里的 registry source/checksum
- 如果 resource 版本和原 `Cargo.lock` 不同：必须在 `fetch` 阶段执行 `cargo update --package <crate>@<version> --precise <version>`，由 Cargo 重新生成 lock，不要手改版本、source 或 checksum
- `cargo update` 必须在 `fetch` 阶段执行，因为后面构建阶段已经断网
- 构建阶段保持 `cargo install --locked`，不要再访问网络
- 不改 `$CARGO_HOME/registry/src`，不依赖“之前 fetch 过”。

### 6.5 处理 Cargo 版本约束

`cargo update` 只修改 `Cargo.lock`，不会修改 `Cargo.toml` 的 semver 约束。如果上游写的是：

```toml
nix = "0.27"
```

而需要升到 `0.29`，`cargo update --package nix --precise 0.29.0` 会直接失败。`0.x` 的 minor 版本在 Cargo 里是 breaking 级别，`^0.27` 不包含 `0.29`。

正确顺序：

1. 先用补丁修改 `Cargo.toml` 的版本约束，不要改 `Cargo.lock`。
2. 再在 `fetch` 阶段执行 `cargo update --package <crate> --precise <version>`，让 Cargo 重新生成 lock。
3. 构建阶段继续使用 `cargo install --locked`。

例如 `fclones`：

```ruby
if OS.ohos?
  # nix < 0.29 treats OHOS as glibc; raise the manifest requirement and let
  # Cargo regenerate the lock.
  patch do
    file "Patches/fclones/0001-use-nix-with-ohos-support.patch"
  end
end

def fetch
  if OS.ohos?
    system "cargo", "update", "--package", "nix", "--precise", "0.29.0"
  end
  system "cargo", "fetch", *std_cargo_fetch_args
end
```

这里的补丁只改 manifest，不是 crate 源码补丁，不需要 `[patch.crates-io]`，也不要手改 lock 的 version/source/checksum。

### 6.6 多个 crate 使用 cargo vendor

需要同时 patch 多个 crate，或需要完整离线依赖树时，用全量 vendor：

```ruby
system "cargo", "vendor", "--locked", "--versioned-dirs", "vendor"
```

然后写：

```toml
[source.crates-io]
replace-with = "vendored-sources"

[source.vendored-sources]
directory = "vendor"
```

再直接修改 `vendor/<crate>`，并同步 `Cargo.lock`。这种方式体积大，但可复现性最好之一。

### 6.7 不要直接修改 cargo cache

不要在 `install` 里用 glob 找：

```text
$CARGO_HOME/registry/src/**/<crate>
```

然后 `inreplace`、`rm_rf` 或复制文件。问题包括：

- 修改的是全局共享 cache，可能污染其他构建
- cache 被清理后补丁消失
- glob 可能命中多个版本
- 补丁不在 `Patches/` 下，无法审计
- 最终源码不对应任何 checksum，许可证和源码交付都说不清。

### 6.8 其它注意事项

- 需要 jemalloc 等平台相关分配器时，注意 OpenHarmony 不可用，可用 `--no-default-features` 或补丁关闭。
- 交叉编译产物检查：`cargo` 默认产物可能是 `aarch64-unknown-linux-musl`，与 OHOS target 不同，注意 `--target`。
- cgo 项目不要把 `CGO_CFLAGS` 用在 `.S` 汇编上，见 5.8。

## 7 Go 与平台适配

Go 的三方模块和 Rust crate 类似，但工具链不同：

- `$GOMODCACHE` 是全局模块缓存，不要直接改。
- 先用 `go.mod` / `go.sum` 升级到认识 OHOS 的版本，能升级就不要打补丁。
- 必须改模块源码时，在 `fetch` 阶段执行 `go mod vendor`，把依赖落到 buildpath 的 `vendor/`，再 patch `vendor/<module>`，构建时使用 `-mod=vendor`。
- 也可以把单个模块 `replace` 到本地目录：

```go
replace example.com/foo => ./vendor/foo
```

`go mod vendor` 的缺点是 vendor 目录大，但它是 Go 原生的可复现方式，比修改全局模块缓存安全。

注意：

- `-mod=vendor` 要和 `go mod vendor` 的输出一起使用，不要只 patch `$GOMODCACHE`。
- cgo 场景需要正确的 `CGO_ENABLED`、`CC` 和 `GOOS/GOARCH`。
- `CGO_CFLAGS` 会传给 `runtime/cgo` 的 `.S` 汇编，如果依赖是预编译静态库，只需要 `CGO_LDFLAGS` 链接 muslext，不要把 C 头文件塞进汇编（见 5.8）。
- `GOFLAGS=-buildmode=pie` 在 OpenHarmony 上通常是必需的。
- 依赖 glibc 的预编译产物（例如 upstream 下载的二进制）在 OpenHarmony 上不可用，应改为本地构建。

参考实现：

- `Formula/g/gascity.rb`：用 `go.mod` 的 `replace` 固定模块版本
- `Formula/b/buildkit.rb`、`Formula/l/lazygit.rb`：`-mod=vendor`
- `Formula/p/payload-dumper-go.rb`：cgo 只加 `CGO_LDFLAGS`，不强制 C 头文件。

## 8 Node.js 与平台适配

### 8.1 原生模块与预编译包

- 原生模块（`node-gyp`）会按 `process.platform` 判断平台，OpenHarmony 上报的值与 Linux 不同，需要在源码里补分支。
- npm 可选依赖里的原生二进制包（`@esbuild/*`、`@opentui/core`、`lightningcss` 等）**不要复用 `linux-arm64-musl` 产物**：要么选带 `openharmony-arm64` 的版本，要么从源码现编。版本太旧没有 OHOS 产物时，用补丁升级该依赖的版本，而不是关掉功能。
- 解释器符号可见性问题见 5.3。

### 8.2 修改 node_modules

`node_modules` 是当前 buildpath 里的本地安装结果，构建结束就丢弃，不属于全局共享 cache。

Node 场景的处理顺序：

1. 依赖版本能被 `package.json` / lockfile / `overrides` 固定时，先把版本固定。
2. 需要改依赖源码时，用 `Patches/<formula>/` 下的 patch 文件修改本地 `node_modules`。
3. 只有“必须在 `npm install` 之前替换这个包”时（例如平台门控的 optional dependency、需要本地源码构建的 native 包），才用 resource 声明该包并 stage 到 `vendor/`，再通过 `overrides` 指向本地路径。

优先用 patch 文件修改本地 `node_modules`，补丁可审计，resource 只用于必须在安装前替换的包。

### 8.3 resource 模式的边界

**不要给依赖很多的项目逐个声明 resource**：每个包都要维护版本、URL、sha256 和 `overrides`，上游一升级就要全部同步，维护成本远高于把 patch 文件放在 `Patches/` 下。

resource 模式的代价是真实的：

- 每个 resource 都要固定版本和 sha256
- 依赖升级时，resource 路径、版本、hash 都要手动更新
- 包之间的版本约束还要通过 `overrides` 重新对齐
- 只适合少量、确实需要在安装前替换的包。

## 9 Python 与平台适配

Python 的“三方库”通常通过 `resource` 声明 sdist，再由 `virtualenv_install_with_resources` 安装。需要修改某个包时，不要等安装完再改 `site-packages`，而应把它从批量安装里排除，先 patch resource，再单独安装。

```ruby
resource "psutil" do
  url "https://files.pythonhosted.org/packages/.../psutil-7.0.0.tar.gz"
  sha256 "..."
end

def install
  venv = virtualenv_install_with_resources(without: %w[psutil]) if OS.ohos?
  if OS.ohos?
    with_env(
      "CFLAGS"   => "#{ENV["CFLAGS"]} -include muslext/utmpx.h",
      "LDFLAGS"  => "#{ENV["LDFLAGS"]} -Wl,--no-as-needed -lmuslext",
      "LDSHARED" => "#{ENV.cc} -shared",
    ) do
      venv.pip_install resource("psutil")
    end
  else
    virtualenv_install_with_resources
  end
end
```

要点：

- 优先从 PyPI sdist 构建，不要用预编译 wheel，wheel 里的 `.so` 通常不能直接用于 OHOS。
- 被 patch 的包必须用 `without:` 从批量安装中排除，否则会被未修补版本覆盖。
- 补丁可以放在 resource 的 `patch do` 块里，也可以 stage 后 patch，再 `pip_install`。
- 编译型扩展要同时设置 `CFLAGS`、`LDFLAGS` 和 `LDSHARED`，`LDSHARED` 往往决定扩展最终怎么链接。
- 需要 muslext 的接口按 5.6 选择最小头文件，不要给所有 Python 包全局注入 umbrella header。
- 不要安装完再 `inreplace` `site-packages`，那是构建产物，不是源码。

参考实现：

- `Formula/m/maigret.rb`、`Formula/m/molecule.rb`：`without: %w[psutil]` + 单独 `pip_install`
- `Formula/p/pip-audit.rb`：PyPI resource 列表
- `Formula/s/standardebooks.rb`：vendored Python 依赖的补丁处理。

## 10 测试与服务

### 10.1 不要仅为本地网络问题更换下载源或规避测试

本节同时适用于下载地址、`sha256` 和测试规避。流水线所在网络通常可以正常访问外网，因此：

- 不要因为“我这里下载失败”就更换 `url` / `sha256`。
- 不要吞掉网络相关的测试错误（如匹配 `LOGIN_REQUIRED` 后提前 `return`）。
- 不要为绕过本地网络问题添加测试依赖或改测试 URL。

如果测试确实依赖不稳定的外部服务，用 `allow_network_access!` 声明并在 upstream 已有 fallback 的前提下复用，而不是新造一套。

**维护者确认的特例**

少数情况下，**Harmonybrew CI 服务器确实无法访问某个具体地址**。这时可以改，但必须满足：

1. 维护者确认 ci-runner 对该地址的访问失败，而不是贡献者本地网络问题。
2. 优先换到官方镜像或官方 GitHub 归档，不用来路不明的第三方镜像。
3. 测试规避要尽量窄：能只跳过单个 URL 就不要删掉整段测试，能只对 OHOS 生效就不要影响所有平台。
4. PR 中写明具体 URL、失败现象、替换来源和影响范围。

这类特例很少，目前有：

| Formula | 无法访问的地址 | 处理 |
| --- | --- | --- |
| `zbar` | `linuxtv.org` | 改用 GitHub 官方源码归档，OHOS 关闭依赖 argp 的上游测试构建 |
| `musepack` | `trac.ffmpeg.org` | 测试样本改用 `fate-suite.ffmpeg.org`，并裁剪到原断言长度 |
| `streamlink` | `www.youtube.com` | 只在 OHOS 下跳过 YouTube 测试并注明原因，其他平台保留上游测试 |

注意：这不是“网络测试都可以改”的通行证。只能针对已确认不可达的具体 host，不能扩大到“所有外网测试都跳过”。

例外情况：如果 upstream 确实重新打包了源码包（`sha256` 因此变化），可以更新 `sha256`，但必须能说明原因（例如上游 release 页面的重新发布记录、实际下载校验结果），并在 PR 中写清楚。判断标准是「换哈希有没有事实依据」，而不是「我这里下不下来」。

### 10.2 service 块

`brew services` 在 OpenHarmony 上不可用，但 `service do ... end` 只是声明式元数据，`brew install` 和 `brew test` 都不会执行它。

因此**不要在 formula 里屏蔽 service 块**，保持 upstream 原样：

```ruby
service do
  run [opt_bin/"foo", "-config", etc/"foo.conf"]
  keep_alive true
end
```

OHOS 的支持已经在包管理器层统一处理：

- `brew services` 命令直接报错 “not supported on OpenHarmony”
- `brew info` 和 `brew info --json` 不显示 service 信息。

不要写：

```ruby
unless OS.ohos?
  service do
    ...
  end
end
```

也不要写 `service do ... end unless OS.ohos?` 或 `if OS.ohos?` 的空分支。给每个 formula 加保护会产生大量无意义 diff，并持续和 upstream 冲突。

### 10.3 辅助函数不要在函数体内 `return`

**错误**：

```ruby
def prepare_ohos_flux
  return unless OS.ohos?
  ...
end
```

**正确**：由调用点判断，函数体保持纯粹。

```ruby
prepare_ohos_flux if OS.ohos?
```

## 11 补丁规范

### 11.1 位置与命名

补丁放在 `Patches/<formula>/` 下。文件名应能说明用途：可以按 `0001-ohos-<简短描述>.patch` 编号，也可以沿用上游文件名或补丁本身的名字，不要使用无意义的编号。

```
Patches/w3m/0001-ohos-getpass.patch
```

### 11.2 头部格式

只保留 diff 本体。**禁止**出现 `git format-patch` 风格的头部：

```
From 0000000000000000000000000000000000000000 Mon Sep 17 00:00:00 2001
From: Harmonybrew <harmonybrew@example.com>
Date: ...
Subject: [PATCH] ...
```

正确的最小形式：

```diff
--- a/src/foo.c
+++ b/src/foo.c
@@ -1,3 +1,4 @@
```

允许带 `diff --git a/... b/...` 或 `diff -ruN a b` 前缀（`make-patch.md` 推荐后者）。

### 11.3 声明补丁来源

按补丁的来源分三种写法：

**仓内自研补丁**（我们自己写的 OHOS 适配）：放在 `Patches/<formula>/`，**不要**写 `type :unofficial`。只有来自其他项目的代码才需要声明。

```ruby
patch do
  file "Patches/xxx/0001-ohos-foo.patch"
end
```

**来自上游 Homebrew 的原始补丁**：用远程 URL 引用，固定到具体 commit，并带上游原来的 `sha256`，不要复制成本地文件：

```ruby
patch do
  url "https://raw.githubusercontent.com/Homebrew/homebrew-core/<commit>/Patches/<formula>/<file>.patch"
  sha256 "..."
  type :unofficial
end
```

`FormulaAudit/Patches` 要求 URL 指向具体 revision，不能写 `refs/heads/main`。

**来自其他项目**（NetBSD、Alpine、Termux、上游 PR 等）：默认不允许，只有维护者事先批准、且许可证兼容时，才用 `type :unofficial` 引入，并在 PR 中注明来源与许可证。参见 [Pull Request 自查指南](self-check-pr.md)。

参考实现：`Formula/m/mysql.rb`、`Formula/m/minidlna.rb`。

能自己写就自己写。引入第三方代码时，务必确认许可证兼容，并在 PR 中说明来源。

### 11.4 补丁优先于 `inreplace`

`inreplace` 只适合一两处简单字符串替换。超过两三处，或需要改结构时，改用补丁文件：

```
Patches/faircamp/0001-ohos-*.patch
```

修改 `node_modules`、cargo 缓存、vendored 依赖里的代码时，也不要在安装阶段 `inreplace`：优先用 patch 文件把改动固化下来。`node_modules` 是 buildpath 本地，可以用 patch 文件修改，Rust 的 `$CARGO_HOME/registry/src` 是全局共享 cache，不再允许直接修改（见第 6 节）。只有必须在安装前替换 npm 包时，才用 resource + `overrides`。

大量 `inreplace` 的 formula 在上游改动后会静默失效，很难排查。

### 11.5 升级依赖版本，而不是绕过它

当某个依赖的旧版本不认识 OHOS（`socket2`、`esbuild`、`nix`、`libc`）时，正确做法是升级依赖版本：

- Rust：改 `Cargo.lock`（配合第 6 节的 resource 方案）。
- Node：用补丁把 `package.json` 里的版本升到带 `openharmony-arm64` 的版本。
- 系统库：优先重编级联依赖的 bottle（见 3.3）。

不要用 `--no-default-features`、`no_xxx` 开关或运行期判断把功能整个关掉。

## 12 速查表

| 现象 | 标准做法 | 参考 |
| --- | --- | --- |
| Rust 包通过 rustls 依赖 aws-lc-sys | `depends_on "cmake" => :build if OS.ohos?` + `ENV["AWS_LC_SYS_NO_JITTER_ENTROPY"] = "1" if OS.ohos?` | `Formula/m/monolith.rb` |
| Rust 包依赖 zstd-sys | `depends_on "pkgconf" => :build if OS.ohos?` + `depends_on "zstd" if OS.ohos?` + `ENV["ZSTD_SYS_USE_PKG_CONFIG"] = "1" if OS.ohos?` | `Formula/o/onefetch.rb` |
| OHOS 适配额外需要构建工具 | 用 `OS.ohos?` 加显式 `:build` 依赖；不要重复声明 CI 已提供的工具 | — |
| 缺少 gettext / util-linux / libomp | 补依赖，不要关功能 | `Formula/f/flex.rb`、`Formula/m/mosquitto.rb`、`Formula/m/msolve.rb` |
| toybox 的 `find`/`xargs` 行为不一致 | `depends_on "findutils" => :build if OS.ohos?`，运行时也要用则去掉 `=> :build` | `Formula/n/newsboat.rb`、`Formula/g/grails.rb` |
| SDK Clang 太旧编不过 | 打补丁兼容，不要为版本问题新增 gcc 依赖，上游已有 gcc 依赖则保留 | — |
| 需要 C++20/23 标准库 | llvm 的 clang++ + 私有 `libc++_shared.so`，需要独立分发时复制到产物目录并设 RPATH。链接 `libLLVM` 时改用系统 `/lib64/libc++_shared.so` | `llvm` |
| 完整第三方库（gettext/libintl、argp、libiconv） | 用独立 formula，不要塞进 muslext | `gettext`、`argp-standalone` |
| 小型 libc 兼容接口（getpass/utmpx/qsort_r/fts） | 用 muslext | `muslext` |
| 一次性旧别名，muslext 未提供（如 `rindex`） | 源码改成标准接口，或局部 `-D` 宏并注释原因，不要扩展 muslext | `rpcsvc-proto` |
| 运行时已导出但头文件没声明，且 muslext 已有 A 类接口 | 优先用 muslext，注释写“does not declare”，不要写“does not export” | `nnn`、`dropbear` |
| 解释器插件找不到符号 | 先确认真的需要，再 `ENV.append "LDFLAGS", "-Wl,-z,global" if OS.ohos?`（修宿主 formula） | `Formula/z/zsh.rb` |
| `sockaddr_storage` 冲突 | 先确认编译命令是否经过 superenv shim，被绕过时补 `CPPFLAGS`/test 参数，自带头遮蔽时打补丁 | `Formula/g/gpsd.rb`、`Formula/lib/libtorrent-rasterbar.rb`、`Patches/strongswan/0001-ohos-sockaddr-storage.patch` |
| 需要 C/C++ 源码平台判断 | 使用 `__OHOS__` 宏，不要伪装其他平台运行时 | 见 5.1 |
| 缺 `getpass()` / `getutxent()` | 优先接 muslext 的最小头文件 | 见 5.6 |
| 缺 `pthread_cancel()` / `pthread_setcancelstate()` | 复用 upstream 源码已有的 musl/Android 分支，或打补丁 | 见 5.5 |
| 需要改上游某条语句 | `if OS.ohos? ... else 原样 end`，`else` 里必须是上游原句 | — |
| 上游已有 `if OS.mac?` 分支 | 追加 `elsif OS.ohos?`，不要在最前面插分支 | — |
| 整块内容在 OHOS 不适用（如 `fails_with`） | `unless OS.ohos? ... end` 包块，不写空分支、不写 `end unless`，`service` 块保持上游原样 | — |
| 需要生成配置文件 | heredoc + 插值，变量直接展开 | — |
| 引用其他 formula 路径 | 新增代码用 `formula_opt_bin` / `formula_opt_lib`，上游原有写法不要改 | `Formula/m/metals.rb` |
| 测试依赖系统工具 | `depends_on "xxx" => :test if OS.ohos?` | — |
| 测试适配（环境变量、路径） | 写进 `test do` 块，不要塞在 `install` 里 | — |
| 需要 CA 证书 | 用 `/etc/ssl/certs/cacert.pem`，不要依赖 `ca-certificates` | `Formula/e/ejabberd.rb` |
| 需要字体 | 运行时加载 `/system/fonts`，不要打包字体 | — |
| 下载 / 测试受网络影响 | 默认不改，只有维护者确认 CI 无法访问具体地址时，按 10.1 的受控特例处理 | `zbar`、`musepack`、`streamlink` |
| 仓内自研补丁 | 不写 `type :unofficial` | — |
| 上游 Homebrew 原始补丁 | 远程 raw URL（固定 commit）+ `sha256` + `type :unofficial`，不复制成本地文件 | `Formula/a/ansible.rb` |
| 补丁来自其他项目 | `type :unofficial` 并注明来源 | `Formula/m/mysql.rb` |
| 需要改 vendored / npm 依赖代码 | node_modules 是 buildpath 本地：优先 patch 文件 + `overrides` 固定版本，只有必须在 install 前替换包时才 resource | — |
| 需要改 Go 三方模块 | `go mod vendor` 到本地 `vendor/`，patch 后 `-mod=vendor`，或 `replace` 到本地路径 | `Formula/g/gascity.rb`、`Formula/b/buildkit.rb` |
| 需要改 Python 三方包 | 用 PyPI sdist resource，`without:` 排除后单独 `pip_install`，不要改 `site-packages` | `Formula/m/maigret.rb`、`Formula/m/molecule.rb` |
| 需要改 1–2 个 Rust crate | `resource` + `[patch.crates-io]` + `cargo update --precise`，不要改 cargo cache | `Formula/z/zellij.rb`、`Formula/s/starship.rb` |
| Cargo.toml 的 semver 不允许目标版本 | 先用补丁改 manifest 约束，再 `cargo update --precise`，不要手改 lock | `Formula/f/fclones.rb` |
| 需要改多个 Rust crate | `cargo vendor --locked --versioned-dirs`，patch `vendor/` | `Formula/r/railway.rb`、`Formula/s/superseedr.rb` |
| 给 C++ 源文件强制注入头文件 | 同时设置 `CXXFLAGS`，不能只设 `CFLAGS` | — |
| CMake 探测被强制头文件干扰 | 显式设置 `-DHAVE_XXX` 或 `ac_cv_func_*`，不要依赖自动探测 | — |
| cgo 的 `CGO_CFLAGS` 进入 `.S` 汇编 | 预编译依赖只加 `CGO_LDFLAGS`，不要把 C 头文件塞进汇编 | — |
| 缺 `getpass` / `utmpx` / `qsort_r` / `fts` / shadow / pthread affinity | 优先接 `muslext`，按最小头文件注入，`pthread_cancel` 仍打补丁 | 见 5.5、5.6 |
| npm 原生包只有 musl 产物 | 升到带 `openharmony-arm64` 的版本，或从源码现编 | `Formula/z/zrok.rb` |
| 依赖的包在 OHOS 上有缺陷 | 先修被依赖的包，再重编本包 | — |
| 上游已 404 / 已 `disable!` | 不录入 | — |
| 包确实无法适配 | 关闭 PR，不硬塞 workaround | — |
