# Remotion 合成专家（remotion）

> 本文档是 `/gsd-bianxiezhuanjia remotion` 子命令的详细执行流程。由主 SKILL.md 在前置门禁通过后加载。
>
> 协议字段（供编排器识别）：
> - `expert-id`: remotion
> - `converges`: Remotion 代码结构混乱 / 可维护性差 / 字幕与文案不同步 / 时长校准缺失
> - `input-contract.required`: state/node-2.copywrite.json + state/node-3.tts.json + state/node-4.digital-human.json（汇聚节点）
> - `output-contract.state`: state/node-5.remotion.json
> - `handoff`: node-6.publish（via state/node-5.remotion.json）
> - `quality-gate`: true

---

## 收敛什么（为什么需要专家而不是直接让大模型写 Remotion）

直接让大模型写 Remotion 合成代码，会发散出四类问题——本专家逐项收敛：

| 发散问题 | 不收敛的后果 | 本专家怎么收敛 |
|---------|------------|--------------|
| 代码结构混乱 | 一个文件堆 500 行，改一处崩一片 | 强制组件分层（见下文目录结构） |
| 硬编码 | 字幕文字、时长、素材路径写死在 JSX | 全部走 props，配置外置 |
| 字幕与文案不同步 | 文案改一个字，要手动改代码 | segments → props 自动映射，文案即数据 |
| 时长校准缺失 | 视频时长和音频对不上，末尾黑屏/截断 | durationInFrames 严格 = TTS 时长 |

---

## 前置检查（汇聚节点门禁）

激活本专家前，编排器必须确认三个输入齐备：

```
state/node-2.copywrite.json    ✓ completed  (字幕源)
state/node-3.tts.json          ✓ completed  (音频 + 时长)
state/node-4.digital-human.json ✓ completed  (数字人视频)
```

缺任一 → 拒绝激活，提示缺哪个上游。

---

## 代码结构规范（BLOCK 级，必须遵守）

### 目录结构

产出的 Remotion 项目必须按以下结构组织：

```
artifacts/remotion/
├── package.json
├── remotion.config.ts
├── src/
│   ├── Root.tsx              # Composition 注册入口（唯一注册点）
│   ├── compositions/
│   │   └── VoiceoverComposition.tsx  # 主合成组件（编排各子组件，不含样式逻辑）
│   ├── components/
│   │   ├── Avatar.tsx        # 数字人图层（只管位置/尺寸，不管内容）
│   │   ├── Caption.tsx       # 字幕组件（props 驱动，按 segments 渲染）
│   │   ├── Bgm.tsx           # 背景音乐（可选）
│   │   └── Transition.tsx    # 转场（可选）
│   ├── props/
│   │   ├── types.ts          # props 类型定义（单一真相源）
│   │   └── default-props.ts  # 默认 props（开发用）
│   └── utils/
│       └── segments-to-captions.ts  # segments → 字幕时间轴的转换
└── public/
    ├── audio.mp3             # TTS 音频（从 node-3 复制）
    ├── avatar.mp4            # 数字人视频（从 node-4 复制）
    └── bgm.mp3               # 背景音乐（可选）
```

### 分层红线

