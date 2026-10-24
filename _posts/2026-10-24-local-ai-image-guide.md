---
layout: post
title: "从零搭一套本地 AI 出图系统：8G 显存的实战指南"
date: 2026-10-24 16:30:00 +0800
categories: 折腾
---

> **说明**：本文由 AI 生成，经 BG6HMA 校对后发布。

> **硬件**：RTX 3060 Ti **8GB** / 32GB RAM / 全部装在 D 盘
> **软件**：ComfyUI 便携版 · Python 3.13 · torch 2.13.0+cu130
> **成果**：一个「打字 → 出图」的本地网页，写实/动漫双模式，支持参考图锁人物、高清修复、脸部精修
> **实测**：832×1216 出图 33 秒；带全部增强 1248×1824 约 110 秒

---

### 这篇适合谁

- 想在**自己电脑上**跑 AI 出图，不想被在线服务的额度、审核、网速卡住的人
- 显存不大（8G 级别），怀疑"我这卡是不是跑不动"的人
- 已经装了 ComfyUI，但被节点画布劝退、或者出图效果一直不对的人
- 想知道**为什么**要这么配，而不只是抄一份参数的人

### 你会得到什么

1. 一套可复现的搭建流程（每一步做什么、为什么）
2. 一套出图参数（写实/动漫/参考图，含实测耗时）
3. 一张"效果不对时怎么排查"的对照表
4. 12 个真实踩过的坑及修法——**这部分能省你最多时间**

### 怎么读

- 只想赶快跑起来 → 读 **第二、三章**
- 想搞懂原理 → 加读 **第四章**
- 出图效果不对 → 直接跳 **第五章** 对照现象找原因

---

## 一、先看清楚：这套系统由什么组成

```
┌─────────────────────────────────────────────────┐
│  你看到的：一个网页                                │
│  ┌───────────────────────────────────────────┐  │
│  │ 输入框 + 尺寸/肤色/开关 + 生成按钮          │  │
│  │ 结果图 + 缩略图画廊                        │  │
│  └───────────────────────────────────────────┘  │
└──────────────────┬──────────────────────────────┘
                   │ HTTP（同一端口，无跨域）
┌──────────────────▼──────────────────────────────┐
│  ComfyUI 服务（127.0.0.1:8188）                  │
│  ├─ 模型加载器（按需加载 / 动态显存搬运）        │
│  ├─ 采样器（扩散模型推理）                       │
│  ├─ 增强链（超分 → 重绘 → 脸部精修）             │
│  └─ 插件（Impact Pack / IPAdapter）              │
└──────────────────┬──────────────────────────────┘
                   │ 读文件
┌──────────────────▼──────────────────────────────┐
│  模型库（独立目录，约 50GB）                      │
│  底模 / VAE / LoRA / 超分 / 人脸检测 / IP-Adapter│
└─────────────────────────────────────────────────┘
```

三个关键设计决定，后面会反复用到：

1. **模型和程序分离** —— 程序可整体升级，模型不用搬
2. **网页与 ComfyUI 同源** —— 放在它的静态目录里，省掉跨域麻烦
3. **一切可用 API 驱动** —— 网页只是壳，背后是 `/prompt` + `/history`

### 为什么不直接用在线服务

| | 本地 | 在线 |
|---|---|---|
| 成本 | 电费 | 订阅/点数 |
| 隐私 | 图不出本机 | 上传到对方服务器 |
| 内容限制 | 只有模型本身的能力边界 | 平台审核 |
| 可控性 | 换底模、调参数、接 API 全自由 | 平台给什么用什么 |
| 门槛 | 要折腾（就是这篇） | 打开就用 |

---

## 二、搭起来：从零到出第一张图

### 2.1 装 ComfyUI 便携版

官方便携包解压即用，自带 Python 和 CUDA 运行时，不污染系统环境。

```
D:\MiniMax-H3\
└── ComfyUI\                      ← 便携根目录
    ├── run_nv.bat                ← 启动脚本
    ├── python_embeded\           ← 内置 Python + torch
    └── ComfyUI\                  ← 应用本体（main.py 在这）
```

> ⚠️ 便携包有**两层** `ComfyUI` 目录，很多路径问题都源于此。记法：
> **外层是运行时，内层是应用**。

### 2.2 决定模型放哪（强烈建议先做这步）

