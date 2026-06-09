<template>
  <section class="solar-motion-page" :class="`phase-${dayPhase}`" :style="pageSkyStyle">
    <div class="bg-grid"></div>
    <div class="scan-line"></div>

    <header class="topbar">
      <div class="brand">
        <div class="sun-logo"></div>
        <div>
          <div class="eyebrow">SOLAR APPARENT MOTION LAB</div>
          <h1>太阳视运动 3D 互动课件</h1>
        </div>
      </div>

      <div class="top-actions">
        <button :class="{ active: controlMode === 'god' }" @click="setControlMode('god')">上帝模式</button>
        <button :class="{ active: controlMode === 'player' }" @click="setControlMode('player')">玩家模式</button>
        <button :class="{ active: isPlaying }" @click="togglePlay">
          {{ isPlaying ? '暂停演示' : '开始演示' }}
        </button>
        <button class="ghost" @click="setStandardScene">标准场景</button>
        <button class="ghost" @click="resetCamera">重置视角</button>
      </div>
    </header>

    <main class="layout">
      <aside class="panel left-panel">
        <div class="panel-title"><span></span>控制面板</div>

        <div class="block">
          <h3>① 观测地点</h3>
          <RangeRow label="纬度" :value="state.latitude" suffix="°" :min="-90" :max="90" :step="0.1" @update:value="state.latitude = $event" />
          <div class="grid-2">
            <button :class="{ active: isApprox(state.latitude, 0) }" @click="state.latitude = 0">赤道</button>
            <button :class="{ active: isApprox(state.latitude, 23.44) }" @click="state.latitude = 23.44">北回归线</button>
            <button :class="{ active: isApprox(state.latitude, 31.23) }" @click="state.latitude = 31.23">上海</button>
            <button :class="{ active: isApprox(state.latitude, 40) }" @click="state.latitude = 40">北纬40°</button>
          </div>
          <p class="tip">纬度决定太阳周日视运动路径的倾斜程度，也决定正午太阳高度。</p>
        </div>

        <div class="block">
          <h3>② 季节日期</h3>
          <div class="grid-4">
            <button :class="{ active: state.dayOfYear === 80 }" @click="setDay(80)">春分</button>
            <button :class="{ active: state.dayOfYear === 172 }" @click="setDay(172)">夏至</button>
            <button :class="{ active: state.dayOfYear === 266 }" @click="setDay(266)">秋分</button>
            <button :class="{ active: state.dayOfYear === 355 }" @click="setDay(355)">冬至</button>
          </div>
          <RangeRow label="日期" :value="state.dayOfYear" :suffix="` / ${dateLabel}`" :min="1" :max="365" :step="1" @update:value="setDay($event)" />
          <p class="tip">日期决定太阳赤纬 δ。6月太阳直射北回归线，北半球路径高、昼长较长；12月太阳直射南回归线，南半球路径高、昼长较长。</p>
        </div>

        <div class="block">
          <h3>③ 时间推进</h3>
          <RangeRow
            label="地方太阳时"
            :value="state.solarTime"
            suffix="时"
            :min="0"
            :max="24"
            :step="0.05"
            @update:value="state.solarTime = $event"
          />
          <RangeRow label="动画速度" :value="state.playSpeed" suffix="x" :min="0.2" :max="8" :step="0.2" @update:value="state.playSpeed = $event" />
          <div class="time-buttons">
            <button
              :class="{ active: !solarMetrics.polarType && isApprox(state.solarTime, solarMetrics.sunrise, 0.06) }"
              :disabled="!!solarMetrics.polarType"
              @click="setSafeSolarTime(solarMetrics.sunrise)"
            >
              日出
            </button>
            <button :class="{ active: isApprox(state.solarTime, 12, 0.06) }" @click="state.solarTime = 12">正午</button>
            <button
              :class="{ active: !solarMetrics.polarType && isApprox(state.solarTime, solarMetrics.sunset, 0.06) }"
              :disabled="!!solarMetrics.polarType"
              @click="setSafeSolarTime(solarMetrics.sunset)"
            >
              日落
            </button>
          </div>
          <p class="tip">
            本课件用“地方太阳时”演示规律：12:00 表示太阳位于当地子午线附近。
            <template v-if="solarMetrics.polarType">当前为{{ solarMetrics.polarType }}，没有普通意义上的日出 / 日落。</template>
          </p>
        </div>

        <div class="block">
          <h3>④ 观察模式</h3>

          <template v-if="controlMode === 'god'">
            <div class="view-grid">
              <button :class="{ active: currentView === 'standard' }" @click="setCameraView('standard')">标准视角</button>
              <button :class="{ active: currentView === 'path' }" @click="setCameraView('path')">路径视角</button>
              <button :class="{ active: currentView === 'top' }" @click="setCameraView('top')">俯视城市</button>
              <button :class="{ active: currentView === 'shadow' }" @click="setCameraView('shadow')">影子视角</button>
              <button :class="{ active: currentView === 'south' }" @click="setCameraView('south')">南北判读</button>
            </div>
            <p class="tip">上帝模式适合老师总览讲解：路径高低、方位变化、影子方向和城市街区中的太阳位置。</p>
          </template>

          <template v-else>
            <div class="view-grid">
              <button :class="{ active: playerViewMode === 'third' }" @click="setPlayerView('third')">第三人称</button>
              <button :class="{ active: playerViewMode === 'first' }" @click="setPlayerView('first')">第一人称</button>
            </div>

            <div class="player-model-select">
              <div class="model-select-title">玩家模型</div>
              <div class="model-buttons">
                <button
                  v-for="model in playerModelButtons"
                  :key="model.key"
                  :class="{ active: selectedPlayerModel === model.key }"
                  @click="setPlayerModel(model.key)"
                >
                  {{ model.label }}
                </button>
              </div>
            </div>

            <p class="tip">
              玩家模式支持 WASD 移动、Shift 奔跑、Space 跳跃、F 飞行、V
              切换视角；第三人称按住鼠标左键拖动画面旋转视角。人物不能走出或飞出圆形平面边界。
            </p>
          </template>
        </div>

        <div class="block">
          <h3>⑤ 显示图层</h3>
          <label class="check-row"><input v-model="layers.dome" type="checkbox" /> 天球半球与高度角网格</label>
          <label class="check-row"><input v-model="layers.paths" type="checkbox" /> 当前日期 + 春分 / 夏至 / 冬至太阳路径</label>
          <label class="check-row"><input v-model="layers.shadow" type="checkbox" /> 城市建筑与树木原生阴影</label>
          <label class="check-row"><input v-model="layers.rays" type="checkbox" /> 太阳光线</label>
          <label class="check-row"><input v-model="layers.altitudeGauge" type="checkbox" /> 太阳高度角雷达扫描</label>
          <label class="check-row"><input v-model="layers.formula" type="checkbox" /> 右侧公式与参数代入</label>
          <label class="check-row"><input v-model="layers.cityTime" type="checkbox" /> 城市时间元素 / 夜间灯光</label>
        </div>
      </aside>

      <section class="stage-card">
        <div ref="canvasWrapRef" class="canvas-wrap"></div>

        <div class="scene-title">
          <b>{{ sceneTitle }}</b>
          <span>{{ sceneSubtitle }}</span>
        </div>

        <!--         <div class="legend-panel">
          <div class="legend-title">图例</div>
          <div><i class="dot yellow"></i> 当前太阳</div>
          <div><i class="dot current"></i> 当前日期路径</div>
          <div><i class="dot blue"></i> 夏至路径</div>
          <div><i class="dot white"></i> 春秋分路径</div>
          <div><i class="dot cyan"></i> 冬至路径</div>
          <div><i class="dot shadow"></i> 太阳定向光原生阴影</div>
          <div><i class="dot current"></i> 太阳高度角扫描</div>
          <div><i class="dot playground"></i> 城市群与道路</div>
        </div> -->
      </section>

      <aside class="panel right-panel">
        <div class="panel-title"><span></span>知识速览</div>

        <div class="data-card">
          <div class="card-head">实时数据</div>
          <div class="big-row">
            <span>太阳高度角 h</span><b>{{ formatDeg(solarMetrics.altitude) }}</b>
          </div>
          <div class="big-row">
            <span>太阳方位角 A</span><b>{{ formatDeg(solarMetrics.azimuth) }}</b>
          </div>
          <div class="big-row">
            <span>地方太阳时</span><b>{{ formatClock(state.solarTime) }}</b>
          </div>
          <div class="small-grid">
            <div>
              <span>日期</span><b>{{ dateLabel }}</b>
            </div>
            <div>
              <span>纬度 φ</span><b>{{ formatDeg(state.latitude) }}</b>
            </div>
            <div>
              <span>赤纬 δ</span><b>{{ formatDeg(solarMetrics.declination) }}</b>
            </div>
            <div>
              <span>时角 H</span><b>{{ formatDeg(solarMetrics.hourAngle) }}</b>
            </div>
            <div>
              <span>日出</span><b>{{ solarMetrics.polarType ? '--' : formatClock(solarMetrics.sunrise) }}</b>
            </div>
            <div>
              <span>日落</span><b>{{ solarMetrics.polarType ? '--' : formatClock(solarMetrics.sunset) }}</b>
            </div>
            <div>
              <span>昼长</span><b>{{ formatDayLength }}</b>
            </div>
            <div>
              <span>正午高度</span><b>{{ formatDeg(solarMetrics.noonAltitude) }}</b>
            </div>
          </div>
        </div>

        <div class="knowledge-card">
          <div class="card-head">城市时间观察</div>
          <ul>
            <li><b>白天</b>：太阳高度角较大，建筑影子短，路灯熄灭，城市整体更明亮。</li>
            <li><b>清晨 / 傍晚</b>：太阳高度角较小，影子拉长，路灯逐渐亮起，适合观察“低太阳高度角”。</li>
            <li><b>夜晚</b>：太阳在地平线下，建筑影子消失，路灯、窗户和城市时钟成为判断时间变化的线索。</li>
          </ul>
        </div>

        <div v-if="layers.formula" class="knowledge-card formula-card">
          <h3>核心公式 · 当前参数代入</h3>
          <div class="formula-item">
            <b>太阳赤纬</b>
            <code>δ ≈ 23.44° × sin[360° × (284 + n) / 365]</code>
            <span>n={{ state.dayOfYear }}，δ={{ formatDeg(solarMetrics.declination) }}</span>
          </div>
          <div class="formula-item">
            <b>时角</b>
            <code>H = 15° × (地方太阳时 - 12)</code>
            <span>太阳时={{ formatClock(state.solarTime) }}，H={{ formatDeg(solarMetrics.hourAngle) }}</span>
          </div>
          <div class="formula-item">
            <b>太阳高度</b>
            <code>sin h = sinφ·sinδ + cosφ·cosδ·cosH</code>
            <span>φ={{ formatDeg(state.latitude) }}，h={{ formatDeg(solarMetrics.altitude) }}</span>
          </div>
          <div class="formula-item">
            <b>正午太阳高度</b>
            <code>hₙ = 90° - |φ - δ|</code>
            <span>hₙ={{ formatDeg(solarMetrics.noonAltitude) }}</span>
          </div>
          <div class="formula-item">
            <b>昼长</b>
            <code>T = 2 × arccos(-tanφ·tanδ) / 15</code>
            <span>{{ solarMetrics.polarType ? solarMetrics.polarType : `T=${formatHour(solarMetrics.dayLength)}` }}</span>
          </div>
        </div>

        <div class="knowledge-card">
          <h3>学生理解路径</h3>
          <ol>
            <li><b>先看路径高低：</b>夏至路径高，冬至路径低，春秋分居中。</li>
            <li><b>再看日出日落：</b>所在半球夏季昼长、冬季昼短；春秋分昼夜接近等长。</li>
            <li><b>最后看影子：</b>太阳越高，影子越短；太阳在东，影子指向西。</li>
          </ol>
        </div>

        <div class="knowledge-card student-task-card">
          <h3>学生观察任务</h3>
          <ol>
            <li><b>站在城市街道观测点：</b>先判断自己面向哪里，再找太阳在东、南、西、北哪一侧。</li>
            <li><b>拖动地方太阳时：</b>观察同一天内太阳从低到高再变低，影子从长到短再变长。</li>
            <li><b>切换夏至和冬至：</b>比较太阳路径高低，解释为什么同一地点夏季正午影子更短。</li>
            <li><b>进入玩家第一人称：</b>用鼠标抬头看太阳路径，或按 F 飞行到高处俯瞰道路、树木和建筑物影子方向。</li>
          </ol>
        </div>

        <div class="knowledge-card student-view-card">
          <h3>从学生视角怎么理解</h3>
          <ul>
            <li><b>我看到的太阳：</b>太阳高度角 h 不是地图上的角，而是我站在地面看到的“太阳离地平线有多高”。</li>
            <li><b>我脚下的影子：</b>影子方向永远背向太阳；太阳越高，影子越短。</li>
            <li><b>我一天的观察：</b>早晨太阳低、影子长；正午太阳高、影子短；傍晚太阳低、影子又变长。</li>
            <li><b>我一年的观察：</b>当地夏季太阳路径高、白昼长；当地冬季太阳路径低、白昼短；春秋分居中。</li>
          </ul>
        </div>

        <div class="knowledge-card shadow-explain-card">
          <h3>影子变化怎么讲</h3>
          <ul>
            <li><b>方向规律：</b>影子永远指向太阳的反方向；太阳在东，影子向西；太阳在西，影子向东。</li>
            <li><b>长短规律：</b>太阳高度角越大，影子越短；清晨、傍晚高度角小，影子最长；正午通常最短。</li>
            <li>
              <b>公式理解：</b>影长 L = 物高 H ÷ tan(h)。当前太阳高度下，1 米杆影长约为 <b>{{ oneMeterShadowText }}</b
              >。
            </li>
            <li><b>3D观察：</b>场景里的黄色量角器显示太阳高度角 h。h 变大时，太阳更接近天顶，影子明显缩短；h 变小时，太阳贴近地平线，影子被拉长。</li>
            <li><b>课堂追问：</b>为什么同一栋建筑或同一根路灯杆，早晨和傍晚影子长，中午影子短？让学生同时看太阳高度角和影长变化。</li>
            <li><b>课堂观察：</b>拖动“地方太阳时”，观察城市建筑、路灯和树木的影子怎样从长变短再变长。</li>
          </ul>
        </div>

        <div class="knowledge-card">
          <h3>易错点提醒</h3>
          <ul>
            <li>太阳视运动是“站在地面看太阳”的表观运动，不是太阳真的绕地球转。</li>
            <li>北半球中纬度，正午太阳通常在南方天空；南半球中纬度则通常在北方天空。</li>
            <li>影子方向永远与太阳方位相反；影长与太阳高度角成反比。</li>
            <li>本模型默认使用地方太阳时，重点演示太阳高度、方位和影子规律。</li>
          </ul>
        </div>

        <div class="summary-card">
          <h3>季节对比</h3>
          <table>
            <thead>
              <tr>
                <th>日期</th>
                <th>赤纬</th>
                <th>路径</th>
                <th>昼长</th>
              </tr>
            </thead>
            <tbody>
              <tr>
                <td>夏至</td>
                <td>+23.44°</td>
                <td>最高</td>
                <td>北半球最长</td>
              </tr>
              <tr>
                <td>春秋分</td>
                <td>约0°</td>
                <td>居中</td>
                <td>昼夜接近等长</td>
              </tr>
              <tr>
                <td>冬至</td>
                <td>-23.44°</td>
                <td>最低</td>
                <td>北半球最短</td>
              </tr>
            </tbody>
          </table>
        </div>
      </aside>
    </main>
  </section>
</template>

<script setup lang="ts">
import { computed, defineComponent, h, nextTick, onBeforeUnmount, onMounted, reactive, ref, watch } from 'vue'
import * as THREE from 'three'
import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js'
import { playerController } from 'three-player-controller'
import { ElSlider } from 'element-plus'
import tommyModelUrl from '@/assets/tommy.glb?url'
import person1ModelUrl from '@/assets/ant.glb?url'

const SKY_RADIUS = 7.6
const GROUND_RADIUS = 7.4

type LayerState = {
  dome: boolean
  paths: boolean
  shadow: boolean
  rays: boolean
  altitudeGauge: boolean
  formula: boolean
  cityTime: boolean
}

type SolarMetrics = {
  declination: number
  hourAngle: number
  altitude: number
  azimuth: number
  noonAltitude: number
  dayLength: number
  sunrise: number
  sunset: number
  polarType: '' | '极昼' | '极夜'
  east: number
  north: number
  up: number
  /** 地方太阳时，给城市太阳时钟牌使用 */
  solarTime: number
}

type ViewPreset = 'standard' | 'path' | 'top' | 'shadow' | 'south'
type ControlMode = 'god' | 'player'
type PlayerViewMode = 'third' | 'first'
type PlayerModelKey = 'tommy' | 'ant'

type ShadowCaster = {
  x: number
  z: number
  height: number
  width: number
  depth: number
  mesh: THREE.Mesh
  target?: THREE.Object3D
}

const BILLBOARD_BACK_CONFIG = {
  eyebrow: '敲代码做 HTML 互动课件',
  title: '码上教育你',
  subtitle: '小红书同名账号',
  footer: '太阳视运动 · 城市观察实验室',
}

const PLAYER_MODEL_CONFIG = {
  // three-player-controller 示例模型通常是小写动画名：idle / walk / run / jump。
  // 如果你的模型动画名不同，就改这里。
  // 如果人物还是偏大，就继续调小 scale，例如 0.0008 / 0.0006。
  // 如果人物太小，就调大，例如 0.0015 / 0.002。
  scale: 0.0012,
  idleAnim: 'idle',
  walkAnim: 'walk',
  runAnim: 'run',
  jumpAnim: 'jump',

  // 如果 person.glb 有飞行动作，就用这里；没有的话可以改回 idle / run。
  // 这个库 GitHub 示例模型常见是小写动画名。
  flyIdleAnim: 'flyidle',
  flyAnim: 'fly',
}

const PLAYER_MODEL_OPTIONS: Record<
  PlayerModelKey,
  {
    label: string
    url: string
    scale: number
    idleAnim: string
    walkAnim: string
    runAnim: string
    jumpAnim: string
    flyIdleAnim: string
    flyAnim: string
  }
> = {
  tommy: {
    label: '汤米',
    url: tommyModelUrl,
    scale: PLAYER_MODEL_CONFIG.scale,
    idleAnim: PLAYER_MODEL_CONFIG.idleAnim,
    walkAnim: PLAYER_MODEL_CONFIG.walkAnim,
    runAnim: PLAYER_MODEL_CONFIG.runAnim,
    jumpAnim: PLAYER_MODEL_CONFIG.jumpAnim,
    flyIdleAnim: PLAYER_MODEL_CONFIG.flyIdleAnim,
    flyAnim: PLAYER_MODEL_CONFIG.flyAnim,
  },
  ant: {
    label: '蜡笔小新',
    url: person1ModelUrl,
    scale: PLAYER_MODEL_CONFIG.scale,
    idleAnim: PLAYER_MODEL_CONFIG.idleAnim,
    walkAnim: PLAYER_MODEL_CONFIG.walkAnim,
    runAnim: PLAYER_MODEL_CONFIG.runAnim,
    jumpAnim: PLAYER_MODEL_CONFIG.jumpAnim,
    flyIdleAnim: PLAYER_MODEL_CONFIG.flyIdleAnim,
    flyAnim: PLAYER_MODEL_CONFIG.flyAnim,
  },
}

const PLAYER_BOUNDARY_CONFIG = {
  // 不能走出 / 飞出圆形平面：用一个透明开口圆柱当边界墙。
  // 半径基本等于地面半径，简单、稳定，不再用一圈 box 拼接，避免卡住玩家。
  radius: GROUND_RADIUS - 0.08,
  wallHeight: 30,
  wallSegments: 96,
  wallThickness: 0.02,
}

const THIRD_PERSON_CAMERA_CONFIG = {
  // 后上方第三人称：更贴近玩家，但仍能看到完整人物和前方街道。
  distance: 1.75,
  height: 1.45,
  lookHeight: 0.5,
  smooth: 0.24,
  defaultYaw: 180,
  defaultPitch: 24,
}

const THIRD_PERSON_MOUSE_CONFIG = {
  sensitivity: 0.12,
  minPitch: -20,
  maxPitch: 30,
}

const PLAYER_MODEL_VISUAL_CONFIG = {
  // 不再每帧强行改 model.position.y。
  // three-player-controller 会自己同步模型和胶囊体位置，外部再改 y 会和控制器打架，造成“起飞/弹跳”。
  // 如果脚底还有轻微穿地，优先调 PLAYER_MODEL_CONFIG.scale 或 capsuleRadiusRatio，不要改模型根节点 y。
  yOffset: 0,
}

