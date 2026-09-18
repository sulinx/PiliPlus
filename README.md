<div align="center">
    <img width="200" height="200" src="assets/images/logo/logo.png">
</div>



<div align="center">
    <h1>PiliPlus</h1>
<div align="center">
    
![GitHub repo size](https://img.shields.io/github/repo-size/bggRGjQaUbCoE/PiliPlus) 
![GitHub Repo stars](https://img.shields.io/github/stars/bggRGjQaUbCoE/PiliPlus) 
![GitHub all releases](https://img.shields.io/github/downloads/bggRGjQaUbCoE/PiliPlus/total) 
</div>
    <p>使用Flutter开发的BiliBili第三方客户端</p>
    
<img src="assets/screenshots/510shots_so.png" width="32%" alt="home" />
<img src="assets/screenshots/174shots_so.png" width="32%" alt="home" />
<img src="assets/screenshots/850shots_so.png" width="32%" alt="home" />
<br/>
<img src="assets/screenshots/main_screen.png" width="96%" alt="home" />
<br/>
</div>


<br/>

## 🔧 本 fork 的改动：Android 原生播放后端 + 杜比视界

> 仓库 [`sulinx/PiliPlus`](https://github.com/sulinx/PiliPlus) ｜ 分支 `feat/dolby-vision` ｜ 基线 = 上游 `main`

在原版之上为 **Android** 接入 **Media3 (ExoPlayer) 原生播放后端**（移植自未合并的 [PR #2145 · PA733 `feat/hdr`](https://github.com/bggRGjQaUbCoE/PiliPlus/pull/2145)，并参照 [cat3399/blbl](https://github.com/cat3399/blbl) 的做法补齐探测与回退）。
**非直播的网络视频不再交给 mpv，改由 ExoPlayer 解码**，杜比视界 / HDR Vivid / HDR 真彩会真正调用平台硬解器；普通画质同样硬解优先，且不会像 mpv 那样静默降到软解。

### 改了什么

| 项目 | 上游 | 本 fork（Android） |
| --- | --- | --- |
| 播放内核 | 全平台 mpv（media_kit） | 非直播网络流 → **Media3/ExoPlayer**；超分 / 镜像 / 听视频 / 直播 / 本地文件 → 仍为 mpv |
| 杜比视界 qn=126 | mpv 打不开 `dvh1/dvhe` → 提示「无法加载解码器…可能切换至软解」→ 软解偏色 | `video/dolby-vision` → 平台 DV 硬解器 → 输出杜比视界 |
| HDR 真彩 / HDR Vivid（125 / 129） | 同上 | 原生硬解 + 窗口 `colorMode=HDR` |
| 普通画质（AVC / HEVC / AV1） | 硬解可能失败并静默软解 | ExoPlayer 硬解优先，失败直接报错或回退 mpv，**不静默软解** |
| 设备能力探测 | 无 | 屏幕申报 HDR 能力 **或** `MediaCodecList` 存在对应硬解器（很多盒子/电视系统的 `Display.getHdrCapabilities()` 恒为空数组） |
| 失败处理 | — | 原生后端创建失败 / 播放报错 / 15 秒无首帧 → 自动回退 mpv 并提示「已使用兼容播放」 |

### 内核分工

- **Media3 原生**：非直播的 DASH 网络流（UGC / PGC 点播），含杜比视界、HDR、普通 SDR 画质
- **mpv**：直播、超分辨率（Anime4K）、镜像翻转（X / Y）、听视频（纯音频）、本地文件播放 —— 这些依赖 mpv 的滤镜与着色器管线，开启时会自动切回 mpv
- 画质菜单旁新增说明：设置 → 播放设置 → **强制 HDR** —— HDR 画质在探测不到设备支持时也强制走原生内核（普通画质无需探测，一律原生）

### 为什么要这么改

1. 很多 Android 盒子 / 电视的 `Display.getHdrCapabilities().supportedHdrTypes` **恒为空**（`dumpsys display` 同样为空），但硬件其实带 `video/dolby-vision` 解码器 —— 只看屏幕申报能力会永远判「不支持」，于是回退 mpv；
2. mpv 的 `mediacodec` 在杜比视界上会 `IllegalStateException` → `Could not open codec` → 软解 → 画面偏色（即上游画质菜单里那句「4k 和杜比视界播放效果可能不佳」）；
3. ExoPlayer 的 `MediaCodecSelector.DEFAULT` 本身就把软解器排在最后（`MediaCodecUtil.getDecoderInfosSortedBySoftwareOnly`），且 `enableDecoderFallback` 默认为 false —— 硬解优先、失败不静默降级，正是这里需要的语义。

### 下载与构建

- 产物：本仓库 **Actions → `Build DV APK` → Run workflow** 手动触发（约 7 分钟），得到 `Android_arm64-v8a` / `Android_armeabi-v7a` 两个 artifact
- **自签名**：使用仓库内固定密钥 `android/dev-release.jks`（配置见 `android/key.properties`），与官方版签名不同 —— **首次安装需先卸载官方版**；此后本 fork 的版本可直接覆盖升级
- 本地构建：`flutter build apk --release --split-per-abi --dart-define-from-file=pili_release.json`

### 验证是否真的点亮了杜比视界

```bash
adb logcat -s PiliPlusHdr flutter | grep DV
# [DV] supportsHdr qn=126 ... dvDecoder=OMX.xxx.dolby-vision.xxx -> true
# [DV] 选中流 video=video/dolby-vision codecs=dvhe.05.07 size=3840x2160 ... 10bit

# Amlogic 盒子可直接查 HDMI 输出状态
adb shell cat /sys/class/amhdmitx/amhdmitx0/hdmi_hdr_status    # → DolbyVision-Std
```

### 已知限制

- 改动**仅对 Android 生效**；iOS / Windows / Linux / macOS 行为与上游一致
- 原生后端下 mpv 专属能力（着色器 / 超分 / 音频归一化）不可用，触发即切回 mpv
- 与上游同步：本分支基于上游 `main`，跟进上游版本时 rebase 即可

<br/>

## 适配平台

- [x] Android
- [x] iOS
- [x] Pad
- [x] Windows
- [x] Linux

[![Packaging status](https://repology.org/badge/vertical-allrepos/piliplus.svg)](https://repology.org/project/piliplus/versions)

## refactor

- [ ] gRPC [wip]
- [x] 用户界面
- [x] 其他

## feat

- [x] 编辑动态
- [x] DLNA 投屏
- [x] 离线缓存/播放
- [x] 移动端支持点击弹幕悬停，点赞、复制、举报 by [@My-Responsitories](https://github.com/My-Responsitories)
- [x] 播放音频
- [x] 跳过番剧片头/片尾
- [x] 安卓端 `loudnorm` 适配 by [@My-Responsitories](https://github.com/My-Responsitories)
- [x] Win/Mac 支持极验、短信登录 by [@My-Responsitories](https://github.com/My-Responsitories)
- [x] 视频截取动图 by [@My-Responsitories](https://github.com/My-Responsitories)
- [x] AI 原声翻译
- [x] SuperChat
- [x] 播放课堂视频
- [x] 发起投票
- [x] 发布动态/评论支持`富文本编辑`/`表情显示`/`@用户`
- [x] 修改消息设置
- [x] 修改聊天设置
- [x] 展示折叠消息
- [x] 查看用户图文
- [x] 动态话题
- [x] 直播分区
- [x] 分享`视频`/`番剧`/`动态`/`专栏`/`直播`至消息
- [x] 创建/修改/删除关注分组
- [x] 移除粉丝
- [x] 直播弹幕发送表情
- [x] 收藏夹排序
- [x] 稍后再看 ~~`未看`~~ / `未看完` / ~~`已看完`~~ 分类
- [x] WebDAV 备份/恢复设置
- [x] 保存评论/动态
- [x] 高级弹幕 by [@My-Responsitories](https://github.com/My-Responsitories)
- [x] 取消/置顶评论
- [x] 记笔记
- [x] 多账号支持 by [@My-Responsitories](https://github.com/My-Responsitories)
- [x] 屏蔽带货动态/评论
- [x] 互动视频
- [x] 发评/动态反诈
- [x] 高能进度条
- [x] 滑动跳转预览视频缩略图
- [x] Live Photo
- [x] 复制/移动/排序收藏夹/稍后再看视频
- [x] 超分辨率
- [x] 合并弹幕
- [x] 会员彩色弹幕
- [x] 播放全部/继续播放/倒序播放
- [x] Cookie登录
- [x] 显示视频分段信息
- [x] 调节字幕大小
- [x] 调节全屏弹幕大小
- [x] 收藏夹/稍后再看多选删除
- [x] 搜索用户动态
- [x] 直播弹幕
- [x] 修改头像/用户名/签名/性别/生日
- [x] 创建/编辑/删除收藏夹
- [x] 评论楼中楼查看对话
- [x] 评论楼中楼定位点击查看的评论
- [x] 评论楼中楼按热度/时间排序
- [x] 评论点踩
- [x] 私信发图
- [x] 投币动画
- [x] 取消/追番，更新追番状态
- [x] 取消/订阅合集
- [x] SponsorBlock
- [x] 显示视频完整合集
- [x] 三连动画
- [x] 番剧三连
- [x] 带图评论
- [x] 视频TAG
- [x] 筛选搜索
- [x] 转发动态
- [x] 合集图片
- [x] 删除/置顶/撤回私信
- [x] 举报用户/评论/视频/动态
- [x] 删除/发布/置顶文本/图片动态
- [x] 其他

## opt

- [x] 专栏界面
- [x] 私信界面
- [x] 收藏面板
- [x] PIP
- [x] 视频封面
- [x] 回复界面
- [x] 系统通知
- [x] 评论显示
- [x] 亮度调节
- [x] 视频播放
- [x] 视频staff
- [x] 防止bottomsheet遮挡全屏视频
- [x] 其他

## fix

- [x] 番剧分集点赞/投币/收藏
- [x] bugs

<br/>

## 功能

- [x] 推荐视频列表(app端)
- [x] 最热视频列表
- [x] 热门直播
- [x] 番剧列表
- [x] 屏蔽黑名单内用户视频
- [x] 无痕模式（播放视为未登录）
- [x] 游客模式（推荐视为未登录）

- [x] 用户相关
  - [x] 粉丝、关注用户、拉黑用户查看
  - [x] 用户主页查看
  - [x] 关注/取关用户
  - [x] 离线缓存
  - [x] 稍后再看
  - [x] 观看记录
  - [x] 我的收藏
  - [x] 站内私信
  
- [x] 动态相关
  - [x] 全部、投稿、番剧分类查看
  - [x] 动态评论查看
  - [x] 动态评论回复功能

- [x] 视频播放相关
  - [x] 双击快进/快退
  - [x] 双击播放/暂停
  - [x] 垂直方向调节亮度/音量
  - [x] 垂直方向上滑全屏、下滑退出全屏
  - [x] 水平方向手势快进/快退
  - [x] 全屏方向设置
  - [x] 倍速选择/长按2倍速
  - [x] 硬件加速（视机型而定）
  - [x] 杜比视界 / HDR 原生硬解（Android，本 fork 新增）
  - [x] 画质选择（高清画质未解锁）
  - [x] 音质选择（视视频而定）
  - [x] 解码格式选择（视视频而定）
  - [x] 弹幕
  - [x] 字幕
  - [x] 记忆播放
  - [x] 视频比例：高度/宽度适应、填充、包含等
     
- [x] 搜索相关
  - [x] 热搜
  - [x] 搜索历史
  - [x] 默认搜索词
  - [x] 投稿、番剧、直播间、用户搜索
  - [x] 视频搜索排序、按时长筛选
    
- [x] 视频详情页相关
  - [x] 视频选集(分p)切换
  - [x] 点赞、投币、收藏/取消收藏
  - [x] 相关视频查看
  - [x] 评论用户身份标识
  - [x] 评论(排序)查看、二楼评论查看
  - [x] 主楼、二楼评论回复功能
  - [x] 评论点赞
  - [x] 评论笔记图片查看、保存

- [x] 设置相关
  - [x] 画质、音质、解码方式预设      
  - [x] 图片质量设定
  - [x] 主题模式：亮色/暗色/跟随系统
  - [x] 震动反馈(可选)
  - [x] 高帧率
  - [x] 自动全屏
  - [x] 横屏适配
- [ ] 等等

<br/>

## 下载

可以通过右侧release进行下载或拉取代码到本地进行编译

本 fork 的 Android 构建产物见：[Actions · Build DV APK](https://github.com/sulinx/PiliPlus/actions/workflows/dv-build.yml)（手动触发，产物为 artifact；已使用固定自签名密钥）

<br/>

## 声明

此项目（PiliPlus）是个人为了兴趣而开发，仅用于学习和测试，请于下载后24小时内删除。
所用API皆从官方网站收集，不提供任何破解内容。
在此致敬原作者：[guozhigq/pilipala](https://github.com/guozhigq/pilipala)
在此致敬上游作者：[orz12/PiliPalaX](https://github.com/orz12/PiliPalaX)
本仓库做了更激进的修改，感谢原作者的开源精神。

感谢使用


<br/>

## 致谢

- [bilibili-API-collect](https://github.com/SocialSisterYi/bilibili-API-collect)
- [flutter_meedu_videoplayer](https://github.com/zezo357/flutter_meedu_videoplayer)
- [media-kit](https://github.com/media-kit/media-kit)
- [dio](https://pub.dev/packages/dio)
- [androidx/media3 (ExoPlayer)](https://github.com/androidx/media) —— 本 fork Android 原生播放后端
- [PA733/PiliPlus `feat/hdr`](https://github.com/PA733/PiliPlus) —— Media3 原生后端的原始移植实现（上游 PR #2145）
- [cat3399/blbl](https://github.com/cat3399/blbl) —— 硬解器探测与失败回退的思路参考
- 等等

<br/>
<br/>
<br/>

## Star History

<a href="https://star-history.dera.page/#bggRGjQaUbCoE/PiliPlus&Date">
 <picture>
   <source media="(prefers-color-scheme: dark)" srcset="https://star-history.dera.page/svg?repos=bggRGjQaUbCoE/PiliPlus&type=Date&theme=dark" />
   <source media="(prefers-color-scheme: light)" srcset="https://star-history.dera.page/svg?repos=bggRGjQaUbCoE/PiliPlus&type=Date" />
   <img alt="Star History Chart" src="https://star-history.dera.page/svg?repos=bggRGjQaUbCoE/PiliPlus&type=Date" />
 </picture>
</a>