默认模型全在 `ComfyUI\ComfyUI\models\`，跟着程序走。这样有两个问题：程序目录膨胀到几十 GB；升级时模型要重搬。

ComfyUI 支持外挂模型目录，在**内层**目录建 `extra_model_paths.yaml`：

```yaml
# ComfyUI\ComfyUI\extra_model_paths.yaml
comfyui_h3:
    base_path: D:/MiniMax-H3/models/

    checkpoints: checkpoints          # 完整底模
    diffusion_models: diffusion_models
    text_encoders: text_encoders
    vae: vae
    loras: loras
    upscale_models: upscale_models    # 超分模型
    ipadapter: ipadapter              # IP-Adapter
    clip_vision: clip_vision          # 图像编码器
    ultralytics_bbox: ultralytics/bbox # 人脸检测
```

搬完之后：

```
D:\MiniMax-H3\
├── models\        ← 50GB，备份只需拷这个
├── ComfyUI\       ← 程序，可整体替换
├── easy\          ← 极简网页
├── scripts\       ← 脚本
└── docs\          ← 文档
```

**收益**：升级 ComfyUI = 覆盖程序目录，模型一个字节不动。

### 2.3 下第一批模型

#### 从哪下

| 源 | 适用 | 实测速度 |
|---|---|---|
| ModelScope（魔搭） | 大部分热门底模 | **18 MB/s** |
| hf-mirror.com | 模型更全（含 IP-Adapter 等） | 4–7 MB/s |

#### 下什么（最小可用集）

| 类型 | 推荐 | 大小 | 说明 |
|---|---|---|---|
| 写实底模 | Juggernaut XL v9 / RealVisXL V5 | 6.5 GB | 二选一即可 |
| 动漫底模 | Animagine XL 4.0 / Illustrious XL | 6.5 GB | 要画动漫才需要 |
| 超分模型 | 4x-UltraSharp | 64 MB | 高清修复用，必装 |

先跑通再扩充，别一上来下 50GB。

#### 下载规矩（这条能救你）

**下载中的文件必须带 `.downloading` 后缀，下完再改名。**

```bash
curl -L -o "model.safetensors.downloading" "URL" \
  && mv "model.safetensors.downloading" "model.safetensors"
```

原因见第五章第 2 条——下了一半的文件会被 ComfyUI 当成可用模型，一用就崩。

#### 看进度

写个脚本扫描 `*.downloading`，对照目标大小算百分比和速率：

```
animagine-xl-4.0.safetensors    5.17 / 6.46 GB   80.0%   10.6 MB/s   ~2 min
```

### 2.4 启动与验证

8G 显存建议带低显存参数：

```bash
cd D:\MiniMax-H3\ComfyUI
python_embeded\python.exe -u run_nv.py --lowvram --int8-only \
  --vae-tile 512 --no-preview-highres --serial-queue
```

**验证三件事**：

```bash
# 1. 端口起来了
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8188/     # 期望 200

# 2. 模型被识别了
curl -s http://127.0.0.1:8188/object_info/CheckpointLoaderSimple

# 3. 增强节点都在
curl -s http://127.0.0.1:8188/object_info/FaceDetailer
```

> 💡 新增模型文件**不用重启**，ComfyUI 会按目录 mtime 自动刷新列表。

### 2.5 出第一张图

打开 <http://127.0.0.1:8188>，把官方示例工作流拖进去，改提示词，点 Run。

命令行党也可以直接打 API：

```bash
curl -X POST http://127.0.0.1:8188/prompt \
  -H "Content-Type: application/json" \
  -d '{"prompt": {"1": {"class_type": "CheckpointLoaderSimple",
                        "inputs": {"ckpt_name": "模型名.safetensors"}}, ... }}'
```

### 2.6 装增强件（可选，但强烈建议）

| 增强件 | 作用 | 安装方式 |
|---|---|---|
| **4x-UltraSharp** | 高清修复的超分模型 | 丢进 `models/upscale_models/` |
| **Impact Pack** | FaceDetailer / 检测器节点 | zip 解压到 `custom_nodes/` + 装依赖 |
| **IP-Adapter** | 参考图锁人物/风格 | 插件 + 两个模型 + 一个编码器 |

**装插件依赖前务必先 dry-run**：

```bash
python -m pip install --dry-run <包名>
```

确认输出里**没有** `torch` / `torchvision` / `numpy` 再装——否则可能把你跑通的 CUDA 环境搞崩。装前存一份快照：

```bash
python -m pip freeze > docs/pip-freeze-before.txt
```

---

## 三、用起来：把出图变成日常操作

### 3.1 为什么自己做个网页

ComfyUI 原界面是**节点画布**——灵活，但对"我就想打几个字出张图"的场景太重了：要理解节点连线、模型接线、采样器参数。

所以做了个极简页：**一个输入框 + 一个按钮**。它没有绕开 ComfyUI，只是把工作流固化成了固定模板，通过 API 提交。

**部署技巧**：把 HTML 放进 ComfyUI 的静态目录，用同一个端口访问，省掉跨域配置。

```
ComfyUI\python_embeded\Lib\site-packages\
  comfyui_frontend_package\static\easy.html
