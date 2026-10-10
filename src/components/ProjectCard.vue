<script setup>
import { computed } from 'vue'

// 这个组件只负责“展示一个项目”。
// 它自己不存数据，数据由父组件 App.vue 通过 props 传进来。
const props = defineProps({
  project: {
    type: Object,
    required: true,
  },
  // 这一条是不是正在被编辑。父组件说了算，卡片自己只是照着显示高亮。
  editing: {
    type: Boolean,
    default: false,
  },
})

// 声明这个组件会往外抛两个事件。
// 卡片自己不知道“删除”“编辑”该怎么做，它只负责喊一声，具体处理交给父组件。
const emit = defineEmits(['delete', 'edit'])

// 状态 → 样式名 的对照表。
// 以后想调整某个状态长什么样，只要改这张表（和下面的 CSS）就行。
const STATUS_STYLE = {
  进行中: 'doing',
  已完成: 'done',
  暂停: 'paused',
}

// 根据状态文字返回对应的 CSS 类名，用来显示不同颜色的标签
function statusClass(status) {
  // 表里查不到就用 other，保证标签不会出现“没有样式”的情况
  return STATUS_STYLE[status] || 'other'
}

/* -------------------- 项目 Logo --------------------
   每张卡片左上角有一个小 logo。它不用手动维护，是“算”出来的：

   - 图标：先拿项目名称 / 简介 / 技术栈去匹配关键词，命中就用对应图标；
           一个都没命中，就按名字算个数，从图标库里挑一个，保证不会没图标。
   - 配色：按项目 id 轮流取，所以相邻的几个项目颜色天然就不一样。

   这样做的好处：以后新增项目时什么都不用填，logo 会自动出现；
   而且同一个项目每次刷新拿到的 logo 都一样，是“稳定”的。
   -------------------------------------------------- */

// 图标库。每个图标只存里面的线条（<path> / <circle> 等），
// 外面那层 <svg> 由下面的 logoMarkup 统一套上，省得每个都写一遍。
const ICONS = {
  // 环保 / 分类
  leaf: '<path d="M11 20A7 7 0 0 1 9.8 6.1C15.5 5 17 4.48 19 2c1 2 2 4.18 2 8 0 5.5-4.78 10-10 10Z"/><path d="M2 21c0-3 1.85-5.36 5.08-6C9.5 14.52 12 13 13 12"/>',
  // 图书 / 教材 / 文档
  book: '<path d="M4 19.5A2.5 2.5 0 0 1 6.5 17H20"/><path d="M6.5 2H20v20H6.5A2.5 2.5 0 0 1 4 19.5v-15A2.5 2.5 0 0 1 6.5 2Z"/>',
  // 预约 / 日程 / 排期
  calendar:
    '<path d="M8 2v4M16 2v4"/><rect x="3" y="4" width="18" height="18" rx="2"/><path d="M3 10h18"/><path d="m9 15.5 2 2 4-4"/>',
  // 视觉 / 摄像头 / 图像识别
  camera:
    '<path d="M14.5 4h-5L7 7H4a2 2 0 0 0-2 2v8a2 2 0 0 0 2 2h16a2 2 0 0 0 2-2V9a2 2 0 0 0-2-2h-3l-2.5-3Z"/><circle cx="12" cy="13" r="3"/>',
  // 硬件 / 嵌入式 / 传感器
  cpu: '<rect x="4" y="4" width="16" height="16" rx="2"/><rect x="9" y="9" width="6" height="6" rx="1"/><path d="M9 2v2M15 2v2M9 20v2M15 20v2M2 9h2M2 15h2M20 9h2M20 15h2"/>',
  // AI / 机器人 / 算法
  robot:
    '<path d="M12 8V4H8"/><rect x="4" y="8" width="16" height="12" rx="2"/><path d="M2 14h2M20 14h2M15 13v2M9 13v2"/>',
  // 数据 / 分析 / 统计
  chart: '<path d="M3 3v18h18"/><path d="M8 17v-5M13 17V8M18 17v-8"/>',
  // 网站 / 平台 / 网络
  globe:
    '<circle cx="12" cy="12" r="10"/><path d="M2 12h20"/><path d="M12 2a15.3 15.3 0 0 1 4 10 15.3 15.3 0 0 1-4 10 15.3 15.3 0 0 1-4-10 15.3 15.3 0 0 1 4-10Z"/>',
  // 实验 / 科研 / 检测
  flask:
    '<path d="M9 3h6"/><path d="M10 3v6.5L4.8 18a2 2 0 0 0 1.7 3h11a2 2 0 0 0 1.7-3L14 9.5V3"/><path d="M7.5 14h9"/>',
  // 系统 / 管理 / 平台
  package:
    '<path d="M21 8a2 2 0 0 0-1-1.73l-7-4a2 2 0 0 0-2 0l-7 4A2 2 0 0 0 3 8v8a2 2 0 0 0 1 1.73l7 4a2 2 0 0 0 2 0l7-4A2 2 0 0 0 21 16Z"/><path d="m3.3 7 8.7 5 8.7-5"/><path d="M12 22V12"/>',
  // 安全 / 权限 / 隐私
  shield:
    '<path d="M20 13c0 5-3.5 7.5-7.66 8.95a1 1 0 0 1-.67-.01C7.5 20.5 4 18 4 13V6a1 1 0 0 1 1-1c2 0 4.5-1.2 6.24-2.72a1.17 1.17 0 0 1 1.52 0C14.51 3.81 17 5 19 5a1 1 0 0 1 1 1Z"/>',
  // 代码 / 编程 / 开发
  code: '<path d="m16 18 6-6-6-6"/><path d="m8 6-6 6 6 6"/>',
}

