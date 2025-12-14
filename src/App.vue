<script setup lang="ts">
import { ref, onMounted, watch } from 'vue'
import { 
  NConfigProvider, NGlobalStyle, NLayout, NLayoutSider, NLayoutContent, 
  NCard, NForm, NFormItem, NInputNumber, NSlider, NButton, NSelect, 
  NSpace, NTag, NDivider, darkTheme 
} from 'naive-ui'

// --- Te: 数据结构与物理常数定义 ---
// 为了模拟效果直观，我们使用模拟单位而非严格SI单位
const ANIMATION_SPEED = 1
const TRAJECTORY_limit = 500

// 1. 新增：能量历史数据定义
const energyHistory = ref<number[]>([])
const energyCanvasRef = ref<HTMLCanvasElement | null>(null)

interface Particle {
  x: number
  y: number
  vx: number
  vy: number
  path: {x: number, y: number}[]
}

// 模拟参数状态
const params = ref({
  q: 1,      // 电荷量 (+/-)
  m: 1,      // 质量
  Ex: 0,     // 电场 X分量
  Ey: 0,     // 电场 Y分量
  B: 0,      // 磁场 (垂直屏幕方向)
  v0: 5,     // 初速度大小
  angle: 0,  // 入射角度
  g: 0       // 重力 (可选)
})

// 运行状态
const isRunning = ref(false)
const time = ref(0)
const particle = ref<Particle>({ x: 50, y: 300, vx: 0, vy: 0, path: [] })
let animationId: number | null = null

// 画布引用
const canvasRef = ref<HTMLCanvasElement | null>(null)

// --- Ni: 高考考点场景预设 (抽象模型的具体化) ---
const scenarios = [
  { label: '自定义模式 (Custom)', value: 'custom' },
  { label: '纯磁场-匀速圆周 (Cyclotron)', value: 'circle' },
  { label: '纯电场-类平抛 (Parabolic)', value: 'parabolic' },
  { label: '速度选择器 (Velocity Selector)', value: 'selector' },
  { label: '直线加速 (Linear Accel)', value: 'linear' },
  { label: '带电微粒-重力平衡 (Milikan)', value: 'gravity_balance' }
]
const currentScenario = ref('custom')

// 监听场景切换，应用 Te 规则
//弃用 数据有问题
/*watch(currentScenario, (val) => {
  resetSimulation()
  switch (val) {
    case 'circle':
      params.value = { ...params.value, Ex: 0, Ey: 0, B: 0.5, v0: 4, angle: 0, q: 1, g: 0 }
      break
    case 'parabolic':
      params.value = { ...params.value, Ex: 0, Ey: 0.5, B: 0, v0: 4, angle: 0, q: 1, g: 0 }
      break
    case 'selector':
      // v = E/B -> 设 v=4, B=0.5, 则 E 需要为 2. 方向要相反
      // F_E = qE (向下), F_B = qvB (向上, 若B垂直向外且v向右)
      params.value = { ...params.value, Ex: 0, Ey: 2, B: -0.5, v0: 4, angle: 0, q: 1, g: 0 }
      break
    case 'gravity_balance':
      // mg = qE
      params.value = { ...params.value, Ex: 0, Ey: -2, B: 0, v0: 0, angle: 0, q: 1, m: 5, g: 0.4 } // 需微调
      break
  }
})
  */
