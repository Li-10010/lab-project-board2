<script setup>
import { computed, nextTick, ref, watch } from 'vue'
import ProjectCard from './components/ProjectCard.vue'
import ProjectForm from './components/ProjectForm.vue'

// localStorage 里用这个“钥匙名”来区分我们的数据。
// 浏览器里可能存了很多网站的东西，靠这个名字知道哪份是自己的。
const STORAGE_KEY = 'lab-project-board:projects'

// 主题的存取用另一个"钥匙名"，和项目数据互不干扰。
const THEME_KEY = 'lab-project-board:theme'

// 页面第一次打开时该用哪套颜色：
// 1) 以前手动选过 → 用记住的那套；
// 2) 没选过 → 跟着系统的偏好走（电脑是深色模式就用深色）。
function getInitialTheme() {
  const saved = localStorage.getItem(THEME_KEY)
  if (saved === 'light' || saved === 'dark') return saved

  // matchMedia 相当于问浏览器一句："系统现在是不是深色模式？"
  const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches
  return prefersDark ? 'dark' : 'light'
}

// 当前主题，只有 'light' 和 'dark' 两种取值
const theme = ref(getInitialTheme())

// 主题一变，就把 data-theme 写到 <html> 上（变成 <html data-theme="dark">）。
// style.css 里已经写好了：data-theme 是 dark 时，整套颜色变量换成深色的那套。
// immediate: true 表示"一开始就先执行一次"，保证页面加载完就已经是正确的主题。
watch(
  theme,
  (value) => {
    document.documentElement.dataset.theme = value
  },
  { immediate: true },
)

// 用户手动点"浅色 / 深色"时调用：切换主题，并把选择记下来，下次打开还记得。
function setTheme(value) {
  theme.value = value
  localStorage.setItem(THEME_KEY, value)
}

// 如果用户还没手动选过，就一直跟着系统走：系统切深色，页面也跟着切。
// 一旦手动选过（localStorage 里有记录），就以用户的选择为准，不再被系统覆盖。
window.matchMedia('(prefers-color-scheme: dark)').addEventListener('change', (event) => {
  if (localStorage.getItem(THEME_KEY)) return
  theme.value = event.matches ? 'dark' : 'light'
})

// 3 个示例项目。
// 它们只在“浏览器里还没存过任何数据”时（也就是第一次打开这个网站）才会用到。
const defaultProjects = [
  {
    id: 1,
    name: '智能垃圾分类助手',
    description: '用摄像头识别垃圾种类，帮助同学在实验室正确分类投放。',
    members: ['张明', '李小雨'],
    techStack: ['Python', 'OpenCV', 'Flask'],
    status: '进行中',
    github: 'https://github.com/example/smart-trash',
  },
  {
    id: 2,
    name: '校园二手书交易平台',
    description: '面向本校学生的二手教材买卖小程序，支持发布、搜索与站内联系。',
    members: ['王浩', '陈思', '刘一诺'],
    techStack: ['Vue 3', 'Node.js', 'MySQL'],
    status: '暂停',
    github: 'https://github.com/example/campus-book',
  },
  {
    id: 3,
    name: '实验室设备预约系统',
    description: '在线查看设备空闲时段并预约，减少线下排队和重复登记。',
    members: ['赵珂'],
    techStack: ['React', 'Express', 'MongoDB'],
    status: '已完成',
    github: 'https://github.com/example/lab-booking',
  },
]

// 老数据兼容：以前的状态里有“规划中”，现在状态固定成三个了。
// 读到旧值就顺手换成新状态，免得卡片上冒出第四个状态。
const LEGACY_STATUS = { 规划中: '暂停' }

function normalizeProjects(list) {
  return list.map((project) => ({
    ...project,
    status: LEGACY_STATUS[project.status] || project.status,
  }))
}

// 页面打开时：先看 localStorage 里有没有存过数据，有就用存的，没有就用上面那 3 个示例。
function loadProjects() {
  const saved = localStorage.getItem(STORAGE_KEY)

  // 第一次打开时读不到东西，saved 是 null
  if (!saved) return defaultProjects

  try {
    // localStorage 里存的是字符串，要解析回真正的数组才能用
    return normalizeProjects(JSON.parse(saved))
  } catch (error) {
    // 万一存的内容坏了，不要让整个页面打不开，退回默认数据就好
    console.warn('读取本地数据失败，已改用默认数据：', error)
    return defaultProjects
  }
}

// 项目的源头数据：初始值来自 localStorage，或者第一次打开时的默认数据
const projects = ref(loadProjects())