const GROUND_SURFACE_Y = 0.08

const RangeRow = defineComponent({
  name: 'RangeRow',
  props: {
    label: { type: String, required: true },
    value: { type: Number, required: true },
    min: { type: Number, required: true },
    max: { type: Number, required: true },
    step: { type: Number, default: 1 },
    suffix: { type: String, default: '' },
  },
  emits: ['update:value'],
  setup(props, { emit }) {
    const formatValue = (value: number) => `${Number(value).toFixed(props.step < 1 ? 2 : 0)}${props.suffix}`

    return () =>
      h('div', { class: 'range-row' }, [
        h('div', { class: 'range-head' }, [
          h('span', { class: 'range-label' }, props.label),
          h('b', { class: 'range-value' }, formatValue(props.value)),
        ]),
        h(ElSlider, {
          class: 'range-slider',
          modelValue: props.value,
          min: props.min,
          max: props.max,
          step: props.step,
          showTooltip: false,
          'onUpdate:modelValue': (value: number | number[]) => {
            const nextValue = Array.isArray(value) ? Number(value[0]) : Number(value)
            emit('update:value', nextValue)
          },
        }),
      ])
  },
})

const canvasWrapRef = ref<HTMLDivElement | null>(null)
const isPlaying = ref(true)
const currentView = ref<ViewPreset>('standard')
const controlMode = ref<ControlMode>('god')
const playerViewMode = ref<PlayerViewMode>('third')
const isPlayerReady = ref(false)
const selectedPlayerModel = ref<PlayerModelKey>('ant')

const state = reactive({
  latitude: 31.23,
  dayOfYear: 172,
  solarTime: 9.2,
  playSpeed: 1.2,
})

const playerModelButtons = computed(() =>
  (Object.keys(PLAYER_MODEL_OPTIONS) as PlayerModelKey[]).map(key => ({
    key,
    label: PLAYER_MODEL_OPTIONS[key].label,
  })),
)

const layers = reactive<LayerState>({
  dome: true,
  paths: true,
  shadow: true,
  rays: true,
  altitudeGauge: true,
  formula: true,
  cityTime: true,
})

let scene: THREE.Scene
let camera: THREE.PerspectiveCamera
let renderer: THREE.WebGLRenderer
let controls: OrbitControls
let resizeObserver: ResizeObserver | null = null
let animationId = 0
let lastTime = 0
let frameCount = 0
let runtimeSolarTime = state.solarTime
let lastSolarTimeUiSync = 0
const SOLAR_TIME_UI_SYNC_INTERVAL = 16
// 性能优化：太阳影子、天空、城市灯光、雷达扫描不需要每帧全部重建。
let lastShadowUpdateTime = 0
let lastGaugeUpdateTime = 0
let lastSkyAndCityUpdateTime = 0
const SHADOW_UPDATE_INTERVAL = 130
const PLAYER_SHADOW_UPDATE_INTERVAL = 999999
const GAUGE_UPDATE_INTERVAL = 240
const SKY_CITY_UPDATE_INTERVAL = 16

let rootGroup: THREE.Group
let domeGroup: THREE.Group
let pathGroup: THREE.Group
let sunGroup: THREE.Group
let rayGroup: THREE.Group
let shadowGroup: THREE.Group
let labelGroup: THREE.Group
let schoolGroup: THREE.Group
let altitudeAngleGroup: THREE.Group
let skyDecorationGroup: THREE.Group
let colliderGroup: THREE.Group

let player: any = null
let playerModelObjectUrl = ''
const playerModelObjectUrls = new Map<PlayerModelKey, string>()
let suppressPlayerViewChange = false

const tempPlayerForward = new THREE.Vector3()
const tempCameraTarget = new THREE.Vector3()
const tempCameraPosition = new THREE.Vector3()
let thirdPersonYaw = 0
let thirdPersonPitch = 10
let thirdPersonDragging = false
let thirdPersonPointerId: number | null = null
let playerInputEnabled = false
let playerInitializingPromise: Promise<void> | null = null

// 修复：第一人称按 Space 跳跃后，three-player-controller 偶发触发 onViewChange，导致视角被切回第三人称。
// 这里记录一个短时间锁，只在玩家第一人称跳跃时生效；锁内如果库把视角切走，就立即恢复第一人称。
let firstPersonJumpViewLockUntil = 0

let ambientLight: THREE.AmbientLight
let keyLight: THREE.DirectionalLight
let rimLight: THREE.DirectionalLight

let sunMesh: THREE.Mesh
let sunGlow: THREE.Sprite
let lightRay: THREE.Line
let hemisphereDome: THREE.Mesh

const shadowCasters: ShadowCaster[] = []

type StreetLightItem = {
  pole: THREE.Mesh
  lamp: THREE.Mesh
  glow: THREE.Sprite
  cone?: THREE.Mesh
}
type WindowLightItem = { material: THREE.MeshBasicMaterial; seed: number }
type TrafficLightItem = { red: THREE.MeshBasicMaterial; yellow: THREE.MeshBasicMaterial; green: THREE.MeshBasicMaterial; seed: number }
type CityClockItem = {
  texture: THREE.CanvasTexture
  ctx: CanvasRenderingContext2D
  material: THREE.MeshBasicMaterial
  group: THREE.Group
  /** 避免每帧重绘 canvas 文字造成性能浪费 */
  lastKey?: string
}

const streetLightItems: StreetLightItem[] = []
const windowLightItems: WindowLightItem[] = []
const trafficLightItems: TrafficLightItem[] = []
const cityRoadMaterials: THREE.MeshStandardMaterial[] = []
const cityClockItems: CityClockItem[] = []

// 城市街道观测点：太阳高度角、太阳光线和观测点标注统一使用这个位置，避免光线/量角弧线错位。
const OBSERVER_POINT = new THREE.Vector3(-0.02, 0.08, -0.98)

const dateLabel = computed(() => dayOfYearToMonthDay(state.dayOfYear))
const solarMetrics = computed<SolarMetrics>(() => computeSolarMetrics(state.latitude, state.dayOfYear, state.solarTime))

const dayPhase = computed<'night' | 'dawn' | 'day' | 'sunset'>(() => {
  const alt = solarMetrics.value.altitude
  // 太阳高度角低于约 -6° 时接近民用晨昏线以下，更适合判定为夜晚；-6°~8° 作为晨昏过渡。
  if (alt <= -6) return 'night'
  if (alt < 8) return state.solarTime < 12 ? 'dawn' : 'sunset'
  return 'day'
})

const pageSkyStyle = computed(() => {
  const sky = getSmoothSkyColors(solarMetrics.value.altitude, state.solarTime)
  return {
    background: `
      radial-gradient(circle at ${sky.sunX}% ${sky.sunY}%, ${sky.glowWarm}, transparent 25%),
      radial-gradient(circle at 16% 18%, ${sky.glowCool}, transparent 30%),
      radial-gradient(circle at 72% 86%, rgba(129, 140, 248, ${sky.nightGlowAlpha}), transparent 38%),
      linear-gradient(180deg, ${sky.top} 0%, ${sky.mid} 48%, ${sky.bottom} 100%)
    `,
  }
})

const formatDayLength = computed(() => (solarMetrics.value.polarType ? solarMetrics.value.polarType : formatHour(solarMetrics.value.dayLength)))
const oneMeterShadowText = computed(() => {
  const altitude = solarMetrics.value.altitude
  if (altitude <= 0) return '太阳在地平线下，暂无影子'
  const length = 1 / Math.tan(degToRad(Math.max(1, altitude)))
  return `${length.toFixed(2)} m`
})

const sceneTitle = computed(() => {
  const isNorth = state.latitude >= 0
  if (state.dayOfYear >= 160 && state.dayOfYear <= 185) {
    return isNorth ? '6月夏至前后 · 北半球路径高、昼长较长' : '6月冬至前后 · 南半球路径低、昼长较短'
  }
  if (state.dayOfYear >= 345 || state.dayOfYear <= 12) {
    return isNorth ? '12月冬至前后 · 北半球路径低、昼长较短' : '12月夏至前后 · 南半球路径高、昼长较长'
  }
  if (Math.abs(state.dayOfYear - 80) < 10 || Math.abs(state.dayOfYear - 266) < 10) return '春秋分前后 · 昼夜接近等长'
  return '太阳周日视运动 · 路径随日期变化'
})

const sceneSubtitle = computed(() => `纬度 ${formatDeg(state.latitude)} · ${dateLabel.value} · 地方太阳时 ${formatClock(state.solarTime)}`)

function degToRad(deg: number) {
  return THREE.MathUtils.degToRad(deg)
}

function radToDeg(rad: number) {
  return THREE.MathUtils.radToDeg(rad)
}

function clamp(v: number, min: number, max: number) {
  return Math.max(min, Math.min(max, v))
}

function normalize360(deg: number) {
  return ((deg % 360) + 360) % 360
}

function normalizeAngle180(angle: number) {
  let next = ((((angle + 180) % 360) + 360) % 360) - 180
  if (next === -180) next = 180
  return next
}

function hexToRgb(hex: number) {
  return { r: (hex >> 16) & 255, g: (hex >> 8) & 255, b: hex & 255 }
}

function mixColor(a: number, b: number, t: number) {
  const ca = hexToRgb(a)
  const cb = hexToRgb(b)
  const k = clamp(t, 0, 1)
  const r = Math.round(ca.r + (cb.r - ca.r) * k)
  const g = Math.round(ca.g + (cb.g - ca.g) * k)
  const bl = Math.round(ca.b + (cb.b - ca.b) * k)
  return `rgb(${r}, ${g}, ${bl})`
}

function mixColorNumber(a: number, b: number, t: number) {
  const ca = hexToRgb(a)
  const cb = hexToRgb(b)
  const k = clamp(t, 0, 1)
  const r = Math.round(ca.r + (cb.r - ca.r) * k)
  const g = Math.round(ca.g + (cb.g - ca.g) * k)
  const bl = Math.round(ca.b + (cb.b - ca.b) * k)
  return (r << 16) + (g << 8) + bl
}

function smoothstep(edge0: number, edge1: number, x: number) {
  const t = clamp((x - edge0) / (edge1 - edge0), 0, 1)
  return t * t * (3 - 2 * t)
}

function getSmoothSkyColors(altitude: number, solarTime: number) {
  const isMorning = solarTime < 12
  const night = { top: 0x020713, mid: 0x071427, bottom: 0x030611, dome: 0x2351a3 }
  const dawn = { top: 0xf1b06f, mid: 0x6fb8dd, bottom: 0x092142, dome: 0xffd19a }
  const day = { top: 0x7ed7ff, mid: 0xbfeeff, bottom: 0x0d3158, dome: 0x9be7ff }
  const sunset = { top: 0x664c94, mid: 0xc66b58, bottom: 0x081529, dome: 0xffa66b }

  let a = night
  let b = isMorning ? dawn : sunset
  let t = smoothstep(-8, 7, altitude)

  if (altitude > 4) {
    a = isMorning ? dawn : sunset
    b = day
    t = smoothstep(4, 28, altitude)
  }

  return {
    top: mixColor(a.top, b.top, t),
    mid: mixColor(a.mid, b.mid, t),
    bottom: mixColor(a.bottom, b.bottom, t),
    clear: mixColorNumber(a.mid, b.mid, t),
    fog: mixColorNumber(a.bottom, b.bottom, t),
    dome: mixColorNumber(a.dome, b.dome, t),
    glowWarm: altitude > -6 ? `rgba(255, 209, 102, ${0.1 + 0.22 * smoothstep(-6, 35, altitude)})` : 'rgba(255, 209, 102, 0.02)',
    glowCool: `rgba(77, 220, 255, ${0.08 + 0.14 * smoothstep(-10, 20, altitude)})`,
    nightGlowAlpha: 0.12 + 0.14 * (1 - smoothstep(-2, 15, altitude)),
    sunX: isMorning ? 22 : 78,
    sunY: altitude > 20 ? 18 : 26,
  }
}

function computeDeclination(dayOfYear: number) {
  return 23.44 * Math.sin(degToRad((360 * (284 + dayOfYear)) / 365))
}

function computeSolarMetrics(latitude: number, dayOfYear: number, solarTime: number): SolarMetrics {
  const phi = degToRad(latitude)
  const deltaDeg = computeDeclination(dayOfYear)
  const delta = degToRad(deltaDeg)
  const hourAngleDeg = 15 * (solarTime - 12)
  const H = degToRad(hourAngleDeg)

  // 本地水平坐标系：z=北，x 的正方向在画面中表示西侧，x 的负方向表示东侧。
  // 这样当学生/相机从北侧面向南方观察时，东在左、西在右，符合真实站立观察的方向感。
  // 方位角 A：北为 0°，东为 90°，南为 180°，西为 270°。
  const east = -Math.cos(delta) * Math.sin(H)
  const north = Math.cos(phi) * Math.sin(delta) - Math.sin(phi) * Math.cos(delta) * Math.cos(H)
  const up = Math.sin(phi) * Math.sin(delta) + Math.cos(phi) * Math.cos(delta) * Math.cos(H)

  const altitude = radToDeg(Math.asin(clamp(up, -1, 1)))
  const azimuth = normalize360(radToDeg(Math.atan2(east, north)))
  const noonAltitude = 90 - Math.abs(latitude - deltaDeg)

  const cosH0 = -Math.tan(phi) * Math.tan(delta)
  let dayLength = 0
  let sunrise = 0
  let sunset = 0
  let polarType: '' | '极昼' | '极夜' = ''

  if (cosH0 < -1) {
    dayLength = 24
    sunrise = 0
    sunset = 24
    polarType = '极昼'
  } else if (cosH0 > 1) {
    dayLength = 0
    sunrise = 12
    sunset = 12
    polarType = '极夜'
  } else {
    const H0 = radToDeg(Math.acos(cosH0))
    dayLength = (2 * H0) / 15
    sunrise = 12 - H0 / 15
    sunset = 12 + H0 / 15
  }

  return {
    declination: deltaDeg,
    hourAngle: hourAngleDeg,
    altitude,
    azimuth,
    noonAltitude,
    dayLength,
    sunrise,
    sunset,
    polarType,
    east,
    north,
    up,
    solarTime,
  }
}

function dayOfYearToMonthDay(dayOfYear: number) {
  const monthDays = [31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31]
  let day = Math.round(clamp(dayOfYear, 1, 365))
  let month = 1
  for (const days of monthDays) {
    if (day <= days) break
    day -= days
    month += 1
  }
  return `${String(month).padStart(2, '0')}月${String(day).padStart(2, '0')}日`
}

function formatDeg(value: number) {
  const sign = value < 0 ? '-' : ''
  const abs = Math.abs(value)
  return `${sign}${abs.toFixed(1)}°`
}

function formatHour(value: number) {
  if (!Number.isFinite(value)) return '--'
  const safe = Math.max(0, value)
  const h = Math.floor(safe)
  const m = Math.round((safe - h) * 60)
  const fixedH = h + Math.floor(m / 60)
  const fixedM = m % 60
  return `${fixedH}时${String(fixedM).padStart(2, '0')}分`
}

function formatClock(value: number) {
  // 修复城市时钟牌 NaN:NaN:NaN：如果传入 undefined / NaN，直接显示占位。
  if (!Number.isFinite(value)) return '--:--:--'

  const totalSeconds = Math.round((((value % 24) + 24) % 24) * 3600) % 86400
  const h = Math.floor(totalSeconds / 3600)
  const m = Math.floor((totalSeconds % 3600) / 60)
  const sec = totalSeconds % 60

  return `${String(h).padStart(2, '0')}:${String(m).padStart(2, '0')}:${String(sec).padStart(2, '0')}`
}

function isApprox(value: number, target: number, tolerance = 0.05) {
  return Math.abs(value - target) <= tolerance
}

function setDay(day: number) {
  state.dayOfYear = Math.round(clamp(day, 1, 365))
}

function setSafeSolarTime(time: number) {
  if (!Number.isFinite(time)) return
  state.solarTime = clamp(time, 0, 24)
}

function togglePlay() {
  isPlaying.value = !isPlaying.value
}

function setStandardScene() {
  state.latitude = 31.23
  state.dayOfYear = 172
  state.solarTime = 9.2
  state.playSpeed = 1.2
  setControlMode('god')
  setCameraView('standard')
}

function setCameraView(view: ViewPreset) {
  currentView.value = view
  controlMode.value = 'god'
  applyControlMode()
  if (!camera || !controls) return

  const views: Record<ViewPreset, { pos: [number, number, number]; target: [number, number, number] }> = {
    standard: { pos: [7.6, 5.2, 8.6], target: [0, 0.55, 0] },
    path: { pos: [0.2, 7.5, 10.2], target: [0, 1.9, 0] },
    top: { pos: [0, 13.6, 0.01], target: [0, 0, 0] },
    shadow: { pos: [6.8, 2.2, -7.8], target: [0.4, 0.25, 0.4] },
    south: { pos: [0, 3.6, -11.5], target: [0, 1.1, 0] },
  }

  const v = views[view]
  camera.position.set(...v.pos)
  controls.target.set(...v.target)
  controls.update()
}

function resetCamera() {
  if (controlMode.value === 'player') {
    resetPlayerToSpawn()
    return
  }
  setCameraView(currentView.value || 'standard')
}

function normalizeModelUrl(url: string, mime = 'model/gltf-binary') {
  if (!url) return ''
  if (url.startsWith('blob:')) return url

  // file:/// 直接 fetch 本地 glb 会被浏览器 CORS 拦截。
  // 所以打包产物必须让 person.glb 变成 data:base64，再转 Blob URL 给 GLTFLoader。
  if (url.startsWith('data:')) {
    const [header, body] = url.split(',')
    if (!body) return url
    const match = header?.match(/data:([^;]+)/)
    const blobMime = match?.[1] || mime
    const binary = atob(body)
    const bytes = new Uint8Array(binary.length)
    for (let i = 0; i < binary.length; i++) bytes[i] = binary.charCodeAt(i)
    return URL.createObjectURL(new Blob([bytes], { type: blobMime }))
  }

  // 开发环境 http://localhost 可以直接加载；file:/// 必须依赖 vite.config.ts 内联。
  // 如果这里仍然是 assets/person.glb，说明 glb 没有被 Vite 内联，file:/// 打开一定会 CORS。
  return url
}

function getNormalizedPlayerModelUrl(modelKey: PlayerModelKey) {
  const existed = playerModelObjectUrls.get(modelKey)
  if (existed) return existed

  const url = normalizeModelUrl(PLAYER_MODEL_OPTIONS[modelKey].url)
  playerModelObjectUrls.set(modelKey, url)
  return url
}

function getCurrentPlayerModelConfig() {
  const option = PLAYER_MODEL_OPTIONS[selectedPlayerModel.value]
  return {
    option,
    url: getNormalizedPlayerModelUrl(selectedPlayerModel.value),
  }
}