// 监听场景切换，应用经过计算的物理参数
watch(currentScenario, (val) => {
  resetSimulation()
  
  // 基础常数预设，方便计算
  // 假设 v0 = 5, m = 1, q = 1
  
  switch (val) {
    case 'custom':
      // 保持当前参数不变，或者重置为默认
      break

    case 'circle': 
      // 场景：纯磁场 - 匀速圆周运动
      // 目标：半径 R = mv/qB。
      // 设 v=5, m=1, q=1. 为了让半径适中 (例如 R=50像素)，B 应该为 0.1
      // R = 5 / 0.1 = 50. 
      // 之前的 B=0.5 会导致 R=10 (太小，像在原地打转)
      
      Object.assign(params.value, {
        q: 1, m: 1, 
        v0: 5, angle: 0, 
        Ex: 0, Ey: 0, 
        B: 0.1, // 修正点：减小磁场以增大半径
        g: 0 
      })
      break

    case 'parabolic': 
      // 场景：纯电场 - 类平抛
      // Y方向偏移 y = 0.5 * (qE/m) * t^2
      // 为了让曲线明显但不过快飞出，E 取 0.5 左右
      Object.assign(params.value, {
        q: 1, m: 1, 
        v0: 5, angle: 0, 
        Ex: 0, Ey: 0.5, // Y轴向下加速
        B: 0, 
        g: 0 
      })
      
      break

    case 'selector': 
      // 场景：速度选择器 (关键修正)
      // 原理：电场力与洛伦兹力平衡。
      // 公式：qE + q(v × B) = 0  =>  E = vB (标量大小)
      // 设定：v0 = 5, B = 0.5 (方向向外, B>0)
      // 洛伦兹力 F_Lorentz 方向：
      // v(右) x B(外) = 力(下, Canvas +Y). 
      // 修正逻辑：代码中 Fby = -q * vx * B. 若 v=5, B=0.5 -> Fby = -2.5 (负号代表向上? 需核对坐标系)
      
      // 重新核对坐标系逻辑：
      // 代码: Fby = -q * vx * B. 
      // 若 q=1, vx=5, B=0.5 => Fby = -2.5. (这是向上的力，因为Canvas Y向下增加)
      // 为了平衡，我们需要一个向下的电场力 F_E = +2.5.
      // 所以 Ey 必须为 2.5
      Object.assign(params.value, {
        q: 1, m: 1, 
        v0: 5, angle: 0, 
        Ex: 0, 
        Ey: 2.5, // 修正点：E = v * B = 5 * 0.5 = 2.5
        B: 0.5, 
        g: 0 
      })
      
      break

    case 'linear': 
      // 场景：复合场直线加速
      // 这是一个特例，通常指 E 和 B 平行，或者 v 平行于 B 导致不受洛伦兹力
      // 这里我们模拟 v // B 的情况
      
      Object.assign(params.value, {
        q: 1, m: 1, 
        v0: 2, angle: 0, 
        Ex: 1, // X方向电场加速
        Ey: 0, 
        B: 0, // 无磁场干扰，或者 B 平行于 v (暂不支持 Bx，这里设为0)
        g: 0 
      })
      break

    case 'gravity_balance': 
      // 场景：密立根油滴 / 重力平衡
      // 原理：mg = qE
      // 重力 Fg = m * g (向下, +Y)
      // 电场力 Fe = q * Ey (需要向上, -Y)
      // 设定 m=1, g=1 => Fg = 1
      // 需要 q*Ey = -1. 设 q=1, 则 Ey = -1
      
      Object.assign(params.value, {
        q: 1, m: 1, 
        v0: 3, angle: 0, // 给一点初速度便于观察水平漂移
        Ex: 0, 
        Ey: -1, // 修正点：负值代表向上的电场力
        B: 0, 
        g: 1 
      })
      break
  }
  
  // 强制刷新一次加速度计算，防止初始跳变
  setTimeout(() => {
    particle.value.vx = params.value.v0 * Math.cos(params.value.angle * Math.PI / 180)
    particle.value.vy = params.value.v0 * Math.sin(params.value.angle * Math.PI / 180)
  }, 0)
  }
  
)
// 1. 新增：纯函数，用于计算特定状态下的加速度
// state = { vx, vy } (位置对加速度没影响，除非场不均匀，这里简化处理)
const getAcceleration = (vx: number, vy: number) => {
  const { q, m, Ex, Ey, B, g } = params.value
  
  // 电场力
  const Fex = q * Ex
  const Fey = q * Ey
  
  // 磁场力 (洛伦兹力)
  const Fbx = q * vy * B
  const Fby = -q * vx * B
  
  // 重力
  const Fgy = m * g

  return {
    ax: (Fex + Fbx) / m,
    ay: (Fey + Fby + Fgy) / m
  }
}

