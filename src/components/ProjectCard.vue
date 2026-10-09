<script setup>
// 这个组件只负责“展示一个项目”。
// 它自己不存数据，数据由父组件 App.vue 通过 props 传进来。
defineProps({
  project: {
    type: Object,
    required: true,
  },
})

// 声明这个组件会往外抛一个 delete 事件。
// 卡片自己不知道“删除”该怎么做，它只负责喊一声，具体处理交给父组件。
const emit = defineEmits(['delete'])

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
</script>

<template>
  <article class="card">
    <!-- 第一行：项目名称 + 状态标签 -->
    <div class="card-head">
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

    <!-- 删除按钮：点击后抛出 delete 事件 -->
    <div class="card-foot">
      <button type="button" class="btn-delete" @click="emit('delete')">删除</button>
    </div>
  </article>
</template>

<style scoped>
/* scoped 表示这些样式只对这个组件生效，不会影响别的文件 */
.card {
  display: flex;
  flex-direction: column;
  gap: 12px;
  padding: 20px;
  background: #ffffff;
  border: 1px solid #e6e8eb;
  border-radius: 12px;
  transition: border-color 0.2s;
}

.card:hover {
  border-color: #c9ced6;
}

.card-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
}

.name {
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
  color: #0c447c;
  background: #e6f1fb;
}

/* 已完成：绿 */
.status.done {
  color: #3b6d11;
  background: #eaf3de;
}

/* 暂停：橙 */
.status.paused {
  color: #854f0b;
  background: #faeeda;
}

/* 兜底：万一出现预料之外的状态值，也别让它没有样式 */
.status.other {
  color: #5f5e5a;
  background: #f1efe8;
}

.desc {
  margin: 0;
  font-size: 14px;
  color: #57606a;
}

.tech {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}

.tech-tag {
  padding: 2px 8px;
  font-size: 12px;
  color: #57606a;
  background: #f5f6f8;
  border: 1px solid #e6e8eb;
  border-radius: 6px;
}

.meta {
  display: flex;
  flex-direction: column;
  gap: 6px;
  padding-top: 12px;
  border-top: 1px solid #f0f1f3;
  font-size: 13px;
  color: #57606a;
}

.meta-row {
  display: flex;
  gap: 8px;
}

.meta-label {
  flex-shrink: 0;
  color: #8b949e;
}

.meta a {
  color: #0c447c;
  word-break: break-all;
}

.card-foot {
  display: flex;
  justify-content: flex-end;
}

.btn-delete {
  padding: 4px 12px;
  font-family: inherit;
  font-size: 13px;
  color: #a32d2d;
  background: #ffffff;
  border: 1px solid #e6e8eb;
  border-radius: 6px;
  cursor: pointer;
}

.btn-delete:hover {
  color: #ffffff;
  background: #e24b4a;
  border-color: #e24b4a;
}
</style>