async function initPlayerController() {
  if (!scene || !camera || !controls || !colliderGroup) return

  try {
    const currentModel = getCurrentPlayerModelConfig()
    playerModelObjectUrl = currentModel.url

    player = new playerController()

    suppressPlayerViewChange = true

    await player.init({
      scene,
      camera,
      controls,
      initPos: getPlayerSpawnPosition(),
      staticCollider: colliderGroup,
      mouseSensitivity: 4,
      minCamDistance: 3.6,
      maxCamDistance: 8.6,
      thirdMouseMode: 3,
      enableZoom: true,
      enableOverShoulderView: false,
      camLookAtHeightRatio: 0.78,
      isFirstPerson: false,
      playerModelConfig: {
        url: playerModelObjectUrl,
        scale: currentModel.option.scale,
        idleAnim: currentModel.option.idleAnim,
        walkAnim: currentModel.option.walkAnim,
        runAnim: currentModel.option.runAnim,
        jumpAnim: currentModel.option.jumpAnim,
        backwardAnim: currentModel.option.walkAnim,
        leftWalkAnim: currentModel.option.walkAnim,
        rightWalkAnim: currentModel.option.walkAnim,
        flyEnabled: true,
        flyIdleAnim: currentModel.option.flyIdleAnim,
        flyAnim: currentModel.option.flyAnim,
        flyHoverForwardAnim: currentModel.option.flyAnim,
        flyHoverBackAnim: currentModel.option.flyIdleAnim,
        flyHoverLeftAnim: currentModel.option.flyIdleAnim,
        flyHoverRightAnim: currentModel.option.flyIdleAnim,
        flyHoverUpAnim: currentModel.option.flyIdleAnim,
        flyHoverDownAnim: currentModel.option.flyIdleAnim,
        speed: 380 * 4.5,
        flySpeed: 1250 * 5.5,
        jumpHeight: 420,
        gravity: -2400,
        capsuleRadiusRatio: 0.62,
        firstPersonCameraOffset: [0, 0.1, 0.1],
      },
    })

    // 初始化完成立刻关闭库的输入监听。只有玩家模式才打开。
    setPlayerInputEnabled(false)

    player.onViewChange = (isFirstPerson: boolean) => {
      if (suppressPlayerViewChange) return

      // 上帝模式下忽略控制器内部视角变化，避免初始化/模型加载后自动把镜头带到第一人称。
      if (controlMode.value !== 'player') {
        playerViewMode.value = 'third'
        syncPlayerModelVisible()
        return
      }

      // 第一人称跳跃期间，Space 只允许触发跳跃，不允许把视角切到第三人称。
      if (isFirstPersonJumpViewLockActive() && playerViewMode.value === 'first') {
        requestAnimationFrame(() => restoreFirstPersonViewAfterJump())
        return
      }

      playerViewMode.value = isFirstPerson ? 'first' : 'third'
      syncPlayerModelVisible()
    }

    isPlayerReady.value = true
    playerViewMode.value = 'third'

    tunePlayerCameraForThirdPerson()

    // 强制第三人称，并播放 idle，避免人物僵硬。
    forcePlayerThirdPerson()
    applyPlayerShadowSettings()
    playPlayerIdleAnimation()
    syncPlayerModelVisible()

    suppressPlayerViewChange = false

    // 初始化后把相机还给上帝模式；如果此时用户仍在玩家模式，再切回第三人称。
    requestAnimationFrame(() => {
      tunePlayerCameraForThirdPerson()

      if (controlMode.value === 'player') {
        setPlayerView('third')
      } else {
        setPlayerInputEnabled(false)
        playerViewMode.value = 'third'
        syncPlayerModelVisible()
        if (controls) {
          controls.enabled = true
          controls.enableDamping = true
          controls.update()
        }
        setCameraView(currentView.value || 'standard')
      }

      // 有些 GLB 动画混合器在首帧后才真正可用，这里再补一次 idle。
      requestAnimationFrame(() => playPlayerIdleAnimation())
    })
  } catch (err) {
    console.error(
      'three-player-controller 初始化失败：如果你正在用 file:/// 打开，请确认 vite.config.ts 已把 .glb 内联成 data URL；否则浏览器会拦截 file:///assets/person.glb。',
      err,
    )
    isPlayerReady.value = false
    playerInitializingPromise = null
    suppressPlayerViewChange = false
    setControlMode('god')
  }
}

async function ensurePlayerControllerReady() {
  if (isPlayerReady.value && player) return
  if (!playerInitializingPromise) {
    playerInitializingPromise = initPlayerController().finally(() => {
      playerInitializingPromise = null
    })
  }
  await playerInitializingPromise
}

function getClampedPlayerPosition(pos: THREE.Vector3) {
  const next = pos.clone()
  const radius = PLAYER_BOUNDARY_CONFIG.radius
  const horizontal = Math.sqrt(next.x * next.x + next.z * next.z)

  if (horizontal > radius) {
    const k = radius / horizontal
    next.x *= k
    next.z *= k
  }

  // 只限制水平圆形平面边界；高度不强行压死，避免影响跳跃/飞行体验。
  return next
}

function clampPlayerToGroundBoundary() {
  if (!player || !isPlayerReady.value) return

  const pos = getPlayerWorldPosition()
  if (!pos) return

  const clamped = getClampedPlayerPosition(pos)
  if (clamped.distanceToSquared(pos) <= 0.01) return

  player.reset?.(clamped)
  updateManualThirdPersonCamera(1 / 60)
}

function getPlayerSpawnPosition() {
  // 城市最南端出生点。当前坐标系 z+ 为北，z- 为南。
  return new THREE.Vector3(0, GROUND_SURFACE_Y + 0.26, -6.15)
}

function resetPlayerToSpawn() {
  player?.reset?.(getPlayerSpawnPosition())
  playPlayerIdleAnimation()
}

function disposeObject3D(root?: THREE.Object3D | null) {
  if (!root) return

  root.traverse(obj => {
    const mesh = obj as THREE.Mesh
    if (!mesh.isMesh) return

    mesh.geometry?.dispose?.()

    const material = mesh.material
    const materials = Array.isArray(material) ? material : [material]
    materials.forEach(mat => {
      if (!mat) return

      // 尽量释放模型材质里可能挂载的贴图。
      Object.keys(mat).forEach(key => {
        const value = (mat as any)[key]
        if (value && value.isTexture && typeof value.dispose === 'function') {
          value.dispose()
        }
      })

      mat.dispose?.()
    })
  })

  root.parent?.remove(root)
}

function destroyCurrentPlayerController() {
  if (!player) return

  const model = player.getPlayerModel?.()
  disposeObject3D(model)

  setPlayerInputEnabled(false)
  player.destroy?.()
  player = null
  isPlayerReady.value = false
  playerInputEnabled = false
}

async function setPlayerModel(modelKey: PlayerModelKey) {
  if (selectedPlayerModel.value === modelKey) return

  const wasPlayerMode = controlMode.value === 'player'
  const wasFirstPerson = playerViewMode.value === 'first'
  const safePos = getClampedPlayerPosition(getPlayerWorldPosition() ?? getPlayerSpawnPosition())

  // 先切换 key，再销毁旧模型/旧控制器，避免新 init 继续拿旧配置。
  selectedPlayerModel.value = modelKey

  // 没进入玩家模式时只记录选择；等用户点击玩家模式再懒加载。
  if (!wasPlayerMode && !player) return

  destroyCurrentPlayerController()
  playerInitializingPromise = null

  if (!wasPlayerMode) return

  controlMode.value = 'player'
  playerViewMode.value = 'third'

  await ensurePlayerControllerReady()

  if (player && isPlayerReady.value) {
    player.reset?.(safePos)
    applyPlayerShadowSettings()
    playPlayerIdleAnimation()
    clampPlayerToGroundBoundary()
    applyControlMode()
    setPlayerView(wasFirstPerson ? 'first' : 'third')
  }
}

async function setControlMode(mode: ControlMode) {
  if (mode === 'god') {
    controlMode.value = 'god'
    playerViewMode.value = 'third'
    suppressPlayerViewChange = true
    forcePlayerThirdPerson()
    suppressPlayerViewChange = false
    setPlayerInputEnabled(false)
    applyControlMode()
    setCameraView(currentView.value || 'standard')
    return
  }

  // 玩家模式懒加载，避免页面初始化后自动切第一人称。
  controlMode.value = 'player'
  playerViewMode.value = 'third'
  await ensurePlayerControllerReady()

  if (!player || !isPlayerReady.value) {
    controlMode.value = 'god'
    applyControlMode()
    return
  }

  applyControlMode()
  setPlayerView('third')
}

function forcePlayerThirdPerson() {
  if (!player || !isPlayerReady.value) return

  const isFirst = Boolean(player.getIsFirstPerson?.())
  if (!isFirst) {
    playerViewMode.value = 'third'
    tunePlayerCameraForThirdPerson()
    syncPlayerModelVisible()
    return
  }

  suppressPlayerViewChange = true
  setPlayerInputEnabled(false)
  player.changeView?.()
  suppressPlayerViewChange = false
  playerViewMode.value = 'third'
  tunePlayerCameraForThirdPerson()
  syncPlayerModelVisible()
}

function applyPlayerShadowSettings() {
  const model = player?.getPlayerModel?.()
  if (!model) return

  model.traverse((obj: THREE.Object3D) => {
    const mesh = obj as THREE.Mesh
    if (!mesh.isMesh) return

    mesh.castShadow = true
    mesh.receiveShadow = true
    mesh.frustumCulled = false

    const material = mesh.material
    const materials = Array.isArray(material) ? material : [material]

    // 如果模型材质是 MeshBasicMaterial，它不会受光照影响，但仍可 castShadow。
    // 为了阴影稳定，这里主要保证 depthWrite / side 合理，不强行替换材质，避免模型贴图丢失。
    materials.forEach(mat => {
      if (!mat) return
      mat.depthWrite = true
      mat.needsUpdate = true
    })
  })
}

function playPlayerIdleAnimation() {
  if (!player || !isPlayerReady.value) return

  // 这个库文档里是 playPlayerAnimationByName(name, fade?)。
  // 如果模型的 idle 动画名不对，请改 PLAYER_MODEL_CONFIG.idleAnim。
  try {
    player.playPlayerAnimationByName?.(PLAYER_MODEL_OPTIONS[selectedPlayerModel.value].idleAnim, 0.25)
  } catch (err) {
    console.warn('idle 动画播放失败，请检查当前 GLB 里的动画名是否为：', PLAYER_MODEL_OPTIONS[selectedPlayerModel.value].idleAnim, err)
  }
}

function isThirdPersonMouseActive() {
  return controlMode.value === 'player' && playerViewMode.value === 'third' && isPlayerReady.value
}

function bindThirdPersonMouseControls(canvas: HTMLCanvasElement) {
  canvas.addEventListener('pointerdown', handleThirdPersonPointerDown)
  canvas.addEventListener('pointermove', handleThirdPersonPointerMove)
  canvas.addEventListener('pointerup', handleThirdPersonPointerUp)
  canvas.addEventListener('pointerleave', handleThirdPersonPointerUp)
  canvas.addEventListener('contextmenu', preventThirdPersonContextMenu)
}

function unbindThirdPersonMouseControls(canvas?: HTMLCanvasElement) {
  if (!canvas) return
  canvas.removeEventListener('pointerdown', handleThirdPersonPointerDown)
  canvas.removeEventListener('pointermove', handleThirdPersonPointerMove)
  canvas.removeEventListener('pointerup', handleThirdPersonPointerUp)
  canvas.removeEventListener('pointerleave', handleThirdPersonPointerUp)
  canvas.removeEventListener('contextmenu', preventThirdPersonContextMenu)
}

function preventThirdPersonContextMenu(e: MouseEvent) {
  if (isThirdPersonMouseActive()) e.preventDefault()
}

function handleThirdPersonPointerDown(e: PointerEvent) {
  if (!isThirdPersonMouseActive()) return
  if (e.button !== 0 && e.button !== 2) return

  thirdPersonDragging = true
  thirdPersonPointerId = e.pointerId

  const canvas = e.currentTarget as HTMLCanvasElement
  canvas.setPointerCapture?.(e.pointerId)
  canvas.style.cursor = 'grabbing'

  e.preventDefault()
}

function handleThirdPersonPointerMove(e: PointerEvent) {
  if (!isThirdPersonMouseActive()) return
  if (!thirdPersonDragging || thirdPersonPointerId !== e.pointerId) return

  // 第三人称视角：按住鼠标拖动旋转镜头。
  thirdPersonYaw = normalizeAngle180(thirdPersonYaw - e.movementX * THIRD_PERSON_MOUSE_CONFIG.sensitivity)
  thirdPersonPitch = clamp(
    thirdPersonPitch - e.movementY * THIRD_PERSON_MOUSE_CONFIG.sensitivity,
    THIRD_PERSON_MOUSE_CONFIG.minPitch,
    THIRD_PERSON_MOUSE_CONFIG.maxPitch,
  )

  e.preventDefault()
}

function handleThirdPersonPointerUp(e: PointerEvent) {
  if (thirdPersonPointerId !== null && thirdPersonPointerId !== e.pointerId) return

  thirdPersonDragging = false
  thirdPersonPointerId = null

  const canvas = e.currentTarget as HTMLCanvasElement
  canvas.releasePointerCapture?.(e.pointerId)
  canvas.style.cursor = isThirdPersonMouseActive() ? 'grab' : 'default'

  e.preventDefault()
}

function initThirdPersonCameraAngle(forceDefault = false) {
  if (!camera || forceDefault) {
    thirdPersonYaw = THIRD_PERSON_CAMERA_CONFIG.defaultYaw
    thirdPersonPitch = THIRD_PERSON_CAMERA_CONFIG.defaultPitch
    return
  }

  const pos = getPlayerWorldPosition()
  if (!pos) {
    thirdPersonYaw = THIRD_PERSON_CAMERA_CONFIG.defaultYaw
    thirdPersonPitch = THIRD_PERSON_CAMERA_CONFIG.defaultPitch
    return
  }

  const offset = camera.position.clone().sub(pos)
  if (offset.lengthSq() < 0.0001) {
    thirdPersonYaw = THIRD_PERSON_CAMERA_CONFIG.defaultYaw
    thirdPersonPitch = THIRD_PERSON_CAMERA_CONFIG.defaultPitch
    return
  }

  thirdPersonYaw = normalizeAngle180(radToDeg(Math.atan2(offset.x, offset.z)))
  thirdPersonPitch = clamp(
    radToDeg(Math.atan2(offset.y - THIRD_PERSON_CAMERA_CONFIG.lookHeight, Math.sqrt(offset.x * offset.x + offset.z * offset.z))),
    THIRD_PERSON_MOUSE_CONFIG.minPitch,
    THIRD_PERSON_MOUSE_CONFIG.maxPitch,
  )
}

function tunePlayerCameraForThirdPerson() {
  if (!player || !isPlayerReady.value) return

  // 第三人称镜头拉远，避免贴脸。
  player.setMinCamDistance?.(THIRD_PERSON_CAMERA_CONFIG.distance)
  player.setMaxCamDistance?.(THIRD_PERSON_CAMERA_CONFIG.distance + 1.8)
  player.setCamLookAtHeightRatio?.(0.68)
  player.setEnableZoom?.(true)
  player.setOverShoulderView?.(false)

  // 3：显示鼠标，拖拽控制相机；不会像 5 那样锁鼠标，能减少刚切换时的疯狂旋转。
  player.setThirdMouseMode?.(3)
}

function applyOrbitControlForMode() {
  if (!controls) return

  // 玩家模式下完全关闭 OrbitControls，避免它和 three-player-controller / 手动第三人称相机抢控制。
  const isPlayerMode = controlMode.value === 'player' && isPlayerReady.value
  controls.enabled = !isPlayerMode
  controls.enableDamping = !isPlayerMode
}

function getPlayerWorldPosition() {
  const pos = player?.getPosition?.()
  if (!pos) return null
  return pos instanceof THREE.Vector3 ? pos.clone() : new THREE.Vector3(pos.x ?? 0, pos.y ?? 0, pos.z ?? 0)
}

function updateManualThirdPersonCamera(dt: number) {
  if (!camera || !player || !isPlayerReady.value) return
  if (controlMode.value !== 'player' || playerViewMode.value !== 'third') return

  const basePos = getPlayerWorldPosition()
  if (!basePos) return

  const yaw = degToRad(thirdPersonYaw)
  const pitch = degToRad(thirdPersonPitch)
  const horizontalDistance = THIRD_PERSON_CAMERA_CONFIG.distance * Math.cos(pitch)

  // 第三人称镜头现在由鼠标控制，不再跟着人物朝向/AD 改变视角。
  // 这样 A/D 只负责人物移动/转向，鼠标负责转动观察方向。
  tempCameraPosition.set(
    basePos.x + Math.sin(yaw) * horizontalDistance,
    basePos.y + THIRD_PERSON_CAMERA_CONFIG.lookHeight + Math.sin(pitch) * THIRD_PERSON_CAMERA_CONFIG.distance,
    basePos.z + Math.cos(yaw) * horizontalDistance,
  )

  tempCameraTarget.copy(basePos).add(new THREE.Vector3(0, THIRD_PERSON_CAMERA_CONFIG.lookHeight, 0))

  const alpha = 1 - Math.pow(1 - THIRD_PERSON_CAMERA_CONFIG.smooth, Math.max(1, dt * 60))
  camera.position.lerp(tempCameraPosition, alpha)
  camera.lookAt(tempCameraTarget)
}

function syncPlayerFlightAnimation() {
  if (!player || !isPlayerReady.value || controlMode.value !== 'player') return

  const isFlying = Boolean(player.getIsFlying?.())
  if (!isFlying) return

  // 飞行时主动补一次飞行动作。若模型里没有 fly / flyidle，请把 PLAYER_MODEL_CONFIG.flyAnim 改成已有动画名。
  const velocity = player.getVelocity?.()
  const moving = velocity && typeof velocity.lengthSq === 'function' ? velocity.lengthSq() > 0.001 : false
  const current = PLAYER_MODEL_OPTIONS[selectedPlayerModel.value]
  const animName = moving ? current.flyAnim : current.flyIdleAnim
  try {
    player.playPlayerAnimationByName?.(animName, 0.18)
  } catch {
    // 如果模型没有飞行动画，静默退回，不影响移动和飞行功能。
  }
}

function isFirstPersonJumpViewLockActive() {
  return performance.now() < firstPersonJumpViewLockUntil
}

function handlePlayerKeyDown(e: KeyboardEvent) {
  if (e.code !== 'Space') return
  if (controlMode.value !== 'player') return
  if (playerViewMode.value !== 'first') return
  if (!player || !isPlayerReady.value) return

  // 不阻止默认事件，不拦截 three-player-controller 的跳跃输入；
  // 只锁定视角，避免 Space 跳跃被误处理成视角切换。
  firstPersonJumpViewLockUntil = performance.now() + 620
}

function restoreFirstPersonViewAfterJump() {
  if (!player || !isPlayerReady.value) return
  if (controlMode.value !== 'player') return
  if (!isFirstPersonJumpViewLockActive()) return

  playerViewMode.value = 'first'

  const isFirst = Boolean(player.getIsFirstPerson?.())
  if (!isFirst) {
    suppressPlayerViewChange = true
    player.changeView?.()
    suppressPlayerViewChange = false
  }

  syncPlayerModelVisible()
}

function syncPlayerModelVisible() {
  const model = player?.getPlayerModel?.()
  if (!model) return

  applyPlayerShadowSettings()

  // 上帝模式不展示人物；玩家第一人称隐藏模型，避免看到模型内部；
  // 玩家第三人称才展示人物。
  model.visible = controlMode.value === 'player' && playerViewMode.value === 'third'
}

function setPlayerInputEnabled(enabled: boolean) {
  if (!player) return
  if (enabled && !playerInputEnabled) {
    player.onAllEvent?.()
    playerInputEnabled = true
    return
  }
  if (!enabled && playerInputEnabled) {
    player.offAllEvent?.()
    playerInputEnabled = false
  }
}

function applyControlMode() {
  if (!renderer || !controls) return

  const isPlayerMode = controlMode.value === 'player' && isPlayerReady.value

  // 第三人称让库接管 OrbitControls；第一人称关闭 OrbitControls，防止鼠标控制打架。
  applyOrbitControlForMode()

  setPlayerInputEnabled(isPlayerMode)

  renderer.domElement.style.cursor = isPlayerMode
    ? playerViewMode.value === 'third'
      ? thirdPersonDragging
        ? 'grabbing'
        : 'grab'
      : 'crosshair'
    : 'default'

  if (!isThirdPersonMouseActive()) {
    thirdPersonDragging = false
    thirdPersonPointerId = null
  }

  syncPlayerModelVisible()
}

function setPlayerView(view: PlayerViewMode) {
  controlMode.value = 'player'

  if (!player || !isPlayerReady.value) {
    playerViewMode.value = view
    applyControlMode()
    return
  }

  // 切视角前临时关闭输入，避免切换瞬间的鼠标 delta 被吃进去导致疯狂旋转。
  setPlayerInputEnabled(false)

  const isFirst = Boolean(player.getIsFirstPerson?.())

  if (view === 'first') {
    playerViewMode.value = 'first'
    applyControlMode()

    // 不用 setFirstPersonCamera，它在这个模型/控制器组合里容易带入异常垂直角；
    // 用 changeView 只做“第三/第一”状态切换，更稳定。
    if (!isFirst) player.changeView?.()
  } else {
    playerViewMode.value = 'third'
    if (isFirst) {
      suppressPlayerViewChange = true
      player.changeView?.()
      suppressPlayerViewChange = false
    }
    tunePlayerCameraForThirdPerson()
    initThirdPersonCameraAngle(true)
    playPlayerIdleAnimation()
    applyControlMode()
    updateManualThirdPersonCamera(1 / 60)
  }

  syncPlayerModelVisible()

  requestAnimationFrame(() => {
    applyControlMode()
    if (view === 'third') {
      tunePlayerCameraForThirdPerson()
      initThirdPersonCameraAngle(true)
      updateManualThirdPersonCamera(1 / 60)
      playPlayerIdleAnimation()
    }
  })
}