// --- 物理引擎核心 (Runge-Kutta 或者是简单的 Euler 积分) ---
// 为了前端性能和代码可读性，这里使用半隐式欧拉法 (Semi-implicit Euler)，稳定性优于标准欧拉
//精度不够 Euler法被弃用
/* 
  const updatePhysics = () => {
  if (!isRunning.value) return

  const dt = 0.1 * ANIMATION_SPEED
  const p = particle.value
  const { q, m, Ex, Ey, B, g } = params.value

  // 1. 计算合力 F
  // 电场力 F_E
  const Fex = q * Ex
  const Fey = q * Ey
  
  // 磁场力 (洛伦兹力) F_B = q(v x B)
  // 设 B 垂直于 xy 平面。若 B > 0 指向屏幕外。
  // v x B => (vy * Bz, -vx * Bz, 0)
  // Fbx = q * vy * B
  // Fby = -q * vx * B
  const Fbx = q * p.vy * B
  const Fby = -q * p.vx * B

  // 重力
  const Fgy = m * g 

  // 牛顿第二定律 a = F/m
  const ax = (Fex + Fbx) / m
  const ay = (Fey + Fby + Fgy) / m

  // 2. 更新速度
  p.vx += ax * dt
  p.vy += ay * dt

  // 3. 更新位置
  p.x += p.vx * dt
  p.y += p.vy * dt

  // --- 新增：能量计算核心逻辑 ---
  // Ek = 0.5 * m * v^2
  const vSq = p.vx * p.vx + p.vy * p.vy
  const ek = 0.5 * m * vSq


  // 限制历史数据长度，与画布宽度对应
  if (time.value % 2 === 0) { // 适当降低采样率平滑曲线
    energyHistory.value.push(ek)
    if (energyHistory.value.length > 400) energyHistory.value.shift()
  }
  // ---------------------------
  // 4. 记录轨迹 (Ni: 模式识别需要可视化的历史数据)
  if (time.value % 5 === 0) { // 减少记录频率优化性能
    p.path.push({ x: p.x, y: p.y })
    if (p.path.length > TRAJECTORY_limit) p.path.shift()
  }

  time.value++
  draw()
  drawEnergy() // <--- 新增：调用能量绘图
  animationId = requestAnimationFrame(updatePhysics)
}
  */
// 2. 替换：使用 RK4 算法的物理更新循环
const updatePhysics = () => {
  if (!isRunning.value) return

  const dt = 0.1 * ANIMATION_SPEED // 时间步长
  const p = particle.value

  // --- RK4 核心算法 ---
  // k1: 当前时刻的斜率
  const a1 = getAcceleration(p.vx, p.vy)
  const k1_vx = a1.ax * dt
  const k1_vy = a1.ay * dt
  const k1_x = p.vx * dt
  const k1_y = p.vy * dt

  // k2: 试探半步 (dt/2) 后的斜率
  const a2 = getAcceleration(p.vx + k1_vx / 2, p.vy + k1_vy / 2)
  const k2_vx = a2.ax * dt
  const k2_vy = a2.ay * dt
  const k2_x = (p.vx + k1_vx / 2) * dt
  const k2_y = (p.vy + k1_vy / 2) * dt

  // k3: 再次试探半步 (用 k2修正)
  const a3 = getAcceleration(p.vx + k2_vx / 2, p.vy + k2_vy / 2)
  const k3_vx = a3.ax * dt
  const k3_vy = a3.ay * dt
  const k3_x = (p.vx + k2_vx / 2) * dt
  const k3_y = (p.vy + k2_vy / 2) * dt

  // k4: 试探整步 (dt) 后的斜率
  const a4 = getAcceleration(p.vx + k3_vx, p.vy + k3_vy)
  const k4_vx = a4.ax * dt
  const k4_vy = a4.ay * dt
  const k4_x = (p.vx + k3_vx) * dt
  const k4_y = (p.vy + k3_vy) * dt

  // 加权平均，更新状态
  p.vx += (k1_vx + 2 * k2_vx + 2 * k3_vx + k4_vx) / 6
  p.vy += (k1_vy + 2 * k2_vy + 2 * k3_vy + k4_vy) / 6
  p.x += (k1_x + 2 * k2_x + 2 * k3_x + k4_x) / 6
  p.y += (k1_y + 2 * k2_y + 2 * k3_y + k4_y) / 6
  // -------------------

  // 计算能量 (Te: 验证数据)
  const vSq = p.vx * p.vx + p.vy * p.vy
  const ek = 0.5 * params.value.m * vSq
  
  if (time.value % 2 === 0) {
    energyHistory.value.push(ek)
    if (energyHistory.value.length > 400) energyHistory.value.shift()
  }

  // 记录轨迹
  if (time.value % 5 === 0) {
    p.path.push({ x: p.x, y: p.y })
    if (p.path.length > TRAJECTORY_limit) p.path.shift()
  }

  time.value++
  draw()
  drawEnergy()
  animationId = requestAnimationFrame(updatePhysics)
}