| 层 | 允许做 | 禁止做 |
|----|--------|--------|
| Root.tsx | 注册 Composition、声明 durationInFrames | 放业务逻辑、写样式 |
| Composition | 编排子组件顺序、传递 props | 写样式细节、硬编码内容 |
| components/* | 单一职责的视觉组件，props 驱动 | 直接读 state 文件、互相 import |
| props/types.ts | 定义所有 props 类型 | 写逻辑 |
| utils/* | 纯函数转换 | 有副作用 |

**最常见的违规**：在 Composition 里硬编码字幕文字、在组件里直接读文件。前者让文案改动需改代码，后者让组件不可复用。

---

## 关键实现约束

### 1. durationInFrames 严格对齐 TTS 时长

```tsx
// Root.tsx
import { Composition } from 'remotion';
import { VoiceoverComposition } from './compositions/VoiceoverComposition';
import { defaultProps } from './props/default-props';

const FPS = 30;
const audioDurationSec = defaultProps.audioDurationSec;  // 来自 node-3 的实际时长

export const RemotionRoot = () => {
  return (
    <Composition
      id="Voiceover"
      component={VoiceoverComposition}
      durationInFrames={Math.ceil(audioDurationSec * FPS)}  // 严格对齐，不手填
      fps={FPS}
      width={1080}
      height={1920}   // 9:16 竖屏（douyin）
      defaultProps={defaultProps}
    />
  );
};
```

### 2. segments → 字幕：数据驱动，非硬编码

字幕文字**绝不**写在 JSX 里，全部来自 props.segments：

```tsx
// ❌ 硬编码（改文案要改代码）
<Caption text="时间管理的三个误区" startFrame={0} />

// ✅ 数据驱动（文案改了，props 更新即可）
{segments.map(seg => (
  <Caption
    key={seg.id}
    text={seg.text}
    startFrame={segToFrame(seg.startSec)}
    durationFrames={segToFrame(seg.durationEst)}
  />
))}
```

`utils/segments-to-captions.ts` 负责 node-2 的 segments → 字幕时间轴的转换。文案改了，只需重新生成 props，代码不动。

### 3. 素材路径走 props，不写死

```tsx
// ❌ 写死
<Avatar src="/static/avatar.mp4" />

// ✅ props 驱动
<Avatar src={props.avatarSrc} />
```

### 4. props 类型集中定义

`props/types.ts` 是 props 的单一真相源，Composition 和所有组件从这里 import 类型：

```ts
// props/types.ts
export interface VoiceoverProps {
  segments: Segment[];
  audioSrc: string;
  audioDurationSec: number;
  avatarSrc: string;
  captionStyle: CaptionStyle;
}

export interface Segment {
  id: number;
  text: string;
  emotion: 'calm' | 'serious' | 'warm' | 'energetic';
  startSec: number;
  durationEst: number;
}
```

---

## 质量门禁（BLOCK 级，全部通过才允许产出 completed）

跑以下检查清单，任一未过则 `status: partial`：

### 代码结构
- [ ] **组件分层**：Root / Composition / components / props / utils 各司其职，无越界
- [ ] **单一注册点**：所有 Composition 只在 Root.tsx 注册
- [ ] **无面条代码**：单文件 ≤ 150 行（utils 纯函数除外）

### 数据驱动
- [ ] **字幕走 props**：Caption 组件零硬编码文字，全来自 segments
- [ ] **素材走 props**：音频/视频路径走 props，不写死
- [ ] **segments 转换有专门 utils**：不在组件里做转换

### 时长校准
- [ ] **durationInFrames = TTS 时长**：用 `Math.ceil(audioDurationSec * FPS)`，不手填
- [ ] **末尾无黑屏/截断**：视频时长 = 音频时长

### 可构建
- [ ] **npm install 成功**：依赖可安装
- [ ] **npm run build 成功**：能产出可播放视频
- [ ] **类型检查通过**：tsc / 类型无报错

---

## 产出

### 状态文件 `state/node-5.remotion.json`

```json
{
  "$schema": "expert-state/v1",
  "expert-id": "remotion",
  "expert-version": "1.0",
  "node-id": "node-5",
  "status": "completed",
  "quality-gate": {
    "passed": true,
    "checks": [
      { "name": "组件分层", "passed": true },
      { "name": "字幕走props", "passed": true },
      { "name": "时长校准", "passed": true, "note": "58s = TTS时长" },
      { "name": "可构建", "passed": true, "note": "npm run build 通过" }
    ]
  },
  "artifacts": ["artifacts/remotion/src/", "artifacts/remotion/package.json"],
  "payload": {
    "component-path": "artifacts/remotion/src/Root.tsx",
    "composition-id": "Voiceover",
    "props": {
      "segments": [...],
      "audioSrc": "audio.mp3",
      "audioDurationSec": 58,
      "avatarSrc": "avatar.mp4"
    },
    "fps": 30,
    "width": 1080,
    "height": 1920,
    "duration-sec": 58,
    "build-command": "cd artifacts/remotion && npm install && npm run build"
  },
  "handoff-ready": [
    { "target": "node-6.publish", "via": "state/node-5.remotion.json" }
  ]
}
```

### 产物

- `artifacts/remotion/src/`：完整可构建的 Remotion 项目
- `artifacts/remotion/package.json`：依赖声明（lockfile 一并产出）
- `public/audio.mp3`、`public/avatar.mp4`：从 node-3/4 复制的素材

---

## 质量门禁未过时怎么办

| 未过项 | 处理建议 |
|--------|---------|
| 组件分层违规 | 按目录结构规范重构，把样式逻辑下沉到 components |
| 字幕硬编码 | 把硬编码文字提取到 props.segments，走数据驱动 |
| 时长不匹配 | 检查 node-3 的 audioDurationSec 是否准确，重算 durationInFrames |
| 构建失败 | 看报错，通常是依赖版本或类型问题，修 package.json |

人也可直接编辑 `artifacts/remotion/src/`，改完手动跑 `npm run build` 验证，再把 status 改 `completed`。

---

## 与其他节点的衔接

| 方向 | 衔接 |
|------|------|
| 上游 | node-2.copywrite（字幕源 segments） |
| 上游 | node-3.tts（音频 + 时长） |
| 上游 | node-4.digital-human（数字人视频） |
| 下游 | node-6.publish（消费 build-command 构建并发布） |