// 只要 projects 发生变化（新增、修改、删除都算），就立刻存进 localStorage。
// 写在同一个地方，就不用“新增时存一次、修改时再存一次”，也就不会漏掉某一处。
watch(
  projects,
  (newProjects) => {
    // JSON.stringify 把数组变成字符串，因为 localStorage 只能存字符串
    localStorage.setItem(STORAGE_KEY, JSON.stringify(newProjects))
  },
  // deep: true 表示“数组里面某个项目的字段变了也算变化”，不只盯着数组本身
  { deep: true },
)

// 控制表单是否显示。新增和编辑共用这同一个表单，只是传进去的内容不一样。
const showForm = ref(false)

// 当前正在编辑的项目；null 表示不在编辑状态（也就是新增状态）。
// 表单靠它区分“我是在新增，还是在改某一条”。
const editingProject = ref(null)

// 表单所在的 DOM 位置，点“编辑”时用它滚过去
const formAnchor = ref(null)

// 表单在页面顶部，如果卡片在下面，点“编辑”后可能根本看不见表单，
// 所以打开表单后主动滚过去一下。
// 要等 nextTick，因为表单是 v-if 控制显隐的，这一刻 DOM 还没渲染出来。
async function scrollToForm() {
  await nextTick()
  formAnchor.value?.scrollIntoView({ behavior: 'smooth', block: 'center' })
}

// 点顶部的“新增项目”按钮
function toggleAddForm() {
  // 已经是新增状态了，再点一下就收起
  if (showForm.value && !editingProject.value) {
    showForm.value = false
    return
  }

  // 否则切回“新增模式”：清掉正在编辑的项目，再打开表单
  editingProject.value = null
  showForm.value = true
}

// 点卡片上的“编辑”：把那个项目交给表单去填，并打开表单
function handleEditProject(project) {
  editingProject.value = project
  showForm.value = true
  scrollToForm()
}

// 收起表单，同时退出编辑状态。
// 新增和编辑都靠它收尾，这样不用担心忘记清掉 editingProject。
function handleCancelForm() {
  showForm.value = false
  editingProject.value = null
}

// 表单提交后，子组件会把填好的数据传进来。
// 是编辑状态就改那一条，否则就新增一条。
function handleSubmitProject(data) {
  if (editingProject.value) {
    // 先记下 id。下面会把整个数组换成新的，到时候 editingProject 就不适合再读了。
    const editingId = editingProject.value.id

    // map 会生成一个新数组：id 对上的那条换成“旧内容 + 新内容”，其余原样保留。
    // 用 { ...project, ...data } 而不是直接改 project，
    // 这样既保住了 id，又不丢别的字段，也符合 Vue“数据要换新对象”的习惯。
    projects.value = projects.value.map((project) =>
      project.id === editingId ? { ...project, ...data } : project,
    )
  } else {
    // 新增：取现有最大的 id 再 +1，保证每个项目的 id 都不重复
    const maxId = projects.value.reduce((max, project) => Math.max(max, project.id), 0)

    // ...data 是展开运算符，意思是“把 data 里的字段都放进这个新对象”
    projects.value.push({ id: maxId + 1, ...data })
  }

  // 存好之后把表单收起来
  handleCancelForm()
}

// 删除项目：先简单确认一次，用户点“确定”才真的删
function handleDeleteProject(id) {
  // 先按 id 找到要删的那个项目，主要是为了在确认框里显示它的名字
  const target = projects.value.find((project) => project.id === id)
  if (!target) return

  // confirm 是浏览器自带的确认框：点“确定”返回 true，点“取消”返回 false
  const confirmed = confirm(`确定要删除「${target.name}」吗？`)
  if (!confirmed) return

  // filter 的意思是“留下所有 id 不等于这一项的元素”，效果就是把它去掉
  projects.value = projects.value.filter((project) => project.id !== id)

  // 如果删掉的正好是正在编辑的那条，顺手把表单收起来，
  // 免得对着一条已经不存在的数据编辑。
  if (editingProject.value?.id === id) handleCancelForm()
}

// 搜索关键词，和搜索框双向绑定。用户输入一个字，它就变一次。
const keyword = ref('')

// ---- 排序 ----
// 排序规则的存取用第三个“钥匙名”，和项目数据、主题互不干扰。
const SORT_KEY = 'lab-project-board:sort'
const SORT_DIR_KEY = 'lab-project-board:sort-dir'

// 当前用哪条规则排序。取值和 <select> 里的 option value 一一对应。
// 存过就用存过的，没存过默认按“添加顺序”（也就是项目在数组里的原始次序）。
const sortBy = ref(localStorage.getItem(SORT_KEY) || 'custom')

