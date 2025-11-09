---
share: "true"
---
通常在开发完成后，想要打包成一个完整的`exe`到其他电脑运行，可以使用`Enigma Virtual Box`来完成。
首先在`QT`执行`release`编译，然后找到该`release`的生成目录，在该目录执行`powershell`或者`终端`：
```shell
windeployqt.exe .\<app>.exe
```
接着会出现很多`dll`库在该目录下。
![[笔记/01 附件/QT打包发布/image 4.png|笔记/01 附件/QT打包发布/image 4.png]]
如果只是打包`Qt5Core.dll`在其他电脑运行，会导致问题：
![[笔记/01 附件/QT打包发布/image 5.png|笔记/01 附件/QT打包发布/image 5.png]]
这是因为缺失文件：
```shell
libstdc++-6.dll
libwinpthread-1.dll
```
我们需要在编译器对应的`bin`目录下找到他们，比如我这里是`Mingw_64`，所以目录位置在：
```shell
xxxx\Qt5.12.12\5.12.12\mingw73_64\bin
```
选可以找到，我们复制：
```shell
libstdc++-6.dll
libwinpthread-1.dll
libgcc_s_seh-1.dll
```

**注意**：有时候`windeploy`会复制`32-bit`的`Qt5Core.dll`到目录下，如果我们是`64-bit`，需手动重新复制，具体查看办法可以上传到在线查看库的文件，下面有链接。
```shell
# PE32+ 为64
# PE32  为32
Magic  PE32+ executable (GUI) x86-64 (stripped to external PDB), for MS Windows
```
## 📋 文件必要性分析

### ✅ **绝对必需的文件**
```
Qt5Core.dll                    ← 核心库，必须！
platforms/qwindows.dll         ← Windows平台插件，必须！
```
如果你的程序使用了 MinGW 编译（从截图看是的）：
```
libgcc_s_seh-1.dll            ← MinGW运行时，必须
libstdc++-6.dll               ← MinGW C++运行时，必须
libwinpthread-1.dll           ← MinGW线程库，必须
```

### 🎨 **GUI 相关（有界面必需）**
```
Qt5Gui.dll                    ← 图形界面，有窗口就必须
Qt5Widgets.dll                ← 控件库，用了按钮等控件就必须
```

### 🖼️ **OpenGL 相关（看情况）**
```
libEGL.dll                    ← 如果用了OpenGL/图形加速
libGLESv2.dll                 ← OpenGL ES支持
opengl32sw.dll                ← 软件OpenGL渲染（备用）
```

**判断方法**：如果你的程序有3D图形、使用了 QOpenGLWidget，就需要。普通2D界面可以不要。

---

## 📁 **各目录文件分析**
### 1️⃣ **platforms/** - 必须！
```
✅ qwindows.dll               ← 必须！没有这个程序无法启动
❌ qwindowsvistastyle.dll     ← 不需要，这是样式插件，styles/里已有
```

### 2️⃣ **imageformats/** - 部分需要
从截图看你有：
```
qgif.dll      ← GIF图片支持
qicns.dll     ← macOS图标格式，可删除
qico.dll      ← ICO图标支持
qjpeg.dll     ← JPEG图片支持
qjpeg.dll     ← 重复了？
qtga.dll      ← TGA图片格式
qwbmp.dll     ← WBMP格式
qwebp.dll     ← WebP格式
```
**保留建议**：
- ✅ **qjpeg.dll** - 如果加载 JPG 图片
- ✅ **qpng.dll** - 如果加载 PNG 图片（你截图里没看到，但应该有）
- ✅ **qgif.dll** - 如果加载 GIF
- ✅ **qico.dll** - 如果加载图标文件
- ❌ 其他格式如果不用可以删除

### 3️⃣ **styles/** - 可选
```
qwindowsvistastyle.dll        ← Windows Vista/7 样式
```
**建议**：如果你没有特别设置样式主题，可以删除。Windows 10/11 会用默认样式。

### 4️⃣ **bearer/** - 通常不需要
```
网络承载插件（用于移动网络、WiFi切换等）
```
**建议**：❌ 删除，桌面应用基本不需要

### 5️⃣ **iconengines/** - 看情况
```
qsvgicon.dll                  ← SVG图标支持
```
**建议**：如果你的程序用了 SVG 格式的图标就保留，否则删除

### 6️⃣ **translations/** - 可选
```
Qt的多语言翻译文件（中文、日文等）
```
**建议**：
- ✅ 如果需要多语言支持，保留需要的语言
- ❌ 如果只用英文界面，全部删除

---

## 🎯 **你截图中其他 DLL 的必要性**
```
Qt5Network.dll                ← 网络功能（HTTP、Socket等）
Qt5Svg.dll                    ← SVG矢量图支持
Qt5Widgets.dll                ← GUI控件（按钮、文本框等）
```
**判断方法**：
- 程序有网络请求？→ 需要 Qt5Network.dll
- 程序显示 SVG 图标/图片？→ 需要 Qt5Svg.dll

---

## 💡 **精简建议（最小打包）**
如果你的程序是**简单的GUI应用**（没有网络、没有SVG、没有OpenGL），最小配置：
```
必需文件：
├─ yourapp.exe
├─ Qt5Core.dll
├─ Qt5Gui.dll
├─ Qt5Widgets.dll
├─ libgcc_s_seh-1.dll
├─ libstdc++-6.dll
├─ libwinpthread-1.dll
└─ platforms/
   └─ qwindows.dll

可选（如果加载图片）：
└─ imageformats/
   ├─ qjpeg.dll
   └─ qpng.dll（你的截图里没显示，但可能需要）
```
这是我的一次打包目录：
![[笔记/01 附件/QT打包发布/image 6.png|笔记/01 附件/QT打包发布/image 6.png]]
当然最直接的办法是使用`Dependency Walker`,来查看未打包的`exe`依赖情况：
![[笔记/01 附件/QT打包发布/image 8.png|笔记/01 附件/QT打包发布/image 8.png]]
或者使用[在线网站]]或者使用[在线网站](https://www.virustotal.com/)上传`未打包的exe文件`，转到`detail`：
![[笔记/01 附件/QT打包发布/image 7.png|笔记/01 附件/QT打包发布/image 7.png]]
当然为了缩小大小，可以使用在打包时选择压缩：
![[笔记/01 附件/QT打包发布/image 9.png|笔记/01 附件/QT打包发布/image 9.png]]