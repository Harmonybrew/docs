# 补丁制作指南

## 1 准备源码

下载源码包，将源码包解压两次，得到两份干净的源码。这里假设源码目录名字分别为 a 和 b。

## 2 修改源码

修改目录 b 中的源码，对其进行业务适配。

本项目不会专门提供鸿蒙适配相关的指导，仅在力所能及的范围内分享一些参考资料：
1. [aports](https://gitlab.alpinelinux.org/alpine/aports)：Alpine Linux 的软件仓库。由于其 C 标准库采用 musl libc，该仓库中积累了大量针对非 glibc 环境的兼容性补丁，是解决 C 标准库差异的重要参考。
2. [termux-packages](https://github.com/termux/termux-packages)：Termux 的软件仓库。Termux 在 Android 平台上积累了丰富的应用沙箱与非 FHS 路径的适配实践，相关补丁对于解决 OpenHarmony 应用沙箱环境下的难题极具借鉴价值。

## 3 制作补丁

适配完成后，需制作标准补丁文件。推荐做法如下：

```sh
# -r: 递归目录; -u: 统一格式; -N: 处理缺失文件;
diff -ruN a b > 0001-add-ohos-support.patch
```

## 4 引入补丁

在核心 tap 仓库的 `Patches` 目录下创建一个与 formula 同名的子目录，将补丁放置其中。示例：`Patches/perl/0001-add-ohos-support.patch`

然后在本地 formula 中加入 `patch` 块，指向补丁文件：

```rb
  if OS.ohos?
    patch do
      file "Patches/perl/0001-add-ohos-support.patch"
    end
  end
```

## 5 验证补丁

执行 `brew install -y -s -v --include-test <formula>`、`brew test <formula>` 验证构建和测试是否通过。如果不通过，需要重新制作补丁、重新验证，直至验证通过。

> 该做法每次都会对软件包进行全量构建。虽耗时长，但操作简单，适用于大多数场景和用户。对于需要调试大型软件包、有增量构建需求的用户，可自行寻找其他方法进行增量构建，提高自身调试效率。

# 6 提交 PR

在本地验证通过并确认无误后，将 formula 和补丁一同提交至 [Harmonybrew/homebrew-core](https://atomgit.com/Harmonybrew/homebrew-core) 仓库。