function addColliderBox(x: number, y: number, z: number, width: number, height: number, depth: number) {
  if (!colliderGroup) return

  const collider = new THREE.Mesh(
    new THREE.BoxGeometry(width, height, depth),
    new THREE.MeshBasicMaterial({
      color: 0xff0000,
      transparent: true,
      opacity: 0,
      depthWrite: false,
    }),
  )
  collider.position.set(x, y, z)
  collider.name = 'static-collider'
  colliderGroup.add(collider)
}

function solarToPosition(metrics: SolarMetrics, radius = SKY_RADIUS) {
  // 世界坐标中 x 轴采用“屏幕/场景方位友好”的方向：x+ 为西，x- 为东。
  // 因此太阳学理计算出的 east 分量，需要映射为 -x。
  const x = -metrics.east * radius
  const y = metrics.up * radius
  const z = metrics.north * radius
  return new THREE.Vector3(x, y, z)
}

function initThree() {
  if (!canvasWrapRef.value) return

  scene = new THREE.Scene()
  scene.fog = new THREE.Fog(0x061022, 11, 28)

  const rect = canvasWrapRef.value.getBoundingClientRect()
  const width = Math.max(1, rect.width)
  const height = Math.max(1, rect.height)

  camera = new THREE.PerspectiveCamera(46, width / height, 0.1, 100)
  camera.position.set(7.6, 5.2, 8.6)

  renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true })
  renderer.setPixelRatio(getRendererPixelRatio())
  renderer.setSize(width, height, false)
  renderer.setClearColor(0x061022, 0)

  // v38：启用 Three.js 原生阴影。太阳方向由 keyLight 绑定太阳位置来控制。
  renderer.shadowMap.enabled = true
  renderer.shadowMap.type = THREE.PCFShadowMap
  // 太阳视运动需要影子连续变化，这里恢复每帧更新原生阴影。
  renderer.shadowMap.autoUpdate = true
  renderer.domElement.style.width = '100%'
  renderer.domElement.style.height = '100%'
  canvasWrapRef.value.appendChild(renderer.domElement)
  bindThirdPersonMouseControls(renderer.domElement)

  controls = new OrbitControls(camera, renderer.domElement)
  controls.enableDamping = true
  controls.dampingFactor = 0.08
  controls.target.set(0, 0.55, 0)
  controls.minDistance = 4.2
  controls.maxDistance = 18
  controls.enablePan = false
  controls.update()

  ambientLight = new THREE.AmbientLight(0xffffff, 1.55)
  scene.add(ambientLight)

  keyLight = new THREE.DirectionalLight(0xffe6a3, 1.75)
  keyLight.position.set(4, 8, 5)
  keyLight.castShadow = true
  keyLight.shadow.mapSize.set(640, 640)
  keyLight.shadow.camera.near = 0.5
  keyLight.shadow.camera.far = 26
  keyLight.shadow.camera.left = -7.2
  keyLight.shadow.camera.right = 7.2
  keyLight.shadow.camera.top = 7.2
  keyLight.shadow.camera.bottom = -7.2
  keyLight.shadow.bias = -0.00018
  keyLight.shadow.normalBias = 0.025
  scene.add(keyLight)

  rimLight = new THREE.DirectionalLight(0x60dcff, 0.95)
  rimLight.position.set(-7, 4, -5)
  scene.add(rimLight)

  rootGroup = new THREE.Group()
  scene.add(rootGroup)

  domeGroup = new THREE.Group()
  pathGroup = new THREE.Group()
  sunGroup = new THREE.Group()
  rayGroup = new THREE.Group()
  shadowGroup = new THREE.Group()
  labelGroup = new THREE.Group()
  schoolGroup = new THREE.Group()
  altitudeAngleGroup = new THREE.Group()
  skyDecorationGroup = new THREE.Group()
  colliderGroup = new THREE.Group()
  colliderGroup.name = '城市静态碰撞体'
  scene.add(colliderGroup)
  rootGroup.add(domeGroup, pathGroup, sunGroup, rayGroup, shadowGroup, labelGroup, schoolGroup, altitudeAngleGroup, skyDecorationGroup)

  createGround()
  createCityScene()
  applyMeshShadowSettings(schoolGroup)
  freezeStaticCityMatrices(schoolGroup)
  createDome()
  createLabels()
  createSun()
  createLightRay()
  createNightSkyDecorations()
  rebuildSolarPaths()
  applyLayerVisibility()
  updateSceneBySolar()

  resizeObserver = new ResizeObserver(() => requestAnimationFrame(resizeRenderer))
  resizeObserver.observe(canvasWrapRef.value)
  window.addEventListener('resize', resizeRenderer, { passive: true })
  window.addEventListener('keydown', handlePlayerKeyDown, true)

  // 玩家控制器改为懒加载：只有点击“玩家模式”时才初始化。
  // 避免 three-player-controller 初始化后自动接管相机，导致刚进页面就切到第一人称。
  lastTime = performance.now()
  animate(lastTime)
}

function applyMeshShadowSettings(root: THREE.Object3D) {
  root.traverse(obj => {
    const mesh = obj as THREE.Mesh
    if (!mesh.isMesh) return

    const material = mesh.material
    const materials = Array.isArray(material) ? material : [material]
    const canUseShadow = materials.some(
      mat => mat instanceof THREE.MeshStandardMaterial || mat instanceof THREE.MeshLambertMaterial || mat instanceof THREE.MeshPhongMaterial,
    )
    if (!canUseShadow) return

    mesh.receiveShadow = true

    // 每帧原生阴影下，所有小装饰都投影会明显变卡。
    // 这里只让建筑主体、屋顶、较大的树冠等参与投影；小窗户、路灯细杆、红绿灯等不投影。
    const geometry = mesh.geometry as THREE.BufferGeometry | undefined
    geometry?.computeBoundingBox()
    const box = geometry?.boundingBox
    if (!box) {
      mesh.castShadow = true
      return
    }

    const size = new THREE.Vector3()
    box.getSize(size)
    const maxSize = Math.max(size.x, size.y, size.z)
    const minSize = Math.min(size.x, size.y, size.z)

    mesh.castShadow = maxSize >= 0.24 && minSize >= 0.018
  })
}

function freezeStaticCityMatrices(root: THREE.Object3D) {
  root.traverse(obj => {
    // 窗户/路灯材质仍会变亮变暗，但物体本身不动，矩阵可以固定。
    obj.updateMatrix()
    obj.matrixAutoUpdate = false
  })
}

function syncThreeJsSunShadow(metrics: SolarMetrics) {
  if (!keyLight || !renderer) return

  const isShadowVisible = layers.shadow && metrics.altitude > 1
  keyLight.castShadow = isShadowVisible
  renderer.shadowMap.enabled = isShadowVisible

  // 太阳高度低时阴影更长；Three.js 阴影用定向光生成，方向与太阳位置绑定。
  const sunDir = solarToPosition(metrics, 1).normalize()
  // 定向光方向必须与太阳高度角一致。只在贴近地平线时留极小正值，避免阴影相机数值不稳定；不要强行抬高到 1.1，否则低太阳角影子会偏短。
  keyLight.position.set(sunDir.x * 9, Math.max(0.08, sunDir.y * 9), sunDir.z * 9)
  keyLight.target.position.set(0, GROUND_SURFACE_Y, 0)
  if (!keyLight.target.parent) scene.add(keyLight.target)
  keyLight.target.updateMatrixWorld()

  const dayK = smoothstep(-2, 30, metrics.altitude)
  keyLight.intensity = isShadowVisible ? 1.35 + dayK * 2.05 : 0.14
  // renderer.shadowMap.needsUpdate = true
}

function createCircularBoundaryColliders() {
  if (!colliderGroup) return

  const geometry = new THREE.CylinderGeometry(
    PLAYER_BOUNDARY_CONFIG.radius,
    PLAYER_BOUNDARY_CONFIG.radius,
    PLAYER_BOUNDARY_CONFIG.wallHeight,
    PLAYER_BOUNDARY_CONFIG.wallSegments,
    1,
    true,
  )

  // 让圆柱墙从地面向上延伸，玩家走路/跳跃/飞行都不能穿出圆形平面。
  geometry.translate(0, GROUND_SURFACE_Y + PLAYER_BOUNDARY_CONFIG.wallHeight / 2, 0)

  const wall = new THREE.Mesh(
    geometry,
    new THREE.MeshBasicMaterial({
      color: 0xff0000,
      transparent: true,
      opacity: 0,
      depthWrite: false,
      side: THREE.DoubleSide,
    }),
  )
  wall.name = 'circular-boundary-cylinder-collider'
  wall.visible = false

  colliderGroup.add(wall)
}

function createGround() {
  const grassTexture = createGrassTexture()
  grassTexture.wrapS = THREE.RepeatWrapping
  grassTexture.wrapT = THREE.RepeatWrapping
  grassTexture.repeat.set(7, 7)

  const ground = new THREE.Mesh(
    new THREE.CylinderGeometry(GROUND_RADIUS, GROUND_RADIUS, 0.08, 192),
    new THREE.MeshStandardMaterial({ map: grassTexture, color: 0x7fcf67, roughness: 0.86, metalness: 0.02 }),
  )
  ground.position.y = GROUND_SURFACE_Y - 0.04
  ground.receiveShadow = true
  rootGroup.add(ground)
  // 让碰撞地面和视觉地面/马路基本一致，避免人物脚陷进路面。
  addColliderBox(0, GROUND_SURFACE_Y - 0.05, 0, GROUND_RADIUS * 2, 0.1, GROUND_RADIUS * 2)
  createCircularBoundaryColliders()

  const ring = new THREE.Mesh(
    new THREE.TorusGeometry(GROUND_RADIUS + 0.03, 0.035, 12, 180),
    new THREE.MeshBasicMaterial({ color: 0x2cc7ff, transparent: true, opacity: 0.85 }),
  )
  ring.rotation.x = Math.PI / 2
  ring.position.y = GROUND_SURFACE_Y + 0.018
  rootGroup.add(ring)

  // v22：改为 3×3 城市群，道路和建筑由 createCityScene 生成。
}

function createGrassTexture() {
  const canvas = document.createElement('canvas')
  canvas.width = 512
  canvas.height = 512
  const ctx = canvas.getContext('2d')!
  ctx.fillStyle = '#5da94e'
  ctx.fillRect(0, 0, canvas.width, canvas.height)

  for (let i = 0; i < 3200; i++) {
    const x = Math.random() * canvas.width
    const y = Math.random() * canvas.height
    const len = 3 + Math.random() * 9
    const alpha = 0.06 + Math.random() * 0.14
    ctx.strokeStyle = Math.random() > 0.55 ? `rgba(210,255,186,${alpha})` : `rgba(33,99,38,${alpha})`
    ctx.lineWidth = 1
    ctx.beginPath()
    ctx.moveTo(x, y)
    ctx.lineTo(x + Math.random() * 6 - 3, y + len)
    ctx.stroke()
  }

  for (let i = 0; i < 18; i++) {
    ctx.fillStyle = `rgba(255,255,255,${0.025 + Math.random() * 0.04})`
    ctx.beginPath()
    ctx.ellipse(
      Math.random() * canvas.width,
      Math.random() * canvas.height,
      20 + Math.random() * 42,
      8 + Math.random() * 20,
      Math.random() * Math.PI,
      0,
      Math.PI * 2,
    )
    ctx.fill()
  }

  const texture = new THREE.CanvasTexture(canvas)
  texture.colorSpace = THREE.SRGBColorSpace
  return texture
}

function createCityScene() {
  shadowCasters.length = 0
  streetLightItems.length = 0
  windowLightItems.length = 0
  trafficLightItems.length = 0
  cityRoadMaterials.length = 0
  cityClockItems.length = 0

  // 3×3 城市群：两横两纵道路，把圆形地面划分为 9 个街区。
  createCityRoadNetwork()
  createCityBlocks()
  createRoadsideTreeBelts()
  createCityTimeElements()
  createCityObservationPoint()
}

function createCityRoadNetwork() {
  const roadMat = new THREE.MeshStandardMaterial({ color: 0x3d4651, roughness: 0.78, metalness: 0.04 })
  cityRoadMaterials.push(roadMat)
  const sidewalkMat = new THREE.MeshStandardMaterial({ color: 0xaeb8c2, roughness: 0.82, metalness: 0.02 })
  const dividerMat = new THREE.MeshBasicMaterial({ color: 0xf8fafc, transparent: true, opacity: 0.68 })
  const roadWidth = 0.62
  const roadLength = 11.2
  const roadOffsets = [-1.95, 1.95]

  function addRoad(x: number, z: number, width: number, length: number, vertical: boolean) {
    const road = new THREE.Mesh(new THREE.BoxGeometry(width, 0.026, length), roadMat)
    road.position.set(x, 0.078, z)
    if (!vertical) road.rotation.y = Math.PI / 2
    schoolGroup.add(road)

    const divider = new THREE.Mesh(new THREE.BoxGeometry(0.032, 0.008, length * 0.94), dividerMat)
    divider.position.set(x, 0.098, z)
    if (!vertical) divider.rotation.y = Math.PI / 2
    schoolGroup.add(divider)

    const sideOffset = roadWidth * 0.5 + 0.15
    ;[-sideOffset, sideOffset].forEach(offset => {
      const walk = new THREE.Mesh(new THREE.BoxGeometry(0.16, 0.018, length), sidewalkMat)
      walk.position.set(vertical ? x + offset : x, 0.092, vertical ? z : z + offset)
      if (!vertical) walk.rotation.y = Math.PI / 2
      schoolGroup.add(walk)
    })
  }

  roadOffsets.forEach(x => addRoad(x, 0, roadWidth, roadLength, true))
  roadOffsets.forEach(z => addRoad(0, z, roadWidth, roadLength, false))

  const crosswalkMat = new THREE.MeshBasicMaterial({ color: 0xffffff, transparent: true, opacity: 0.72 })
  roadOffsets.forEach(x => {
    roadOffsets.forEach(z => {
      for (let i = -2; i <= 2; i++) {
        const a = new THREE.Mesh(new THREE.BoxGeometry(0.055, 0.006, 0.68), crosswalkMat)
        a.position.set(x + i * 0.1, 0.112, z - 0.45)
        schoolGroup.add(a)
        const b = new THREE.Mesh(new THREE.BoxGeometry(0.68, 0.006, 0.055), crosswalkMat)
        b.position.set(x - 0.45, 0.113, z + i * 0.1)
        schoolGroup.add(b)
      }
    })
  })
}

type CityBuildingOpts = {
  x: number
  z: number
  width: number
  depth: number
  height: number
  color: number
  roof?: number
  floors?: number
}

const buildingsInBlock = 8

function createCityBlocks() {
  const blockCenters = [-3.75, 0, 3.75]
  const palettes = [0x8ecae6, 0xffb703, 0xfb8500, 0xbde0fe, 0xcdb4db, 0xa7c957, 0xffafcc, 0x90dbf4, 0xfed9b7, 0x98f5e1, 0xf4a261, 0xa5b4fc]
  const modernTowerBlocks = new Set([4, 5, 7])
  let blockIndex = 0

  for (const z of blockCenters) {
    for (const x of blockCenters) {
      for (let i = 0; i < buildingsInBlock; i++) {
        // 每个街区固定 10 幢：按 5×2 排布，并适当拉大间距，让街区更疏朗一点。
        // 每个街区固定 15 幢建筑：3x5 排布
        const col = i % 5
        const row = Math.floor(i / 5)
        const jitterX = (col - 2) * 0.27 + ((blockIndex + i) % 2 ? 0.03 : -0.03)
        const jitterZ = (row - 1) * 0.48 + (((blockIndex + i) % 3) - 1) * 0.04
        const width = 0.16 + ((blockIndex + i) % 3) * 0.03
        const depth = 0.16 + ((blockIndex + i + 1) % 3) * 0.025
        const height = 0.36 + ((blockIndex * 2 + i) % 6) * 0.14
        createCartoonCityBuilding({
          x: x + jitterX,
          z: z + jitterZ,
          width,
          depth,
          height,
          color: palettes[(blockIndex + i) % palettes.length]!,
          roof: [0x1f2937, 0x334155, 0x475569, 0x7c2d12][(blockIndex + i) % 4],
          floors: Math.max(2, Math.round(height / 0.25)),
        })
      }
      if (modernTowerBlocks.has(blockIndex)) {
        createModernCityTower({
          x: x + (blockIndex % 2 === 0 ? 0.52 : -0.52),
          z: z + (blockIndex % 3 === 0 ? -0.44 : 0.44),
          width: 0.5 + (blockIndex % 2) * 0.08,
          depth: 0.5 + (blockIndex % 3) * 0.04,
          height: 1.85 + (blockIndex % 3) * 0.24,
          color: 0x7dd3fc,
          glassColor: [0x0f5f8f, 0x1d4ed8, 0x3730a3, 0x0f766e][blockIndex % 4],
          accentColor: [0x0f172a, 0x1e293b, 0x172554, 0x312e81][blockIndex % 4],
          floors: 8 + (blockIndex % 3) * 2,
          spire: blockIndex === 4,
        })
      }

      if (blockIndex % 2 === 0) createPocketPark(x + 0.58, z - 0.58, 0.3)
      else createSmallPlaza(x - 0.58, z + 0.58)
      blockIndex += 1
    }
  }
}

function createModernCityTower(opts: CityBuildingOpts & { glassColor?: number; accentColor?: number; spire?: boolean }) {
  const floors = opts.floors ?? Math.max(9, Math.round(opts.height / 0.17))

  // 主体改成不透明的深色玻璃幕墙，不再像透明盒子。
  const bodyMat = new THREE.MeshStandardMaterial({
    color: opts.glassColor ?? 0x1d4ed8,
    roughness: 0.18,
    metalness: 0.42,
    transparent: false,
    opacity: 1,
    emissive: new THREE.Color(0x061525),
    emissiveIntensity: 0.12,
  })

  const body = new THREE.Mesh(new THREE.BoxGeometry(opts.width, opts.height, opts.depth), bodyMat)
  body.position.set(opts.x, opts.height / 2 + 0.08, opts.z)
  schoolGroup.add(body)

  const accentMat = new THREE.MeshStandardMaterial({
    color: opts.accentColor ?? 0x0f172a,
    roughness: 0.28,
    metalness: 0.58,
  })

  // 四角金属竖边，让楼更像现代玻璃塔楼。
  const edgeSize = 0.026
  const cornerList = [
    [-opts.width / 2, -opts.depth / 2],
    [opts.width / 2, -opts.depth / 2],
    [-opts.width / 2, opts.depth / 2],
    [opts.width / 2, opts.depth / 2],
  ]
  cornerList.forEach(([cx, cz]) => {
    const edge = new THREE.Mesh(new THREE.BoxGeometry(edgeSize, opts.height + 0.04, edgeSize), accentMat)
    edge.position.set(opts.x + cx!, opts.height / 2 + 0.1, opts.z + cz!)
    schoolGroup.add(edge)
  })

  // 楼顶退台和设备层。
  const roof = new THREE.Mesh(new THREE.BoxGeometry(opts.width * 1.08, 0.07, opts.depth * 1.08), accentMat)
  roof.position.set(opts.x, opts.height + 0.14, opts.z)
  schoolGroup.add(roof)

  const crown = new THREE.Mesh(
    new THREE.BoxGeometry(opts.width * 0.68, 0.16, opts.depth * 0.62),
    new THREE.MeshStandardMaterial({ color: 0x111827, roughness: 0.24, metalness: 0.52 }),
  )
  crown.position.set(opts.x, opts.height + 0.255, opts.z)
  schoolGroup.add(crown)

  if (opts.spire) {
    const spire = new THREE.Mesh(
      new THREE.ConeGeometry(opts.width * 0.16, 0.42, 5),
      new THREE.MeshStandardMaterial({ color: 0xfde68a, roughness: 0.26, metalness: 0.62 }),
    )
    spire.position.set(opts.x, opts.height + 0.57, opts.z)
    schoolGroup.add(spire)
  }

  // 发光窗户：按层和列排布，白天偏蓝，夜晚会被 updateCityLights 控制发光。
  const winMat = new THREE.MeshBasicMaterial({
    color: 0xbff4ff,
    transparent: true,
    opacity: 0.86,
    side: THREE.DoubleSide,
  })
  windowLightItems.push({ material: winMat, seed: opts.x * 11.7 + opts.z * 5.3 + opts.height })

  const floorStep = opts.height / floors
  for (let r = 1; r < floors; r++) {
    const y = 0.14 + r * floorStep

    for (let c = -1; c <= 1; c++) {
      const front = new THREE.Mesh(new THREE.PlaneGeometry(opts.width * 0.16, Math.min(0.052, floorStep * 0.38)), winMat)
      front.position.set(opts.x + c * opts.width * 0.24, y, opts.z - opts.depth / 2 - 0.009)
      front.rotation.y = Math.PI
      schoolGroup.add(front)

      const back = front.clone()
      back.position.z = opts.z + opts.depth / 2 + 0.009
      back.rotation.y = 0
      schoolGroup.add(back)
    }

    for (let c = -1; c <= 1; c++) {
      const side = new THREE.Mesh(new THREE.PlaneGeometry(opts.depth * 0.15, Math.min(0.05, floorStep * 0.36)), winMat)
      side.position.set(opts.x + opts.width / 2 + 0.009, y, opts.z + c * opts.depth * 0.23)
      side.rotation.y = Math.PI / 2
      schoolGroup.add(side)

      const side2 = side.clone()
      side2.position.x = opts.x - opts.width / 2 - 0.009
      side2.rotation.y = -Math.PI / 2
      schoolGroup.add(side2)
    }
  }

  // 底部裙房和入口，增强现代大楼识别度。
  const podium = new THREE.Mesh(
    new THREE.BoxGeometry(opts.width * 1.18, 0.24, opts.depth * 1.16),
    new THREE.MeshStandardMaterial({ color: 0x111827, roughness: 0.3, metalness: 0.36 }),
  )
  podium.position.set(opts.x, 0.2, opts.z)
  schoolGroup.add(podium)

  const lobby = new THREE.Mesh(
    new THREE.BoxGeometry(opts.width * 0.58, 0.18, 0.035),
    new THREE.MeshBasicMaterial({ color: 0xfde68a, transparent: true, opacity: 0.9 }),
  )
  lobby.position.set(opts.x, 0.28, opts.z - opts.depth / 2 - 0.025)
  lobby.rotation.y = Math.PI
  schoolGroup.add(lobby)

  addShadowCaster(opts.x, opts.z, opts.width * 1.18, opts.depth * 1.16, opts.height)
}

