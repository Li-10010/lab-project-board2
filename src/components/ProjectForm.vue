<script setup>
import { ref } from 'vue'

// 这个组件负责收集用户填写的内容，填完把结果“抛”给父组件 App.vue。
// emit 就是“抛出事件”的意思，后面用 emit('submit', 数据) 来发。
const emit = defineEmits(['submit', 'cancel'])

// 每个输入框对应一个 ref，初始值都是空的。
// v-model 会把这个变量和输入框双向绑定：用户输入 → 变量跟着变；变量改 → 输入框跟着变。
const name = ref('')
const description = ref('')
const members = ref('') // 先让用户用逗号分隔着填，之后再拆成数组
const techStack = ref('') // 同上
const status = ref('进行中')
const github = ref('')

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

  // 把收集到的数据交给父组件处理
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

// 清空所有输入，方便下次重新填写
function clearForm() {
  name.value = ''
  description.value = ''
  members.value = ''
  techStack.value = ''
  status.value = '进行中'
  github.value = ''
}
</script>

<template>
  <!-- @submit.prevent 表示“提交时执行 handleSubmit，并阻止浏览器默认的整页刷新” -->
  <form class="form" @submit.prevent="handleSubmit">
    <h2 class="form-title">新增项目</h2>

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
      <button type="submit" class="btn btn-primary">提交</button>
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
  background: #ffffff;
  border: 1px solid #e6e8eb;
  border-radius: 12px;
}

.form-title {
  margin: 0;
  font-size: 16px;
  font-weight: 600;
}

.field {
  display: flex;
  flex-direction: column;
  gap: 6px;
}

label {
  font-size: 13px;
  color: #57606a;
}

input,
textarea,
select {
  width: 100%;
  padding: 8px 10px;
  font-family: inherit;
  font-size: 14px;
  color: #1f2328;
  background: #ffffff;
  border: 1px solid #d8dce1;
  border-radius: 8px;
}

textarea {
  resize: vertical;
}

input:focus,
textarea:focus,
select:focus {
  outline: none;
  border-color: #378add;
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
}

.btn-ghost {
  color: #57606a;
  background: #ffffff;
  border: 1px solid #d8dce1;
}

.btn-ghost:hover {
  background: #f5f6f8;
}

.btn-primary {
  color: #ffffff;
  background: #1f2328;
  border: 1px solid #1f2328;
}

.btn-primary:hover {
  background: #3a4048;
}
</style>