→ 访问 http://127.0.0.1:8188/easy.html
```

### 3.2 参数怎么填

**一张表讲完**（这些是实测出来的，不是拍脑袋）：

| 参数 | 写实 | 动漫 | 说明 |
|---|---|---|---|
| 步数 | 30 | 28 | 再高收益很小 |
| CFG | 6 | 7 | 写实超过 8 容易"过饱和塑料感" |
| 采样器 | dpmpp_2m | dpmpp_2m | 通用稳定 |
| 调度器 | karras | karras | 配合上面 |
| 尺寸 | 832×1216 | 832×1216 | SDXL 训练分辨率附近最稳 |
| 高清倍数 | 1.5 | 1.5 | 1248×1824 |
| 高清重绘 | 0.4 | 0.38 | 太低没效果，太高会改内容 |
| 脸部重绘 | 0.32 | 0.25 | 高了皮肤出噪点 |

### 3.3 写实人物：四步调好

**① 换成写实底模**

动漫模型画不出照片，写实模型画不出二次元——这是底模先验，调参数救不回来。

**② 拍近一点**

`close-up portrait` / `head and shoulders` / `medium shot`，别写 `full body`，除非你要的就是全身。
全身构图时脸只有 100 来像素，神仙也画不清。

**③ 写清五官 + 肤色**

```
detailed face, symmetrical features, clear skin texture,
natural makeup, catchlight in eyes,
fair porcelain skin, bright luminous complexion
```

**④ 负面词拦住缺陷**

```
(worst quality, low quality:1.4),
(anime, cartoon, illustration, 3d render, doll:1.3),
ugly, deformed face, asymmetric eyes, crooked teeth,
plastic skin, dark skin, tanned skin, dull skin
```

那个 `(anime, ... :1.3)` 是逼它别往动漫跑的关键。

### 3.4 动漫：模型和质量词要对上

不同动漫底模的**质量标签体系完全不同**，混用会掉效果：

| 模型系 | 质量词 |
|---|---|
| Pony | `score_9, score_8_up, score_7_up` |
| Animagine | `masterpiece, high score, great score, absurdres` |
| Illustrious / NoobAI | `masterpiece, best quality, amazing quality` |

页面里做了自动切换：按 checkpoint 文件名判断体系，你不用记。

风格差异（同提示词同参数）：

| | Animagine XL 4.0 | Illustrious XL v1.0 |
|---|---|---|
| 味道 | 厚涂、质感细腻、成熟向 | 干净色块、通透、现代插画 |
| 耗时 | 127 秒 | 81 秒 |
| 适合 | 国风 / 厚涂半身像 | 日系插画 / 角色立绘 |

### 3.5 参考图：三种模式别混用

| 模式 | 原理 | 参数 | 什么时候用 |
|---|---|---|---|
| **改这张图** | 图生图（整图重绘） | 强度 0.2–0.85 | 在现有图上改细节、换衣服 |
| **锁人物** | IP-Adapter plus-face | 权重 0.6–1.0 | 同一张脸，换场景换姿势 |
| **锁风格** | IP-Adapter 通用版 | 权重 0.5–0.9 | 借画风，构图自由 |

**强度怎么理解**：

- 0.2–0.35：几乎不动原图，只补细节
- 0.50–0.65：保留人物和构图，换细节
- 0.75–0.85：只留大致氛围

### 3.6 显存不够怎么办

三档策略，按需切：

| 状态 | 显存占用 | 网页 | 下次出图 |
|---|---|---|---|
| 正常使用 | 3.5–6 GB | 能开 | 无等待 |
| **卸载模型** | ~0.6 GB | **能开** | 30–60 秒重新加载 |
| 停止服务 | ~0.6 GB | 不能 | 先启动（约 40 秒） |

卸载不用重启服务，一个接口搞定：

```bash
curl -X POST http://127.0.0.1:8188/free \
  -H "Content-Type: application/json" \
  -d '{"unload_models":true,"free_memory":true}'
