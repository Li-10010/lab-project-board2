<script setup>
import { computed, ref, watch } from 'vue'
import ProjectCard from './components/ProjectCard.vue'
import ProjectForm from './components/ProjectForm.vue'

// localStorage 里用这个“钥匙名”来区分我们的数据。
// 浏览器里可能存了很多网站的东西，靠这个名字知道哪份是自己的。
const STORAGE_KEY = 'lab-project-board:projects'

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

// 只要 projects 发生变化（新增、删除，以后做修改也算），就立刻存进 localStorage。
// 写在同一个地方，就不用“新增时存一次、删除时再存一次”，也就不会漏掉某一处。
watch(
  projects,
  (newProjects) => {
    // JSON.stringify 把数组变成字符串，因为 localStorage 只能存字符串
    localStorage.setItem(STORAGE_KEY, JSON.stringify(newProjects))
  },
  // deep: true 表示“数组里面某个项目的字段变了也算变化”，不只盯着数组本身
  { deep: true },
)

// 控制“新增项目”表单是否显示
const showForm = ref(false)

// 表单提交后，子组件会把填好的数据传进来，这里把它加进列表
function handleAddProject(data) {
  // 取现有最大的 id 再 +1，保证每个项目的 id 都不重复
  const maxId = projects.value.reduce((max, project) => Math.max(max, project.id), 0)

  // ...data 是展开运算符，意思是“把 data 里的字段都放进这个新对象”
  projects.value.push({ id: maxId + 1, ...data })

  // 加完顺手把表单收起来
  showForm.value = false
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
}

// 搜索关键词，和搜索框双向绑定。用户输入一个字，它就变一次。
const keyword = ref('')

// 根据关键词算出“当前要显示哪些项目”。
// computed 是“算出来的数据”：只要 keyword 或 projects 变了，它会自动重新算一遍。
const filteredProjects = computed(() => {
  // 去掉前后空格并转成小写，这样搜 "vue" 也能匹配到 "Vue 3"
  const text = keyword.value.trim().toLowerCase()
  if (!text) return projects.value

  // includes 判断“名称里是否包含这段文字”，所以只搜一部分也能找到
  return projects.value.filter((project) => project.name.toLowerCase().includes(text))
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
      <button class="btn-add" @click="showForm = !showForm">
        {{ showForm ? '收起' : '新增项目' }}
      </button>
    </header>

    <!-- 搜索框：v-model 让输入内容和 keyword 实时同步，所以边打字边过滤 -->
    <input
      v-model="keyword"
      class="search"
      type="text"
      placeholder="搜索项目名称，例如：巡检"
    />

    <!-- 表单只在 showForm 为 true 的时候才显示 -->
    <ProjectForm v-if="showForm" @submit="handleAddProject" @cancel="showForm = false" />

    <!-- 项目列表：v-for 会拿数组里的每一项生成一张卡片 -->
    <section class="list">
      <ProjectCard
        v-for="project in filteredProjects"
        :key="project.id"
        :project="project"
        @delete="handleDeleteProject(project.id)"
      />
    </section>
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

.btn-add {
  flex-shrink: 0;
  padding: 8px 16px;
  font-family: inherit;
  font-size: 14px;
  color: #ffffff;
  background: #1f2328;
  border: 1px solid #1f2328;
  border-radius: 8px;
  cursor: pointer;
}

.btn-add:hover {
  background: #3a4048;
}

h1 {
  margin: 0 0 6px;
  font-size: 26px;
  font-weight: 600;
}

.subtitle {
  margin: 0;
  font-size: 14px;
  color: #8b949e;
}

.search {
  width: 100%;
  padding: 10px 12px;
  margin-bottom: 20px;
  font-family: inherit;
  font-size: 14px;
  color: #1f2328;
  background: #ffffff;
  border: 1px solid #d8dce1;
  border-radius: 8px;
}

.search:focus {
  outline: none;
  border-color: #378add;
}

/* 卡片列表：屏幕宽时自动排成两列，窄时变成一列 */
.list {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(320px, 1fr));
  gap: 16px;
}
</style>