// 关键词 → 图标名 的对照表。
// 顺序有讲究：越具体的越往前放，先匹配到谁就用谁。
// 比如“智能垃圾分类助手”同时含“智能”和“垃圾”，因为“垃圾”排在前面，所以会用叶子图标。
const KEYWORD_ICONS = [
  [['垃圾', '分类', '回收', '环保', '低碳', '碳排'], 'leaf'],
  [['书', '教材', '图书', '阅读', '文档', '笔记'], 'book'],
  [['预约', '排期', '日程', '考勤', '会议', '值班'], 'calendar'],
  [['摄像', '视觉', '识别', '图像', '拍照', '监控', '视频'], 'camera'],
  [['硬件', '嵌入式', '单片机', '传感', '电路', '芯片', 'arduino', '树莓'], 'cpu'],
  [['机器人', 'robot', '智能', '算法', '模型', '神经', '人工智能'], 'robot'],
  [['数据', '分析', '统计', '报表', '可视化', '图表', '指标', '推荐'], 'chart'],
  [['实验', '化学', '材料', '生物', '科研', '检测', '测量'], 'flask'],
  [['安全', '加密', '隐私', '权限', '防护', '认证', '鉴权'], 'shield'],
  [['代码', '编程', '开发', '开源', 'github', '重构', '插件'], 'code'],
  [['系统', '管理', '平台', '教务', '库存', '后台', '网站'], 'package'],
]

// 12 套渐变色（都是“浅 → 深”的一对颜色）。
// 按项目 id 轮流取，id 是唯一的，所以相邻项目基本不会撞色。
// 顺序是特意排的：前几个颜色对上了最常见的图标语义
// （叶子配绿、书配紫、日历配粉、摄像头配蓝），默认的几条示例看起来更顺眼。
const LOGO_PALETTES = [
  ['#4ade80', '#059669'], // 绿  → 环保 / 分类
  ['#a78bfa', '#7c3aed'], // 紫  → 图书 / 文档
  ['#f472b6', '#db2777'], // 粉  → 日程 / 预约
  ['#5b9dff', '#2563eb'], // 蓝  → 视觉 / 平台
  ['#fb923c', '#ea580c'], // 橙  → 硬件 / 系统
  ['#22d3ee', '#0e7490'], // 青  → AI / 算法
  ['#fbbf24', '#c2740e'], // 琥珀 → 数据 / 分析
  ['#fb7185', '#e11d48'], // 玫红 → 安全 / 检测
  ['#a3e635', '#4d7c0f'], // 黄绿 → 实验 / 科研
  ['#818cf8', '#4338ca'], // 靛蓝 → 代码 / 开发
  ['#2dd4bf', '#0f766e'], // 蓝绿 → 绿色科技
  ['#f87171', '#b91c1c'], // 红  → 告警 / 风险
]