// 4. 新增：绘制能量曲线的函数 (Te: 自适应坐标系)
const drawEnergy = () => {
  const ctx = energyCanvasRef.value?.getContext('2d')
  if (!ctx || !energyCanvasRef.value) return
  
  const w = energyCanvasRef.value.width
  const h = energyCanvasRef.value.height
  const history = energyHistory.value

  // 清空画布
  ctx.fillStyle = '#18181c' // 比主背景稍亮
  ctx.fillRect(0, 0, w, h)

  if (history.length < 2) return

  // 计算 Y 轴比例 (Auto-scaling)
  // 找出当前最大动能，作为图表的顶端 (留出 20% padding)
  const maxEk = Math.max(...history, 1) // 避免除以0
  const minEk = Math.min(...history)
  const range = maxEk * 1.2 

  // 绘制网格线
  ctx.strokeStyle = '#333'
  ctx.lineWidth = 0.5
  ctx.beginPath(); ctx.moveTo(0, h/2); ctx.lineTo(w, h/2); ctx.stroke(); // 中线

  // 绘制曲线
  ctx.beginPath()
  ctx.strokeStyle = '#f2c97d' // Naive Warning Color (黄色代表能量)
  ctx.lineWidth = 2

  // 将历史数据映射到画布坐标
  // x轴: 0 -> w
  // y轴: 0 -> range (注意 canvas y轴向下，需要翻转)
  const stepX = w / 400 // 假设最大存储400个点

  history.forEach((ek, i) => {
    const x = i * stepX
    // 归一化 Y 值
    const normalizedY = 1 - (ek / range) 
    const y = normalizedY * h
    
    if (i === 0) ctx.moveTo(x, y)
    else ctx.lineTo(x, y)
  })
  ctx.stroke()

  // 绘制填充 (让图表看起来更现代)
  ctx.lineTo(history.length * stepX, h)
  ctx.lineTo(0, h)
  ctx.closePath()
  ctx.fillStyle = 'rgba(242, 201, 125, 0.1)'
  ctx.fill()

  // 绘制数值标签 (HUD)
  ctx.font = '12px monospace'
  ctx.fillStyle = '#f2c97d'
  const currentEk = history[history.length - 1] || 0
  ctx.fillText(`Ek: ${currentEk.toFixed(2)} J`, 10, 20)
  ctx.fillStyle = '#666'
  ctx.fillText(`Max: ${maxEk.toFixed(2)}`, w - 80, 20)
  ctx.fillText(`Min: ${minEk.toFixed(2)}`, w - 80, 35)
}


// --- 渲染逻辑 ---
const draw = () => {
  const ctx = canvasRef.value?.getContext('2d')
  if (!ctx || !canvasRef.value) return
  
  const width = canvasRef.value.width
  const height = canvasRef.value.height

  // 清空背景
  ctx.fillStyle = '#101014' // Naive UI Dark bg
  ctx.fillRect(0, 0, width, height)

  // 绘制网格 (Te: 坐标系参照)
  drawGrid(ctx, width, height)

  // 绘制场区指示 (Ni: 可视化不可见场)
  drawFieldIndicators(ctx, width, height)

  // 绘制轨迹
  ctx.beginPath()
  ctx.strokeStyle = '#63e2b7' // Naive Primary Color
  ctx.lineWidth = 2
  particle.value.path.forEach((pos, index) => {
    if (index === 0) ctx.moveTo(pos.x, pos.y)
    else ctx.lineTo(pos.x, pos.y)
  })
  ctx.stroke()

  // 绘制粒子
  ctx.beginPath()
  ctx.arc(particle.value.x, particle.value.y, 6, 0, Math.PI * 2)
  ctx.fillStyle = params.value.q > 0 ? '#ff6666' : '#66ccff' // 正负电荷颜色区分
  ctx.fill()
  ctx.strokeStyle = '#fff'
  ctx.stroke()
}