function createCartoonCityBuilding(opts: CityBuildingOpts) {
  const floors = opts.floors ?? Math.max(2, Math.round(opts.height / 0.3))
  const body = new THREE.Mesh(new THREE.BoxGeometry(opts.width, opts.height, opts.depth), createCartoonFacadeMaterial(opts.color, floors))
  body.position.set(opts.x, opts.height / 2 + 0.08, opts.z)
  schoolGroup.add(body)

  const roof = new THREE.Mesh(
    new THREE.BoxGeometry(opts.width * 1.08, 0.09, opts.depth * 1.08),
    new THREE.MeshStandardMaterial({ color: opts.roof ?? 0x334155, roughness: 0.55, metalness: 0.08 }),
  )
  roof.position.set(opts.x, opts.height + 0.15, opts.z)
  schoolGroup.add(roof)

  const antenna = new THREE.Mesh(
    new THREE.CylinderGeometry(0.012, 0.012, 0.24, 8),
    new THREE.MeshStandardMaterial({ color: 0xdbeafe, roughness: 0.42, metalness: 0.28 }),
  )
  antenna.position.set(opts.x + opts.width * 0.23, opts.height + 0.31, opts.z - opts.depth * 0.15)
  schoolGroup.add(antenna)

  const frontZ = opts.z - opts.depth / 2 - 0.006
  for (let r = 0; r < floors; r++) {
    for (let c = -1; c <= 1; c++) {
      const winMat = new THREE.MeshBasicMaterial({ color: 0xdff7ff, transparent: true, opacity: 0.78, side: THREE.DoubleSide })
      windowLightItems.push({ material: winMat, seed: opts.x * 13.17 + opts.z * 7.31 + r * 0.37 + c * 0.19 })
      const win = new THREE.Mesh(new THREE.PlaneGeometry(opts.width * 0.16, Math.min(0.09, (opts.height / floors) * 0.34)), winMat)
      win.position.set(opts.x + c * opts.width * 0.24, 0.22 + r * (opts.height / floors), frontZ)
      win.rotation.y = Math.PI
      schoolGroup.add(win)
    }
  }
  addShadowCaster(opts.x, opts.z, opts.width, opts.depth, opts.height)
}

function createCartoonFacadeMaterial(baseColor: number, floors: number) {
  const canvas = document.createElement('canvas')
  canvas.width = 512
  canvas.height = 512
  const ctx = canvas.getContext('2d')!
  const base = new THREE.Color(baseColor)
  const gradient = ctx.createLinearGradient(0, 0, 0, 512)
  gradient.addColorStop(
    0,
    `rgb(${Math.min(255, Math.round(base.r * 255 + 28))}, ${Math.min(255, Math.round(base.g * 255 + 28))}, ${Math.min(255, Math.round(base.b * 255 + 28))})`,
  )
  gradient.addColorStop(1, `rgb(${Math.round(base.r * 210)}, ${Math.round(base.g * 210)}, ${Math.round(base.b * 210)})`)
  ctx.fillStyle = gradient
  ctx.fillRect(0, 0, 512, 512)

  ctx.strokeStyle = 'rgba(255,255,255,0.16)'
  ctx.lineWidth = 3
  for (let y = 0; y <= 512; y += Math.max(54, 512 / floors)) {
    ctx.beginPath()
    ctx.moveTo(0, y)
    ctx.lineTo(512, y)
    ctx.stroke()
  }

  const rows = Math.max(2, floors)
  for (let r = 0; r < rows; r++) {
    for (let c = 0; c < 3; c++) {
      const x = 94 + c * 124
      const y = 58 + r * (390 / rows)
      const w = 62
      const h = Math.min(48, 240 / rows)
      ctx.fillStyle = r % 2 === 0 ? 'rgba(215,247,255,0.92)' : 'rgba(255,240,180,0.86)'
      roundRect(ctx, x, y, w, h, 8)
      ctx.fill()
      ctx.strokeStyle = 'rgba(15,23,42,0.28)'
      ctx.stroke()
      ctx.strokeStyle = 'rgba(255,255,255,0.42)'
      ctx.beginPath()
      ctx.moveTo(x + 8, y + 8)
      ctx.lineTo(x + w - 8, y + h - 8)
      ctx.stroke()
    }
  }

  const texture = new THREE.CanvasTexture(canvas)
  texture.colorSpace = THREE.SRGBColorSpace
  return new THREE.MeshStandardMaterial({ map: texture, roughness: 0.58, metalness: 0.03 })
}

function roundRect(ctx: CanvasRenderingContext2D, x: number, y: number, w: number, h: number, r: number) {
  ctx.beginPath()
  ctx.moveTo(x + r, y)
  ctx.arcTo(x + w, y, x + w, y + h, r)
  ctx.arcTo(x + w, y + h, x, y + h, r)
  ctx.arcTo(x, y + h, x, y, r)
  ctx.arcTo(x, y, x + w, y, r)
  ctx.closePath()
}

function createPocketPark(x: number, z: number, radius = 0.34) {
  const park = new THREE.Mesh(new THREE.CircleGeometry(radius, 32), new THREE.MeshStandardMaterial({ color: 0x6fbf73, roughness: 0.86 }))
  park.rotation.x = -Math.PI / 2
  park.position.set(x, 0.096, z)
  schoolGroup.add(park)
  createTree(x - radius * 0.35, z, 0.28)
  createTree(x + radius * 0.28, z + radius * 0.16, 0.24)
}

function createSmallPlaza(x: number, z: number) {
  const plaza = new THREE.Mesh(new THREE.BoxGeometry(0.56, 0.018, 0.42), new THREE.MeshStandardMaterial({ color: 0xcbd5e1, roughness: 0.82 }))
  plaza.position.set(x, 0.096, z)
  schoolGroup.add(plaza)
  const deco = new THREE.Mesh(
    new THREE.CylinderGeometry(0.09, 0.09, 0.04, 20),
    new THREE.MeshStandardMaterial({ color: 0x38bdf8, roughness: 0.38, metalness: 0.1 }),
  )
  deco.position.set(x, 0.13, z)
  schoolGroup.add(deco)
}

function createRoadsideTreeBelts() {
  const roadOffsets = [-1.95, 1.95]
  const treeOffset = 0.52
  const positions: Array<[number, number]> = []

  for (const roadOffset of roadOffsets) {
    for (let t = -5.25; t <= 5.25; t += 1.25) {
      if (roadOffsets.some(offset => Math.abs(t - offset) < 0.42)) continue
      positions.push([roadOffset - treeOffset, t], [roadOffset + treeOffset, t])
      positions.push([t, roadOffset - treeOffset], [t, roadOffset + treeOffset])
    }
  }

  positions.forEach(([x, z], i) => {
    if (Math.hypot(x, z) > GROUND_RADIUS - 0.55) return
    createTree(x, z, 0.21 + (i % 3) * 0.032)
  })
}

function createCityTimeElements() {
  // 路灯：沿两横两纵主干道布置，白天熄灭，夜晚渐亮。
  const roadOffsets = [-1.95, 1.95]
  const lampOffset = 0.52

  for (const roadOffset of roadOffsets) {
    for (let t = -5.25; t <= 5.25; t += 1.38) {
      if (roadOffsets.some(offset => Math.abs(t - offset) < 0.4)) continue
      createStreetLamp(roadOffset - lampOffset, t, 0)
      createStreetLamp(roadOffset + lampOffset, t, Math.PI)
      createStreetLamp(t, roadOffset - lampOffset, Math.PI / 2)
      createStreetLamp(t, roadOffset + lampOffset, -Math.PI / 2)
    }
  }

  // 十字路口红绿灯：3×3 街区共有 2×2 个主要路口。
  roadOffsets.forEach(x => {
    roadOffsets.forEach(z => createTrafficLight(x + 0.31, z + 0.31, (x + z) * 0.17))
  })

  // 城市时间牌：放在城市中心上空，高于楼群，像城市中心的数字广告牌。
  createCityClockBillboard(0, 0)
}

function createStreetLamp(x: number, z: number, rotation = 0) {
  if (Math.hypot(x, z) > GROUND_RADIUS - 0.45) return
  const group = new THREE.Group()
  group.position.set(x, 0.09, z)
  group.rotation.y = rotation

  const poleMat = new THREE.MeshStandardMaterial({ color: 0x334155, roughness: 0.48, metalness: 0.22 })
  const pole = new THREE.Mesh(new THREE.CylinderGeometry(0.018, 0.024, 0.72, 10), poleMat)
  pole.position.y = 0.36
  group.add(pole)

  const arm = new THREE.Mesh(new THREE.BoxGeometry(0.26, 0.022, 0.022), poleMat)
  arm.position.set(0.12, 0.7, 0)
  group.add(arm)

  const lampMat = new THREE.MeshBasicMaterial({ color: 0xfff3b0 })
  const lamp = new THREE.Mesh(new THREE.SphereGeometry(0.055, 14, 10), lampMat)
  lamp.position.set(0.26, 0.69, 0)
  group.add(lamp)

  const glow = new THREE.Sprite(
    new THREE.SpriteMaterial({ map: createGlowTexture(), color: 0xffd36a, transparent: true, opacity: 0, depthWrite: false }),
  )
  glow.position.set(0.26, 0.69, 0)
  glow.scale.set(0.55, 0.55, 1)
  group.add(glow)

  const coneMaterial = new THREE.MeshBasicMaterial({
    color: 0xffd36a,
    transparent: true,
    opacity: 0,
    side: THREE.DoubleSide,
    depthWrite: false,
  })
  const cone = new THREE.Mesh(new THREE.ConeGeometry(0.42, 0.9, 32, 1, true), coneMaterial)
  cone.position.set(0.26, 0.23, 0)
  // ConeGeometry 默认沿 Y 轴生成，不再额外旋转，避免光锥方向被翻转。
  group.add(cone)

  schoolGroup.add(group)

  // 玩家碰撞体：路灯杆是细高物体，视觉上能撞到，必须给简化碰撞盒，避免人物穿模。
  addColliderBox(x, GROUND_SURFACE_Y + 0.42, z, 0.16, 0.84, 0.16)

  // 灯臂朝向随 rotation 改变，再给灯头位置一个小碰撞盒，避免第三人称贴近时穿过灯头。
  const lampLocal = new THREE.Vector3(0.25, 0, 0).applyAxisAngle(new THREE.Vector3(0, 1, 0), rotation)
  addColliderBox(x + lampLocal.x, GROUND_SURFACE_Y + 0.7, z + lampLocal.z, 0.18, 0.18, 0.18)

  streetLightItems.push({ pole, lamp, glow, cone })
}

function createTrafficLight(x: number, z: number, seed = 0) {
  const group = new THREE.Group()
  group.position.set(x, 0.1, z)
  const pole = new THREE.Mesh(
    new THREE.CylinderGeometry(0.016, 0.018, 0.46, 8),
    new THREE.MeshStandardMaterial({ color: 0x1f2937, metalness: 0.15, roughness: 0.52 }),
  )
  pole.position.y = 0.23
  group.add(pole)
  const box = new THREE.Mesh(
    new THREE.BoxGeometry(0.12, 0.24, 0.075),
    new THREE.MeshStandardMaterial({ color: 0x0f172a, roughness: 0.45, metalness: 0.1 }),
  )
  box.position.y = 0.52
  group.add(box)
  const red = new THREE.MeshBasicMaterial({ color: 0x6b1111 })
  const yellow = new THREE.MeshBasicMaterial({ color: 0x6b5a11 })
  const green = new THREE.MeshBasicMaterial({ color: 0x105c2d })
  ;[
    [red, 0.59],
    [yellow, 0.52],
    [green, 0.45],
  ].forEach(([mat, y]) => {
    const light = new THREE.Mesh(new THREE.SphereGeometry(0.018, 10, 8), mat as THREE.MeshBasicMaterial)
    light.position.set(0, y as number, -0.04)
    group.add(light)
  })
  schoolGroup.add(group)
  // 玩家碰撞体：红绿灯也在路口附近，防止角色直接穿过灯杆。
  addColliderBox(x, GROUND_SURFACE_Y + 0.34, z, 0.16, 0.68, 0.16)
  trafficLightItems.push({ red, yellow, green, seed })
}

function createAdBillboardTexture() {
  const canvas = document.createElement('canvas')
  canvas.width = 768
  canvas.height = 256
  const ctx = canvas.getContext('2d')!

  const bg = ctx.createLinearGradient(0, 0, 768, 256)
  bg.addColorStop(0, '#111827')
  bg.addColorStop(0.52, '#0f172a')
  bg.addColorStop(1, '#1e1b4b')
  ctx.fillStyle = bg
  roundRect(ctx, 16, 12, 736, 232, 28)
  ctx.fill()

  ctx.strokeStyle = 'rgba(255, 209, 102, 0.82)'
  ctx.lineWidth = 5
  ctx.stroke()

  ctx.fillStyle = 'rgba(255,255,255,0.70)'
  ctx.font = '700 22px Microsoft YaHei, Arial'
  ctx.textAlign = 'center'
  ctx.fillText(BILLBOARD_BACK_CONFIG.eyebrow, 384, 58)

  ctx.fillStyle = '#ffd166'
  ctx.font = '900 54px Microsoft YaHei, Arial'
  ctx.fillText(BILLBOARD_BACK_CONFIG.title, 384, 130)

  ctx.fillStyle = 'rgba(224,242,254,0.88)'
  ctx.font = '700 23px Microsoft YaHei, Arial'
  ctx.fillText(BILLBOARD_BACK_CONFIG.subtitle, 384, 176)

  ctx.fillStyle = 'rgba(255,255,255,0.55)'
  ctx.font = '600 18px Microsoft YaHei, Arial'
  ctx.fillText(BILLBOARD_BACK_CONFIG.footer, 384, 214)

  const texture = new THREE.CanvasTexture(canvas)
  texture.colorSpace = THREE.SRGBColorSpace
  texture.needsUpdate = true
  return texture
}

function createCityClockBillboard(x: number, z: number) {
  const canvas = document.createElement('canvas')
  canvas.width = 768
  canvas.height = 256
  const ctx = canvas.getContext('2d')!
  const texture = new THREE.CanvasTexture(canvas)
  texture.colorSpace = THREE.SRGBColorSpace

  const group = new THREE.Group()
  group.position.set(x, 0, z)

  // 放在城市中心上空，比楼群更高，避免被建筑挡住。
  const boardY = 3.08
  const boardWidth = 2.25
  const boardHeight = 0.86

  const material = new THREE.MeshBasicMaterial({ map: texture, transparent: true, side: THREE.FrontSide, opacity: 0.96 })
  const board = new THREE.Mesh(new THREE.PlaneGeometry(boardWidth, boardHeight), material)
  board.position.set(0, boardY, 0.006)
  // 正面：城市太阳时钟，面向初始相机方向。
  board.rotation.y = 0

  // 背面：小红书账号广告牌。单独用一张贴图，避免动态时钟刷新影响广告内容。
  const adTexture = createAdBillboardTexture()
  const adMaterial = new THREE.MeshBasicMaterial({ map: adTexture, transparent: true, side: THREE.FrontSide, opacity: 0.96 })
  const adBoard = new THREE.Mesh(new THREE.PlaneGeometry(boardWidth, boardHeight), adMaterial)
  adBoard.position.set(0, boardY, -0.006)
  adBoard.rotation.y = Math.PI

  group.add(board, adBoard)

  const frameMat = new THREE.MeshStandardMaterial({ color: 0x0f172a, roughness: 0.36, metalness: 0.32 })
  const frameTop = new THREE.Mesh(new THREE.BoxGeometry(boardWidth + 0.1, 0.035, 0.035), frameMat)
  frameTop.position.set(0, boardY + boardHeight / 2 + 0.025, -0.004)
  const frameBottom = frameTop.clone()
  frameBottom.position.y = boardY - boardHeight / 2 - 0.025
  const frameLeft = new THREE.Mesh(new THREE.BoxGeometry(0.035, boardHeight + 0.1, 0.035), frameMat)
  frameLeft.position.set(-boardWidth / 2 - 0.032, boardY, -0.004)
  const frameRight = frameLeft.clone()
  frameRight.position.x = boardWidth / 2 + 0.032
  group.add(frameTop, frameBottom, frameLeft, frameRight)

  const poleMat = new THREE.MeshStandardMaterial({ color: 0x334155, roughness: 0.48, metalness: 0.2 })
  const poleHeight = boardY - 0.38
  const pole = new THREE.Mesh(new THREE.CylinderGeometry(0.024, 0.032, poleHeight, 12), poleMat)
  pole.position.set(0, poleHeight / 2 + 0.1, 0)
  group.add(pole)

  const base = new THREE.Mesh(
    new THREE.CylinderGeometry(0.18, 0.24, 0.08, 28),
    new THREE.MeshStandardMaterial({ color: 0x1e293b, roughness: 0.62, metalness: 0.08 }),
  )
  base.position.set(0, 0.14, 0)
  group.add(base)

  // 科技感微光底座，不参与碰撞，只作为城市中心视觉焦点。
  const glow = new THREE.Mesh(
    new THREE.TorusGeometry(0.36, 0.012, 8, 72),
    new THREE.MeshBasicMaterial({ color: 0x38bdf8, transparent: true, opacity: 0.58, depthWrite: false }),
  )
  glow.rotation.x = Math.PI / 2
  glow.position.set(0, 0.2, 0)
  group.add(glow)

  schoolGroup.add(group)
  cityClockItems.push({ texture, ctx, material, group })
}

function createCityObservationPoint() {
  const marker = new THREE.Mesh(
    new THREE.CylinderGeometry(0.08, 0.08, 0.035, 24),
    new THREE.MeshBasicMaterial({ color: 0xffd166, transparent: true, opacity: 0.92 }),
  )
  marker.position.set(OBSERVER_POINT.x, 0.115, OBSERVER_POINT.z)
  schoolGroup.add(marker)

  const ring = new THREE.Mesh(
    new THREE.TorusGeometry(0.16, 0.008, 8, 48),
    new THREE.MeshBasicMaterial({ color: 0xffffff, transparent: true, opacity: 0.82 }),
  )
  ring.rotation.x = Math.PI / 2
  ring.position.set(OBSERVER_POINT.x, 0.13, OBSERVER_POINT.z)
  schoolGroup.add(ring)
  schoolGroup.add(createSpriteText('城市街道观测点', '#ffffff', OBSERVER_POINT.clone().add(new THREE.Vector3(0, 0.5, 0)), 0.17))
}