// 是升序还是降序。只有 'asc' / 'desc' 两种取值。
// 存过就用存过的，没存过默认升序。
const sortAsc = ref(localStorage.getItem(SORT_DIR_KEY) !== 'desc')

// 规则或方向一变就记下来，下次打开还是你选的那个。
watch([sortBy, sortAsc], () => {
  localStorage.setItem(SORT_KEY, sortBy.value)
  localStorage.setItem(SORT_DIR_KEY, sortAsc.value ? 'asc' : 'desc')
})

// 状态的“轻重”顺序：进行中要盯着 → 排前面，已完成收工了 → 排后面。
// 万一遇到预料之外的状态值，用 99 兜底，别让它乱插队。
const STATUS_ORDER = { 进行中: 0, 暂停: 1, 已完成: 2 }

// 比较两个项目，返回负数表示 a 排前面。
// 这里只算“升序”的结果，最后统一按方向翻转，省得每个分支都写两遍。
function compareProjects(a, b) {
  switch (sortBy.value) {
    case 'name':
      // localeCompare 配合 'zh-Hans-CN' 会按拼音排，而不是按字符编码排。
      // （按编码排的话“赵”会跑到“张”前面，明显不符合直觉）
      return a.name.localeCompare(b.name, 'zh-Hans-CN')

    case 'status':
      return (STATUS_ORDER[a.status] ?? 99) - (STATUS_ORDER[b.status] ?? 99)

    case 'members':
      // 老数据可能没有 members 字段，用可选链 + 兜底 0，避免报错
      return (a.members?.length ?? 0) - (b.members?.length ?? 0)

    case 'tech':
      return (a.techStack?.length ?? 0) - (b.techStack?.length ?? 0)

    case 'custom':
    default:
      return 0 // 返回 0 表示“不分先后”，交给下面保持原顺序
  }
}

// 最后要显示在页面上的项目 = 先按关键词筛，再按规则排。
// 拆成 filter 和 sort 两步，逻辑更清楚：搜索只管“留哪些”，排序只管“怎么放”。
const visibleProjects = computed(() => {
  // 第一步：筛。去掉前后空格并转小写，这样搜 "vue" 也能匹配到 "Vue 3"
  const text = keyword.value.trim().toLowerCase()
  const list = text
    ? // includes 判断“名称里是否包含这段文字”，所以只搜一部分也能找到
      projects.value.filter((project) => project.name.toLowerCase().includes(text))
    : // 复制一份再排，绝不直接改 projects（不然会连带写进 localStorage）
      [...projects.value]

  // 第二步：排。“添加顺序”就是什么也不做，保持数组原本的次序。
  if (sortBy.value === 'custom') return list

  // 现代浏览器的 sort 是“稳定排序”：两条项目比较结果为 0 时，它俩的先后不会被打乱。
  // 所以按状态排时，同一状态内部仍保持添加顺序，看起来很自然。
  const sorted = list.sort(compareProjects)

  // 降序就是把升序的结果整个倒过来。
  return sortAsc.value ? sorted : sorted.reverse()
})
</script>