const drawGrid = (ctx: CanvasRenderingContext2D, w: number, h: number) => {
  ctx.strokeStyle = '#333'
  ctx.lineWidth = 0.5
  for (let x = 0; x <= w; x += 50) {
    ctx.beginPath(); ctx.moveTo(x, 0); ctx.lineTo(x, h); ctx.stroke()
  }
  for (let y = 0; y <= h; y += 50) {
    ctx.beginPath(); ctx.moveTo(0, y); ctx.lineTo(w, y); ctx.stroke()
  }
}

const drawFieldIndicators = (ctx: CanvasRenderingContext2D, _w: number, _h: number) => {
  ctx.font = '14px sans-serif'
  ctx.fillStyle = '#aaa'
  
  // 显示 B 场方向
  const bText = params.value.B > 0 ? '⊗ 磁场向里' : (params.value.B < 0 ? '⊙ 磁场向外' : '无磁场')
  ctx.fillText(`B: ${bText}`, 20, 30)

  // 显示 E 场方向
  let eText = '无电场'
  if (params.value.Ex !== 0 || params.value.Ey !== 0) eText = `E: (${params.value.Ex}, ${params.value.Ey})`
  ctx.fillText(eText, 20, 50)
}

// --- 控制逻辑 ---
const startSimulation = () => {
  if (isRunning.value) return
  
  // 智能判断：如果当前粒子已经飞出了边界，或者跑了很远，
  // 用户点"开始"可能其实是想"重来"。
  const canvas = canvasRef.value
  if (canvas) {
    const p = particle.value
    const isOut = p.x < 0 || p.x > canvas.width || p.y < 0 || p.y > canvas.height
    // 如果粒子出界了，点开始自动重置
    if (isOut) {
      resetSimulation()
    }
  }
  
  isRunning.value = true
  updatePhysics()
}
const pauseSimulation = () => {
  isRunning.value = false
  if (animationId) cancelAnimationFrame(animationId)
}

const resetSimulation = () => {
  pauseSimulation()
  time.value = 0
  energyHistory.value = [] // <--- 新增
  // 根据角度计算初始 vx, vy
  const rad = params.value.angle * (Math.PI / 180)
  particle.value = {
    x: 100, // 初始位置固定在左侧便于观察
    y: 300,
    vx: params.value.v0 * Math.cos(rad),
    vy: params.value.v0 * Math.sin(rad),
    path: []
  }
  setTimeout(draw, 10) // 确保各种状态更新后重绘一次
}


// --- 修复：参数响应逻辑 ---

// 分类 1: 初始运动学参数 (改变这些应当立即重置回到起点)
watch(
  [() => params.value.v0, () => params.value.angle, () => params.value.m, () => params.value.q], 
  () => {
    // 只有当参数发生实质变化时才触发
    // 逻辑：用户改变了发射条件，默认他们想看新的发射结果，所以自动重置
    resetSimulation() 
    // 可选：如果希望改参数后自动开始跑，可以加上 startSimulation()
    // 但通常保持暂停让用户看清初始状态更好
  }
)

// 分类 2: 场强参数 (改变这些应当即时生效，无需重置)
// 实际上 updatePhysics 已经在实时读取了，但为了确保 UI 也就是数值输入框的
// "Enter" 或 "Blur" 行为能正确同步，我们可以强制 Vue 进行一次重绘
watch(
  [() => params.value.Ex, () => params.value.Ey, () => params.value.B, () => params.value.g],
  () => {
    // 如果处于暂停状态，我们需要手动画一帧，否则用户看不到场强指示器的文字变化
    if (!isRunning.value) {
      draw() 
    }
  }
)

onMounted(() => {
  resetSimulation()
  draw()
  drawEnergy()
  
})
</script>