function createTree(x: number, z: number, scale = 0.46) {
  const tree = new THREE.Group()
  tree.position.set(x, GROUND_SURFACE_Y, z)

  const trunkHeight = scale * 1.12
  const trunk = new THREE.Mesh(
    new THREE.CylinderGeometry(scale * 0.045, scale * 0.07, trunkHeight, 9),
    new THREE.MeshStandardMaterial({ color: 0x7a4a2a, roughness: 0.86 }),
  )
  trunk.position.y = trunkHeight / 2
  tree.add(trunk)

  // 树枝：斜向伸出，不再是“一根棍子”。
  const branchMaterial = new THREE.MeshStandardMaterial({ color: 0x68401f, roughness: 0.82 })
  const branchAngles = [0, Math.PI * 0.46, Math.PI * 0.92, Math.PI * 1.38]
  branchAngles.forEach((angle, i) => {
    const branch = new THREE.Mesh(new THREE.CylinderGeometry(scale * 0.015, scale * 0.026, scale * 0.42, 7), branchMaterial)
    branch.position.set(Math.sin(angle) * scale * 0.12, trunkHeight * (0.66 + (i % 2) * 0.08), Math.cos(angle) * scale * 0.12)
    branch.rotation.z = Math.sin(angle) * 0.48
    branch.rotation.x = Math.cos(angle) * 0.48
    tree.add(branch)
  })

  const leafColors = [0x4f9f4c, 0x65b95b, 0x7fca68, 0x3f8f47]
  const leafMaterial = (color: number) => new THREE.MeshStandardMaterial({ color, roughness: 0.72 })

  const clusters = [
    { x: 0, y: trunkHeight + scale * 0.16, z: 0, r: scale * 0.34, color: leafColors[1] },
    { x: -scale * 0.18, y: trunkHeight + scale * 0.04, z: scale * 0.04, r: scale * 0.26, color: leafColors[0] },
    { x: scale * 0.18, y: trunkHeight + scale * 0.06, z: -scale * 0.02, r: scale * 0.27, color: leafColors[2] },
    { x: scale * 0.02, y: trunkHeight + scale * 0.3, z: -scale * 0.12, r: scale * 0.23, color: leafColors[3] },
  ]

  clusters.forEach((c, i) => {
    const crown = new THREE.Mesh(new THREE.DodecahedronGeometry(c.r, 1), leafMaterial(c.color!))
    crown.position.set(c.x, c.y, c.z)
    crown.rotation.set(i * 0.25, i * 0.45, i * 0.18)
    tree.add(crown)
  })

  // 树下小阴影底盘，增强真实感。
  const base = new THREE.Mesh(
    new THREE.CircleGeometry(scale * 0.42, 24),
    new THREE.MeshBasicMaterial({ color: 0x1f3b22, transparent: true, opacity: 0.22, depthWrite: false }),
  )
  base.rotation.x = -Math.PI / 2
  base.position.y = 0.006
  tree.add(base)

  schoolGroup.add(tree)
  addShadowCaster(x, z, scale * 0.62, scale * 0.62, trunkHeight + scale * 0.42)
}

function addShadowCaster(x: number, z: number, width: number, depth: number, height: number, target?: THREE.Object3D) {
  const geometry = new THREE.BufferGeometry()
  const mesh = new THREE.Mesh(
    geometry,
    new THREE.MeshBasicMaterial({ color: 0x061019, transparent: true, opacity: 0.36, side: THREE.DoubleSide, depthWrite: false }),
  )
  mesh.position.y = GROUND_SURFACE_Y + 0.004
  shadowGroup.add(mesh)
  shadowCasters.push({ x, z, width, depth, height, mesh, target })

  // 玩家模式碰撞体：用简化 BoxCollider 包住建筑，避免角色穿楼。
  addColliderBox(x, height / 2 + 0.08, z, width, height + 0.16, depth)
}

function createDome() {
  hemisphereDome = new THREE.Mesh(
    new THREE.SphereGeometry(SKY_RADIUS, 96, 32, 0, Math.PI * 2, 0, Math.PI / 2),
    new THREE.MeshBasicMaterial({ color: 0x72d8ff, transparent: true, opacity: 0.06, side: THREE.BackSide, depthWrite: false }),
  )
  domeGroup.add(hemisphereDome)

  for (const alt of [15, 30, 45, 60, 75]) {
    const y = SKY_RADIUS * Math.sin(degToRad(alt))
    const r = SKY_RADIUS * Math.cos(degToRad(alt))
    const ring = new THREE.Mesh(
      new THREE.TorusGeometry(r, 0.007, 8, 160),
      new THREE.MeshBasicMaterial({ color: 0x6ee7ff, transparent: true, opacity: alt % 30 === 0 ? 0.42 : 0.23 }),
    )
    ring.rotation.x = Math.PI / 2
    ring.position.y = y
    domeGroup.add(ring)
  }

  for (let az = 0; az < 360; az += 15) {
    const p = azimuthAltitudeToVec3(az, 0, SKY_RADIUS)
    const line = makeLine(
      [new THREE.Vector3(0, 0.012, 0), new THREE.Vector3(p.x, 0.012, p.z)],
      az % 90 === 0 ? 0xffd166 : 0x38779b,
      az % 90 === 0 ? 0.55 : 0.18,
    )
    domeGroup.add(line)
  }
}

function createLabels() {
  const labels = [
    { text: '北 N', az: 0, color: '#9defff' },
    { text: '东 E', az: 90, color: '#ffe08a' },
    { text: '南 S', az: 180, color: '#9defff' },
    { text: '西 W', az: 270, color: '#ffe08a' },
  ]
  labels.forEach(item => {
    const p = azimuthAltitudeToVec3(item.az, 0, GROUND_RADIUS + 0.38)
    // 方位标签统一 15 号字，避免在场景里过大抢画面。
    labelGroup.add(createLabelSpriteText(item.text, item.color, new THREE.Vector3(p.x, 0.18, p.z), 15, 1))
  })
  labelGroup.add(createLabelSpriteText('天顶', '#ffffff', new THREE.Vector3(0, SKY_RADIUS + 0.28, 0), 15, 1))
}

function createSun() {
  sunMesh = new THREE.Mesh(new THREE.SphereGeometry(0.22, 32, 24), new THREE.MeshBasicMaterial({ color: 0xffd166 }))
  sunGroup.add(sunMesh)

  const spriteMaterial = new THREE.SpriteMaterial({ map: createGlowTexture(), color: 0xffd166, transparent: true, opacity: 0.95, depthWrite: false })
  sunGlow = new THREE.Sprite(spriteMaterial)
  sunGlow.scale.set(1.7, 1.7, 1)
  sunGroup.add(sunGlow)
}

function createGlowTexture() {
  const canvas = document.createElement('canvas')
  canvas.width = 128
  canvas.height = 128
  const ctx = canvas.getContext('2d')!
  const gradient = ctx.createRadialGradient(64, 64, 0, 64, 64, 64)
  gradient.addColorStop(0, 'rgba(255,255,255,1)')
  gradient.addColorStop(0.25, 'rgba(255,219,103,0.9)')
  gradient.addColorStop(0.55, 'rgba(255,177,46,0.35)')
  gradient.addColorStop(1, 'rgba(255,177,46,0)')
  ctx.fillStyle = gradient
  ctx.fillRect(0, 0, 128, 128)
  const texture = new THREE.CanvasTexture(canvas)
  texture.colorSpace = THREE.SRGBColorSpace
  return texture
}

function createLightRay() {
  lightRay = makeLine([new THREE.Vector3(), new THREE.Vector3()], 0xffe082, 0.68)
  rayGroup.add(lightRay)
}

function createNightSkyDecorations() {
  if (!skyDecorationGroup) return
  clearGroup(skyDecorationGroup)

  const starCount = 190
  const positions: number[] = []
  const colors: number[] = []
  for (let i = 0; i < starCount; i++) {
    const az = Math.random() * 360
    const alt = 8 + Math.random() * 76
    const p = azimuthAltitudeToVec3(az, alt, SKY_RADIUS + 1.15 + Math.random() * 1.1)
    positions.push(p.x, p.y, p.z)
    const warm = Math.random() > 0.78
    colors.push(warm ? 1 : 0.78, warm ? 0.88 : 0.92, warm ? 0.62 : 1)
  }
  const starGeo = new THREE.BufferGeometry()
  starGeo.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3))
  starGeo.setAttribute('color', new THREE.Float32BufferAttribute(colors, 3))
  const starMat = new THREE.PointsMaterial({ size: 0.034, vertexColors: true, transparent: true, opacity: 0, depthWrite: false })
  const stars = new THREE.Points(starGeo, starMat)
  stars.userData.kind = 'stars'
  skyDecorationGroup.add(stars)

  // 已按课堂学理简化：夜晚只保留星空，不再展示流星雨，避免分散太阳视运动主线。
  const meteorCount = 0
  for (let i = 0; i < meteorCount; i++) {
    const group = new THREE.Group()
    group.userData.kind = 'meteor'
    group.userData.phase = Math.random()
    group.userData.speed = 0.18 + Math.random() * 0.32
    group.userData.baseAz = Math.random() * 360
    group.userData.baseAlt = 20 + Math.random() * 56
    const start = new THREE.Vector3(-0.36, 0.14, 0)
    const end = new THREE.Vector3(0.42, -0.08, 0)
    const streak = makeLine([start, end], 0xdbeafe, 0)
    streak.userData.kind = 'meteor'
    group.add(streak)
    const head = new THREE.Sprite(
      new THREE.SpriteMaterial({ map: createGlowTexture(), color: 0xffffff, transparent: true, opacity: 0, depthWrite: false }),
    )
    head.userData.kind = 'meteor'
    head.scale.set(0.12, 0.12, 1)
    head.position.copy(end)
    group.add(head)
    group.scale.setScalar(0.6 + Math.random() * 0.55)
    skyDecorationGroup.add(group)
  }
}

function updateNightSkyDecorations(nightAlpha: number) {
  if (!skyDecorationGroup) return
  skyDecorationGroup.visible = nightAlpha > 0.02
  const time = performance.now() * 0.001

  skyDecorationGroup.children.forEach(obj => {
    if (obj.userData.kind === 'stars') {
      const material = (obj as THREE.Points).material as THREE.PointsMaterial
      material.transparent = true
      material.opacity = 0.18 + 0.78 * nightAlpha
      return
    }

    if (obj.userData.kind === 'meteor') {
      const phase = (((time * obj.userData.speed + obj.userData.phase) % 1) + 1) % 1
      const visiblePulse = phase < 0.28 ? Math.sin((phase / 0.28) * Math.PI) : 0
      const az = obj.userData.baseAz + phase * 48
      const alt = obj.userData.baseAlt - phase * 18
      obj.position.copy(azimuthAltitudeToVec3(az, alt, SKY_RADIUS + 1.35))
      obj.rotation.z = -0.42

      obj.traverse(child => {
        const material = (child as THREE.Line | THREE.Sprite).material as THREE.Material | undefined
        if (!material) return
        const mat = material as THREE.Material & { opacity?: number; transparent?: boolean }
        mat.transparent = true
        mat.opacity = nightAlpha * visiblePulse * 0.92
      })
    }
  })
}

function rebuildSolarPaths() {
  clearGroup(pathGroup)
  if (!layers.paths) return

  const pathDefs = [
    { day: state.dayOfYear, name: `${dateLabel.value}路径`, color: 0xffd166, opacity: 1, radius: 0.018 },
    { day: 172, name: '夏至路径', color: 0x3687ff, opacity: 0.78, radius: 0.011 },
    { day: 80, name: '春秋分路径', color: 0xffffff, opacity: 0.62, radius: 0.01 },
    { day: 355, name: '冬至路径', color: 0x45e8ff, opacity: 0.76, radius: 0.011 },
  ]

  pathDefs.forEach((def, index) => {
    const points = buildSunPathPoints(state.latitude, def.day)
    if (points.length < 2) return
    pathGroup.add(makeTubeLine(points, def.color, def.radius, def.opacity))

    const mid = points[Math.floor(points.length / 2)]!
    const textColor = def.color === 0xffffff ? '#ffffff' : def.color === 0xffd166 ? '#ffe28a' : def.color === 0x3687ff ? '#9fc0ff' : '#8af6ff'
    pathGroup.add(
      createSpriteText(
        def.name,
        textColor,
        mid
          .clone()
          .multiplyScalar(1.035)
          .add(new THREE.Vector3(0, index === 0 ? 0.18 : 0, 0)),
        index === 0 ? 0.28 : 0.22,
      ),
    )

    if (index === 0) {
      const currentPathMetrics = computeSolarMetrics(state.latitude, def.day, 12)
      if (!currentPathMetrics.polarType) {
        const first = points[0]!
        const last = points[points.length - 1]!
        pathGroup.add(createSmallMarker(first, 0xffd166, '日出'))
        pathGroup.add(createSmallMarker(last, 0xff8f70, '日落'))
      }
    }
  })
}

function buildSunPathPoints(latitude: number, dayOfYear: number) {
  const m = computeSolarMetrics(latitude, dayOfYear, 12)
  const points: THREE.Vector3[] = []
  if (m.polarType === '极夜') return points

  const start = m.polarType === '极昼' ? 0 : m.sunrise
  const end = m.polarType === '极昼' ? 24 : m.sunset
  const steps = 180

  for (let i = 0; i <= steps; i++) {
    const t = start + ((end - start) * i) / steps
    const metrics = computeSolarMetrics(latitude, dayOfYear, t)
    if (metrics.altitude >= -0.1 || m.polarType === '极昼') points.push(solarToPosition(metrics, SKY_RADIUS))
  }
  return points
}

function createSmallMarker(position: THREE.Vector3, color: number, text: string) {
  const group = new THREE.Group()
  const mesh = new THREE.Mesh(new THREE.SphereGeometry(0.055, 12, 8), new THREE.MeshBasicMaterial({ color, transparent: true, opacity: 0.95 }))
  mesh.position.copy(position)
  group.add(mesh)
  group.add(createSpriteText(text, color === 0xffd166 ? '#ffdc82' : '#ffad96', position.clone().add(new THREE.Vector3(0, 0.22, 0)), 0.18))
  return group
}

function updateSceneBySolar(metrics: SolarMetrics = solarMetrics.value) {
  if (!sunMesh || !sunGlow || !lightRay) return

  const sunPos = solarToPosition(metrics, SKY_RADIUS)
  const isAbove = metrics.altitude > 0
  const now = performance.now()

  sunMesh.position.copy(sunPos)
  sunGlow.position.copy(sunPos)
  sunMesh.visible = isAbove
  sunGlow.visible = isAbove
  sunGlow.material.opacity = metrics.altitude > 8 ? 0.88 : 1

  updateLightRay(sunPos, isAbove)

  // 原生阴影每帧更新：太阳方向 / 定向光方向必须跟 rAF 的 runtimeMetrics 同步，
  // 不能放在低频的 updateSkyByTime 里，否则影子会像右侧时间面板一样一跳一跳。
  syncThreeJsSunShadow(metrics)

  // v43：阴影由 Three.js 原生 shadowMap 每帧生成，旧手绘阴影不再更新。
  if (shadowGroup?.visible) shadowGroup.visible = false

  // 太阳高度角雷达扫描原来每帧 clearGroup + 重建几何和文字，比较费。
  // 这里单独限频，保留扫描感，同时避免大量 dispose / new。
  const gaugeInterval = controlMode.value === 'player' ? GAUGE_UPDATE_INTERVAL * 1.8 : GAUGE_UPDATE_INTERVAL
  if (now - lastGaugeUpdateTime >= gaugeInterval) {
    lastGaugeUpdateTime = now
    updateAltitudeAngleGauge(metrics)
  }

  // 天空颜色、城市灯光、时钟牌无需逐帧更新，限频即可。
  if (now - lastSkyAndCityUpdateTime >= SKY_CITY_UPDATE_INTERVAL) {
    lastSkyAndCityUpdateTime = now
    updateSkyByTime(metrics)
  }
}

function updateAllShadows(_metrics: SolarMetrics) {
  // v38：阴影由 Three.js DirectionalLight + shadowMap 自动生成。
  // 旧版手绘 BufferGeometry 阴影不再每帧更新，减少 CPU 压力，同时让影子真实依附在地面上。
  if (shadowGroup) shadowGroup.visible = false
}

function updateLightRay(sunPos: THREE.Vector3, visible: boolean) {
  lightRay.visible = layers.rays && visible
  if (!lightRay.visible) return
  const position = (lightRay.geometry as THREE.BufferGeometry).getAttribute('position') as THREE.BufferAttribute
  position.setXYZ(0, sunPos.x, sunPos.y, sunPos.z)
  position.setXYZ(1, OBSERVER_POINT.x, OBSERVER_POINT.y, OBSERVER_POINT.z)
  position.needsUpdate = true
}