```

实测 3538 MiB → 856 MiB。

---

## 四、为什么这么设计（原理）

### 4.1 两级采样：高清修复为什么有效

一次出图时，模型在 832×1216 的画布上"均匀用力"，脸部只分到很少的算力。两级采样的思路是：

```
第一遍：832×1216 出草稿（30 步）
   ↓ 4x-UltraSharp 超分 → 缩到 1248×1824
第二遍：以放大图为底再画一遍（18 步，denoise 0.4）
```

第二遍不是"放大"，是**在更大画布上重新生成细节**。denoise 0.4 意味着保留 60% 原结构、重画 40% 细节——所以构图不会变，但五官、发丝、布料纹理都会变清晰。

**为什么用 AI 超分而不是直接拉伸**：lanczos 插值只是复制像素，超分模型会"猜"出合理的高频细节，第二遍采样的起点质量就不同。所以流程是"4x 超分 → 缩回 1.5x"，而不是直接 1.5x 插值。

### 4.2 FaceDetailer 在做什么

```
检测人脸（YOLO）→ 裁出脸部区域 → 放大到 guide_size
  → 单独跑一次采样 → 缩回原尺寸 → 羽化贴回
```

它解决的是"脸在整张图里占比太小"的结构性问题：**脸被单独当成一张图来画**，分辨率自然够了。

参数里有三个关键值：

| 参数 | 作用 | 调错的后果 |
|---|---|---|
| `denoise` | 脸部重画幅度 | 0.5 会出颗粒噪点，0.32 最自然 |
| `guide_size` | 脸部重绘分辨率 | 512 细节不足，768 刚好 |
| `feather` | 边缘羽化 | 太小会有"贴脸"的硬边 |

### 4.3 IP-Adapter 与编码器匹配

IP-Adapter 的结构是：**图像 → CLIP 编码器 → 投影层 → 注入 UNet**。

所以它有两个必须配套的部件：

```
ip-adapter-*-vit-h.safetensors  ← 投影层（输入维度 1280）
        ↕ 必须匹配
ViT-H 图像编码器（输出 1280 维）

ip-adapter_sdxl.safetensors     ← 投影层（输入维度 1664）
        ↕ 必须匹配