<template>
  <NConfigProvider :theme="darkTheme">
    <NGlobalStyle />
    <NLayout has-sider style="height: 100vh">
      
      <NLayoutSider 
        width="320" 
        content-style="padding: 20px;" 
        bordered
      >
        <NSpace vertical size="large">
          <div class="header">
            <h2>Physics Lab <span style="font-size:0.6em; color:#63e2b7">drHo Edition</span></h2>
            <p style="opacity: 0.6; font-size: 0.8rem">带电粒子在复合场中的运动</p>
          </div>

          <NCard title="Scenario (场景)" size="small">
            <NSelect v-model:value="currentScenario" :options="scenarios" />
          </NCard>

          <NCard title="Parameters (参数)" size="small">
            <NForm label-placement="left" label-width="auto" size="small">
              
              <NDivider title-placement="left">粒子属性</NDivider>
              <NFormItem label="电荷量 q">
                <NInputNumber v-model:value="params.q" :step="0.1" />
              </NFormItem>
              <NFormItem label="质量 m">
                <NInputNumber v-model:value="params.m" :min="0.1" :step="0.1" />
              </NFormItem>

              <NDivider title-placement="left">初始条件</NDivider>
              <NFormItem label="初速度 v0">
                <NSlider v-model:value="params.v0" :max="20" :step="0.05" />
              </NFormItem>
              <NFormItem label="入射角 θ">
                <NSlider v-model:value="params.angle" :min="-180" :max="180" :step="1" />
              </NFormItem>

              <NDivider title-placement="left">场强设置</NDivider>
              <NFormItem label="磁场 B">
                <NInputNumber v-model:value="params.B" :step="0.02" placeholder="正为向内" />
              </NFormItem>
              <NFormItem label="电场 Ex">
                <NInputNumber v-model:value="params.Ex" :step="0.02" />
              </NFormItem>
              <NFormItem label="电场 Ey">
                <NInputNumber v-model:value="params.Ey" :step="0.02" />
              </NFormItem>
              <NFormItem label="重力 g">
                <NInputNumber v-model:value="params.g" :step="0.02" />
              </NFormItem>
            </NForm>
          </NCard>

          <NSpace justify="space-between">
            <NButton type="primary" @click="startSimulation" :disabled="isRunning">开始 (Start)</NButton>
            <NButton type="warning" @click="pauseSimulation" :disabled="!isRunning">暂停 (Pause)</NButton>
            <NButton type="error" ghost @click="resetSimulation">重置 (Reset)</NButton>
          </NSpace>

          <NCard size="small" style="background: #26262a;">
            <div style="font-family: monospace; font-size: 0.8rem; color: #aaa;">
              <div>v_x: {{ particle.vx.toFixed(2) }} | v_y: {{ particle.vy.toFixed(2) }}</div>
              <div>x: {{ particle.x.toFixed(0) }} | y: {{ particle.y.toFixed(0) }}</div>
            </div>
          </NCard>

        </NSpace>
      </NLayoutSider>

      <NLayoutContent style="background-color: #101014; height: 100%; display: flex; flex-direction: column; align-items: center; justify-content: center; padding: 20px;">
  
  <div style="position: relative;">
    <canvas 
      ref="canvasRef" 
      width="1100" 
      height="500" 
      style="border: 1px solid #333; border-radius: 4px 4px 0 0; display: block;"
    ></canvas>
    <div style="position: absolute; bottom: 10px; right: 10px; pointer-events: none;">
      <NTag type="success" size="small" :bordered="false" style="opacity: 0.8">
        轨迹视图 (Trajectory)
      </NTag>
    </div>
  </div>

  <div style="position: relative; margin-top: -1px;">
    <canvas 
      ref="energyCanvasRef" 
      width="1100" 
      height="180" 
      style="border: 1px solid #333; border-radius: 0 0 4px 4px; display: block;"
    ></canvas>
    <div style="position: absolute; bottom: 10px; right: 10px; pointer-events: none;">
      <NTag type="warning" size="small" :bordered="false" style="opacity: 0.8">
        动能视图 (Kinetic Energy)
      </NTag>
    </div>
  </div>

</NLayoutContent>
    </NLayout>
  </NConfigProvider>
</template>

<style scoped>
.header h2 {
  margin: 0;
  font-weight: 300;
  letter-spacing: 1px;
}
</style>