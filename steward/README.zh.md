# Steward

[English](README.md) | 中文

机器人 Ego Harness 可选用 `rgbd_video_gen` 和 `NovelAnything` 两个 skill 调用远程新视角生成服务。`scripts/setup-dsh-profiles.sh`（Linux/macOS）和 `scripts/setup-dsh-profiles.ps1`（Windows）会在机器人 profile 中安装两者；Windows 上也可用 `scripts/setup-rgbd-video-gen.ps1`、`scripts/setup-novelanything.ps1` 加到已有 profile。使用方法见 [rgbd_video_gen](dsh/packages/dsh-bundle-rgbd-video-gen/README.zh.md) 和 [NovelAnything](dsh/packages/dsh-bundle-novelanything/README.zh.md)。

模型证据沿用原有 `head_camera`；连续录制按仿真时钟采样并保存机器人里程计和编码器，手爪目标由机器人自身运动学计算。快照工具使用现有头部／广角相机各自的标定，不改变两者的布局、视场或感知分工。此次集成不包含 SLARM 或新增三目相机。

可直接使用的机器人提示词、相机约定和包职责见[感知基模skill说明](docs/novel-view-generation/README.zh.md)。

Steward 是基于 DeepSeek Harness 的具身智能机器人系统。设计见仓库根目录的 [steward.md](../steward.md)，当前实现进度与运行命令见其中的「13. Implementation Progress」；Phase 1 感知如何从 RGB-D 得到几何与类别、以及泛化路线见 [13.7](../steward.md#137-phase-1-感知方法与泛化)。

## 快速开始

```sh
uv sync
.venv/bin/steward demo          # 构建厨房并运行「把杯子放到桌上」，按 World State 判定成败
.venv/bin/steward up --scenario phase0-shuttle --no-dispatch-task   # 常驻运行世界与机器人，搬运交给下面的 shuttle
open http://127.0.0.1:7400/console   # 浏览器控制台：斜视主视图、相机、地图、指标、控制
.venv/bin/steward shuttle --api http://127.0.0.1:7420 --steps 0   # 在餐桌、柜子、餐边柜之间循环搬运
./scripts/check.sh              # Python 测试、DSH 插件编译与测试、组合配置快照比对
.venv/bin/steward build-kitchen # 机器人模型改动后（如头部加装的广角相机）重建默认场景
```

机器人头部有两台 RGB-D 相机：`head_camera` 看前方，`wide_camera` 是短焦广角相机，看身前台面与夹爪；两台都进入感知，见 [steward.md 13.8](../steward.md#138-机器人本体的双相机)。

`cobot-magic` 左右臂 joint1～joint6 初始角度均为 0 弧度，夹爪初始张开。`cobot-magic` 夹爪使用简化的掌体、导轨、滑块与手指几何。手指沿工具轴延伸，对称张开时内侧开口最大为 100 mm；抓取点位于两侧指垫之间、指尖后方 5 mm。这些显示几何不代表厂家 CAD，也不提供接触式抓取。

## 导入 USD 机器人

可选的 `usd` extra 使用 MuJoCo 4.0 的原生 OpenUSD 解码器。在独立的 Python 3.12 环境中安装它，以保留常规仿真环境。从仓库根目录运行：

```sh
uv venv --python 3.12 steward/var/usd-env
uv pip install --python steward/var/usd-env/bin/python -e './steward[usd]'
steward/var/usd-env/bin/steward import-usd "$HOME/Downloads/robot.usd" --name agilex-usd \
  --meters-per-unit 1 --orient-joints --fixed-base \
  --bound-mass 0.001 --bound-inertia 0.000001 --inertia-from-bounds --drive-frequency 5
steward/.venv/bin/steward world --scenario agilex-usd --phase phase0 \
  --wca-port 7450 --robot-port 7460 --ego-api '' --world-harness '' --robot-harness ''
```

打开 `http://127.0.0.1:7450/console`。导入会创建带地面的检查场景，并将 `template.json`、二进制网格和 `report.json` 写入 `var/steward/imports/<name>/`。已有名称会被拒绝。网格引用使用绝对路径，因此应保留该目录的位置。查看结果也支持 MuJoCo 3.17，不再读取源 USD。源文件始终不会被修改。

示例选项用于提供的 AgileX 资产：文件将米制几何数值标记为厘米，一个底盘关节方向相反，且缺少惯量。质量和惯量下限、均匀盒体惯量估算以及 5 Hz 临界阻尼驱动均为显式仿真近似，会记入报告。正确编写的资产可省略这些选项；依赖动力学结果前应检查报告。驱动在启动时保持导入的静止姿态，不执行 USD 中的驱动目标。

导入器支持一棵刚体机器人树，包括固定、旋转和平移关节、网格与基本几何体、力式位置驱动以及标量 PhysX mimic 联动约束。转换保留关节限位、枢轴和显式惯量。网格碰撞使用 MuJoCo 凸包。着色网络、贴图和相机不会转移；缺失的 MDL 着色器会记入报告，缺失的几何依赖则会中止导入。闭环和不支持的动力学会被拒绝。通用导入不会自动分配任务控制器；可通过机器人端口的 `/command` 接口按名称驱动关节。提供的双 Piper 模型另有已注册的适配器 `agilex-usd`，使用方式如下。


将提供的 `robot.usd` 以 `agilex-usd` 名称导入后，可在任务场景中启动：

```sh
steward/.venv/bin/steward up --scenario cobot-table --body agilex-usd --phase phase0 \
  --speed 1 --no-dispatch-task
```

打开 `http://127.0.0.1:7400/console`，在本体选择器中选择 **AgileX USD 双 Piper**。也可用它替换厨房场景中的机器人。`up` 会启动其 Ego Runtime；导航、抓取和放置使用与其他本体相同的预测回执和技能 API。原始导入检查场景仍独立保留，不注册为可执行任务的本体。

[仿真标定配置](config/agilex-usd.toml) 指定模板位置、伺服增益、升降初始位置及仿真相机俯角。路径相对此文件解析；导入到其他数据根目录时，可用 `STEWARD_AGILEX_USD_CONFIG` 指定另一份配置。模板缺失或不兼容会明确报错。适配器保留 USD 网格、手臂变换、关节限位、惯量和手指联动，添加抓取点，并将移动底盘映射为平面位置控制。轮子关节固定。两种双 Piper 本体（`cobot-magic` 和 `agilex-usd`）的左右臂 joint1～joint6 初始角度均为 0 弧度。两个仿真 RGB-D 视角安装在 `top_camera_link` 上，随升降运动；其标定不来自 USD。Phase 0 使用理想附着，不依赖夹爪接触抓取。此功能支持仿真任务，不提供真机控制或经过验证的轮式动力学。

本地资产集成测试命令为 `STEWARD_TEST_AGILEX_USD=1 steward/.venv/bin/python -m pytest -q steward/tests/test_agilex.py`。它通过 Ego API 导航、抓起碗并放到桌面另一处，最后核对世界坐标。运动学测试夹具包含源关节数据，以小盒体替代网格；集成测试使用本地真实网格，因为仓库不分发这些文件，所以需显式启用。

## 与 DeepSeek Harness 一起运行

```sh
echo 'DEEPSEEK_API_KEY=sk-...' > .env     # 已被 Git 忽略，运行脚本自动加载
.venv/bin/steward up                      # WCA :7400，机器人端口 :7410，Ego API :7420；场景任务自动交给 Ego Harness
.venv/bin/steward up --phase phase1-sam   # 机器人用 SAM 3.1 自行分割 RGB-D（设置在 config/segmentation.toml）
./scripts/setup-dsh-profiles.sh           # 构建插件，创建 ego、world、world-task profile
./scripts/run-ego.sh "把杯子放到桌上"       # 机器人智能体
./scripts/run-world.sh "按描述构建一间厨房……"  # World Harness 一次性任务
./scripts/run-world.sh --port 3091        # World Harness 浏览器界面
./scripts/install-desktop-app.sh          # 在桌面安装 Steward.app：选场景后启动世界、Ego Harness 并打开控制台
.venv/bin/steward services                # 检查远程感知服务（SAM 3.1、NovelAnything、NVS、SLARM）是否可达
```

## Windows

Windows 上使用同样的命令：`uv sync` 把 `steward` 可执行文件装在 `.venv\Scripts` 下，每个 `.sh` 脚本都有同名、同参数的 PowerShell 版本。如果 PowerShell 拒绝运行未签名脚本，先执行一次 `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`。

```powershell
uv sync
.venv\Scripts\steward up --phase phase1-sam
.\scripts\setup-dsh-profiles.ps1          # 构建插件，创建 ego、world、world-task profile
.\scripts\run-ego.ps1 "把杯子放到桌上"      # 机器人智能体
.\scripts\run-world.ps1 --port 3091       # World Harness 浏览器界面
.\scripts\check.ps1                       # Python 测试、DSH 插件编译与测试、组合配置快照比对
.\scripts\install-desktop-app.ps1         # 在桌面创建 Steward 快捷方式：选场景后启动世界、Ego Harness 并打开控制台
```

桌面启动器的日志写在 `%LOCALAPPDATA%\Steward\Logs`；macOS 上写在 `~/Library/Logs/Steward`。

`ego` profile 是机器人的 Ego Harness，只能经 Ego API 感知与行动，shell、文件、网页等通用工具已禁用；`world` 与 `world-task` profile 是 World Harness，经世界控制接口构建资产、组合场景、控制 Episode。机器人与世界使用不同的 `DSH_HOME`。

## 目录

| 路径 | 内容 |
|---|---|
| `src/steward/scene` | WS 与 ES 共用的 SceneState、校验器、MJCF 转换 |
| `src/steward/assets`、`authoring` | 资产生成、校验、预览、资产库；场景与 Scenario |
| `src/steward/world` | WSim 与 Steward 主机（WCA、机器人端口） |
| `src/steward/ego` | Ego Runtime：估计器、ESim、技能、Ego API |
| `src/steward/observer` | ES 与 World State 的对比评估 |
| `dsh/` | DeepSeek Harness 插件与 bundle（锁定 `@deepseek-ai/dsh@0.2.1-rc.3`） |
| `scripts/` | profile 创建、检查、多种子回归 |
| `config/perception-services.toml` | 远程感知服务清单（地址、健康检查、接口），见 steward.md 13.7.6 |
| `config/segmentation.toml` | `phase1-sam` 的机器人侧分割设置：词表、阈值、裁剪与语义融合，见 steward.md 13.7.7 |
| `compat/snapshots` | 组合配置快照 |

运行数据写入 `var/`，不纳入版本管理。
