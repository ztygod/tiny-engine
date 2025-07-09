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

<style lang="less" scoped>
.version-history {
  .commit-item {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 8px 12px;
    border-bottom: 1px solid #e5e7eb; // 浅灰色
    cursor: pointer;

    &:hover {
      background-color: #f9fafb;
    }

    .commit-meta {
      display: flex;
      align-items: center;

      .commit-avatar {
        width: 32px;
        height: 32px;
        border-radius: 50%;
        margin-right: 8px;
      }

      .commit-author-info {
        display: flex;
        flex-direction: column;

        .commit-author {
          font-weight: bold;
        }

        .commit-date {
          font-size: 12px;
          color: #6b7280; // 灰色
        }
      }
    }

    .commit-message {
      flex: 1;
      margin: 0 16px;
      overflow: hidden;
      text-overflow: ellipsis;
      white-space: nowrap;
    }

    .commit-actions {
      display: flex;
      align-items: center;

      .commit-hash {
        font-size: 12px;
        color: #9ca3af;
        margin-right: 8px;
      }
    }
  }

  .load-more {
    display: flex;
    justify-content: center;
    margin-top: 16px;
  }
}
</style>
