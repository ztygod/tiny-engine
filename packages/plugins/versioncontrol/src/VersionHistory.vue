<template>
  <div class="version-history">
    <div v-for="commit in commits" :key="commit.hash" class="commit-item" @click="showDetails(commit)">
      <!-- 左侧：头像+作者+时间 -->
      <div class="commit-meta">
        <img :src="commit.avatar" alt="avatar" class="commit-avatar" />
        <div class="commit-author-info">
          <div class="commit-author">{{ commit.author }}</div>
          <div class="commit-date">{{ commit.date }}</div>
        </div>
      </div>

      <!-- 中间：提交信息 -->
      <div class="commit-message">
        {{ commit.message }}
      </div>

      <!-- 右侧：hash 和按钮 -->
      <div class="commit-actions">
        <span class="commit-hash">{{ commit.hash.slice(0, 7) }}</span>
        <tiny-button size="small">详情</tiny-button>
      </div>
    </div>

    <!-- 加载更多 -->
    <div class="load-more">
      <tiny-button @click="loadMore">加载更多</tiny-button>
    </div>

    <!-- 弹窗：提交详情 -->
    <tiny-dialog v-model:visible="dialogVisible" title="提交详情" width="500px">
      <div v-if="selectedCommit">
        <p><strong>作者：</strong>{{ selectedCommit.author }}</p>
        <p><strong>时间：</strong>{{ selectedCommit.date }}</p>
        <p><strong>提交信息：</strong>{{ selectedCommit.message }}</p>
        <p><strong>Hash：</strong>{{ selectedCommit.hash }}</p>
      </div>
    </tiny-dialog>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import { Button as TinyButton, Dialog as TinyDialog } from '@opentiny/vue'

// 模拟数据
const commits = ref([
  {
    hash: 'a1b2c3d',
    author: 'Alice',
    date: '2025-07-09 14:30',
    message: 'Initial commit',
    avatar: 'https://avatars.githubusercontent.com/u/1?v=4'
  },
  {
    hash: 'd4e5f6g',
    author: 'Bob',
    date: '2025-07-08 12:10',
    message: 'Add README',
    avatar: 'https://avatars.githubusercontent.com/u/2?v=4'
  }
])

const dialogVisible = ref(false)
const selectedCommit = ref(null)

function showDetails(commit) {
  selectedCommit.value = commit
  dialogVisible.value = true
}

function loadMore() {}
</script>
