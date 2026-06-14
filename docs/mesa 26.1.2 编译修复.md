# mesa 26.1.2 编译修复：pan_kmod_stub.h 补全 gpu_id + hunk 行数修正

> 环境：zagwrt · aarch64_generic · musl · openwrt-25.12 · 2026-06-14

---

## 错误现象

所有 `libpan-arch-vN.a.p/` 目标（v4~v13）均报：

```
../src/panfrost/lib/kmod/pan_kmod_stub.h:26: error: unterminated #if
   26 | #if defined(__cplusplus)
```

---

## 根因分析

### 问题一：stub 结构体缺少 `gpu_id` 字段

`pan_image.h` 第 297 行访问了 `dprops->gpu_id`，但 patch 200 创建的 stub 结构体只有 3 个 thread 字段：

```c
// 修复前（缺少 gpu_id）
struct pan_kmod_dev_props {
   uint32_t max_threads_per_core;
   uint32_t max_tasks_per_core;
   uint32_t max_threads_per_wg;
};
```

### 问题二：hunk 行数未同步

追加 `gpu_id` 字段后文件从 27 行变为 28 行，但 hunk 头仍为：

```
@@ -0,0 +1,27 @@
```

导致 patch 应用时文件被截断，`#if defined(__cplusplus)` 之后的 `#endif` 丢失。

---

## 修复步骤

### 步骤一：追加 `gpu_id` 字段

```bash
python3 << 'PYEOF'
with open('feeds/video/libs/mesa/patches/200-panfrost-Enable-cross-compilation-of-precompilers-on.patch', 'r') as f:
    content = f.read()

old = (
    '+struct pan_kmod_dev_props {\n'
    '+   uint32_t max_threads_per_core;\n'
    '+   uint32_t max_tasks_per_core;\n'
    '+   uint32_t max_threads_per_wg;\n'
    '+};\n'
)
new = (
    '+struct pan_kmod_dev_props {\n'
    '+   uint64_t gpu_id;\n'
    '+   uint32_t max_threads_per_core;\n'
    '+   uint32_t max_tasks_per_core;\n'
    '+   uint32_t max_threads_per_wg;\n'
    '+};\n'
)

content_new = content.replace(old, new, 1)
with open('feeds/video/libs/mesa/patches/200-panfrost-Enable-cross-compilation-of-precompilers-on.patch', 'w') as f:
    f.write(content_new)
print('OK')
PYEOF
```

### 步骤二：修正 hunk 行数 27 → 28

```bash
python3 << 'PYEOF'
with open('feeds/video/libs/mesa/patches/200-panfrost-Enable-cross-compilation-of-precompilers-on.patch', 'r') as f:
    content = f.read()

content_new = content.replace('@@ -0,0 +1,27 @@', '@@ -0,0 +1,28 @@', 1)
with open('feeds/video/libs/mesa/patches/200-panfrost-Enable-cross-compilation-of-precompilers-on.patch', 'w') as f:
    f.write(content_new)
print('OK')
PYEOF
```

### 步骤三：清理重编

```bash
rm -rf build_dir/hostpkg/mesa-26.1.2/
make package/feeds/video/mesa/host/compile V=s -j$(nproc) 2>&1 | \
  grep -E 'FAILED|error:|time:|ninja: build stopped' | head -20
```

---

## 经验教训

| 操作 | 注意事项 |
|------|---------|
| 向 patch 新文件 hunk 追加行 | **必须同步更新** `@@ -0,0 +1,N @@` 中的 N |
| patch 应用后文件被截断 | 优先检查 hunk 行数是否与实际内容匹配 |
| stub 结构体补全 | 只需补充被实际代码访问的字段，不必完整复制真实结构体 |

---

✅ **编译成功**（mesa 26.1.2 host build）
