# MHTML 在 WPF WebView2 中嵌入显示

## 目标

在 WPF 应用内嵌展示 MHTML 格式的帮助文档（`用户操作说明书.mhtml`），要求样式、图片、目录跳转完整可用。

## 演进过程

### 第1步：WebBrowser + IE11 注册表（不理想）

WPF 原生 `WebBrowser` 控件底层是 IE ActiveX，默认 IE7 模式。

```csharp
// App.xaml.cs OnStartup
Registry.CurrentUser.CreateSubKey(
    @"SOFTWARE\Microsoft\Internet Explorer\Main\FeatureControl\FEATURE_BROWSER_EMULATION")
    .SetValue("Bubble.exe", 11001, RegistryValueKind.DWord);
```

- `11001` (0x2AF9) = IE11 Edge 模式
- 必须在 WebBrowser 创建**之前**写入注册表
- 结果：IE11 对现代 CSS 支持有限，且仍需依赖注册表魔改

### 第2步：迁移到 WebView2

安装 NuGet 包 `Microsoft.Web.WebView2`，基于 Edge Chromium 内核。

**XAML 变更：**
```xml
<!-- 旧 -->
<WebBrowser x:Name="webBrowser" />

<!-- 新 -->
xmlns:wv2="clr-namespace:Microsoft.Web.WebView2.Wpf;assembly=Microsoft.Web.WebView2.Wpf"
<wv2:WebView2 x:Name="webView" />
```

**初始化模式：**
```csharp
await webView.EnsureCoreWebView2Async(); // 必须先异步初始化
```

### 第3步：file:// 加载 → CSS 正常，图片丢失

```csharp
webView.Source = new Uri(helpFilePath);
```

- CSS 正常渲染（IE11 方案直接淘汰）
- 图片不显示：MHTML 内图片通过 `cid:` 引用，`file:///` 协议下 Chromium 将其视为跨域请求拦截

### 第4步：虚拟主机加载 → 触发下载

```csharp
webView.CoreWebView2.SetVirtualHostNameToFolderMapping(
    "help.local", basePath, CoreWebView2HostResourceAccessKind.Allow);
webView.CoreWebView2.Navigate("https://help.local/用户操作说明书.mhtml");
```

- Chromium 不识别 MHTML 格式的 HTTP 响应，触发文件下载而非渲染
- `SetVirtualHostNameToFolderMapping` 对 `.mhtml` 文件无效

### 第5步：NavigateToString → 大小超限

解析 MHTML → 转为内嵌 HTML → `NavigateToString(html)`

- `NavigateToString` 有约 **2MB 限制**
- MHTML 中 base64 图片转 data URI 后极易超限
- 异常：`System.ArgumentException` — "值不在预期的范围内"

### 第6步：临时文件 + 虚拟主机（最终方案）

```
MHTML 文件
  → 解析 MIME 多部分结构
  → 提取 HTML + CSS + 图片
  → CSS 内联为 <style>
  → 图片转 data:image/...;base64,...
  → 修复锚点链接
  → 写入临时 _help_temp.html
  → SetVirtualHostNameToFolderMapping + Navigate
```

纯 HTML 文件（非 MHTML）通过虚拟主机加载没有下载问题，图片作为 data URI 内嵌不存在跨域。

## MHTML 解析器核心逻辑

### MHTML 文件结构

```
From: <Saved by Blink>
MIME-Version: 1.0
Content-Type: multipart/related; boundary="----Boundary----"

------Boundary----
Content-Type: text/html
Content-Transfer-Encoding: quoted-printable

<html>...</html>

------Boundary----
Content-Type: text/css
Content-Location: cid:css-xxx@mhtml.blink

body { ... }

------Boundary----
Content-Type: image/png
Content-Transfer-Encoding: base64
Content-Location: image001.png

iVBORw0KGgo...
```

### 解析步骤