function updateAltitudeAngleGauge(metrics: SolarMetrics) {
  if (!altitudeAngleGroup) return
  clearGroup(altitudeAngleGroup)
  if (!layers.altitudeGauge || metrics.altitude <= 0) return

  /**
   * 太阳高度角 h：太阳光线与观测地平面的夹角。
   *
   * 这版把“雷达扫描”真正套在太阳入射光线附近：
   * - 黄色长光线从城市街道观测点一直连到太阳；
   * - 雷达扫描盘悬浮在这条光线上；
   * - 灰色基准线表示与地面平行的地平线方向；
   * - 扫描扇面从地平基准线扫到太阳光线，表达高度角 h。
   */
  const rayTarget = OBSERVER_POINT.clone()
  const sunPos = solarToPosition(metrics, SKY_RADIUS)

  const sunDir = sunPos.clone().sub(rayTarget).normalize()
  const horizontal = new THREE.Vector3(sunDir.x, 0, sunDir.z)
  if (horizontal.lengthSq() < 0.0001) horizontal.set(-(metrics.east || 0), 0, metrics.north || -1)
  if (horizontal.lengthSq() < 0.0001) horizontal.set(0, 0, -1)
  horizontal.normalize()

  const up = new THREE.Vector3(0, 1, 0)
  // 用场景中 rayTarget -> sunPos 的实际方向角来画扫描扇面，确保弧线终点和太阳光线严格重合。
  // 标签仍显示地理公式计算的太阳高度角 h。
  const visualAltitude = THREE.MathUtils.radToDeg(Math.atan2(sunDir.y, Math.sqrt(sunDir.x * sunDir.x + sunDir.z * sunDir.z)))
  const shownAltitude = clamp(visualAltitude, 0, 89.5)

  // 扫描盘中心放在观测点到太阳的连线上，不贴地面。
  const origin = rayTarget.clone().add(sunDir.clone().multiplyScalar(1.55))
  const r = 1.62

  // 太阳入射光线必须连到太阳，避免学生误以为只是局部装饰线。
  altitudeAngleGroup.add(makeTubeLine([rayTarget, sunPos], 0xffd166, 0.016, 0.96))
  altitudeAngleGroup.add(makeLine([rayTarget, sunPos], 0xfff4bd, 0.42))

  const horizonEnd = origin.clone().add(horizontal.clone().multiplyScalar(r * 1.35))
  const horizonBack = origin.clone().add(horizontal.clone().multiplyScalar(-r * 0.28))
  const sunEdge = origin.clone().add(sunDir.clone().multiplyScalar(r * 1.1))

  // 同一高度处的地平基准线：表示“与地面平行的方向”，不贴在地面上。
  altitudeAngleGroup.add(makeTubeLine([horizonBack, horizonEnd], 0xe5e7eb, 0.01, 0.58))

  // 加粗扫描盘附近的太阳光线边，和完整太阳光线重合。
  altitudeAngleGroup.add(makeTubeLine([origin, sunEdge], 0xffd166, 0.022, 1))

  // 观测点引导线：说明这个浮动扫描盘对应城市观测者的位置。
  altitudeAngleGroup.add(makeLine([rayTarget, origin], 0x9ca3af, 0.32))
  altitudeAngleGroup.add(createSpriteText('观测点', '#cbd5e1', rayTarget.clone().add(new THREE.Vector3(0, 0.16, 0)), 0.105))

  const arc: THREE.Vector3[] = []
  const steps = Math.max(12, Math.ceil(shownAltitude / 2.2))
  for (let i = 0; i <= steps; i++) {
    if (i === steps) {
      // 最后一个点直接落在太阳光线上，避免弧线越过直射光线后多冒出一截。
      arc.push(origin.clone().add(sunDir.clone().multiplyScalar(r * 0.82)))
      continue
    }
    const a = degToRad((shownAltitude * i) / steps)
    arc.push(
      origin
        .clone()
        .add(horizontal.clone().multiplyScalar(Math.cos(a) * r * 0.82))
        .add(up.clone().multiplyScalar(Math.sin(a) * r * 0.82)),
    )
  }
  if (arc.length > 1) altitudeAngleGroup.add(makeTubeLine(arc, 0xffd166, 0.02, 0.98))

  // 雷达扫描扇面：悬浮在太阳光线旁，从地平基准线扫到太阳光线。
  const fanVertices: number[] = [origin.x, origin.y, origin.z]
  for (let i = 0; i <= steps; i++) {
    const p =
      i === steps
        ? origin.clone().add(sunDir.clone().multiplyScalar(r * 0.78))
        : origin
            .clone()
            .add(horizontal.clone().multiplyScalar(Math.cos(degToRad((shownAltitude * i) / steps)) * r * 0.78))
            .add(up.clone().multiplyScalar(Math.sin(degToRad((shownAltitude * i) / steps)) * r * 0.78))
    fanVertices.push(p.x, p.y, p.z)
  }
  const fanIndices: number[] = []
  for (let i = 1; i < steps + 1; i++) fanIndices.push(0, i, i + 1)
  const fanGeo = new THREE.BufferGeometry()
  fanGeo.setAttribute('position', new THREE.Float32BufferAttribute(fanVertices, 3))
  fanGeo.setIndex(fanIndices)
  fanGeo.computeVertexNormals()
  const fan = new THREE.Mesh(
    fanGeo,
    new THREE.MeshBasicMaterial({ color: 0xffd166, transparent: true, opacity: 0.17, side: THREE.DoubleSide, depthWrite: false }),
  )
  altitudeAngleGroup.add(fan)

  // 动态扫描线：像雷达一样在“地平基准线—太阳光线”之间往返。
  const sweepPhase = (Math.sin(performance.now() * 0.0028) + 1) / 2
  const sweepAlt = shownAltitude * sweepPhase
  const sweepA = degToRad(sweepAlt)
  const sweepEnd = origin
    .clone()
    .add(horizontal.clone().multiplyScalar(Math.cos(sweepA) * r * 0.95))
    .add(up.clone().multiplyScalar(Math.sin(sweepA) * r * 0.95))
  altitudeAngleGroup.add(makeTubeLine([origin, sweepEnd], 0xffffff, 0.011, 0.82))

  for (let alt = 15; alt <= 75; alt += 15) {
    if (alt > shownAltitude + 0.5) break
    const a = degToRad(alt)
    const p = origin
      .clone()
      .add(horizontal.clone().multiplyScalar(Math.cos(a) * r * 0.82))
      .add(up.clone().multiplyScalar(Math.sin(a) * r * 0.82))
    const tangent = horizontal
      .clone()
      .multiplyScalar(-Math.sin(a))
      .add(up.clone().multiplyScalar(Math.cos(a)))
      .normalize()
    const tickA = p.clone().add(tangent.clone().multiplyScalar(-0.058))
    const tickB = p.clone().add(tangent.clone().multiplyScalar(0.058))
    altitudeAngleGroup.add(makeLine([tickA, tickB], 0xfff1a8, 0.76))
  }

  const labelPos = origin
    .clone()
    .add(horizontal.clone().multiplyScalar(Math.cos(degToRad(shownAltitude * 0.52)) * (r * 1.02)))
    .add(up.clone().multiplyScalar(Math.sin(degToRad(shownAltitude * 0.52)) * (r * 1.02) + 0.15))
  altitudeAngleGroup.add(createSpriteText(`h=${formatDeg(metrics.altitude)}`, '#ffd166', labelPos, 0.18))
  altitudeAngleGroup.add(createSpriteText('地平基准线', '#e5e7eb', horizonEnd.clone().add(new THREE.Vector3(0, 0.12, 0)), 0.11))
  altitudeAngleGroup.add(
    createSpriteText(
      '太阳入射光线',
      '#fff2a8',
      sunPos
        .clone()
        .multiplyScalar(0.93)
        .add(new THREE.Vector3(0, 0.16, 0)),
      0.13,
    ),
  )
}

function updateCityTimeElements(nightK: number, dayK: number, metrics: SolarMetrics) {
  const enabled = layers.cityTime
  const nightVisible = enabled ? nightK : 0
  const time = performance.now() * 0.001

  streetLightItems.forEach((item, index) => {
    const flicker = 0.92 + Math.sin(time * 3.2 + index) * 0.04
    const opacity = nightVisible * flicker
    const lampMat = item.lamp.material as THREE.MeshBasicMaterial
    lampMat.color.set(opacity > 0.12 ? 0xfff1a6 : 0x4b5563)
    const glowMat = item.glow.material as THREE.SpriteMaterial
    glowMat.opacity = opacity * 0.92
    const coneMat = item.cone?.material as THREE.MeshBasicMaterial | undefined
    if (coneMat) coneMat.opacity = opacity * 0.16
  })

  cityRoadMaterials.forEach(mat => {
    const c = new THREE.Color(0x3d4651).lerp(new THREE.Color(0x171c25), nightVisible * 0.72)
    mat.color.copy(c)
    mat.roughness = 0.78 + nightVisible * 0.08
  })

  windowLightItems.forEach(item => {
    const lit = nightVisible * (0.52 + 0.48 * Math.abs(Math.sin(time * 0.85 + item.seed)))
    item.material.color.set(lit > 0.18 ? 0xffe7a3 : 0xdff7ff)
    item.material.opacity = 0.45 + lit * 0.55
  })

  trafficLightItems.forEach(item => {
    const phase = Math.floor((metrics.solarTime * 60 + item.seed * 10) / 18) % 3
    item.red.color.set(phase === 0 ? 0xff2a2a : 0x5a1111)
    item.yellow.color.set(phase === 1 ? 0xffd84d : 0x5a4a11)
    item.green.color.set(phase === 2 ? 0x2dff75 : 0x105c2d)
  })

  cityClockItems.forEach(item => {
    item.group.visible = enabled
    item.material.opacity = enabled ? 0.96 : 0
    if (!enabled) return

    // 只在显示内容真的变化时重绘 canvas。
    // 否则每帧 fillText + texture.needsUpdate 会比较耗性能。
    const clockKey = [formatClock(metrics.solarTime), metrics.altitude.toFixed(1), oneMeterShadowText.value, nightVisible.toFixed(1)].join('|')

    if (item.lastKey !== clockKey) {
      item.lastKey = clockKey
      drawCityClockTexture(item.ctx, item.texture, metrics, nightVisible)
    }
  })
}

function drawCityClockTexture(ctx: CanvasRenderingContext2D, texture: THREE.CanvasTexture, metrics: SolarMetrics, nightK: number) {
  const width = ctx.canvas.width
  const height = ctx.canvas.height
  const centerX = width / 2
  ctx.clearRect(0, 0, width, height)

  const bg = ctx.createLinearGradient(0, 0, width, height)
  bg.addColorStop(0, nightK > 0.35 ? '#0b1220' : '#102a43')
  bg.addColorStop(1, nightK > 0.35 ? '#172554' : '#075985')
  ctx.fillStyle = bg
  roundRect(ctx, 16, 12, width - 32, height - 24, 28)
  ctx.fill()
  ctx.strokeStyle = 'rgba(125, 211, 252, 0.75)'
  ctx.lineWidth = 5
  ctx.stroke()

  const shadowText = metrics.altitude <= 0 ? '太阳在地平线下' : `${(1 / Math.tan(degToRad(Math.max(1, metrics.altitude)))).toFixed(2)} m`

  ctx.fillStyle = '#bae6fd'
  ctx.font = '700 32px Microsoft YaHei, Arial'
  ctx.textAlign = 'center'
  ctx.fillText('城市太阳时钟', centerX, 56)

  ctx.fillStyle = '#fde68a'
  ctx.font = '900 56px DIN Alternate, Microsoft YaHei, Arial'
  ctx.fillText(formatClock(metrics.solarTime), centerX, 123)

  ctx.fillStyle = '#e0f2fe'
  ctx.font = '700 24px Microsoft YaHei, Arial'
  ctx.fillText(`太阳高度 h ${formatDeg(metrics.altitude)}`, centerX, 174)

  ctx.fillStyle = 'rgba(255,255,255,0.72)'
  ctx.font = '600 20px Microsoft YaHei, Arial'
  ctx.fillText('看路灯、窗户、影子一起判断时间', centerX, 212)

  texture.needsUpdate = true
}

function updateSkyByTime(metrics: SolarMetrics) {
  if (!renderer || !scene) return
  const colors = getSmoothSkyColors(metrics.altitude, state.solarTime)

  renderer.setClearColor(colors.clear, 0)
  if (scene.fog instanceof THREE.Fog) {
    scene.fog.color.set(colors.fog)
    scene.fog.near = metrics.altitude <= -4 ? 10 : 12.5
    scene.fog.far = metrics.altitude <= -4 ? 28 : 32
  }
  if (hemisphereDome?.material) {
    const material = hemisphereDome.material as THREE.MeshBasicMaterial
    material.color.set(colors.dome)
    material.opacity = metrics.altitude > 20 ? 0.08 : metrics.altitude <= -4 ? 0.045 : 0.065
  }

  // 白天更亮：同时调 Three 场景灯光，不只换外层容器背景。
  const dayK = smoothstep(-2, 30, metrics.altitude)
  const nightK = 1 - smoothstep(-6, 6, metrics.altitude)
  // 原生阴影想更明显，不能只加太阳光，还要适当降低环境光和边缘补光。
  if (ambientLight) ambientLight.intensity = 0.42 + dayK * 0.62
  if (rimLight) rimLight.intensity = 0.22 + dayK * 0.36

  updateNightSkyDecorations(nightK)
  updateCityTimeElements(nightK, dayK, metrics)
}

function azimuthAltitudeToVec3(azimuth: number, altitude: number, radius: number) {
  const az = degToRad(azimuth)
  const alt = degToRad(altitude)
  const horizontal = Math.cos(alt)

  // 方位角仍按地理定义：北0°、东90°、南180°、西270°。
  // 三维世界中 x+ 作为西侧、x- 作为东侧，保证从北侧面向南方观察时，东在左、西在右。
  const x = -Math.sin(az) * horizontal * radius
  const y = Math.sin(alt) * radius
  const z = Math.cos(az) * horizontal * radius
  return new THREE.Vector3(x, y, z)
}

function makeLine(points: THREE.Vector3[], color: number, opacity = 1) {
  const geometry = new THREE.BufferGeometry().setFromPoints(points)
  return new THREE.Line(geometry, new THREE.LineBasicMaterial({ color, transparent: true, opacity }))
}

function makeTubeLine(points: THREE.Vector3[], color: number, radius = 0.01, opacity = 1) {
  const curve = new THREE.CatmullRomCurve3(points)
  const geometry = new THREE.TubeGeometry(curve, Math.max(8, points.length * 2), radius, 8, false)
  return new THREE.Mesh(geometry, new THREE.MeshBasicMaterial({ color, transparent: true, opacity, depthWrite: false }))
}

function createLabelSpriteText(text: string, color: string, position: THREE.Vector3, fontSize = 15, worldSize = 0.34) {
  const canvas = document.createElement('canvas')
  const ctx = canvas.getContext('2d')!
  const padding = Math.ceil(fontSize * 1.2)
  canvas.width = Math.max(96, Math.ceil(text.length * fontSize * 1.25 + padding * 2))
  canvas.height = Math.max(48, Math.ceil(fontSize * 2.6))

  ctx.clearRect(0, 0, canvas.width, canvas.height)
  ctx.font = `900 ${fontSize}px Microsoft YaHei, Arial`
  ctx.textAlign = 'center'
  ctx.textBaseline = 'middle'
  ctx.lineWidth = Math.max(2, Math.round(fontSize * 0.18))
  ctx.strokeStyle = 'rgba(0,0,0,0.72)'
  ctx.fillStyle = color
  ctx.strokeText(text, canvas.width / 2, canvas.height / 2)
  ctx.fillText(text, canvas.width / 2, canvas.height / 2)

  const texture = new THREE.CanvasTexture(canvas)
  texture.colorSpace = THREE.SRGBColorSpace
  const sprite = new THREE.Sprite(new THREE.SpriteMaterial({ map: texture, transparent: true, depthWrite: false }))
  sprite.position.copy(position)
  sprite.scale.set(worldSize * (canvas.width / canvas.height), worldSize, 1)
  return sprite
}

function createSpriteText(text: string, color: string, position: THREE.Vector3, size = 0.28) {
  const canvas = document.createElement('canvas')
  const ctx = canvas.getContext('2d')!
  const fontSize = 48
  const lines = text.split('\n')
  canvas.width = 512
  canvas.height = Math.max(128, lines.length * 72)
  ctx.clearRect(0, 0, canvas.width, canvas.height)
  ctx.font = `900 ${fontSize}px Microsoft YaHei, Arial`
  ctx.textAlign = 'center'
  ctx.textBaseline = 'middle'
  ctx.lineWidth = 8
  ctx.strokeStyle = 'rgba(0,0,0,0.72)'
  ctx.fillStyle = color
  lines.forEach((line, index) => {
    const y = canvas.height / 2 + (index - (lines.length - 1) / 2) * 60
    ctx.strokeText(line, canvas.width / 2, y)
    ctx.fillText(line, canvas.width / 2, y)
  })

  const texture = new THREE.CanvasTexture(canvas)
  texture.colorSpace = THREE.SRGBColorSpace
  const sprite = new THREE.Sprite(new THREE.SpriteMaterial({ map: texture, transparent: true, depthWrite: false }))
  sprite.position.copy(position)
  sprite.scale.set(size * 3.2, size * (canvas.height / canvas.width) * 3.2, 1)
  return sprite
}

function clearGroup(group: THREE.Group) {
  group.traverse(obj => {
    const mesh = obj as THREE.Mesh
    if (mesh.geometry) mesh.geometry.dispose()
    const mat = mesh.material as THREE.Material | THREE.Material[] | undefined
    if (Array.isArray(mat)) mat.forEach(m => m.dispose())
    else mat?.dispose()
  })
  group.clear()
}

function applyLayerVisibility() {
  if (domeGroup) domeGroup.visible = layers.dome
  if (pathGroup) pathGroup.visible = layers.paths

  // v38：影子改用 Three.js 原生阴影，shadowGroup 旧的手绘薄片不再显示。
  // layers.shadow 现在控制 keyLight.castShadow / renderer.shadowMap.enabled。
  if (shadowGroup) shadowGroup.visible = false

  if (rayGroup) rayGroup.visible = layers.rays
  if (altitudeAngleGroup) altitudeAngleGroup.visible = layers.altitudeGauge

  if (keyLight && renderer) {
    const metrics = solarMetrics.value
    const isShadowVisible = layers.shadow && metrics.altitude > 1
    keyLight.castShadow = isShadowVisible
    renderer.shadowMap.enabled = isShadowVisible
    // renderer.shadowMap.needsUpdate = true
  }
}

function getRendererPixelRatio() {
  // 玩家版优先保证操控流畅。高分屏下 DPR 过高会让白天楼群、阴影、路径线渲染发卡。
  // 这里限制到 1.5，优先保证“玩起来顺”。
  return Math.min(window.devicePixelRatio || 1, 1.35)
}

function resizeRenderer() {
  if (!canvasWrapRef.value || !renderer || !camera) return
  const rect = canvasWrapRef.value.getBoundingClientRect()
  const width = Math.max(1, Math.floor(rect.width))
  const height = Math.max(1, Math.floor(rect.height))
  const ratio = getRendererPixelRatio()

  camera.aspect = width / height
  camera.updateProjectionMatrix()

  renderer.setPixelRatio(ratio)
  renderer.setSize(width, height, false)
  renderer.domElement.style.width = '100%'
  renderer.domElement.style.height = '100%'
}

function animate(now: number) {
  animationId = requestAnimationFrame(animate)
  frameCount += 1

  const dt = Math.min(0.05, (now - lastTime) / 1000 || 0)
  lastTime = now

  if (isPlaying.value) {
    runtimeSolarTime += dt * state.playSpeed * 0.45
    if (runtimeSolarTime > 24) runtimeSolarTime -= 24

    // 太阳位置和光照计算仍然走 requestAnimationFrame；
    // 但是右侧数据、顶部背景 CSS、文字等 Vue DOM 不必每帧刷新，否则白天渐变背景和数据面板会拖慢操作。
    if (now - lastSolarTimeUiSync >= SOLAR_TIME_UI_SYNC_INTERVAL) {
      lastSolarTimeUiSync = now
      state.solarTime = runtimeSolarTime
    }
  } else {
    runtimeSolarTime = state.solarTime
  }

  const runtimeMetrics = computeSolarMetrics(state.latitude, state.dayOfYear, runtimeSolarTime)

  if (controlMode.value === 'player' && isPlayerReady.value) {
    applyOrbitControlForMode()
    player?.update?.(dt)
    restoreFirstPersonViewAfterJump()
    // 边界现在交给透明圆柱碰撞体处理，不再每帧 reset，避免玩家被重置到动不了。

    // 飞行动画不用每帧重复触发，隔帧同步即可，减少动画切换/混合开销。
    if (frameCount % 2 === 0) syncPlayerFlightAnimation()

    updateManualThirdPersonCamera(dt)
  } else {
    controls?.update()
  }

  updateSceneBySolar(runtimeMetrics)
  renderer?.render(scene, camera)
}

watch(
  () => [state.latitude, state.dayOfYear],
  () => {
    if (pathGroup) rebuildSolarPaths()
    updateSceneBySolar()
  },
)

watch(
  () => [layers.dome, layers.paths, layers.shadow, layers.rays, layers.altitudeGauge, layers.cityTime],
  () => {
    if (pathGroup) rebuildSolarPaths()
    applyLayerVisibility()
    updateSceneBySolar()
  },
)

watch(
  () => state.solarTime,
  value => {
    if (!isPlaying.value) runtimeSolarTime = value
  },
)

onMounted(async () => {
  await nextTick()
  initThree()
})

onBeforeUnmount(() => {
  cancelAnimationFrame(animationId)
  resizeObserver?.disconnect()
  window.removeEventListener('resize', resizeRenderer)
  window.removeEventListener('keydown', handlePlayerKeyDown, true)
  setPlayerInputEnabled(false)
  unbindThirdPersonMouseControls(renderer?.domElement)
  destroyCurrentPlayerController()
  playerModelObjectUrls.forEach(url => {
    if (url.startsWith('blob:')) URL.revokeObjectURL(url)
  })
  controls?.dispose()
  if (renderer) {
    renderer.dispose()
    renderer.domElement.remove()
  }
  if (rootGroup) clearGroup(rootGroup)
})
</script>

<style scoped>
:global(html),
:global(body),
:global(#app) {
  width: 100%;
  height: 100%;
  margin: 0;
}

* {
  box-sizing: border-box;
}