<template>
  <div class="page">
    <!-- 页面标题区 -->
    <header class="page-head">
      <div>
        <h1>实验室项目展示板</h1>
        <p class="subtitle">共 {{ projects.length }} 个项目</p>
      </div>
      <div class="head-actions">
        <!-- 主题开关：两个按钮之间有一个滑块，点击就左右滑动 -->
        <div class="theme-switch" role="group" aria-label="切换配色主题">
          <!-- 滑块本身不显示文字，纯装饰，所以标了 aria-hidden -->
          <span class="theme-thumb" :class="{ dark: theme === 'dark' }" aria-hidden="true"></span>

          <button
            type="button"
            class="theme-option"
            :class="{ active: theme === 'light' }"
            :aria-pressed="theme === 'light'"
            title="浅色模式"
            @click="setTheme('light')"
          >
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
              <circle cx="12" cy="12" r="4" />
              <path d="M12 2v2M12 20v2M4.93 4.93l1.41 1.41M17.66 17.66l1.41 1.41M2 12h2M20 12h2M4.93 19.07l1.41-1.41M17.66 6.34l1.41-1.41" />
            </svg>
            <span>浅色</span>
          </button>

          <button
            type="button"
            class="theme-option"
            :class="{ active: theme === 'dark' }"
            :aria-pressed="theme === 'dark'"
            title="深色模式"
            @click="setTheme('dark')"
          >
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
              <path d="M21 12.79A9 9 0 1 1 11.21 3 7 7 0 0 0 21 12.79z" />
            </svg>
            <span>深色</span>
          </button>
        </div>

        <!-- 编辑状态下按钮文字也跟着变，不然用户会以为点了要新增 -->
        <button class="btn-add" @click="toggleAddForm">
          {{ showForm && !editingProject ? '收起' : '新增项目' }}
        </button>
      </div>
    </header>

    <!-- 工具栏：左边搜索、右边排序，一行放不下时窄屏会自动换行 -->
    <div class="toolbar">
      <!-- 搜索框：v-model 让输入内容和 keyword 实时同步，所以边打字边过滤 -->
      <input
        v-model="keyword"
        class="search"
        type="text"
        placeholder="搜索项目名称，例如：巡检"
      />

      <!-- 排序区：一个下拉选规则，右边一个按钮切换升 / 降序 -->
      <div class="sort-box">
        <label class="sort-label" for="sort-select">排序</label>

        <select id="sort-select" v-model="sortBy" class="sort-select">
          <option value="custom">添加顺序</option>
          <option value="name">名称</option>
          <option value="status">状态</option>
          <option value="members">成员数量</option>
          <option value="tech">技术栈数量</option>
        </select>

        <!-- 升 / 降序切换。按钮上只放一个箭头，靠翻转表示方向。 -->
        <button
          type="button"
          class="sort-dir"
          :class="{ desc: !sortAsc }"
          :title="sortAsc ? '当前升序，点击改为降序' : '当前降序，点击改为升序'"
          :aria-label="sortAsc ? '当前升序，点击改为降序' : '当前降序，点击改为升序'"
          @click="sortAsc = !sortAsc"
        >
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
            <path d="M12 19V5" />
            <path d="M5 12l7-7 7 7" />
          </svg>
        </button>
      </div>
    </div>

    <!-- 筛选结果提示：只有“搜了并且数量变少”时才出现，避免平时噪音 -->
    <p v-if="keyword.trim()" class="hint">
      找到 {{ visibleProjects.length }} 个匹配「{{ keyword.trim() }}」的项目
    </p>

    <!-- 表单：新增和编辑共用。
         :project 传 null 就是新增模式，传某个项目就是编辑它。
         formAnchor 只是用来在点“编辑”时把页面滚到表单这里。 -->
    <div v-if="showForm" ref="formAnchor">
      <ProjectForm
        :project="editingProject"
        @submit="handleSubmitProject"
        @cancel="handleCancelForm"
      />
    </div>

    <!-- 项目列表：v-for 会拿数组里的每一项生成一张卡片。
         数据源是 visibleProjects（已按关键词筛过、按规则排过）。 -->
    <section class="list">
      <ProjectCard
        v-for="project in visibleProjects"
        :key="project.id"
        :project="project"
        :editing="editingProject?.id === project.id"
        @edit="handleEditProject(project)"
        @delete="handleDeleteProject(project.id)"
      />
    </section>

    <!-- 一个都没搜到时给个提示，不然页面上会白茫茫一片让人以为坏了 -->
    <p v-if="visibleProjects.length === 0" class="empty">没有找到匹配的项目</p>
  </div>
</template>

<style scoped>
.page {
  max-width: 900px;
  margin: 0 auto;
  padding: 48px 24px 64px;
}

.page-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
  margin-bottom: 28px;
}

.head-actions {
  display: flex;
  align-items: center;
  gap: 10px;
  flex-shrink: 0;
}

/* ---- 主题开关 ---- */
.theme-switch {
  position: relative; /* 滑块要相对这个盒子定位 */
  display: flex;
  padding: 3px;
  background: var(--switch-bg);
  border: 1px solid var(--border);
  border-radius: 999px;
  transition:
    background-color 0.25s ease,
    border-color 0.25s ease;
}

/* 滑块：靠 transform 在左右两格之间移动，左右位置各占一半 */
.theme-thumb {
  position: absolute;
  top: 3px;
  left: 3px;
  width: calc(50% - 3px);
  height: calc(100% - 6px);
  background: var(--switch-thumb);
  border-radius: 999px;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.15);
  transition:
    transform 0.25s ease,
    background-color 0.25s ease;
}

/* 深色模式下把滑块推到右边。translateX(100%) = 移动"自己宽度"那么远，正好一格 */
.theme-thumb.dark {
  transform: translateX(100%);
}