ViT-bigG 图像编码器（输出 1664 维）
```

配错就报 `size mismatch for proj_in.weight: [1280,1280] vs [1280,1664]`。

坑在于：h94 官方仓库 `sdxl_models/image_encoder/` 里放的**其实是 bigG**，而 `sdxl_models/` 下的适配器大多叫 `*-vit-h`。名字和内容对不上，我第一次就直接踩了。

### 4.4 质量词为什么必须按模型换

质量词本质是**训练时被打上的高分标签**。Pony 用 Danbooru 的 `score_9` 体系，Animagine 用 `masterpiece/high score`，Illustrious 用 `masterpiece/best quality`——它们在各自训练集里代表"好图"。

换模型不换词，等于对着新模型说它没学过的暗号，效果自然打折。

### 4.5 8G 显存怎么跑 20GB 的模型

ComfyUI 的动态显存加载：模型权重常驻**内存**，推理时按层搬到显存，用完就换出去。

```
内存（32GB）：完整模型 19.5GB 常驻
显存（8GB）：  当前层 + 激活值，峰值约 6.9GB
```

代价是速度（每层都要搬运），收益是**能跑**。这就是为什么 `--lowvram` 不是"降质"，而是"降速保命"。

---

## 五、踩坑实录

按影响从大到小排，每条都给了现象、根因、修法。

### 5.1 `os.execv` 在 Windows 上会丢进程

**现象**：启动器运行 940 毫秒后退出，退出码 0，日志只有启动器自己的几行输出，`main.py` 一个字都没打印。

**根因**：Windows 上 `os.execv` 不是真正的进程替换，而是"新起子进程 + 父进程退出"的模拟。那个子进程在某些环境下会被当孤儿回收，于是"启动即死亡"，还不报错。

**修法**：

```python
import subprocess
sys.exit(subprocess.call(cmd))   # 显式等待，退出码透传
```

**教训**：跨平台代码里，`os.exec*` 系列在 Windows 上都是模拟实现，跑长驻服务时别用。

### 5.2 下了一半的模型也是"合法模型"

**现象**：
```
RuntimeError: shape '[10240, 1280]' is invalid for input of size 6358940
```

**根因**：ComfyUI 只按**后缀**判断模型存在（`.safetensors`），不看完整性。一个下了 40% 的文件照样出现在下拉框里，加载器按头部声明的形状去读，读到文件尾就崩。

**修法**：下载中一律用 `.downloading` 后缀，下完改名。

**教训**：任何"按文件名枚举资源"的系统，都要考虑半成品文件。

### 5.3 写实需求却生成动漫

**根因**：① 用了动漫底模（Pony 是 Danbooru 训练集）；② SDXL 几乎不懂中文，中文提示词等于没写。

**修法**：换写实底模 + 提示词改英文 + 负面词加 `(anime, cartoon, 3d render:1.3)`。

**教训**：先选对模型，再调参数。

### 5.4 人物丑

四个原因，按影响排序：

| 原因 | 修法 |
|---|---|
| 脸在画面里太小 | 用特写/半身构图 |
| 提示词没写五官 | `detailed face, symmetrical features` |
| 负面词没拦缺陷 | `ugly, deformed face, asymmetric eyes` |
| 没有二次重绘 | 开高清修复 + 脸部精修 |

### 5.5 装插件把环境搞崩（未遂）

**风险**：FaceDetailer 依赖 `ultralytics`、`scikit-image`、`opencv` 等一串包，pip 有可能顺手把 `torch` / `numpy` 升级或降级，直接废掉 CUDA 环境。

**修法**：

```bash
python -m pip install --dry-run <包名>     # 先看会装什么
python -m pip freeze > pip-freeze-before.txt  # 存快照
```

看到 `Would install` 里没有 torch/numpy/torchvision 才真装；装完立刻验证：

```
torch 2.13.0+cu130 | cuda available: True
```

**教训**：动别人环境的依赖前，先 dry-run；再留快照能回滚。

### 5.6 模型堆在程序目录里

**修法**：`extra_model_paths.yaml` 外挂（见 2.2）。

**顺带**：ultralytics / matplotlib 默认往 C 盘写缓存，启动器里改掉：

```python
os.environ["YOLO_CONFIG_DIR"] = r"D:\MiniMax-H3\.cache\ultralytics"
os.environ["MPLCONFIGDIR"]    = r"D:\MiniMax-H3\.cache\matplotlib"
```

### 5.7 IP-Adapter 编码器不匹配

见 4.3。**修法**：ViT-H 适配器配 ViT-H 编码器（2.4GB），bigG 单独存放并改名区分。

### 5.8 `curl` 遇到 502 也算成功

**现象**：服务明明挂了，启动脚本判断"已在运行"，于是不启动。

**根因**：`curl` 默认只在**连接失败**时返回非零，HTTP 4xx/5xx 依然返回 0。中间隔了个代理返回 502，就被当成成功。

**修法**：加 `-f`（fail on HTTP error）。

```bash
curl -s -f -o nul --max-time 3 http://127.0.0.1:8188/system_stats
```

### 5.9 PowerShell 写 UTF-8 带 BOM 把脚本搞死

**现象**：进度脚本的速率永远显示 0。

**根因**：`Set-Content -Encoding UTF8` 在 PowerShell 5.1 会写 BOM。读回来第一行的 key 变成 `\ufeff文件名`，和内存里的 key 对不上，永远算不出差值。

**修法**：写用 `-Encoding ASCII`，读时兜底 `-replace '^\ufeff',''`。

### 5.10 快捷方式建不出来

`New-Object -ComObject WScript.Shell` 被安全策略拦截；`pywin32` 没有对应 Python 版本的 wheel。

**修法**：直接把 `.bat` 副本放桌面，文件名带序号。

**顺带一个隐蔽 bug**：bat 里用 `%~dp0`（脚本自身目录）定位，复制到桌面后路径全错。改成写死根目录：

```bat
set "ROOT=D:\MiniMax-H3"
```

### 5.11 东亚脸被画成高颧骨

**根因**：写实底模训练数据以西方人像为主，对 "Asian" 有统计偏差。

**修法**（实测有效）：

```
正向：natural East Asian features, monolid eyes, soft rounded jawline,
      flat nose bridge, warm beige skin
负向：deep-set eyes, exaggerated high cheekbones,
      overly angular jaw, caucasian features