.solar-motion-page {
  --bg: #061022;
  --panel: rgba(10, 26, 48, 0.82);
  --panel2: rgba(8, 22, 40, 0.94);
  --line: rgba(114, 214, 255, 0.2);
  --line2: rgba(255, 209, 102, 0.28);
  --text: #eaf6ff;
  --muted: #93a9bf;
  --gold: #ffd166;
  --cyan: #4ddcff;
  --green: #45f4a8;
  --danger: #ff6b6b;
  position: relative;
  width: 100%;
  height: 100vh;
  min-height: 680px;
  overflow: hidden;
  color: var(--text);
  padding: 12px;
  display: grid;
  grid-template-rows: 62px 1fr;
  gap: 14px;
  font-family:
    ui-sans-serif,
    system-ui,
    -apple-system,
    BlinkMacSystemFont,
    'Segoe UI',
    'Microsoft YaHei',
    Arial,
    sans-serif;
  background:
    radial-gradient(circle at 18% 14%, rgba(77, 220, 255, 0.16), transparent 28%),
    radial-gradient(circle at 78% 20%, rgba(255, 209, 102, 0.12), transparent 24%),
    radial-gradient(circle at 55% 92%, rgba(129, 140, 248, 0.15), transparent 38%), linear-gradient(135deg, #020713, #071427 48%, #030611);
  transition: background 1.6s ease;
}

.solar-motion-page.phase-day {
  --panel: rgba(12, 45, 76, 0.72);
  --panel2: rgba(10, 34, 58, 0.88);
  --line: rgba(255, 255, 255, 0.24);
  background:
    radial-gradient(circle at 28% 16%, rgba(255, 232, 150, 0.26), transparent 23%),
    radial-gradient(circle at 68% 18%, rgba(135, 213, 255, 0.36), transparent 28%), linear-gradient(180deg, #7ed7ff 0%, #bfeeff 48%, #0d3158 100%);
}

.solar-motion-page.phase-dawn {
  background:
    radial-gradient(circle at 20% 20%, rgba(255, 225, 145, 0.32), transparent 28%),
    radial-gradient(circle at 78% 18%, rgba(255, 116, 96, 0.18), transparent 30%), linear-gradient(180deg, #f1b06f 0%, #6fb8dd 42%, #092142 100%);
}

.solar-motion-page.phase-sunset {
  background:
    radial-gradient(circle at 78% 20%, rgba(255, 168, 87, 0.34), transparent 27%),
    radial-gradient(circle at 24% 15%, rgba(125, 98, 255, 0.22), transparent 30%), linear-gradient(180deg, #664c94 0%, #c66b58 46%, #081529 100%);
}

/* 白天时降低中间舞台暗色蒙版，让 Three 场景真正亮起来，而不是只换页面背景。 */
.solar-motion-page.phase-day .stage-card {
  background:
    radial-gradient(circle at 48% 32%, rgba(255, 255, 255, 0.28), transparent 24%),
    radial-gradient(circle at 72% 78%, rgba(255, 209, 102, 0.14), transparent 26%),
    linear-gradient(135deg, rgba(126, 215, 255, 0.34), rgba(191, 238, 255, 0.22));
  box-shadow:
    0 24px 80px rgba(0, 54, 94, 0.22),
    inset 0 0 70px rgba(255, 255, 255, 0.08),
    inset 0 0 0 1px rgba(255, 255, 255, 0.1);
}

.solar-motion-page.phase-night .stage-card {
  background:
    radial-gradient(circle at 30% 22%, rgba(129, 140, 248, 0.12), transparent 24%),
    radial-gradient(circle at 70% 74%, rgba(77, 220, 255, 0.08), transparent 28%),
    linear-gradient(135deg, rgba(2, 7, 19, 0.62), rgba(7, 20, 39, 0.94));
}

.solar-motion-page.phase-night {
  background:
    radial-gradient(circle at 18% 14%, rgba(77, 220, 255, 0.13), transparent 28%),
    radial-gradient(circle at 78% 20%, rgba(129, 140, 248, 0.18), transparent 24%),
    radial-gradient(circle at 55% 92%, rgba(71, 85, 105, 0.22), transparent 38%), linear-gradient(135deg, #020713, #071427 48%, #030611);
}

.bg-grid {
  position: absolute;
  inset: 0;
  pointer-events: none;
  background-image:
    linear-gradient(rgba(255, 255, 255, 0.06) 1px, transparent 1px), linear-gradient(90deg, rgba(255, 255, 255, 0.06) 1px, transparent 1px);
  background-size: 44px 44px;
  mask-image: radial-gradient(circle at center, black, transparent 78%);
}

.scan-line {
  position: absolute;
  inset: 0;
  pointer-events: none;
  background: linear-gradient(180deg, transparent, rgba(77, 220, 255, 0.05), transparent);
  animation: scan 6s linear infinite;
}

@keyframes scan {
  from {
    transform: translateY(-100%);
  }
  to {
    transform: translateY(100%);
  }
}

.topbar {
  position: relative;
  z-index: 3;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 14px;
  padding: 10px 14px;
  border: 1px solid var(--line);
  border-radius: 22px;
  background: rgba(7, 17, 31, 0.72);
  backdrop-filter: blur(16px);
  box-shadow:
    0 18px 60px rgba(0, 0, 0, 0.28),
    inset 0 0 40px rgba(77, 220, 255, 0.055);
}

.brand {
  display: flex;
  align-items: center;
  gap: 13px;
}
.sun-logo {
  width: 44px;
  height: 44px;
  border-radius: 16px;
  background: radial-gradient(circle at 35% 30%, #fff4bd, #ffd166 40%, #ff8a30 72%, #221309 100%);
  box-shadow: 0 0 30px rgba(255, 209, 102, 0.55);
}
.eyebrow {
  color: var(--cyan);
  font-size: 10px;
  letter-spacing: 0.14em;
}
h1 {
  margin: 2px 0 0;
  font-size: 16px;
  line-height: 1.1;
}
.top-actions {
  display: flex;
  align-items: center;
  gap: 10px;
}

button {
  border: 1px solid rgba(125, 211, 252, 0.22);
  color: var(--text);
  background: rgba(16, 36, 62, 0.78);
  border-radius: 12px;
  padding: 7px 10px;
  cursor: pointer;
  transition: 0.18s ease;
  font-size: 10.5px;
}
button:hover {
  transform: translateY(-1px);
  border-color: rgba(255, 209, 102, 0.65);
}
button.active {
  color: #101722;
  border-color: rgba(255, 209, 102, 0.85);
  background: linear-gradient(135deg, #ffe185, #ffb02e);
  box-shadow: 0 0 22px rgba(255, 209, 102, 0.35);
  font-weight: 900;
}
button.ghost {
  background: rgba(14, 30, 52, 0.55);
}

.layout {
  position: relative;
  z-index: 2;
  min-height: 0;
  display: grid;
  grid-template-columns: 252px minmax(0, 1.9fr) 292px;
  gap: 12px;
}

.panel,
.stage-card {
  min-height: 0;
  border: 1px solid rgba(77, 220, 255, 0.22);
  border-radius: 24px;
  background: linear-gradient(180deg, rgba(8, 22, 42, 0.86), rgba(5, 14, 28, 0.78));
  backdrop-filter: blur(18px);
  box-shadow:
    0 18px 70px rgba(0, 0, 0, 0.32),
    inset 0 0 60px rgba(77, 220, 255, 0.045);
  overflow: hidden;
}

.panel {
  padding: 10px;
  overflow: auto;
  scrollbar-width: thin;
  scrollbar-color: rgba(77, 220, 255, 0.35) transparent;
  font-size: 10px;
}

.panel-title {
  position: sticky;
  top: -10px;
  z-index: 4;
  margin: -10px -10px 9px;
  padding: 10px 11px 9px;
  background: linear-gradient(180deg, rgba(10, 24, 43, 0.98), rgba(10, 24, 43, 0.82));
  border-bottom: 1px solid var(--line);
  display: flex;
  align-items: center;
  gap: 8px;
  font-weight: 900;
  font-size: 11px;
}
.panel-title span {
  width: 10px;
  height: 10px;
  border-radius: 50%;
  background: var(--gold);
  box-shadow: 0 0 16px var(--gold);
}

.block,
.knowledge-card,
.data-card,
.summary-card {
  margin-bottom: 9px;
  padding: 9px;
  border-radius: 16px;
  border: 1px solid rgba(125, 211, 252, 0.14);
  background: rgba(8, 22, 40, 0.66);
}
.block h3,
.knowledge-card h3,
.summary-card h3 {
  margin: 0 0 9px;
  font-size: 10px;
  color: #f6fbff;
  letter-spacing: 0.02em;
}

.range-row {
  width: 100%;
  margin: 10px 0 14px;
}
.range-head {
  width: 100%;
  display: grid !important;
  grid-template-columns: minmax(0, 1fr) max-content;
  align-items: center;
  column-gap: 12px;
  margin-bottom: 8px;
  line-height: 1;
}
.range-label {
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  color: #d9eaff;
  font-size: 9.6px;
  font-weight: 800;
  letter-spacing: 0.02em;
}
.range-value {
  justify-self: end;
  min-width: 58px;
  max-width: 138px;
  height: 18px;
  padding: 0 8px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  border-radius: 999px;
  color: #121824;
  white-space: nowrap;
  font-size: 9.4px;
  font-weight: 950;
  text-align: center;
  overflow: hidden;
  text-overflow: ellipsis;
  border: 1px solid rgba(255, 229, 148, 0.9);
  background: linear-gradient(135deg, #fff4b8, #ffd166 48%, #ffad2f);
  box-shadow:
    0 0 14px rgba(255, 209, 102, 0.42),
    inset 0 0 0 1px rgba(255, 255, 255, 0.42);
}
.range-slider {
  width: 100%;
  margin-top: 0;
}
.range-slider :deep(.el-slider) {
  --el-slider-main-bg-color: #4da3ff;
  --el-slider-runway-bg-color: rgba(226, 232, 240, 0.88);
  --el-slider-stop-bg-color: transparent;
  width: 100%;
}
.range-slider :deep(.el-slider__runway) {
  height: 6px;
  margin: 11px 0 5px;
  border-radius: 999px;
  background: rgba(226, 232, 240, 0.86);
}
.range-slider :deep(.el-slider__bar) {
  height: 6px;
  border-radius: 999px;
  background: linear-gradient(90deg, #38bdf8, #60a5fa, #ffd166);
  box-shadow: 0 0 12px rgba(96, 165, 250, 0.35);
}
.range-slider :deep(.el-slider__button-wrapper) {
  top: -15px;
  width: 30px;
  height: 30px;
}
.range-slider :deep(.el-slider__button) {
  width: 14px;
  height: 14px;
  border: 2px solid #f8fbff;
  background: linear-gradient(135deg, #38bdf8, #3b82f6);
  box-shadow: 0 0 10px rgba(56, 189, 248, 0.55);
}
.grid-2 {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 7px;
}
.grid-4 {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 6px;
  margin-bottom: 12px;
}
.view-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 7px;
}
.time-buttons {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 7px;
  margin-top: 10px;
}
.tip,
.knowledge-card li {
  color: var(--muted);
  font-size: 9.6px;
  line-height: 1.58;
}
.block > .tip {
  margin: 15px 0 0;
  padding-top: 10px;
  border-top: 1px solid rgba(125, 211, 252, 0.1);
}
.block .grid-2 + .tip,
.block .grid-4 + .tip,
.block .view-grid + .tip,
.block .time-buttons + .tip {
  margin-top: 14px;
}
.block .grid-4 + .range-row {
  margin-top: 2px;
}
.left-panel .grid-2 button.active,
.left-panel .grid-4 button.active,
.left-panel .view-grid button.active,
.left-panel .time-buttons button.active {
  color: #101722;
  border-color: rgba(255, 209, 102, 0.85);
  background: linear-gradient(135deg, #ffe185, #ffb02e);
  box-shadow:
    0 0 22px rgba(255, 209, 102, 0.35),
    inset 0 0 0 1px rgba(255, 255, 255, 0.32);
  font-weight: 900;
}
.check-row {
  display: flex;
  align-items: center;
  gap: 7px;
  padding: 7px 0;
  color: #d6eaff;
  font-size: 9.8px;
  line-height: 1.35;
}
.check-row input {
  accent-color: #ffd166;
}

.sub-title {
  margin: 10px 0 4px;
  color: var(--cyan);
  font-size: 12px;
  font-weight: 900;
  letter-spacing: 0.05em;
}
.fp-tip {
  margin: 6px 0 8px;
  padding: 7px 8px;
  border-radius: 10px;
  color: #dceeff;
  font-size: 10px;
  line-height: 1.5;
  background: rgba(77, 220, 255, 0.08);
  border: 1px solid rgba(77, 220, 255, 0.14);
}
.student-fp-controls {
  margin-top: 10px;
  padding: 10px;
  border-radius: 16px;
  border: 1px solid rgba(77, 220, 255, 0.18);
  background: linear-gradient(180deg, rgba(77, 220, 255, 0.08), rgba(255, 209, 102, 0.05));
}

.stage-card {
  position: relative;
  min-width: 0;
  background:
    radial-gradient(circle at 48% 34%, rgba(77, 220, 255, 0.2), transparent 27%),
    radial-gradient(circle at 72% 78%, rgba(255, 209, 102, 0.1), transparent 26%),
    linear-gradient(135deg, rgba(3, 12, 24, 0.62), rgba(7, 20, 38, 0.88));
  box-shadow:
    0 24px 90px rgba(0, 0, 0, 0.38),
    inset 0 0 80px rgba(77, 220, 255, 0.07),
    inset 0 0 0 1px rgba(255, 255, 255, 0.04);
}
.stage-card::before {
  content: '';
  position: absolute;
  inset: 10px;
  z-index: 4;
  pointer-events: none;
  border-radius: 18px;
  border: 1px solid rgba(77, 220, 255, 0.16);
  box-shadow:
    inset 0 0 28px rgba(77, 220, 255, 0.08),
    0 0 30px rgba(77, 220, 255, 0.08);
}
.stage-card::after {
  content: '';
  position: absolute;
  left: 16px;
  right: 16px;
  bottom: 12px;
  height: 1px;
  z-index: 4;
  pointer-events: none;
  background: linear-gradient(90deg, transparent, rgba(255, 209, 102, 0.5), rgba(77, 220, 255, 0.55), transparent);
}
.canvas-wrap {
  touch-action: none;
  user-select: none;
  position: absolute;
  inset: 0;
  min-height: 420px;
}
.canvas-wrap :deep(canvas) {
  display: block;
  width: 100%;
  height: 100%;
}
.stage-card:has(.canvas-wrap) {
  outline: 1px solid rgba(77, 220, 255, 0.05);
}

.scene-title {
  position: absolute;
  top: 18px;
  left: 50%;
  transform: translateX(-50%);
  text-align: center;
  pointer-events: none;
  text-shadow: 0 4px 18px rgba(0, 0, 0, 0.65);
}
.scene-title b {
  display: block;
  color: var(--gold);
  font-size: 18px;
  margin-bottom: 6px;
}
.scene-title span {
  color: #dceeff;
  font-size: 11.5px;
}

.legend-panel {
  position: absolute;
  z-index: 5;
  border: 1px solid rgba(125, 211, 252, 0.18);
  border-radius: 18px;
  background: rgba(5, 17, 31, 0.72);
  backdrop-filter: blur(14px);
  box-shadow: 0 14px 44px rgba(0, 0, 0, 0.32);
}
.legend-title,
.card-head {
  color: var(--gold);
  font-weight: 900;
  margin-bottom: 10px;
}
.small-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 8px;
}
.small-grid div {
  padding: 8px;
  border-radius: 12px;
  background: rgba(255, 255, 255, 0.05);
}
.small-grid span {
  display: block;
  color: var(--muted);
  font-size: 11px;
  margin-bottom: 4px;
}
.small-grid b {
  color: #eaf6ff;
  font-size: 11.5px;
}

.legend-panel {
  left: 16px;
  bottom: 16px;
  padding: 14px;
  width: 230px;
  color: #dceeff;
  font-size: 11.5px;
  line-height: 1.9;
}
.dot {
  display: inline-block;
  width: 10px;
  height: 10px;
  border-radius: 50%;
  margin-right: 7px;
}
.dot.yellow {
  background: #ffd166;
  box-shadow: 0 0 10px #ffd166;
}
.dot.current {
  background: #ffb02e;
  box-shadow: 0 0 12px #ffb02e;
}
.dot.blue {
  background: #3687ff;
  box-shadow: 0 0 10px #3687ff;
}
.dot.white {
  background: #ffffff;
  box-shadow: 0 0 10px #ffffff;
}
.dot.cyan {
  background: #45e8ff;
  box-shadow: 0 0 10px #45e8ff;
}
.dot.shadow {
  background: rgba(0, 0, 0, 0.55);
  border: 1px solid #999;
}
.dot.playground {
  background: #c9634a;
  box-shadow: 0 0 10px rgba(201, 99, 74, 0.8);
}

.data-card {
  background: linear-gradient(180deg, rgba(15, 43, 75, 0.82), rgba(7, 21, 38, 0.72));
}
.big-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 10px 0;
  border-bottom: 1px solid rgba(125, 211, 252, 0.12);
}
.big-row span {
  color: var(--muted);
}
.big-row b {
  color: var(--gold);
  font-size: 22px;
}
.knowledge-card ol,
.knowledge-card ul {
  margin: 0;
  padding-left: 18px;
}
.knowledge-card {
  padding: 11px 12px;
}
.knowledge-card h3 {
  margin-bottom: 10px;
  color: #f8fbff;
  text-shadow: 0 0 14px rgba(125, 211, 252, 0.18);
}
.knowledge-card p {
  margin: 7px 0;
  color: #c8d9ee;
  font-size: 10px;
  line-height: 1.72;
}
.knowledge-card li {
  margin: 6px 0;
  padding-left: 2px;
  color: #c8d9ee;
  font-size: 9.8px;
  line-height: 1.68;
}
.knowledge-card b,
.knowledge-card strong,
.formula-item b {
  color: #ffd166;
  font-weight: 950;
  text-shadow: 0 0 12px rgba(255, 209, 102, 0.32);
}
.knowledge-card code,
.formula-item code {
  color: #8ee7ff;
  background: rgba(56, 189, 248, 0.08);
  border: 1px solid rgba(125, 211, 252, 0.12);
  border-radius: 6px;
  padding: 1px 4px;
}
.formula-card {
  background: rgba(5, 17, 31, 0.66);
}
.shadow-explain-card {
  border-color: rgba(255, 209, 102, 0.24);
  background: linear-gradient(180deg, rgba(255, 209, 102, 0.09), rgba(5, 17, 31, 0.66));
}
.formula-item {
  display: grid;
  grid-template-columns: 60px minmax(0, 1fr);
  gap: 8px 10px;
  align-items: center;
  padding: 9px 0;
  border-top: 1px dashed rgba(125, 211, 252, 0.13);
}
.formula-item b {
  color: #ffffff;
}
.formula-item code {
  color: #bcecff;
  font-family: 'JetBrains Mono', Consolas, monospace;
  font-size: 11.5px;
  white-space: normal;
  word-break: break-word;
}
.formula-item span {
  grid-column: 2;
  color: var(--muted);
  font-size: 11.5px;
}
.summary-card table {
  width: 100%;
  border-collapse: collapse;
  font-size: 11.5px;
}
.summary-card th,
.summary-card td {
  border: 1px solid rgba(125, 211, 252, 0.14);
  padding: 8px;
  text-align: center;
}
.summary-card th {
  color: var(--gold);
  background: rgba(255, 255, 255, 0.05);
}

@media (max-width: 1320px) {
  .layout {
    grid-template-columns: 238px minmax(0, 1.55fr) 276px;
  }
}

@media (max-width: 1100px) {
  .solar-motion-page {
    height: auto;
    min-height: 100vh;
    overflow: auto;
  }
  .layout {
    grid-template-columns: 1fr;
  }
  .stage-card {
    height: 720px;
  }
}

:deep(.range-head) {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  width: 100%;
  margin-bottom: 8px;
}

:deep(.range-label) {
  flex: 1;
  min-width: 0;
  font-size: 12px;
  font-weight: 700;
  color: rgba(226, 232, 240, 0.92);
  line-height: 1.2;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

:deep(.range-value) {
  flex-shrink: 0;
  min-width: 58px;
  padding: 3px 8px;
  border-radius: 999px;
  text-align: center;
  font-size: 12px;
  font-weight: 900;
  line-height: 1.2;
  color: #ffd166;
  background: linear-gradient(135deg, rgba(255, 209, 102, 0.18), rgba(255, 176, 32, 0.08));
  border: 1px solid rgba(255, 209, 102, 0.35);
  box-shadow:
    0 0 12px rgba(255, 209, 102, 0.18),
    inset 0 0 10px rgba(255, 209, 102, 0.08);
}

:deep(.range-row) {
  width: 100%;
  margin-top: 8px;
}

:deep(.range-row .el-slider) {
  width: 100%;
}

.player-model-select {
  margin-top: 12px;
  padding-top: 10px;
  border-top: 1px solid rgba(148, 163, 184, 0.16);
}

.model-select-title {
  margin-bottom: 8px;
  font-size: 12px;
  font-weight: 800;
  letter-spacing: 0.08em;
  color: rgba(226, 232, 240, 0.86);
}

.model-buttons {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 8px;
}

.model-buttons button {
  min-height: 30px;
  border-radius: 10px;
  border: 1px solid rgba(77, 220, 255, 0.2);
  background: rgba(15, 23, 42, 0.58);
  color: rgba(226, 232, 240, 0.9);
  font-size: 12px;
  font-weight: 800;
  cursor: pointer;
}

.model-buttons button.active {
  color: #06121f;
  border-color: rgba(255, 209, 102, 0.75);
  background: linear-gradient(135deg, #ffd166, #4ddcff);
  box-shadow: 0 0 16px rgba(77, 220, 255, 0.24);
}

.time-buttons button:disabled {
  cursor: not-allowed;
  opacity: 0.48;
  filter: grayscale(0.35);
  box-shadow: none;
}

.time-buttons button:disabled.active {
  color: rgba(226, 232, 240, 0.72);
  background: rgba(15, 23, 42, 0.65);
  border-color: rgba(148, 163, 184, 0.2);
}
</style>