.theme-option {
  position: relative;
  z-index: 1; /* 压在滑块上面，不然点不到 */
  flex: 1 1 0; /* 两个按钮等宽，滑块的一半才对得上 */
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
  padding: 6px 12px;
  font-family: inherit;
  font-size: 13px;
  color: var(--text-muted);
  background: transparent;
  border: 0;
  border-radius: 999px;
  cursor: pointer;
  transition: color 0.25s ease;
}

.theme-option.active {
  color: var(--text-primary);
  font-weight: 500;
}

.theme-option svg {
  flex-shrink: 0;
  width: 15px;
  height: 15px;
}

.btn-add {
  flex-shrink: 0;
  padding: 8px 16px;
  font-family: inherit;
  font-size: 14px;
  color: var(--btn-text);
  background: var(--btn-bg);
  border: 1px solid var(--btn-bg);
  border-radius: 8px;
  cursor: pointer;
  transition:
    background-color 0.2s ease,
    border-color 0.2s ease;
}

.btn-add:hover {
  background: var(--btn-bg-hover);
  border-color: var(--btn-bg-hover);
}

h1 {
  margin: 0 0 6px;
  font-size: 26px;
  font-weight: 600;
}

.subtitle {
  margin: 0;
  font-size: 14px;
  color: var(--text-muted);
}

/* 工具栏：搜索框自动伸长，排序区固定宽度，两者底部对齐 */
.toolbar {
  display: flex;
  align-items: center;
  gap: 12px;
  margin-bottom: 20px;
}

.search {
  flex: 1 1 auto; /* 有剩余空间就都给它，所以它会占满左边 */
  min-width: 0; /* 允许在窄屏时被压缩，否则会把排序区挤出去 */
  padding: 10px 12px;
  font-family: inherit;
  font-size: 14px;
  color: var(--text-primary);
  background: var(--bg-panel);
  border: 1px solid var(--border-strong);
  border-radius: 8px;
  transition:
    background-color 0.25s ease,
    border-color 0.2s ease,
    color 0.25s ease;
}

.search:focus {
  outline: none;
  border-color: var(--accent);
}

/* ---- 排序区 ---- */
.sort-box {
  flex-shrink: 0;
  display: flex;
  align-items: center;
  gap: 8px;
}

.sort-label {
  font-size: 13px;
  color: var(--text-muted);
}

.sort-select {
  padding: 10px 12px;
  font-family: inherit;
  font-size: 14px;
  color: var(--text-primary);
  background: var(--bg-panel);
  border: 1px solid var(--border-strong);
  border-radius: 8px;
  cursor: pointer;
  transition:
    background-color 0.25s ease,
    border-color 0.2s ease,
    color 0.25s ease;
}

.sort-select:focus {
  outline: none;
  border-color: var(--accent);
}

/* 升 / 降序按钮：正方形，和旁边的输入框高度一致 */
.sort-dir {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  width: 40px;
  height: 40px;
  padding: 0;
  color: var(--text-primary);
  background: var(--bg-panel);
  border: 1px solid var(--border-strong);
  border-radius: 8px;
  cursor: pointer;
  transition:
    background-color 0.25s ease,
    border-color 0.2s ease,
    color 0.25s ease;
}

.sort-dir:hover {
  border-color: var(--accent);
}

.sort-dir svg {
  width: 18px;
  height: 18px;
  /* 箭头默认朝上（升序）；降序时转 180° 朝下 */
  transition: transform 0.25s ease;
}

.sort-dir.desc svg {
  transform: rotate(180deg);
}

/* 筛选结果提示 */
.hint {
  margin: -8px 0 16px;
  font-size: 13px;
  color: var(--text-muted);
}

/* 空结果提示 */
.empty {
  padding: 48px 0;
  margin: 0;
  font-size: 14px;
  text-align: center;
  color: var(--text-muted);
}

/* 卡片列表：屏幕宽时自动排成两列，窄时变成一列 */
.list {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(320px, 1fr));
  gap: 16px;
}

/* 窄屏时标题和按钮改成上下两行，开关上不显示文字，只留图标省地方 */
@media (max-width: 560px) {
  .page-head {
    flex-direction: column;
    align-items: stretch;
  }

  .head-actions {
    justify-content: space-between;
  }

  .theme-option span {
    display: none;
  }

  /* 窄屏时搜索框和排序区改成一上一下，各自占满整行 */
  .toolbar {
    flex-direction: column;
    align-items: stretch;
  }

  .sort-box {
    justify-content: space-between;
  }

  .sort-select {
    flex: 1 1 auto; /* 下拉自己撑开，把“排序”标签和箭头推到两侧 */
  }
}
</style>
