# 天梯榜 · RankLadder

电子产品性能排行榜 Android 应用（**v1.1.6**，覆盖**手机**与**电脑**两条天梯）。

- 底栏液态玻璃效果：[Kyant0/AndroidLiquidGlass](https://github.com/Kyant0/AndroidLiquidGlass)（`io.github.kyant0:backdrop:2.0.1`，Apache-2.0）
- 主题：Jetpack Compose **Material 3** + **莫奈（Monet）动态取色**
- 数据：**全部由爬虫自动抓取**（构建期 + 运行时在线抓取），没有手敲成绩
- 底栏：**直接使用原项目（Kyant0/AndroidLiquidGlass）的 `LiquidBottomTabs` 组件**，其中 5 个源文件与上游**逐字节一致**（零改动），第 6 个（`Coroutines.kt`）是上游 expect/actual 的 Android 实现收敛
- 产物：已签名的 release APK，两份同一文件（md5 `40e4d6ba52a22e18151afae8997952ec`，2,151,418 字节）
  - 工作区根目录 `RankLadder-v1.1.6-release.apk`
  - `out/RankLadder-v1.1.6-release.apk`

---

## 1. 数据从哪来（v1.1 新增）

### 1.1 构建期爬虫 `tools/fetch_rankings.py`

```bash
python3 tools/fetch_rankings.py            # 抓两个源 → 生成快照 + 兜底表
python3 tools/fetch_rankings.py --limit 300  # 电脑榜保留条数
```

| 榜单 | 来源 | 抓取方式 | 本次结果（2026-09-17 抓取） |
| --- | --- | --- | --- |
| 手机 | [安兔兔官方性能榜](https://www.antutu.com/web/ranking) | 页面是 Nuxt 应用，成绩藏在 `<script type="application/json" data-nuxt-data="nuxt-app">` 的 **devalue 引用表**里；脚本自己实现了解码器把扁平数组还原成对象树 | **118 款机型**，榜期 `202608`，含总分 / CPU / GPU / 内存 / UX 五项子分 |
| 电脑 | [Notebookcheck CPU Benchmark List](https://www.notebookcheck.net/Mobile-Processors-Benchmark-List.2436.0.html) | 解析 5 MB 的 HTML 表格（22 列），按型号后缀/TDP 判定笔记本·台式机·工作站，剔除 EPYC/Xeon 服务器平台 | **200 颗处理器**，Cinebench R23 单/多核、GeekBench 6 单/多核、7-Zip，榜首 Threadripper PRO 7995WX（R23 多核 107681） |

脚本一次产出两份文件：

- `app/src/main/assets/rankings.json`（约 115 KB）—— APK 内置快照，App 启动即读；
- `data/FallbackData.kt` —— **由脚本生成**的兜底表（两个榜各取前几名），万一内置快照读不出来也有真实数据可显示。

任一来源抓取失败脚本会**非 0 退出**，不会写出半截数据；原始 HTML 缓存在 `/var/tmp/rankladder-crawler`。

> 榜单只有跑分。屏幕 / 电池 / 机身 / 影像这些参数和实拍图由第二个爬虫
> `tools/fetch_phone_details.py` 补齐，产物是 `assets/phone_details.json` + `assets/photos/*.jpg`，
> 详见**第 4 节**。

### 1.2 运行时在线抓取（App 内）

`data/AntutuCrawler.kt` 在手机上直接抓安兔兔官方榜：下载页面 → 用 `data/Devalue.kt`（Kotlin 版 devalue 解码器）
还原 Nuxt 状态 → `SnapshotJson.parseAnTuTuPayload` 转成设备行 → 替换手机天梯，原始载荷缓存到
`filesDir/antutu-payload.json`，下次离线启动也能用。

- 设置页 →「数据」卡片：显示当前来源 / 数据版本 / 抓取时间 / 收录条目，可**立即抓取安兔兔最新榜单**、清除本地缓存、开启**启动时自动抓取**；
- 抓取失败会显示原因，并继续用上一次的数据，不会白屏；
- 电脑榜体量太大（5 MB 页面），只在构建期抓取；
- 入口地址是 `https://www.antutu.com/web/ranking`（旧入口 `/en/ranking/rank.htm` 会 301 跳到 **http**，`HttpURLConnection` 不跨协议跟随，v1.1.4 已改为自己跟随重定向 —— 见第 3 节）。

数据流：`内置兜底样本 → APK 内置快照(assets) → 本地抓取缓存 → 在线抓取`，越往后越新，全部走
`Repository` 的 `StateFlow<LadderData>`，界面自动重组。

---

## 2. 深浅色与切换动画修复（v1.1.3）

三个问题都是实测定位的，不是猜的。

### ① 深色模式下大量文字是黑色

探针测试实测四种模式的主题色，结论：

```
DARK/SYSTEM: bg=#FF121316(L=0.01) onSurface=#FFE3E2E6(L=0.76) contentColor=#FF000000
LIGHT/SYSTEM: bg=#FFFAF9FD(L=0.95) onSurface=#FF1B1B1F(L=0.01) contentColor=#FF000000
```

配色方案本身是对的，但 `contentColor` 四种模式**全是纯黑**。原因是
**Material3 的 `MaterialTheme` 并不提供 `LocalContentColor`，只有 `Surface` 会**；
我的页面全是裸 `Column` + `Modifier.background(...)`，于是没有显式颜色的 `Text`
一律继承根节点的黑色 —— 浅色下恰好能看，深色下就是黑底黑字。

修复：`theme/Theme.kt` 在 `MaterialTheme` 内层套一层
`Surface(color = scheme.background, contentColor = scheme.onBackground)`，
所有未指定颜色的文字自动跟随配色方案。回归测试 `ThemeContrastTest` 断言
浅色下文字亮度 < 0.3、深色下 > 0.5 且不等于纯黑。

### ② 浅色模式下底栏偏黑

上游 `LiquidBottomTabs` 用 `isSystemInDarkTheme()` 取色，它只看**系统**深色开关，
和「设置 → 主题模式」无关。系统深色 + App 浅色时，底栏就会取到深色的
`containerColor = #121212@0.4`，看起来发黑（反之系统浅色 + App 深色时底栏会发白）。

修复：不改上游源码，而是在调用处用 `CompositionLocalProvider` 覆盖 `LocalConfiguration`
的夜间位（`forcedNightUiMode()`，只改 `UI_MODE_NIGHT_MASK`，保留 `UI_MODE_TYPE_MASK`），
让上游的 `isSystemInDarkTheme()` 报告 App 实际解析出的深浅色。
`forcedNightUiMode` 抽成纯函数单测，另有一个组合测试验证覆盖后
`isSystemInDarkTheme()` 确实翻转。

### ③ 界面切换没有动画

`MainScreen` 的页签内容改用 `AnimatedContent`：按页签顺序决定方向
（向右进 / 向左退），`slideInHorizontally(1/6 宽) + fadeIn(220ms)` 与
`slideOutHorizontally + fadeOut(160ms)` 组合，260ms `tween`。
详情页本身是 `ModalBottomSheet`，自带上滑动画。

---

## 3. 安兔兔 301 修复 + 可折叠筛选（v1.1.4）

### ① App 内抓取失败：`HTTP 301`

**根因（本次用 `curl` 实测，不是猜的）**：安兔兔把旧入口 `https://www.antutu.com/en/ranking/rank.htm`
永久搬走了，返回的是

```
HTTP/1.1 301 Moved Permanently
Location: http://www.antutu.com/web/ranking
```

注意 `Location` 是 **http**（协议降级）。Java 的 `HttpURLConnection` 只在**同协议内**自动跟随重定向，
遇到 https → http 会直接停在 301，于是 `responseCode` 就是 301，旧代码把它当成失败抛出，
设置页显示 `抓取失败：HTTP 301`。

**两处修复**（`data/AntutuCrawler.kt`）：

1. 入口直接用最终地址 `RANKING_URL = https://www.antutu.com/web/ranking`
   （`curl` 实测该地址返回 **200 / 254,050 字节**），构建期爬虫 `tools/fetch_rankings.py` 同步改掉；
2. 重定向改成**自己跟随**：`instanceFollowRedirects = false`，循环最多 `MAX_REDIRECTS` 次读
   `Location`，交给 `resolveRedirect(base, location)` 解析——绝对地址、协议相对（`//host/…`）、
   相对路径三种形式都能处理，**跨协议也照跟**；没有 `Location` 或非 2xx 都抛带 URL 的 `IOException`，
   超过上限明确报 `more than N redirects from …`，不会再出现「只有一个裸状态码」的报错。

新增 2 个用例：`AntutuPayloadTest.redirects are resolved across protocols and relative paths`（覆盖
`resolveRedirect` 的三种形式与跨协议）、`the crawler entry point is the final address that answers 200`
（钉住入口地址，防止以后再被 301 打到）。

### ② 手机 / 电脑榜的筛选区可折叠

- 搜索框下方的「筛选与排序」区**默认收起**，点整行标题展开 / 收起（`ui/RankingScreen.kt` 的
  `FilterToggleButton` + `AnimatedVisibility`：展开 `expandVertically(240ms) + fadeIn(200ms)`，
  收起 `shrinkVertically(180ms) + fadeOut(140ms)`，右侧箭头 `animateFloatAsState` 旋转 180°）；
- 收起时标题行右侧显示**当前选择摘要**（指标短名 + 品牌 / 形态），有生效筛选时再显示一个**数量徽标**，
  避免「列表被筛过却看不出来」；
- 展开状态放在 `UiState.Filters.expanded` 里，**手机榜与电脑榜各自记住**，切页签不丢；
  电脑榜的「设备类型」ChipRow 也在折叠区内。
- 新增 `FilterCollapseTest`（3 个用例）：默认收起且点击可展开、电脑榜有独立折叠区且含设备类型、
  选中品牌后出现数量徽标与摘要。

---

## 4. 详情页：完整参数 + 实拍图（v1.1.5）

点一台手机进详情页，除了跑分，现在还有**屏幕 / 电池 / 机身 / 影像 / 平台 / 连接 / 上市**七个分区
（每台 35～53 行参数）和一张**实拍图**。

### 4.1 数据从哪来（实测过的路）

榜单页只有跑分：安兔兔那 254 KB 页面里 grep 不到任何 `screen` / `battery` / `display` 字段，
只有一张站点 logo 图。所以参数得另找来源，逐个试下来：

| 来源 | 结果 |
| --- | --- |
| 安兔兔榜单载荷 | 只有跑分与 RAM/ROM，没有屏幕/电池 |
| GSMArena 站内搜索 `res.php3?sSearch=` | **403 之后是 Cloudflare Turnstile**，搜不了 |
| GSMArena `sitemap.xml → sitemaps/phones.xml` | 4 MB / 42,244 条，但**只更新到 2025 年中**（最大 id 13950），榜单上的新机一个都不在里面 |
| GSMArena **品牌列表页** | **能直接访问且是最新的** → 用它当索引 |
| kimovil / 91mobiles / phonearena | 403；ZOL 跳 `service.zol.com.cn/checking`；nanoreview 404 |

于是 `tools/fetch_phone_details.py` 这么走：

1. `makers.php3` 现取各品牌的数字 id（不写死）；Poco/Redmi 归在 Xiaomi 名下、iQOO 归在 vivo 名下、
   nubia / Red Magic 归在 **ZTE** 名下（GSMArena 没有独立的 nubia maker）；
2. 翻品牌列表页 `https://www.gsmarena.com/<brand>-phones-f-<id>-0-p<N>.php`（按发布时间倒序，
   翻到没有新条目为止），拿到 **机型页 URL + 大图 URL + 一句话摘要**；
3. 名字对齐（见 4.2）；
4. 抓机型页，解析 `<td class="ttl">`（字段）+ `<td class="nfo">`（值）表格，按 `SPEC_PLAN`
   映射成中文分区；
5. 下 `https://fdn2.gsmarena.com/vv/bigpic/<slug>.jpg`，缩到 360 px、JPEG q74，
   写进 `app/src/main/assets/photos/<机型 id>.jpg`（平均 ~8 KB，116 张共 ~940 KB）。

**两个踩过的坑**（都留在代码注释里）：

- 品牌页必须**逐个 `<li>` 解析**。早先用一条跨条目的正则，遇到一个没有 `<strong><span>` 的条目，
  就会把**下一台机型的名字安到上一台的链接上**（realme GT 7T 就是这么丢的）；
- 抓太快会被整站 429（实测封了约 1 小时，连首页都 429）。所以默认 2 s 间隔 + `10/30/90/180 s` 退避 +
  **断点续跑**（已抓到的机型不重抓，图片已存在就跳过）。被封期间还发现：**主站 429 时图片 CDN
  `fdn2.gsmarena.com` 仍然 200**，于是加了 `--photos-only`（先只下图）和 `--via-archive`
  （参数页改走 `https://web.archive.org/web/2id_/<原地址>` 的原始快照）。
  顺带一个坑：归档会把 gzip 原样吐出来却不一定带 `Content-Encoding`，必须按魔数 `1f 8b` 再解一次，
  否则整页参数全丢（本次 116 台的参数全部来自归档快照，字段与线上一致）。

### 4.2 名字对齐：宁可少一台，也不要张冠李戴

榜单名（`REDMI K90 Pro Max`）和 GSMArena 名（`Xiaomi Redmi K90 Pro Max 5G`）写法差很多，
所以按 token 做**子集匹配 + 分级放宽**：严格子集 → 数字粘连（`Neo 10` = `Neo10`）→ 忽略 `5G/4G` 后缀
（GSMArena 对 5G-only 机型反而不写，如 `Galaxy A55`、`Poco X6 Pro`）。
**多出来的词只允许是品牌词 / 网络制式 / 地区版本**（`Xiaomi`、`5G`、`(Global)`），
否则判为不同机型：

- `Galaxy Z Fold8` **不会**匹配到 `Galaxy Z Fold8 Ultra`；
- `Poco F3` **不会**匹配到 `Poco F3 GT`。

结果：**116/118 命中**，`分辨率`/`电池容量`/`屏幕尺寸`/`重量`/`充电`/`后置摄像头` 六项覆盖率 100%。
剩下 2 台（`Galaxy Z Fold8`、`TECNO POVA 7 Neo`）GSMArena 确实没有对应页面，
详情页显示「暂未收录该机型更详细的参数」，不留空卡片。

### 4.3 App 里怎么用

- `assets/phone_details.json`（按**机型 id** 对齐，所以在线重抓榜单后老机型仍能对上）
  + `assets/photos/<id>.jpg`；
- `data/PhoneDetails.kt` 解析成 `PhoneDetailBook`，`RankViewModel.bootstrap` 在 IO 线程加载，
  走 `StateFlow` 暴露；
- `ui/Photo.kt`：`BitmapFactory` + `inSampleSize` 降采样解码，**不引第三方图片库**；解不出来就画占位图，
  布局不塌；
- 详情页七个分区做成**可折叠卡片**（前三个默认展开），沿用筛选区那套 `expandVertically` /
  箭头旋转动画；底部写明「详细参数与实拍图来源 GSMArena · 抓取于 …」。

> **版权提示**：GSMArena 的参数与图片版权归原作者，这里仅作演示内置。正式分发前请替换为自有或已授权素材。

---

## 5. 横向对比（v1.1.6）

底栏新增「**对比**」页签（手机 / 电脑 / **对比** / 收藏 / 设置，共 5 个 tab）：

- 两个 **A / B 卡片**各选一台设备。挑选器带搜索；一旦 A 选定，B 的挑选器会**自动限定同一榜单**，
  避免拿手机和 CPU 互比；
- 选满两台后出**逐项对比表**：每台的综合指数 + 实拍图，下面把该榜单的每一项跑分（总分 / CPU / GPU /
  内存 / UX）和详细参数里可比的数值项（分辨率按像素面积、刷新率、峰值亮度、电池容量、充电功率、
  重量——重量是越小越好）排成两列，**优胜一格用主题色加粗高亮**；再附系统 / 芯片 / GPU / 后置摄像头
  等只读信息行；
- 详情页里也加了「**加入对比**」按钮，和收藏并排，选中后变成「已在对比」；
- 优胜判定是纯函数（`ui/Compare.kt`），不依赖 Compose，直接拿真实数据做单元测试。

收藏页原有的勾选对比保留，但上限从 3 收到 2，与对比页共用同一份选择，两边互相可见。

---

## 6. 底栏：原项目的组件，零改动（v1.1.2；v1.1.3 ~ v1.1.6 都没动过它的源码）

底栏不是我手写的，而是把 **Kyant0/AndroidLiquidGlass 官方 catalog 里的 `LiquidBottomTabs`**
整套搬进来用（Apache-2.0，每个文件头注明来源路径）：

```
app/src/main/java/com/kyant/backdrop/catalog/
├── components/LiquidBottomTabs.kt     官方底栏组件（三层玻璃 + 胶囊拖动）
├── components/LiquidBottomTab.kt      官方单个页签
└── utils/DampedDragAnimation.kt       官方阻尼拖动动画（胶囊手感来自这里）
    utils/DragGestureInspector.kt      官方手势识别
    utils/InteractiveHighlight.kt      官方按压高光（RuntimeShader）
    utils/Coroutines.kt                awaitFrame()（多平台 expect/actual 收敛为 Android 实现）
```

**下列 5 个文件与上游逐字节一致，没有任何功能改动**，校验方式（去掉文件头的署名注释后 diff 上游，
实测输出为「OK …」）；`utils/Coroutines.kt` 例外——上游是多平台 `expect/actual`，
非多平台模块里无法照搬，因此收敛为它的 androidMain 实现（`kotlinx.coroutines.android.awaitFrame()`，
全限定调用避免自递归）：

```bash
for f in components/LiquidBottomTabs.kt components/LiquidBottomTab.kt \
         utils/DampedDragAnimation.kt utils/DragGestureInspector.kt utils/InteractiveHighlight.kt; do
  diff <(tail -n +6 app/src/main/java/com/kyant/backdrop/catalog/$f) \
       <上游克隆>/app/src/commonMain/kotlin/com/kyant/backdrop/catalog/$f && echo "OK $f"
done
```

官方组件的结构（保持原样）：
1. **底层**：整条胶囊 `drawBackdrop(shape = { Capsule() }, effects = { vibrancy(); blur(8dp); lens(24dp, 24dp) })`；
2. **中层**：一份 `alpha(0f)` + `layerBackdrop` 的隐藏页签层，用 `ColorFilter.tint(accentColor)` 把选中态颜色录进去；
3. **顶层胶囊**：`rememberCombinedBackdrop(backdrop, tabsBackdrop)` 同时采样「背后内容」和「上色后的页签」，
   再叠 `lens(chromaticAberration = true)` + `Highlight` + `Shadow` + `InnerShadow`，
   并用 `DampedDragAnimation` 的 `velocity` 做拉伸形变（拖得快会被拉长，松手回弹）。

拖动是**上游原生行为**：按住胶囊左右滑动即可切换页签，胶囊跟手、按最近页签吸附，
拖动过程中内容实时切换（所以玻璃里折射的是新页面）。先按住再滑同样成立。

> 代价（明确说明）：上游把强调色写死成蓝色 `0xFF0088FF`（深色 `0xFF0091FF`），
> 所以**底栏胶囊的高亮色不跟随莫奈取色**；App 其余部分（列表、筛选、详情页、纯色底栏回退）
> 仍然是 Material 3 + 莫奈配色。若要让底栏也跟随取色，需要给上游组件加颜色参数——
> 那属于修改原项目源码，本次按要求没有做。

我这边只有一个薄封装 `ui/GlassBottomBar.kt`：提供页签内容（图标 + 文字）、窗口内边距、
以及关闭玻璃时的纯色回退；它只用上游组件的**原有参数**调用。

### 一次真实排错

第一版封装把 `selectedTabIndex` 写成每次重组都新建的 lambda，而上游用
`remember(selectedTabIndex)` 保存当前下标 —— 于是每次重组都被重置，**向左拖回上一个页签失效**。
改成 `rememberUpdatedState` 后正常（这属于我的封装层，不涉及改动上游文件）。
拖动测试用真实触摸事件跑：先按住 ~450ms，再走 80dp（≈1.19 个页签宽），
断言切到下一个页签 / 拖回上一个页签。

### 补充依赖

上游示例用的 `Capsule()` 形状来自另一个 artifact，已按官方用法加入
`io.github.kyant0:shapes:1.2.1`（Maven 中央仓库，minCompileSdk 37，与本项目一致）。

---

## 7. 开发环境（本机已补全）

沙箱里原本只有 JDK 11、没有 Android SDK、没有 Gradle。已补全为一套完整的 Android 构建链，
**全部安装在工作区之外**（`/var/tmp`、`/usr/lib/jvm`），因此不会占用工作区的体积与文件数配额：

| 组件 | 版本 | 安装位置 |
| --- | --- | --- |
| JDK | OpenJDK 21.0.12（apt `openjdk-21-jdk-headless`） | `/usr/lib/jvm/java-21-openjdk-amd64` |
| Android SDK 命令行工具 | 11076708 | `/var/tmp/android-sdk/cmdline-tools/latest` |
| Platform | `android-36`、`android-37.0` | `/var/tmp/android-sdk/platforms` |
| Build-Tools | `36.0.0`、`37.0.0` | `/var/tmp/android-sdk/build-tools` |
| Gradle | 9.7.1 | `/var/tmp/gradle/gradle-9.7.1`（`GRADLE_USER_HOME=/var/tmp/gradle-home`） |
| AGP / Kotlin | 9.3.2 / 2.4.10（Compose 编译器插件） | Gradle 依赖 |

> `androidx.compose.*:1.12.x` 与 `backdrop 2.0.1` 的 AAR 元数据要求 `minCompileSdk=37`、
> `minAndroidGradlePluginVersion=9.1.0`，所以必须使用 compileSdk 37 + AGP 9.x。
> 2 GB 内存的机器上 R8 会被 OOM Killer 杀掉，因此额外挂了 3 GB swapfile（`/var/tmp/swapfile`）。

### 构建命令

```bash
export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
export ANDROID_HOME=/var/tmp/android-sdk
export GRADLE_USER_HOME=/var/tmp/gradle-home          # 放在工作区外，避免撑爆配额

python3 tools/fetch_rankings.py                                          # ① 抓榜单数据
python3 tools/fetch_phone_details.py                                     # ② 抓手机详细参数 + 实拍图
gradle --project-cache-dir=/var/tmp/pcache :app:testDebugUnitTest        # ③ 52 个测试
gradle --project-cache-dir=/var/tmp/pcache :app:lintRelease              # ④ 静态检查
gradle --project-cache-dir=/var/tmp/pcache :app:assembleRelease          # ⑤ 出 APK
```

在自己电脑上：Android Studio 直接打开 `rankladder/` 即可（`local.properties` 里的 `sdk.dir`
需要改成你本机的 SDK 路径，该文件不入库）。

### 签名

`keystore/rankladder.jks` 是**演示用自签证书**（口令见 `keystore.properties`），
仅为让 `assembleRelease` 直接产出可安装的 APK。正式发布请换成你自己的密钥。

---

## 8. 功能

| 页签 | 内容 |
| --- | --- |
| 手机 | 118 款机型天梯（安兔兔官方榜 202608）。综合指数 / 安兔兔总分 / CPU / GPU / 内存 / UX 排序，品牌筛选、关键词搜索、成绩条、星标收藏。**筛选与排序区可折叠**（默认收起）|
| 电脑 | 200 颗处理器天梯（笔记本 120 / 台式机 79 / 工作站 1）。Cinebench R23 单·多核 / GeekBench 6 单·多核 / 7-Zip 排序，形态与品牌筛选。**独立的折叠筛选区**（默认收起，含设备类型）|
| 对比 | 选 A/B 两台设备横向对比：逐项跑分 + 可比参数（分辨率/刷新率/亮度/电池/充电/重量），优胜项高亮；挑选器自动限定同榜（v1.1.6，见第 5 节）|
| 收藏 | 收藏列表 + 最多 3 款横向对比表 |
| 设置 | 主题模式、取色方式、自定义主色、玻璃开关、紧凑列表、**数据抓取**、数据说明、关于 |

点击任意一行弹出详情：**实拍图** + 综合指数与排名、全部跑分（相对榜首的成绩条）、硬件规格，
以及七个可折叠的详细参数分区（屏幕 / 电池与充电 / 机身与外观 / 影像 / 平台与存储 / 连接与传感器 /
上市与价格，手机榜 116/118 台有，见第 4 节），底部标注数据来源。

**综合性能指数**：每项跑分除以该项最佳成绩，按权重加权平均，再归一化到「榜首 = 100」。
权重由抓取的指标表决定（写在 `rankings.json` 里，加指标不用改 App）：
手机 安兔兔总分 40% / GPU 25% / CPU 20% / UX 10% / 内存 5%；
电脑 R23 多核 35% / GB6 多核 30% / GB6 单核 15% / R23 单核 10% / 7-Zip 10%。

---

## 9. 实现要点

### 底栏液态玻璃（Backdrop）

`ui/MainScreen.kt` 把除底栏以外的全部内容录进一个 backdrop 图层，底栏再去采样它：

```kotlin
val backdrop = rememberLayerBackdrop {
    drawRect(background)   // 背景色也要画进去，否则玻璃里是透明的
    drawContent()
}
Box(Modifier.fillMaxSize().layerBackdrop(backdrop)) { /* 列表、设置页… */ }

GlassBottomBar(backdrop = backdrop, selected = ui.tab, glassEnabled = settings.glassEnabled, …)
```

`ui/GlassBottomBar.kt` 里两层玻璃：外层胶囊（`vibrancy()` → `blur(8dp)` → `lens(18dp, 40dp)`
+ 半透明 surface 保证可读性），内层是随页签滑动的玻璃胶囊（用 `BlendMode.Hue` 把主题色混进折射画面，
这是库文档推荐的做法）。低于 API 31/33 时库会自动跳过 `RenderEffect` / `RuntimeShader`，
此时退化为纯色胶囊（`glassEnabled = false` 也走这条路径）。

### 莫奈取色

`theme/Monet.kt` + `theme/Theme.kt`：

- Android 12+：`dynamicLightColorScheme` / `dynamicDarkColorScheme`（系统 Monet 引擎）。
- Android 8~11（或用户选择「壁纸取色 / 自定义主色」）：
  先用 `WallpaperManager.getWallpaperColors()`（API 27+，无需权限）拿主色；
  拿不到就自己采样壁纸位图（按色相分桶，用饱和度×明度权重挑代表色）；
  再由种子色经 **Oklch** 生成整套 Material 3 色板（明度阶 0–100），这是对 HCT 色板的近似实现。
- 回到前台会重新取色（`LifecycleResumeEffect`），换壁纸即时生效。

### 数据层

- `data/Models.kt` —— `Device` / `Metric` / `LadderData`（筛选、排序、综合指数都在快照对象上算，每个分类只算一次并缓存）；
- `data/SnapshotJson.kt` —— 解析 `rankings.json`，以及把安兔兔载荷转成设备行（含品牌归一：`Galaxy…→三星 Samsung`、`Mi 17→小米 Xiaomi`）；
- `data/Devalue.kt` —— Nuxt devalue 引用表解码器；
- `data/Repository.kt` —— 快照持有者（`StateFlow`），负责 assets / 缓存 / 在线三种来源的装载与合并；
- `data/AntutuCrawler.kt` —— HTTPS 抓取（gzip、浏览器 UA、超时、重定向）。

新增一条天梯只需要：爬虫多写一个 `categories.XXX` 节点（指标 + 设备），App 端加一个 `Category`。

---

## 10. 验证结果

在本机（2 vCPU / 2 GB）实跑：

- `tools/fetch_rankings.py` — 本次（2026-09-17 12:02:32Z，走新的最终地址）重新抓取成功：`118 款机型，榜期 202608，最高分 4213827`（归一化基准，榜首 Red Magic 11 Pro 实得 4013168）；`200 颗处理器（榜首 AMD Ryzen Threadripper PRO 7995WX / R23 多核 107681）`；写出 `rankings.json`（118,127 字节 / 115.4 KB）+ 兜底表
- `tools/fetch_phone_details.py` — 详细参数 **116/118 台**（七分区齐全，每台 35～53 行；`分辨率`/`电池容量`/`屏幕尺寸`/`重量`/`充电`/`后置摄像头` 覆盖率 100%），**实拍图 116 张 / 992 KB**；未收录 2 台（`Galaxy Z Fold8`、`TECNO POVA 7 Neo`，GSMArena 无对应页面）；写出 `phone_details.json`（365,074 字节）
- `:app:testDebugUnitTest :app:lintRelease :app:assembleRelease` — **BUILD SUCCESSFUL in 4m33s，58 个测试全部通过，0 失败 0 错误**（14 个测试类）
  - `CompareTest`（4）：`firstNumber`/`resolutionPixels` 解析、跑分行优胜方向与数值一致且榜首赢「总分」、重量行越小越好（v1.1.6 新增）
  - `CompareScreenTest`（2，Robolectric + Compose）：对比页选 A/B 两台出逐项表（电池/分辨率/重量行 + 两张实拍图）、清空回到选择态；B 的挑选器自动限定与 A 同榜（v1.1.6 新增）
  - `RankingTest`（9）：排序、综合指数归一化、缺失跑分隐藏、未知指标回退、搜索/品牌/形态筛选、收藏标记、指标权重
  - `CrawledSnapshotTest`（6，Robolectric）：**加载真实的 `assets/rankings.json`**，校验两个榜都 ≥100 条、id 唯一、权重和为 1、指标 id 与来源一致、provenance 含来源名、内置兜底表也能独立成榜
  - `AntutuPayloadTest`（7，Robolectric）：devalue 引用表解码、载荷→设备行、品牌归一、从 HTML 抽取载荷、缺载荷时明确报错、**`resolveRedirect` 三种 Location 形式与跨协议跳转**、**入口地址钉在返回 200 的最终地址**（v1.1.4 新增 2 个）
  - `FilterCollapseTest`（3，Robolectric + Compose）：筛选区默认收起且点击展开、电脑榜有独立折叠区（含设备类型）、选中品牌后出现数量徽标与摘要（v1.1.4 新增）
  - `GlassPillDragTest`（3，Robolectric + Compose）：**长按胶囊右拖切到电脑榜、左拖切回手机榜**、普通点击仍可用
  - `AppSmokeTest`（5，Robolectric，真实启动 `MainActivity`）：手机榜渲染（榜首 Red Magic 11 Pro）、切电脑榜、切换排序指标、设置页主题控件、**设置页的抓取控件**
  - `ThemeContrastTest`（3）：**深浅色内容色对比**（深色下文字不得为黑）、`forcedNightUiMode` 纯函数、
    覆盖 `LocalConfiguration` 后 `isSystemInDarkTheme()` 跟随 App 主题
  - `PhoneDetailsTest`（6，Robolectric）：**加载真实的 `assets/phone_details.json`**，校验覆盖 ≥3/4 的手机榜、没有孤儿条目、榜首有分辨率/电池/尺寸/重量、**每个 photo 路径都能在 assets 里打开且 >2 KB**、内置图能被 `decodeAssetPhoto` 解成 ≤480 px 的真实位图、缺参数的机型降级不崩（v1.1.5 新增）
  - `DeviceDetailSheetTest`（2，Robolectric + Compose）：真点开榜首详情页，断言屏幕尺寸/分辨率/电池容量/充电/重量都渲染出来、实拍图节点存在、`影像` 分区默认收起点开才有后置/前置、来源署名可见；电脑榜机型走「暂未收录」降级文案（v1.1.5 新增）
  - `MonetSchemeTest`（4）、`AndroidRuntimeTest`（3，含 `autoRefresh` 持久化）、`GlassFallbackTest`（1）
- `:app:lintRelease` — **0 error / 0 fatal**，3 条无害 warning（`UnusedAttribute` / `NewerVersionAvailable` / `ObsoleteSdkInt`）
- `:app:assembleRelease` — **BUILD SUCCESSFUL**
- `aapt2 dump badging` — `package=dev.arena.rankladder`，`versionCode=8`，`versionName=1.1.6`，`minSdk=26`，`targetSdk=37`，`application-label='天梯榜'`，`uses-permission: INTERNET`
- `apksigner verify --verbose` — **Verifies**（APK Signature Scheme v2）
- `unzip -l` — 包内含 `assets/rankings.json`（118,127 字节）、`assets/phone_details.json`（365,074 字节）与 **116 个 `assets/photos/*.jpg`**，即榜单、详细参数、实拍图都确实打进了 APK
- `mapping.txt` — R8 之后仍保留 `com.kyant.backdrop.catalog.*`（30 个类，即搬入的官方底栏）、
  `com.kyant.shapes.*`（5 个类，`Capsule`）与 `com.kyant.backdrop.*`（73 个类），说明原项目的
  玻璃底栏与形状库都真实参与运行，没有被裁掉（v1.1.5 比上版各少 1 个类，是 R8 按新调用图裁掉的未用成员）

**未覆盖的部分**：沙箱没有 `/dev/kvm`，跑不了模拟器或真机，因此
① API 33+ 上 `RuntimeShader` 折射的实际渲染效果、② 真机上手指拖动胶囊的手感与触感反馈、
③ 真实网络下 App 内在线抓取（`curl` 已实测最终地址返回 200 / 254,050 字节，`HttpURLConnection` 的重定向与解析逻辑也有单元测试，但没在真机上点过那个按钮）、④ 折叠区展开动画的观感、⑤ 详情页实拍图在真机上的清晰度与排版（Robolectric 里验证的是「位图能解出来、节点在语义树里、文字都渲染了」，不是像素效果）
没有肉眼验证过。Robolectric 自带的 AGSL 编译器不支持该库 lens 着色器的写法
（`cannot swizzle value of type 'shader'`），所以 UI 运行时测试固定在 SDK 30。

---

## 11. 目录结构

```
rankladder/
├── settings.gradle.kts / build.gradle.kts / gradle.properties
├── keystore/ + keystore.properties           演示签名
├── tools/fetch_rankings.py                   榜单爬虫（安兔兔 + Notebookcheck）
├── tools/fetch_phone_details.py              详细参数 + 实拍图爬虫（GSMArena，v1.1.5）
├── tools/make_demo_keystore.sh               生成演示签名（v1.1.6）
├── LICENSE / NOTICE / CONTRIBUTING.md        开源三件套（v1.1.6）
└── app/src/
    ├── main/assets/rankings.json             抓取到的榜单快照（构建产物，已入库）
    ├── main/assets/phone_details.json        手机详细参数（按机型 id 对齐，构建产物）
    ├── main/assets/photos/*.jpg              116 张实拍图缩略图（360 px，共约 1 MB）
    ├── main/java/dev/arena/rankladder/
    │   ├── MainActivity.kt
    │   ├── data/       Models · LadderData · SnapshotJson · Devalue · Repository
    │   │               AntutuCrawler · PhoneDetails · FallbackData(生成)
    │   ├── prefs/      SettingsStore（SharedPreferences）
    │   ├── theme/      Theme · Monet（Oklch 色板 + 壁纸取色）
    │   └── ui/         MainScreen · GlassBottomBar · RankingScreen · DeviceDetail
    │                   Photo · Compare(纯逻辑) · CompareScreen · FavoritesScreen
    │                   SettingsScreen · Components · RankViewModel
    ├── main/res/       drawable(矢量图标) · mipmap(自适应图标) · values(主题)
    └── test/java/      14 个测试类（58 个用例）
```

## 12. 数据来源与免责

手机成绩来自**安兔兔官方性能榜**（榜期 202608，抓取于 2026-09-17），
电脑成绩来自 **Notebookcheck CPU Benchmark List**（Cinebench R23 / GeekBench 6 / 7-Zip，抓取于 2026-09-17）。
两者均为公开榜单，版权归原作者所有，本应用只做转载与归一化计算。
不同系统版本、散热与后台负载会带来明显波动（笔记本尤甚），成绩仅供横向参考，不构成购买建议。

手机的**详细参数与实拍图**来自 **GSMArena** 公开机型页（v1.1.5，抓取于 2026-09-17；因主站限流，
本次参数页取自 Internet Archive 的原始快照，字段与线上一致），图片已缩到 360 px 内置。
这些内容的版权归原作者，本仓库仅作演示，正式分发前请替换为自有或已授权素材。

## 13. 许可

应用代码 MIT；`com/kyant/backdrop/catalog/**` 下的文件版权归 Kyant0，遵循 **Apache-2.0**
（底栏液态玻璃效果与拖动手感均来自该项目）。


---

## 14. 开源准备（v1.1.6）

本仓库按可直接公开的标准整理：

- **LICENSE / NOTICE**：应用代码 MIT；`com/kyant/backdrop/catalog/**` 与 backdrop/shapes 库为
  Apache-2.0（版权归 Kyant0）；榜单与参数/图片数据版权归原来源（见第 12 节），NOTICE 里逐一列出。
- **.gitignore**：忽略 Gradle/IDE/Python 产物、`local.properties` 与**签名秘密**
  （`keystore.properties`、`keystore/*.jks` 均不入库）。
- **签名可复现且无秘密入库**：`keystore.properties.example` 附带演示值；
  `tools/make_demo_keystore.sh` 用 keytool 现场生成演示 keystore。缺 keystore 时
  `assembleRelease` 自动回退 debug 签名，任何干净克隆都能出可安装 APK。
- **路径不再写死**：两个爬虫的 HTML 缓存目录可用 `RANKLADDER_CACHE` 环境变量覆盖
  （默认 `/var/tmp/rankladder-crawler`）；SDK 走 `local.properties` / `ANDROID_HOME`（不入库）。
- **CONTRIBUTING.md**：环境、数据刷新、签名、以及几条底线（底栏上游文件保持逐字节一致、
  新榜单走加法、UI 文案用简体中文、测试先红后绿）。
- 数据资产（`rankings.json` / `phone_details.json` / `photos/`）作为构建产物入库，便于评审 diff；
  重新抓取用第 1、4 节的命令即可。
