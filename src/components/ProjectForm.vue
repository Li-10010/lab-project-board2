<script setup>
import { computed, ref, watch } from 'vue'

// 这个组件负责收集用户填写的内容，填完把结果“抛”给父组件 App.vue。
// emit 就是“抛出事件”的意思，后面用 emit('submit', 数据) 来发。
const emit = defineEmits(['submit', 'cancel'])

// 这个表单同时管两件事：
//   - 不传 project → 新增模式，输入框从空开始；
//   - 传了 project → 编辑模式，输入框先填上这个项目原来的内容。
// 这样一个组件就够了，不用再写第二个几乎一模一样的表单。
const props = defineProps({
  project: {
    type: Object,
    default: null,
  },
})

// 有 id 就说明是在编辑某一条。
// 用 computed 而不是普通变量，是因为 props 随时可能变，computed 会跟着自动重算。
const isEditing = computed(() => Boolean(props.project && props.project.id))

// 每个输入框对应一个 ref，初始值都是空的。
// v-model 会把这个变量和输入框双向绑定：用户输入 → 变量跟着变；变量改 → 输入框跟着变。
const name = ref('')
const description = ref('')
const members = ref('') // 先让用户用逗号分隔着填，之后再拆成数组
const techStack = ref('') // 同上
const status = ref('进行中')
const github = ref('')

// 清空所有输入，方便下次重新填写
function clearForm() {
  name.value = ''
  description.value = ''
  members.value = ''
  techStack.value = ''
  status.value = '进行中'
  github.value = ''
}

// 把传进来的项目内容“倒”进各个输入框。
// 数组要转回“逗号分隔”的文字，因为输入框里只能是一段文字。
function fillFrom(project) {
  if (!project) {
    // 新增模式：全部清空
    clearForm()
    return
  }

  name.value = project.name || ''
  description.value = project.description || ''
  members.value = (project.members || []).join(', ')
  techStack.value = (project.techStack || []).join(', ')
  status.value = project.status || '进行中'
  github.value = project.github || ''
}

// props.project 一变就重新填充——比如表单开着的时候，用户又点了另一张卡片的“编辑”。
// immediate: true 让组件第一次显示时也先执行一次，否则编辑模式下输入框会是空的。
watch(() => props.project, fillFrom, { immediate: true })

// 把“张三, 李四”这种输入切成数组 ['张三', '李四']
// 中文逗号和英文逗号都支持，空的内容会自动被过滤掉
function toArray(text) {
  return text
    .split(/[,，]/)
    .map((item) => item.trim())
    .filter(Boolean)
}

function handleSubmit() {
  // 最简单的校验：名称必须填
  if (!name.value.trim()) {
    alert('请先填写项目名称')
    return
  }

  // 把收集到的数据交给父组件处理。
  // 新增还是修改由父组件按 editingProject 判断，这里不用操心。
  emit('submit', {
    name: name.value.trim(),
    description: description.value.trim(),
    members: toArray(members.value),
    techStack: toArray(techStack.value),
    status: status.value,
    github: github.value.trim(),
  })

  clearForm()
}

function handleCancel() {
  clearForm()
  emit('cancel')
}
</script>

<template>
  <!-- @submit.prevent 表示“提交时执行 handleSubmit，并阻止浏览器默认的整页刷新” -->
  <!-- :class 给编辑模式加一个 editing 类，好把它和新增模式在外观上区分开 -->
  <form class="form" :class="{ editing: isEditing }" @submit.prevent="handleSubmit">
    <h2 class="form-title">
      {{ isEditing ? '编辑项目' : '新增项目' }}
      <!-- 编辑时把项目名显示出来，用户一眼就知道自己在改哪一条 -->
      <span v-if="isEditing" class="editing-name">{{ project.name }}</span>
    </h2>

    <div class="field">
      <label for="name">项目名称</label>
      <input id="name" v-model="name" type="text" placeholder="例如：智能垃圾分类助手" />
    </div>

    <div class="field">
      <label for="description">项目简介</label>
      <textarea
        id="description"
        v-model="description"
        rows="2"
        placeholder="一句话说明这个项目是做什么的"
      ></textarea>
    </div>

    <div class="field">
      <label for="members">成员</label>
      <input id="members" v-model="members" type="text" placeholder="多人用逗号分隔，例如：张明, 李小雨" />
    </div>

    <div class="field">
      <label for="techStack">技术栈</label>
      <input id="techStack" v-model="techStack" type="text" placeholder="用逗号分隔，例如：Vue 3, Node.js" />
    </div>

    <div class="field">
      <label for="status">项目状态</label>
      <select id="status" v-model="status">
        <option value="进行中">进行中</option>
        <option value="已完成">已完成</option>
        <option value="暂停">暂停</option>
      </select>
    </div>

    <div class="field">
      <label for="github">GitHub 地址</label>
      <input id="github" v-model="github" type="text" placeholder="https://github.com/..." />
    </div>

    <div class="actions">
      <button type="button" class="btn btn-ghost" @click="handleCancel">取消</button>
      <button type="submit" class="btn btn-primary">
        {{ isEditing ? '保存修改' : '提交' }}
      </button>
    </div>
  </form>
</template>

<style scoped>
.form {
  display: flex;
  flex-direction: column;
  gap: 14px;
  padding: 20px;
  margin-bottom: 20px;
  background: var(--bg-panel);
  border: 1px solid var(--border);
  border-radius: 12px;
  transition:
    background-color 0.25s ease,
    border-color 0.25s ease;
}

/* 编辑模式下换个边框颜色，让用户清楚现在是在改东西而不是新增 */
.form.editing {
  border-color: var(--accent);
}

.form-title {
  display: flex;
  align-items: center;
  gap: 8px;
  margin: 0;
  font-size: 16px;
  font-weight: 600;
}

/* 标题后面那个小标签，显示正在编辑的项目名 */
.editing-name {
  /* min-width: 0 是让它在名字很长时能被压缩，否则会把标题顶出去 */
  min-width: 0;
  padding: 2px 8px;
  font-size: 12px;
  font-weight: 500;
  color: var(--accent);
  background: var(--bg-subtle);
  border-radius: 6px;
  /* 名字太长就用省略号截断，不要撑破布局 */
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.field {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

label {
  font-size: 13px;
  color: var(--text-secondary);
}

input,
textarea,
select {
  width: 100%;
  padding: 8px 10px;
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

textarea {
  resize: vertical;
}

input:focus,
textarea:focus,
select:focus {
  outline: none;
  border-color: var(--accent);
}

.actions {
  display: flex;
  justify-content: flex-end;
  gap: 8px;
  margin-top: 4px;
}

.btn {
  padding: 8px 16px;
  font-family: inherit;
  font-size: 14px;
  border-radius: 8px;
  cursor: pointer;
  transition:
    background-color 0.2s ease,
    border-color 0.2s ease,
    color 0.2s ease;
}

.btn-ghost {
  color: var(--text-secondary);
  background: var(--bg-panel);
  border: 1px solid var(--border-strong);
}

.btn-ghost:hover {
  background: var(--bg-subtle);
}

.btn-primary {
  color: var(--btn-text);
  background: var(--btn-bg);
  border: 1px solid var(--btn-bg);
}

.btn-primary:hover {
  background: var(--btn-bg-hover);
  border-color: var(--btn-bg-hover);
}
</style>