// 一个很简单的字符串“哈希”：把文字变成一个数字。
// 要求是“同一个名字每次都得到同一个数”，这样 logo 才不会每次刷新都换。
function hashText(text) {
  let hash = 0
  for (let i = 0; i < text.length; i += 1) {
    hash = (hash * 31 + text.charCodeAt(i)) % 1000000007
  }
  return hash
}

// 决定这个项目用哪个图标
function pickIconName(project) {
  // 把名称、简介、技术栈拼成一段文字，一起拿去匹配关键词。
  // toLowerCase 是为了让 "AI"、"Vue" 这种大小写不敏感。
  const text = [project.name, project.description, ...(project.techStack || [])]
    .join(' ')
    .toLowerCase()

  for (const [keywords, icon] of KEYWORD_ICONS) {
    // some 的意思是“只要有一个关键词命中就算命中”
    if (keywords.some((keyword) => text.includes(keyword))) return icon
  }

  // 一个关键词都没命中：按名字算个数，从图标库里挑一个，至少不会没图标
  const names = Object.keys(ICONS)
  return names[hashText(project.name || '') % names.length]
}

// 拼出完整的 SVG 代码：外面那层 <svg> 由这里统一加。
// v-html 会把这段文字当成 HTML 插进页面里（内容是我们自己写死的，所以是安全的）。
// 注意：width/height 直接写在标签上，因为 v-html 插进来的元素
//       拿不到 scoped 样式（Vue 只给模板里写死的元素加 data-v 属性）。
const logoMarkup = computed(
  () =>
    `<svg viewBox="0 0 24 24" width="22" height="22" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">${
      ICONS[pickIconName(props.project)]
    }</svg>`,
)

// logo 的背景：从配色表里按 id 挑一套，做成左上到右下的斜向渐变
const logoStyle = computed(() => {
  const id = Number(props.project.id) || 0
  const total = LOGO_PALETTES.length

  // 先取整，再取余；多加一个 total 是为了让负数也能落到正确的下标上
  const index = (((Math.trunc(id) - 1) % total) + total) % total
  const [from, to] = LOGO_PALETTES[index]

  return { backgroundImage: `linear-gradient(135deg, ${from}, ${to})` }
})
</script>

<template>
  <!-- editing 为 true 时加一个 editing 类，给卡片描个高亮边 -->
  <article class="card" :class="{ editing }">
    <!-- 第一行：logo + 项目名称 + 状态标签 -->
    <div class="card-head">
      <!-- 项目 logo：颜色和图标都是按项目自动算出来的 -->
      <span class="logo" :style="logoStyle" v-html="logoMarkup" aria-hidden="true"></span>
      <h2 class="name">{{ project.name }}</h2>
      <span class="status" :class="statusClass(project.status)">{{ project.status }}</span>
    </div>

    <!-- 项目简介 -->
    <p class="desc">{{ project.description }}</p>

    <!-- 技术栈：用循环把数组里的每一项渲染成一个小标签 -->
    <div class="tech">
      <span v-for="tech in project.techStack" :key="tech" class="tech-tag">{{ tech }}</span>
    </div>

    <!-- 项目成员 + GitHub 地址 -->
    <div class="meta">
      <div class="meta-row">
        <span class="meta-label">项目成员</span>
        <span>{{ project.members.join('、') }}</span>
      </div>
      <div class="meta-row">
        <span class="meta-label">GitHub</span>
        <a :href="project.github" target="_blank" rel="noreferrer">{{ project.github }}</a>
      </div>
    </div>

    <!-- 编辑 / 删除按钮：点击后各自抛出一个事件，具体怎么做交给父组件 -->
    <div class="card-foot">
      <button type="button" class="btn-edit" @click="emit('edit')">编辑</button>
      <button type="button" class="btn-delete" @click="emit('delete')">删除</button>
    </div>
  </article>
</template>

<style scoped>
/* scoped 表示这些样式只对这个组件生效，不会影响别的文件。
   颜色一律用 style.css 里定义好的变量，这样深色模式会自动生效，这里不用写两遍。 */
.card {
  display: flex;
  flex-direction: column;
  gap: 12px;
  padding: 20px;
  background: var(--bg-panel);
  border: 1px solid var(--border);
  border-radius: 12px;
  transition:
    background-color 0.25s ease,
    border-color 0.2s ease,
    box-shadow 0.2s ease;
}

.card:hover {
  border-color: var(--border-hover);
}