1. **提取 boundary** — 正则 `boundary\s*=\s*"?([^"\s;\r\n]+)"?`
2. **拆分 MIME 部分** — `--boundary` 为分隔符
3. **逐个解析头部**：
   - `Content-Type` — 判断类型（text/html / image/* / text/css）
   - `Content-ID` — cid 引用标识
   - `Content-Location` — 文件路径或 cid URL
   - `Content-Transfer-Encoding` — 编码方式
4. **解码正文** — 支持 3 种编码：
   - `base64` — 去空白后 `Convert.FromBase64String`
   - `quoted-printable` — 去软换行 `=\r\n`，解码 `=XX` 十六进制
   - 无编码 — 原样使用
5. **组装输出** — CSS 内联、图片转 data URI、锚点修复

### Content-Transfer-Encoding 处理

```csharp
// quoted-printable 解码
body = Regex.Replace(body, "=\r\n", "");    // 去掉软换行
body = Regex.Replace(body, "=\n", "");
// 逐个解码 =XX
for (int i = 0; i < input.Length; i++)
{
    if (input[i] == '=' && TryParseHex(input[i+1..i+3], out byte val))
        result.Add(val), i += 2;
    else
        result.Add((byte)input[i]);
}
```

### 图片 → data URI

```csharp
string base64 = Regex.Replace(body, @"\s+", "");
return $"data:{contentType};base64,{base64}";
```

支持的引用方式替换：
- `src="cid:xxx"` / `src='cid:xxx'` / `src=cid:xxx`
- `src="filename.png"` / `src='filename.png'`
- `url("...")` / `url('...')` / `url(...)`

### CSS 内联

```csharp
// <link href="cid:css-xxx@mhtml.blink" /> → <style>...</style>
Regex.Replace(htmlContent,
    $@"<link\s+[^>]*href=""{Regex.Escape(cssLocation)}""[^>]*/?>",
    $"<style>{cssContent}</style>");
```

### 锚点修复

MHTML 中链接是绝对 `file:///E:/.../xxx.html#section1`，需剥离为 `#section1`：

```csharp
htmlContent = Regex.Replace(htmlContent, @"href=""file:///[^""#]*#", @"href=""#");
htmlContent = Regex.Replace(htmlContent, @"href='file:///[^'#]*#", @"href='#");
```

## 关键踩坑

| 坑 | 现象 | 原因 | 解决 |
|---|------|------|------|
| verbatim 字符串引号 | CS1525 编译错误 | `@"..."` 内 `"` 必须写 `""` | 正则中每个 `"` 写成 `""` |
| NavigateToString 大小限制 | ArgumentException | 字符串 > 2MB | 写临时文件 + 虚拟主机加载 |
| file:// 协议 | 图片不显示 | 跨域拦截 cid: 子资源 | 内联 data URI 避免网络请求 |
| 虚拟主机 + MHTML | 触发下载 | Chromium 不渲染 MHTML | 先转换为纯 HTML 再加载 |
| MHTML CSS 丢失 | 无样式 | 未处理 text/css 部分 | 提取 CSS 部分内联为 `<style>` |
| 锚点不跳转 | 点目录无反应 | 链接含绝对 file:// 路径 | 正则剥离只保留 `#fragment` |

## 最终文件结构

```
HelpView.xaml        — WebView2 控件声明
HelpView.xaml.cs     — 加载逻辑 + MHTML 解析器
  ├── HelpView_Loaded()          — 入口，协调加载流程
  ├── ConvertMhtmlToHtml()       — MHTML → HTML 主方法
  ├── DecodeContent()            — 解码 quoted-printable / base64
  ├── DecodeQuotedPrintableBytes() — QP 字节级解码
  ├── BuildDataUri()             — 图片 → data URI
  └── ShowHtmlContent()          — 错误页面
```

## 可选优化

- 关闭 HelpView 时删除 `_help_temp.html` 临时文件
- 对超大 MHTML 可分块处理避免内存峰值
- 支持更多 MIME 类型（如 `image/svg+xml`、字体文件）