```

**更彻底**：给一张参考图，用 IP-Adapter 锁人物，直接绕过模型先验。

### 5.12 显存被占满

先分清是"服务占的"还是"别的程序占的"：

```bash
nvidia-smi
```

服务占的 → 卸载模型；别的程序占的（游戏、壁纸引擎）→ 别硬杀，先关它。

---

## 六、数据与清单

### 6.1 实测性能（RTX 3060 Ti 8GB）

| 任务 | 输出 | 耗时 |
|---|---|---|
| SDXL 基础出图 | 832×1216 | 33 秒 |
| + 高清修复 | 1248×1824 | 44 秒 |
| + 脸部精修 | 1248×1824 | 110 秒 |
| 参考图锁人物 + 全增强 | 1248×1824 | 71–98 秒 |
| 动漫（Animagine 4.0） | 1248×1824 | 127 秒 |
| 动漫（Illustrious v1.0） | 1248×1824 | 81 秒 |
| 视频（MiniMax H3） | 640×352 / 124 帧 | 210 秒 |
| 显存峰值 | — | 约 6.9 GB |

### 6.2 模型清单

| 类型 | 模型 | 大小 |
|---|---|---|
| 写实底模 | Juggernaut XL v9 | 6.62 GB |
| 写实底模 | RealVisXL V5 (fp16) | 6.46 GB |
| 动漫底模 | Animagine XL 4.0 | 6.46 GB |
| 动漫底模 | Illustrious XL v1.0 | 6.46 GB |
| 动漫底模 | Pony Diffusion V6 XL | 6.46 GB |
| 超分 | 4x-UltraSharp | 64 MB |
| 人脸检测 | face_yolov8m | 50 MB |
| IP-Adapter | plus-face (ViT-H) | 808 MB |
| IP-Adapter | 通用版 (ViT-H) | 666 MB |
| 图像编码器 | ViT-H | 2.4 GB |
| 图像编码器 | ViT-bigG | 3.5 GB |

### 6.3 目录结构

```
D:\MiniMax-H3\
├── models\                 模型库（50GB，独立）
│   ├── checkpoints\        底模
│   ├── upscale_models\     超分
│   ├── ipadapter\          IP-Adapter
│   ├── clip_vision\        图像编码器
│   └── ultralytics\bbox\   人脸检测
├── ComfyUI\                程序本体
│   └── ComfyUI\
│       ├── custom_nodes\   Impact Pack / IPAdapter
│       └── extra_model_paths.yaml
├── easy\                   极简网页（源文件）
├── scripts\                测试/批量脚本
├── docs\                   文档 + 依赖快照
├── Start-EasyImage.bat     启动
├── Unload-Models.bat       卸载模型
├── Stop-EasyImage.bat      停止
└── Show-Status.bat         状态
```

### 6.4 API 速查

| 接口 | 方法 | 用途 |
|---|---|---|
| `/prompt` | POST | 提交工作流 |
| `/history/{id}` | GET | 查任务结果 |
| `/view?filename=` | GET | 取图 |
| `/upload/image` | POST | 上传参考图 |
| `/free` | POST | 卸载模型释放显存 |
| `/object_info/{节点}` | GET | 查节点参数 |
| `/ws?clientId=` | WS | 实时进度 |

---

## 七、复盘：如果重来一次

**会直接照做的**

1. 一开始就配 `extra_model_paths.yaml`——后期迁移 50GB 虽然只是改名，但容易踩"服务在跑时文件被占用"
2. 一开始就立 `.downloading` 规矩
3. 模型优先选**主流 + 有明确文档**的（Animagine、RealVis 都有官方推荐参数），冷门模型省不了时间
4. 装任何插件前 dry-run + 快照

**会少走弯路的**

5. 别用动漫底模试写实——这是最初浪费最多时间的判断
6. 别用中文提示词喂 SDXL——中文理解能力基本为零
7. 人脸问题先解决"构图距离"，再考虑加装增强件

**还会再做的**

8. 网页壳 + API 驱动的架构值得复制到别的模型上（视频、放大、抠图都能套同一个壳）
9. 文档和依赖快照一起做，两个月后回头看能省大量考古时间

---

### 最后

整套东西跑通之后，真正难的不是"AI 怎么用"，而是这些边角：进程模型、文件完整性、编码、架构匹配、代理行为。

模型本身反而最省心——**选对了，它就老老实实出图**。

希望这篇能让你少踩几个我已经踩过的坑。