/* 正在编辑的那张卡片描个蓝色高亮，和上面打开的表单对应起来 */
.card.editing {
  border-color: var(--accent);
  box-shadow: 0 0 0 3px rgba(120, 170, 255, 0.25);
}

.card-head {
  display: flex;
  align-items: center;
  gap: 12px;
}

/* ---- logo 小方块 ---- */
.logo {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  width: 40px;
  height: 40px;
  /* 图标线条用白色，currentColor 会继承这里的颜色 */
  color: #ffffff;
  border-radius: 11px;
  box-shadow:
    /* 顶部一条淡淡的高光，让方块看起来有点立体 */
    inset 0 1px 0 rgba(255, 255, 255, 0.28),
    /* 外面一点点投影，把方块从卡片上“托”起来 */
    0 2px 6px rgba(15, 23, 42, 0.18);
}

/* :deep() 是 Vue 的“穿透”写法。
   因为 svg 是用 v-html 插进来的，它身上没有 data-v 标记，
   普通的 .logo svg 选择器匹配不到它，必须用 :deep() 才能选中。 */
.logo :deep(svg) {
  width: 22px;
  height: 22px;
  /* 加一道很轻的阴影，保证白色线条压在浅色渐变上也看得清 */
  filter: drop-shadow(0 1px 1px rgba(0, 0, 0, 0.25));
}

.name {
  /* flex: 1 让标题占满剩下的位置，把状态标签挤到最右边 */
  flex: 1;
  /* min-width: 0 是为了超长的项目名能正常换行，而不是把卡片撑破 */
  min-width: 0;
  margin: 0;
  font-size: 17px;
  font-weight: 600;
}

.status {
  display: inline-flex;
  align-items: center;
  gap: 5px;
  flex-shrink: 0;
  padding: 3px 10px;
  font-size: 12px;
  font-weight: 500;
  border-radius: 999px;
  white-space: nowrap;
}

/* 状态前面的小圆点。
   currentColor 表示“跟随文字颜色”，所以不用为每种状态单独写一遍圆点颜色。 */
.status::before {
  content: '';
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: currentColor;
}

/* 进行中：蓝 */
.status.doing {
  color: var(--status-doing-text);
  background: var(--status-doing-bg);
}

/* 已完成：绿 */
.status.done {
  color: var(--status-done-text);
  background: var(--status-done-bg);
}

/* 暂停：橙 */
.status.paused {
  color: var(--status-paused-text);
  background: var(--status-paused-bg);
}

/* 兜底：万一出现预料之外的状态值，也别让它没有样式 */
.status.other {
  color: var(--status-other-text);
  background: var(--status-other-bg);
}

.desc {
  margin: 0;
  font-size: 14px;
  color: var(--text-secondary);
}

.tech {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}

.tech-tag {
  padding: 2px 8px;
  font-size: 12px;
  color: var(--text-secondary);
  background: var(--bg-subtle);
  border: 1px solid var(--border);
  border-radius: 6px;
}

.meta {
  display: flex;
  flex-direction: column;
  gap: 6px;
  padding-top: 12px;
  border-top: 1px solid var(--divider);
  font-size: 13px;
  color: var(--text-secondary);
}

.meta-row {
  display: flex;
  gap: 8px;
}

.meta-label {
  flex-shrink: 0;
  color: var(--text-muted);
}

.meta a {
  color: var(--link);
  word-break: break-all;
}

.card-foot {
  display: flex;
  justify-content: flex-end;
  gap: 8px;
}

/* 编辑和删除长得像一对，只有悬停时的颜色不一样 */
.btn-edit,
.btn-delete {
  padding: 4px 12px;
  font-family: inherit;
  font-size: 13px;
  background: var(--bg-panel);
  border: 1px solid var(--border);
  border-radius: 6px;
  cursor: pointer;
  transition:
    color 0.2s ease,
    background-color 0.2s ease,
    border-color 0.2s ease;
}

.btn-edit {
  color: var(--text-secondary);
}

.btn-edit:hover {
  color: var(--text-primary);
  background: var(--bg-subtle);
  border-color: var(--border-hover);
}

.btn-delete {
  color: var(--danger);
}

.btn-delete:hover {
  color: var(--btn-text);
  background: var(--danger-bg);
  border-color: var(--danger-bg);
}
</style>
