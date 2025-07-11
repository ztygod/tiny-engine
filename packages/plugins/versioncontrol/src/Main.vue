<template>
  <div class="version-control-container">
    <!-- <version-control-header @close="close"></version-control-header> -->
    <header class="version-control-header">
      <div class="header-left">
        <span class="title">版本控制</span>
        <link-button :href="docsUrl" class="link-button"></link-button>
      </div>

      <div class="header-center">
        <!-- 分支选择器 -->
        <div class="branch-selector">
          <select v-model="currentBranch" @change="onBranchChange" class="branch-select">
            <option v-for="branch in branches" :key="branch" :value="branch">
              {{ branch }}
            </option>
          </select>
        </div>

        <!-- 搜索框 -->
        <div class="search-container">
          <input
            v-model="searchQuery"
            @input="onSearch"
            placeholder="搜索提交信息、作者或Hash..."
            class="search-input"
          />
          <svg class="search-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor">
            <circle cx="11" cy="11" r="8"></circle>
            <path d="m21 21-4.35-4.35"></path>
          </svg>
        </div>
      </div>

      <div class="header-right">
        <!-- 操作按钮 -->
        <button @click="createTag" class="action-btn">
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor">
            <path d="M20.59 13.41l-7.17 7.17a2 2 0 0 1-2.83 0L2 12V2h10l8.59 8.59a2 2 0 0 1 0 2.82z"></path>
            <line x1="7" y1="7" x2="7.01" y2="7"></line>
          </svg>
          标签
        </button>
        <button @click="createBranch" class="action-btn">
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor">
            <line x1="6" y1="3" x2="6" y2="15"></line>
            <circle cx="18" cy="6" r="3"></circle>
            <circle cx="6" cy="18" r="3"></circle>
            <path d="m18 9a9 9 0 0 1-9 9"></path>
          </svg>
          分支
        </button>
        <close-icon @close="close"></close-icon>
      </div>
    </header>

    <main class="version-control-content">
      <!-- 筛选器 -->
      <div class="filters-container">
        <div class="filter-group">
          <label>作者:</label>
          <select v-model="authorFilter" @change="applyFilters" class="filter-select">
            <option value="">全部</option>
            <option v-for="author in uniqueAuthors" :key="author" :value="author">
              {{ author }}
            </option>
          </select>
        </div>

        <div class="filter-group">
          <label>时间范围:</label>
          <select v-model="timeFilter" @change="applyFilters" class="filter-select">
            <option value="">全部</option>
            <option value="today">今天</option>
            <option value="week">本周</option>
            <option value="month">本月</option>
          </select>
        </div>

        <div class="filter-group">
          <button @click="clearFilters" class="clear-filters-btn">清除筛选</button>
        </div>

        <div class="filter-group stats">
          <span class="stats-text"> 共 {{ filteredCommits.length }} 个提交 | {{ uniqueAuthors.length }} 位贡献者 </span>
        </div>
      </div>

      <!-- 主要内容区域 -->
      <div class="main-content">
        <!-- 左侧：图形化分支线 -->
        <div class="timeline-container">
          <div class="timeline-header">
            <h3>提交时间线</h3>
            <div class="timeline-controls">
              <button
                @click="timelineView = 'compact'"
                :class="{ active: timelineView === 'compact' }"
                class="view-toggle"
              >
                紧凑
              </button>
              <button
                @click="timelineView = 'detailed'"
                :class="{ active: timelineView === 'detailed' }"
                class="view-toggle"
              >
                详细
              </button>
            </div>
          </div>

          <div class="timeline-content" :class="timelineView">
            <div
              v-for="(commit, index) in filteredCommits"
              :key="commit.hash"
              class="timeline-item"
              :class="{
                selected: selectedCommit?.hash === commit.hash,
                'merge-item': commit.type === 'merge'
              }"
              @click="selectCommit(commit)"
            >
              <!-- 分支线 -->
              <div class="timeline-line">
                <div class="timeline-dot" :class="getCommitTypeClass(commit)">
                  <!-- 合并提交图标 -->
                  <svg v-if="commit.type === 'merge'" viewBox="0 0 24 24" fill="currentColor">
                    <path d="M8 16l2.879-2.879a3 3 0 0 1 4.242 0L18 16M8 8l2.879 2.879a3 3 0 0 1 4.242 0L18 8" />
                  </svg>
                  <!-- 标签提交图标 -->
                  <svg v-else-if="commit.tags && commit.tags.length > 0" viewBox="0 0 24 24" fill="currentColor">
                    <path d="M20.59 13.41l-7.17 7.17a2 2 0 0 1-2.83 0L2 12V2h10l8.59 8.59a2 2 0 0 1 0 2.82z" />
                  </svg>
                  <!-- 普通提交点 -->
                  <div v-else class="commit-dot"></div>
                </div>
                <!-- 连接线 -->
                <div
                  v-if="index < filteredCommits.length - 1"
                  class="timeline-connector"
                  :class="getConnectorClass(commit, filteredCommits[index + 1])"
                ></div>
              </div>

              <!-- 提交信息预览 -->
              <div class="timeline-info">
                <div class="commit-hash-mini">{{ commit.hash.slice(0, 7) }}</div>
                <div class="commit-time">{{ formatTime(commit.date) }}</div>
                <div v-if="timelineView === 'detailed'" class="commit-message-mini">
                  {{ commit.message.slice(0, 30) }}{{ commit.message.length > 30 ? '...' : '' }}
                </div>
                <div v-if="commit.tags && commit.tags.length > 0" class="commit-tags-mini">
                  <span v-for="tag in commit.tags.slice(0, 2)" :key="tag" class="tag-mini">{{ tag }}</span>
                </div>
              </div>
            </div>
          </div>
        </div>

        <!-- 右侧：提交详情列表 -->
        <div class="commits-container">
          <div class="commits-header">
            <h3>提交详情</h3>
            <div class="header-controls">
              <div class="view-options">
                <button @click="viewMode = 'list'" :class="{ active: viewMode === 'list' }" class="view-btn">
                  <svg viewBox="0 0 24 24" fill="none" stroke="currentColor">
                    <line x1="8" y1="6" x2="21" y2="6"></line>
                    <line x1="8" y1="12" x2="21" y2="12"></line>
                    <line x1="8" y1="18" x2="21" y2="18"></line>
                    <line x1="3" y1="6" x2="3.01" y2="6"></line>
                    <line x1="3" y1="12" x2="3.01" y2="12"></line>
                    <line x1="3" y1="18" x2="3.01" y2="18"></line>
                  </svg>
                  列表
                </button>
                <button @click="viewMode = 'detailed'" :class="{ active: viewMode === 'detailed' }" class="view-btn">
                  <svg viewBox="0 0 24 24" fill="none" stroke="currentColor">
                    <rect x="3" y="3" width="18" height="18" rx="2" ry="2"></rect>
                    <line x1="9" y1="9" x2="15" y2="9"></line>
                    <line x1="9" y1="15" x2="15" y2="15"></line>
                  </svg>
                  详细
                </button>
              </div>

              <div class="sort-options">
                <select v-model="sortBy" @change="applySorting" class="sort-select">
                  <option value="date-desc">时间 ↓</option>
                  <option value="date-asc">时间 ↑</option>
                  <option value="author">作者</option>
                  <option value="message">消息</option>
                </select>
              </div>
            </div>
          </div>

          <div class="commits-list" :class="viewMode">
            <div
              v-for="commit in paginatedCommits"
              :key="commit.hash"
              class="commit-item"
              :class="{
                selected: selectedCommit?.hash === commit.hash,
                'merge-commit': commit.type === 'merge'
              }"
              @click="selectCommit(commit)"
            >
              <!-- 提交头部信息 -->
              <div class="commit-header">
                <div class="commit-meta">
                  <img :src="commit.avatar" alt="avatar" class="commit-avatar" />
                  <div class="commit-author-info">
                    <div class="commit-author">{{ commit.author }}</div>
                    <div class="commit-date">{{ formatDate(commit.date) }}</div>
                  </div>
                  <div v-if="commit.type === 'merge'" class="merge-badge">
                    <svg viewBox="0 0 24 24" fill="currentColor">
                      <path d="M8 16l2.879-2.879a3 3 0 0 1 4.242 0L18 16M8 8l2.879 2.879a3 3 0 0 1 4.242 0L18 8" />
                    </svg>
                    合并
                  </div>
                </div>

                <div class="commit-actions">
                  <span class="commit-hash">{{ commit.hash.slice(0, 7) }}</span>
                  <div class="action-buttons">
                    <button @click.stop="showCommitDetails(commit)" class="action-btn small" title="查看详情">
                      <svg viewBox="0 0 24 24" fill="none" stroke="currentColor">
                        <circle cx="12" cy="12" r="3"></circle>
                        <path d="M12 1v6m0 6v6m11-7h-6m-6 0H1"></path>
                      </svg>
                    </button>
                    <button @click.stop="compareCommit(commit)" class="action-btn small" title="比较差异">
                      <svg viewBox="0 0 24 24" fill="none" stroke="currentColor">
                        <path d="M18 20V10M6 20V4m6 16v-8"></path>
                      </svg>
                    </button>
                    <button @click.stop="revertToCommit(commit)" class="action-btn small danger" title="回滚">
                      <svg viewBox="0 0 24 24" fill="none" stroke="currentColor">
                        <path d="M3 7v6h6M21 17v-6h-6"></path>
                        <path d="M21 3l-9 9-9-9"></path>
                      </svg>
                    </button>
                  </div>
                </div>
              </div>

              <!-- 提交消息 -->
              <div class="commit-message">
                <span class="commit-type-prefix" :class="commit.type">
                  {{ getCommitTypePrefix(commit.type) }}
                </span>
                {{ commit.message }}
              </div>

              <!-- 标签和分支信息 -->
              <div v-if="commit.tags?.length || commit.branches?.length" class="commit-labels">
                <span v-for="tag in commit.tags" :key="tag" class="commit-tag">
                  <svg viewBox="0 0 24 24" fill="none" stroke="currentColor">
                    <path d="M20.59 13.41l-7.17 7.17a2 2 0 0 1-2.83 0L2 12V2h10l8.59 8.59a2 2 0 0 1 0 2.82z"></path>
                  </svg>
                  {{ tag }}
                </span>
                <span v-for="branch in commit.branches" :key="branch" class="commit-branch">
                  <svg viewBox="0 0 24 24" fill="none" stroke="currentColor">
                    <line x1="6" y1="3" x2="6" y2="15"></line>
                    <circle cx="18" cy="6" r="3"></circle>
                    <circle cx="6" cy="18" r="3"></circle>
                  </svg>
                  {{ branch }}
                </span>
              </div>

              <!-- 详细视图额外信息 -->
              <div v-if="viewMode === 'detailed'" class="commit-details">
                <div class="commit-stats">
                  <span class="stat-item">
                    <svg viewBox="0 0 24 24" fill="none" stroke="currentColor">
                      <path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"></path>
                      <polyline points="14,2 14,8 20,8"></polyline>
                    </svg>
                    <span class="stat-value">{{ commit.filesChanged || 0 }} 文件</span>
                  </span>
                  <span class="stat-item additions">
                    <span class="stat-value">+{{ commit.additions || 0 }}</span>
                  </span>
                  <span class="stat-item deletions">
                    <span class="stat-value">-{{ commit.deletions || 0 }}</span>
                  </span>
                </div>

                <div v-if="commit.changedFiles?.length" class="commit-files">
                  <div class="files-header">变更文件:</div>
                  <div class="files-list">
                    <div v-for="file in commit.changedFiles.slice(0, 3)" :key="file" class="changed-file">
                      <svg viewBox="0 0 24 24" fill="none" stroke="currentColor">
                        <path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"></path>
                        <polyline points="14,2 14,8 20,8"></polyline>
                      </svg>
                      {{ file }}
                    </div>
                    <div v-if="commit.changedFiles.length > 3" class="more-files">
                      +{{ commit.changedFiles.length - 3 }} 个文件...
                    </div>
                  </div>
                </div>
              </div>
            </div>

            <!-- 空状态 -->
            <div v-if="filteredCommits.length === 0" class="empty-state">
              <svg viewBox="0 0 24 24" fill="none" stroke="currentColor">
                <circle cx="11" cy="11" r="8"></circle>
                <path d="m21 21-4.35-4.35"></path>
              </svg>
              <h3>未找到匹配的提交</h3>
              <p>尝试调整搜索条件或筛选器</p>
              <button @click="clearFilters" class="clear-btn">清除所有筛选</button>
            </div>
          </div>

          <!-- 分页控制 -->
          <div v-if="filteredCommits.length > 0" class="pagination-container">
            <button @click="loadMore" :disabled="isLoading || !hasMore" class="load-more-btn">
              <svg v-if="isLoading" class="loading-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor">
                <path d="M21 12a9 9 0 11-6.219-8.56"></path>
              </svg>
              <span v-if="isLoading">加载中...</span>
              <span v-else-if="hasMore">加载更多 ({{ filteredCommits.length - paginatedCommits.length }} 个)</span>
              <span v-else>已加载全部</span>
            </button>

            <div class="pagination-info">
              显示 {{ Math.min(currentPage * pageSize, filteredCommits.length) }} / {{ filteredCommits.length }} 个提交
            </div>
          </div>
        </div>
      </div>
    </main>

    <!-- 提交详情对话框 -->
    <div v-if="dialogVisible" class="dialog-overlay" @click="closeDialog">
      <div class="dialog-content" @click.stop>
        <div class="dialog-header">
          <h3>提交详情</h3>
          <button @click="closeDialog" class="close-btn">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor">
              <line x1="18" y1="6" x2="6" y2="18"></line>
              <line x1="6" y1="6" x2="18" y2="18"></line>
            </svg>
          </button>
        </div>

        <div class="dialog-body" v-if="selectedCommit">
          <div class="detail-section">
            <h4>基本信息</h4>
            <div class="detail-grid">
              <div class="detail-item">
                <label>作者:</label>
                <div class="author-info">
                  <img :src="selectedCommit.avatar" alt="avatar" class="author-avatar" />
                  <span>{{ selectedCommit.author }}</span>
                </div>
              </div>
              <div class="detail-item">
                <label>时间:</label>
                <span>{{ formatDate(selectedCommit.date) }}</span>
              </div>
              <div class="detail-item">
                <label>Hash:</label>
                <span class="hash-full">{{ selectedCommit.hash }}</span>
              </div>
              <div class="detail-item">
                <label>类型:</label>
                <span class="commit-type-badge" :class="selectedCommit.type">
                  {{ getCommitTypeText(selectedCommit.type) }}
                </span>
              </div>
              <div class="detail-item full-width">
                <label>提交信息:</label>
                <div class="commit-message-full">{{ selectedCommit.message }}</div>
              </div>
            </div>
          </div>

          <div class="detail-section" v-if="selectedCommit.changedFiles?.length">
            <h4>变更文件 ({{ selectedCommit.changedFiles.length }})</h4>
            <div class="files-list-detailed">
              <div v-for="file in selectedCommit.changedFiles" :key="file" class="file-item">
                <svg viewBox="0 0 24 24" fill="none" stroke="currentColor">
                  <path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"></path>
                  <polyline points="14,2 14,8 20,8"></polyline>
                </svg>
                {{ file }}
              </div>
            </div>
          </div>

          <div class="detail-section" v-if="selectedCommit.tags?.length || selectedCommit.branches?.length">
            <h4>标签和分支</h4>
            <div class="labels-container">
              <div v-if="selectedCommit.tags?.length" class="label-group">
                <span class="label-type">标签:</span>
                <span v-for="tag in selectedCommit.tags" :key="tag" class="label-item tag">{{ tag }}</span>
              </div>
              <div v-if="selectedCommit.branches?.length" class="label-group">
                <span class="label-type">分支:</span>
                <span v-for="branch in selectedCommit.branches" :key="branch" class="label-item branch">{{
                  branch
                }}</span>
              </div>
            </div>
          </div>

          <div class="detail-actions">
            <button @click="compareWithCurrent" class="action-btn primary">
              <svg viewBox="0 0 24 24" fill="none" stroke="currentColor">
                <path d="M18 20V10M6 20V4m6 16v-8"></path>
              </svg>
              与当前版本比较
            </button>
            <button @click="createBranchFromCommit" class="action-btn">
              <svg viewBox="0 0 24 24" fill="none" stroke="currentColor">
                <line x1="6" y1="3" x2="6" y2="15"></line>
                <circle cx="18" cy="6" r="3"></circle>
                <circle cx="6" cy="18" r="3"></circle>
              </svg>
              从此版本创建分支
            </button>
            <button @click="revertToCommit(selectedCommit)" class="action-btn danger">
              <svg viewBox="0 0 24 24" fill="none" stroke="currentColor">
                <path d="M3 7v6h6M21 17v-6h-6"></path>
                <path d="M21 3l-9 9-9-9"></path>
              </svg>
              回滚到此版本
            </button>
          </div>
        </div>
      </div>
    </div>

    <!-- 标签创建对话框 -->
    <div v-if="tagDialogVisible" class="dialog-overlay" @click="closeTagDialog">
      <div class="dialog-content small" @click.stop>
        <div class="dialog-header">
          <h3>创建标签</h3>
          <button @click="closeTagDialog" class="close-btn">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor">
              <line x1="18" y1="6" x2="6" y2="18"></line>
              <line x1="6" y1="6" x2="18" y2="18"></line>
            </svg>
          </button>
        </div>

        <div class="dialog-body">
          <div class="form-group">
            <label>标签名称:</label>
            <input v-model="newTagName" placeholder="例如: v1.0.0" class="form-input" @keyup.enter="confirmCreateTag" />
          </div>
          <div class="form-group">
            <label>描述 (可选):</label>
            <textarea v-model="newTagDescription" placeholder="标签描述..." class="form-textarea"></textarea>
          </div>
          <div class="form-group">
            <label>目标提交:</label>
            <select v-model="tagTargetCommit" class="form-select">
              <option value="">选择提交 (默认为最新)</option>
              <option v-for="commit in commits.slice(0, 10)" :key="commit.hash" :value="commit.hash">
                {{ commit.hash.slice(0, 7) }} - {{ commit.message.slice(0, 50) }}
              </option>
            </select>
          </div>

          <div class="dialog-actions">
            <button @click="closeTagDialog" class="action-btn secondary">取消</button>
            <button @click="confirmCreateTag" class="action-btn primary" :disabled="!newTagName.trim()">
              创建标签
            </button>
          </div>
        </div>
      </div>
    </div>

    <!-- 比较差异对话框 -->
    <div v-if="compareDialogVisible" class="dialog-overlay" @click="closeCompareDialog">
      <div class="dialog-content large" @click.stop>
        <div class="dialog-header">
          <h3>版本比较</h3>
          <button @click="closeCompareDialog" class="close-btn">
            <svg viewBox="0 0 24 24" fill="none" stroke="currentColor">
              <line x1="18" y1="6" x2="6" y2="18"></line>
              <line x1="6" y1="6" x2="18" y2="18"></line>
            </svg>
          </button>
        </div>

        <div class="dialog-body">
          <div class="compare-header">
            <div class="compare-info">
              <div class="compare-item">
                <label>基础版本:</label>
                <span class="version-info">
                  {{ compareData.base?.hash?.slice(0, 7) }} - {{ compareData.base?.message }}
                </span>
              </div>
              <div class="compare-item">
                <label>目标版本:</label>
                <span class="version-info">
                  {{ compareData.target?.hash?.slice(0, 7) }} - {{ compareData.target?.message }}
                </span>
              </div>
            </div>
          </div>

          <div class="compare-stats">
            <div class="stat-card">
              <div class="stat-number">{{ compareData.filesChanged || 0 }}</div>
              <div class="stat-label">文件变更</div>
            </div>
            <div class="stat-card additions">
              <div class="stat-number">+{{ compareData.additions || 0 }}</div>
              <div class="stat-label">新增行</div>
            </div>
            <div class="stat-card deletions">
              <div class="stat-number">-{{ compareData.deletions || 0 }}</div>
              <div class="stat-label">删除行</div>
            </div>
          </div>

          <div class="compare-files">
            <h4>变更文件列表</h4>
            <div class="files-diff-list">
              <div v-for="file in compareData.changedFiles || []" :key="file.name" class="file-diff-item">
                <div class="file-diff-header">
                  <svg viewBox="0 0 24 24" fill="none" stroke="currentColor">
                    <path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"></path>
                    <polyline points="14,2 14,8 20,8"></polyline>
                  </svg>
                  <span class="file-name">{{ file.name }}</span>
                  <div class="file-stats">
                    <span class="additions">+{{ file.additions || 0 }}</span>
                    <span class="deletions">-{{ file.deletions || 0 }}</span>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
// import { ref, computed, watch } from 'vue'
// import { LinkButton, CloseIcon } from '@opentiny/tiny-engine-common'
// import { useHelp } from '@opentiny/tiny-engine-controller'

// export default {
//   components: {
//     LinkButton,
//     CloseIcon,
//     VersionControlHeader,
//   },
//   emits: ['close'],
//   setup(props, { emit }) {
//     const docsUrl = useHelp().getDocsUrl('script')

//     // 响应式数据
//     const currentBranch = ref('main')
//     const branches = ref(['main', 'develop', 'feature/new-ui', 'hotfix/bug-fix', 'release/v2.1'])
//     const searchQuery = ref('')
//     const authorFilter = ref('')
//     const timeFilter = ref('')
//     const sortBy = ref('date-desc')
//     const viewMode = ref('list')
//     const timelineView = ref('compact')
//     const selectedCommit = ref(null)
//     const dialogVisible = ref(false)
//     const tagDialogVisible = ref(false)
//     const compareDialogVisible = ref(false)
//     const isLoading = ref(false)
//     const currentPage = ref(1)
//     const pageSize = ref(20)

//     // 标签创建相关
//     const newTagName = ref('')
//     const newTagDescription = ref('')
//     const tagTargetCommit = ref('')

//     // 比较数据
//     const compareData = ref({
//       base: null,
//       target: null,
//       filesChanged: 0,
//       additions: 0,
//       deletions: 0,
//       changedFiles: []
//     })

//     // 模拟提交数据
//     const commits = ref([
//       {
//         hash: 'a1b2c3d4e5f6789012345678901234567890abcd',
//         author: 'Alice Johnson',
//         date: '2025-07-10 14:30:25',
//         message: 'feat: 添加用户认证功能和权限管理系统',
//         avatar: 'https://avatars.githubusercontent.com/u/1?v=4',
//         type: 'feature',
//         tags: ['v2.1.0'],
//         branches: ['main'],
//         filesChanged: 8,
//         additions: 156,
//         deletions: 23,
//         changedFiles: ['src/auth/login.js', 'src/auth/register.js', 'src/components/AuthForm.vue', 'tests/auth.test.js', 'src/middleware/auth.js', 'src/utils/token.js', 'docs/auth.md', 'package.json']
//       },
//       {
//         hash: 'd4e5f6g7h8i9012345678901234567890123abcd',
//         author: 'Bob Smith',
//         date: '2025-07-10 11:45:10',
//         message: 'fix: 修复登录页面样式问题和响应式布局',
//         avatar: 'https://avatars.githubusercontent.com/u/2?v=4',
//         type: 'bugfix',
//         tags: [],
//         branches: ['main', 'develop'],
//         filesChanged: 3,
//         additions: 12,
//         deletions: 8,
//         changedFiles: ['src/styles/login.css', 'src/components/LoginForm.vue', 'src/styles/responsive.css']
//       },
//       {
//         hash: 'g7h8i9j0k1l2345678901234567890123456abcd',
//         author: 'Charlie Brown',
//         date: '2025-07-09 16:20:33',
//         message: 'merge: 合并 feature/dashboard 分支到主分支',
//         avatar: 'https://avatars.githubusercontent.com/u/3?v=4',
//         type: 'merge',
//         tags: [],
//         branches: ['main'],
//         filesChanged: 15,
//         additions: 234,
//         deletions: 67,
//         changedFiles: ['src/dashboard/Dashboard.vue', 'src/dashboard/widgets/Chart.vue', 'src/dashboard/widgets/Stats.vue', 'src/api/dashboard.js', 'src/store/dashboard.js']
//       },
//       {
//         hash: 'j0k1l2m3n4o5678901234567890123456789abcd',
//         author: 'Diana Prince',
//         date: '2025-07-09 09:15:42',
//         message: 'docs: 更新API文档和开发指南',
//         avatar: 'https://avatars.githubusercontent.com/u/4?v=4',
//         type: 'docs',
//         tags: [],
//         branches: ['main'],
//         filesChanged: 5,
//         additions: 89,
//         deletions: 12,
//         changedFiles: ['docs/api.md', 'README.md', 'docs/getting-started.md', 'docs/deployment.md', 'CHANGELOG.md']
//       },
//       {
//         hash: 'm3n4o5p6q7r8901234567890123456789012abcd',
//         author: 'Eve Wilson',
//         date: '2025-07-08 15:30:18',
//         message: 'refactor: 重构数据库连接模块和查询优化',
//         avatar: 'https://avatars.githubusercontent.com/u/5?v=4',
//         type: 'refactor',
//         tags: ['v2.0.1'],
//         branches: ['main'],
//         filesChanged: 12,
//         additions: 178,
//         deletions: 145,
//         changedFiles: ['src/database/connection.js', 'src/database/models/User.js', 'src/database/models/Post.js', 'src/config/database.js', 'src/utils/query.js']
//       },
//       {
//         hash: 'p6q7r8s9t0u1234567890123456789012345abcd',
//         author: 'Frank Miller',
//         date: '2025-07-08 10:45:55',
//         message: 'feat: 实现实时通知系统',
//         avatar: 'https://avatars.githubusercontent.com/u/6?v=4',
//         type: 'feature',
//         tags: [],
//         branches: ['main', 'develop'],
//         filesChanged: 6,
//         additions: 98,
//         deletions: 5,
//         changedFiles: ['src/components/Notification.vue', 'src/services/websocket.js', 'src/store/notifications.js', 'src/utils/notification.js']
//       },
//       {
//         hash: 's9t0u1v2w3x4567890123456789012345678abcd',
//         author: 'Grace Lee',
//         date: '2025-07-07 14:22:11',
//         message: 'style: 统一代码格式和ESLint规则',
//         avatar: 'https://avatars.githubusercontent.com/u/7?v=4',
//         type: 'style',
//         tags: [],
//         branches: ['main'],
//         filesChanged: 20,
//         additions: 45,
//         deletions: 38,
//         changedFiles: ['.eslintrc.js', '.prettierrc', 'src/components/*.vue', 'src/utils/*.js']
//       },
//       {
//         hash: 'v2w3x4y5z6a7890123456789012345678901abcd',
//         author: 'Henry Davis',
//         date: '2025-07-07 09:33:44',
//         message: 'test: 添加单元测试和集成测试',
//         avatar: 'https://avatars.githubusercontent.com/u/8?v=4',
//         type: 'test',
//         tags: [],
//         branches: ['main'],
//         filesChanged: 8,
//         additions: 156,
//         deletions: 0,
//         changedFiles: ['tests/unit/auth.test.js', 'tests/unit/dashboard.test.js', 'tests/integration/api.test.js', 'jest.config.js']
//       }
//     ])

//     // 计算属性
//     const uniqueAuthors = computed(() => {
//       return [...new Set(commits.value.map(commit => commit.author))]
//     })

//     const filteredCommits = computed(() => {
//       let filtered = [...commits.value]

//       // 搜索筛选
//       if (searchQuery.value) {
//         const query = searchQuery.value.toLowerCase()
//         filtered = filtered.filter(commit =>
//           commit.message.toLowerCase().includes(query) ||
//           commit.author.toLowerCase().includes(query) ||
//           commit.hash.toLowerCase().includes(query)
//         )
//       }

//       // 作者筛选
//       if (authorFilter.value) {
//         filtered = filtered.filter(commit => commit.author === authorFilter.value)
//       }

//       // 时间筛选
//       if (timeFilter.value) {
//         const now = new Date()
//         const today = new Date(now.getFullYear(), now.getMonth(), now.getDate())

//         filtered = filtered.filter(commit => {
//           const commitDate = new Date(commit.date)

//           switch (timeFilter.value) {
//             case 'today':
//               return commitDate >= today
//             case 'week':
//               const weekAgo = new Date(today.getTime() - 7 * 24 * 60 * 60 * 1000)
//               return commitDate >= weekAgo
//             case 'month':
//               const monthAgo = new Date(today.getTime() - 30 * 24 * 60 * 60 * 1000)
//               return commitDate >= monthAgo
//             default:
//               return true
//           }
//         })
//       }

//       // 排序
//       filtered.sort((a, b) => {
//         switch (sortBy.value) {
//           case 'date-desc':
//             return new Date(b.date) - new Date(a.date)
//           case 'date-asc':
//             return new Date(a.date) - new Date(b.date)
//           case 'author':
//             return a.author.localeCompare(b.author)
//           case 'message':
//             return a.message.localeCompare(b.message)
//           default:
//             return 0
//         }
//       })

//       return filtered
//     })

//     const paginatedCommits = computed(() => {
//       return filteredCommits.value.slice(0, currentPage.value * pageSize.value)
//     })

//     const hasMore = computed(() => {
//       return filteredCommits.value.length > currentPage.value * pageSize.value
//     })

//     // 方法
//     const close = () => {
//       emit('close')
//     }

//     const onBranchChange = () => {
//       console.log('切换到分支:', currentBranch.value)
//       // 这里可以实现分支切换逻辑
//     }

//     const onSearch = () => {
//       // 搜索逻辑已在计算属性中处理
//       currentPage.value = 1 // 重置分页
//     }

//     const applyFilters = () => {
//       // 筛选逻辑已在计算属性中处理
//       currentPage.value = 1 // 重置分页
//     }

//     const applySorting = () => {
//       // 排序逻辑已在计算属性中处理
//       currentPage.value = 1 // 重置分页
//     }

//     const clearFilters = () => {
//       searchQuery.value = ''
//       authorFilter.value = ''
//       timeFilter.value = ''
//       sortBy.value = 'date-desc'
//       currentPage.value = 1
//     }

//     const selectCommit = (commit) => {
//       selectedCommit.value = commit
//     }

//     const showCommitDetails = (commit) => {
//       selectedCommit.value = commit
//       dialogVisible.value = true
//     }

//     const closeDialog = () => {
//       dialogVisible.value = false
//     }

//     const compareCommit = (commit) => {
//       // 模拟比较数据
//       compareData.value = {
//         base: commits.value[commits.value.indexOf(commit) + 1] || null,
//         target: commit,
//         filesChanged: commit.filesChanged || 0,
//         additions: commit.additions || 0,
//         deletions: commit.deletions || 0,
//         changedFiles: commit.changedFiles?.map(name => ({
//           name,
//           additions: Math.floor(Math.random() * 20),
//           deletions: Math.floor(Math.random() * 10)
//         })) || []
//       }
//       compareDialogVisible.value = true
//     }

//     const closeCompareDialog = () => {
//       compareDialogVisible.value = false
//     }

//     const revertToCommit = (commit) => {
//       if (confirm(`确定要回滚到提交 ${commit.hash.slice(0, 7)} 吗？\n\n这将撤销此提交之后的所有更改。`)) {
//         console.log('回滚到提交:', commit.hash)
//         // 这里实现回滚逻辑
//         alert('回滚操作已提交，请等待处理完成。')
//       }
//     }

//     const compareWithCurrent = () => {
//       console.log('与当前版本比较:', selectedCommit.value.hash)
//       compareCommit(selectedCommit.value)
//       closeDialog()
//     }

//     const createBranchFromCommit = () => {
//       const branchName = prompt('请输入新分支名称:', `feature/from-${selectedCommit.value.hash.slice(0, 7)}`)
//       if (branchName && branchName.trim()) {
//         console.log('从提交创建分支:', selectedCommit.value.hash, '分支名:', branchName)
//         branches.value.push(branchName.trim())
//         alert(`分支 "${branchName}" 创建成功！`)
//         closeDialog()
//       }
//     }

//     const createTag = () => {
//       tagDialogVisible.value = true
//     }

//     const closeTagDialog = () => {
//       tagDialogVisible.value = false
//       newTagName.value = ''
//       newTagDescription.value = ''
//       tagTargetCommit.value = ''
//     }

//     const confirmCreateTag = () => {
//       if (!newTagName.value.trim()) {
//         alert('请输入标签名称')
//         return
//       }

//       const targetHash = tagTargetCommit.value || commits.value[0].hash
//       const targetCommit = commits.value.find(c => c.hash === targetHash)

//       if (targetCommit) {
//         if (!targetCommit.tags) {
//           targetCommit.tags = []
//         }
//         targetCommit.tags.push(newTagName.value.trim())
//       }

//       console.log('创建标签:', {
//         name: newTagName.value.trim(),
//         description: newTagDescription.value,
//         commit: targetHash
//       })

//       alert(`标签 "${newTagName.value.trim()}" 创建成功！`)
//       closeTagDialog()
//     }

//     const createBranch = () => {
//       const branchName = prompt('请输入新分支名称:', 'feature/new-feature')
//       if (branchName && branchName.trim()) {
//         console.log('创建分支:', branchName.trim())
//         branches.value.push(branchName.trim())
//         alert(`分支 "${branchName.trim()}" 创建成功！`)
//       }
//     }

//     const loadMore = () => {
//       if (hasMore.value && !isLoading.value) {
//         isLoading.value = true
//         setTimeout(() => {
//           currentPage.value++
//           isLoading.value = false
//         }, 500)
//       }
//     }

//     const getCommitTypeClass = (commit) => {
//       return {
//         'merge-commit': commit.type === 'merge',
//         'tag-commit': commit.tags && commit.tags.length > 0,
//         'feature-commit': commit.type === 'feature',
//         'bugfix-commit': commit.type === 'bugfix',
//         'docs-commit': commit.type === 'docs',
//         'refactor-commit': commit.type === 'refactor',
//         'style-commit': commit.type === 'style',
//         'test-commit': commit.type === 'test'
//       }
//     }

//     const getConnectorClass = (currentCommit, nextCommit) => {
//       if (!nextCommit) return ''

//       if (currentCommit.type === 'merge' || nextCommit.type === 'merge') {
//         return 'merge-connector'
//       }

//       return 'normal-connector'
//     }

//     const getCommitTypePrefix = (type) => {
//       const prefixes = {
//         feature: 'feat:',
//         bugfix: 'fix:',
//         docs: 'docs:',
//         style: 'style:',
//         refactor: 'refactor:',
//         test: 'test:',
//         merge: 'merge:'
//       }
//       return prefixes[type] || ''
//     }

//     const getCommitTypeText = (type) => {
//       const texts = {
//         feature: '新功能',
//         bugfix: '修复',
//         docs: '文档',
//         style: '样式',
//         refactor: '重构',
//         test: '测试',
//         merge: '合并'
//       }
//       return texts[type] || '其他'
//     }

//     const formatDate = (dateString) => {
//       const date = new Date(dateString)
//       return date.toLocaleString('zh-CN', {
//         year: 'numeric',
//         month: '2-digit',
//         day: '2-digit',
//         hour: '2-digit',
//         minute: '2-digit',
//         second: '2-digit'
//       })
//     }

//     const formatTime = (dateString) => {
//       const date = new Date(dateString)
//       const now = new Date()
//       const diff = now - date

//       if (diff < 60000) { // 1分钟内
//         return '刚刚'
//       } else if (diff < 3600000) { // 1小时内
//         return `${Math.floor(diff / 60000)}分钟前`
//       } else if (diff < 86400000) { // 1天内
//         return `${Math.floor(diff / 3600000)}小时前`
//       } else if (diff < 604800000) { // 1周内
//         return `${Math.floor(diff / 86400000)}天前`
//       } else {
//         return date.toLocaleDateString('zh-CN', {
//           month: '2-digit',
//           day: '2-digit'
//         })
//       }
//     }

//     // 监听选中的提交变化
//     watch(selectedCommit, (newCommit) => {
//       if (newCommit) {
//         console.log('选中提交:', newCommit.hash.slice(0, 7), newCommit.message)
//       }
//     })

//     return {
//       docsUrl,
//       currentBranch,
//       branches,
//       searchQuery,
//       authorFilter,
//       timeFilter,
//       sortBy,
//       viewMode,
//       timelineView,
//       selectedCommit,
//       dialogVisible,
//       tagDialogVisible,
//       compareDialogVisible,
//       isLoading,
//       currentPage,
//       pageSize,
//       newTagName,
//       newTagDescription,
//       tagTargetCommit,
//       compareData,
//       commits,
//       uniqueAuthors,
//       filteredCommits,
//       paginatedCommits,
//       hasMore,
//       close,
//       onBranchChange,
//       onSearch,
//       applyFilters,
//       applySorting,
//       clearFilters,
//       selectCommit,
//       showCommitDetails,
//       closeDialog,
//       compareCommit,
//       closeCompareDialog,
//       revertToCommit,
//       compareWithCurrent,
//       createBranchFromCommit,
//       createTag,
//       closeTagDialog,
//       confirmCreateTag,
//       createBranch,
//       loadMore,
//       getCommitTypeClass,
//       getConnectorClass,
//       getCommitTypePrefix,
//       getCommitTypeText,
//       formatDate,
//       formatTime
//     }
//   }
// }
</script>

<style lang="less" scoped>
.version-control-container {
  width: 50vw;
  height: 100%;
  background-color: var(--ti-lowcode-plugin-version-control-bg, #ffffff);
  box-shadow: 6px 0px 3px 0px rgba(0, 0, 0, 0.05);
  display: flex;
  flex-direction: column;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;

  .version-control-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    border-bottom: 1px solid var(--ti-lowcode-plugin-version-control-head-border-bottom-color, #e5e7eb);
    padding: 12px 16px;
    background: linear-gradient(135deg, #fafafa 0%, #f5f5f5 100%);
    min-height: 60px;

    .header-left {
      display: flex;
      align-items: center;

      .title {
        color: var(--ti-lowcode-plugin-panel-title-color, #1f2937);
        font-weight: var(--ti-lowcode-plugin-panel-title-font-weight, 600);
        font-size: 16px;
      }

      .link-button {
        display: inline-block;
        width: 24px;
        height: 24px;
        margin-left: 8px;
        cursor: pointer;
        color: var(--ti-lowcode-plugin-version-control-help-link-color, #6b7280);
        transition: color 0.2s;
        &:hover {
          color: #3b82f6;
        }
      }
    }

    .header-center {
      display: flex;
      align-items: center;
      gap: 16px;
      flex: 1;
      justify-content: center;

      .branch-selector {
        .branch-select {
          padding: 8px 12px;
          border: 1px solid #d1d5db;
          border-radius: 8px;
          background: white;
          font-size: 14px;
          min-width: 140px;
          cursor: pointer;
          transition: all 0.2s;

          &:focus {
            outline: none;
            border-color: #3b82f6;
            box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
          }

          &:hover {
            border-color: #9ca3af;
          }
        }
      }

      .search-container {
        position: relative;

        .search-input {
          padding: 8px 12px 8px 40px;
          border: 1px solid #d1d5db;
          border-radius: 8px;
          width: 320px;
          font-size: 14px;
          transition: all 0.2s;

          &:focus {
            outline: none;
            border-color: #3b82f6;
            box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
          }

          &:hover {
            border-color: #9ca3af;
          }

          &::placeholder {
            color: #9ca3af;
          }
        }

        .search-icon {
          position: absolute;
          left: 12px;
          top: 50%;
          transform: translateY(-50%);
          width: 16px;
          height: 16px;
          color: #6b7280;
          pointer-events: none;
        }
      }
    }

    .header-right {
      display: flex;
      align-items: center;
      gap: 8px;

      .action-btn {
        display: flex;
        align-items: center;
        gap: 6px;
        padding: 8px 12px;
        border: 1px solid #d1d5db;
        border-radius: 8px;
        background: white;
        color: #374151;
        font-size: 14px;
        cursor: pointer;
        transition: all 0.2s;

        svg {
          width: 16px;
          height: 16px;
        }

        &:hover {
          background: #f9fafb;
          border-color: #9ca3af;
          transform: translateY(-1px);
        }

        &:active {
          transform: translateY(0);
        }
      }
    }
  }

  .version-control-content {
    flex: 1;
    display: flex;
    flex-direction: column;
    overflow: hidden;

    .filters-container {
      display: flex;
      align-items: center;
      gap: 20px;
      padding: 12px 16px;
      background: #f9fafb;
      border-bottom: 1px solid #e5e7eb;
      flex-wrap: wrap;

      .filter-group {
        display: flex;
        align-items: center;
        gap: 8px;

        label {
          font-size: 14px;
          color: #374151;
          font-weight: 500;
          white-space: nowrap;
        }

        .filter-select {
          padding: 6px 10px;
          border: 1px solid #d1d5db;
          border-radius: 6px;
          font-size: 14px;
          background: white;
          cursor: pointer;
          transition: all 0.2s;

          &:focus {
            outline: none;
            border-color: #3b82f6;
          }

          &:hover {
            border-color: #9ca3af;
          }
        }

        .clear-filters-btn {
          padding: 6px 12px;
          border: 1px solid #d1d5db;
          border-radius: 6px;
          background: white;
          color: #6b7280;
          font-size: 14px;
          cursor: pointer;
          transition: all 0.2s;

          &:hover {
            background: #f3f4f6;
            color: #374151;
          }
        }

        &.stats {
          margin-left: auto;

          .stats-text {
            font-size: 14px;
            color: #6b7280;
            font-weight: 500;
          }
        }
      }
    }

    .main-content {
      flex: 1;
      display: flex;
      overflow: hidden;

      .timeline-container {
        width: 320px;
        border-right: 1px solid #e5e7eb;
        background: #fafafa;
        display: flex;
        flex-direction: column;

        .timeline-header {
          padding: 16px;
          border-bottom: 1px solid #e5e7eb;
          display: flex;
          justify-content: space-between;
          align-items: center;

          h3 {
            margin: 0;
            font-size: 16px;
            color: #1f2937;
            font-weight: 600;
          }

          .timeline-controls {
            display: flex;
            gap: 4px;

            .view-toggle {
              padding: 4px 8px;
              border: 1px solid #d1d5db;
              background: white;
              color: #6b7280;
              font-size: 12px;
              cursor: pointer;
              transition: all 0.2s;

              &:first-child {
                border-radius: 4px 0 0 4px;
              }

              &:last-child {
                border-radius: 0 4px 4px 0;
                border-left: none;
              }

              &.active {
                background: #3b82f6;
                color: white;
                border-color: #3b82f6;
              }

              &:hover:not(.active) {
                background: #f3f4f6;
              }
            }
          }
        }

        .timeline-content {
          flex: 1;
          overflow-y: auto;
          padding: 8px;

          &.detailed .timeline-item {
            .timeline-info {
              .commit-message-mini {
                display: block;
              }
            }
          }

          .timeline-item {
            display: flex;
            align-items: flex-start;
            gap: 12px;
            padding: 10px 8px;
            border-radius: 8px;
            cursor: pointer;
            transition: all 0.2s;
            margin-bottom: 4px;

            &:hover {
              background: rgba(59, 130, 246, 0.05);
              transform: translateX(2px);
            }

            &.selected {
              background: rgba(59, 130, 246, 0.1);
              border: 1px solid #3b82f6;
              transform: translateX(4px);
            }

            &.merge-item {
              background: rgba(139, 92, 246, 0.05);

              &:hover {
                background: rgba(139, 92, 246, 0.1);
              }

              &.selected {
                background: rgba(139, 92, 246, 0.15);
                border-color: #8b5cf6;
              }
            }

            .timeline-line {
              display: flex;
              flex-direction: column;
              align-items: center;
              position: relative;

              .timeline-dot {
                width: 28px;
                height: 28px;
                border-radius: 50%;
                display: flex;
                align-items: center;
                justify-content: center;
                position: relative;
                z-index: 2;
                border: 2px solid white;
                box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);

                &.merge-commit {
                  background: #8b5cf6;
                  color: white;
                }

                &.tag-commit {
                  background: #f59e0b;
                  color: white;
                }

                &.feature-commit {
                  background: #10b981;
                  color: white;
                }

                &.bugfix-commit {
                  background: #ef4444;
                  color: white;
                }

                &.docs-commit {
                  background: #6b7280;
                  color: white;
                }

                &.refactor-commit {
                  background: #3b82f6;
                  color: white;
                }

                &.style-commit {
                  background: #ec4899;
                  color: white;
                }

                &.test-commit {
                  background: #84cc16;
                  color: white;
                }

                svg {
                  width: 14px;
                  height: 14px;
                }

                .commit-dot {
                  width: 10px;
                  height: 10px;
                  background: #d1d5db;
                  border-radius: 50%;
                }
              }

              .timeline-connector {
                width: 2px;
                height: 40px;
                background: #e5e7eb;
                margin-top: 4px;

                &.merge-connector {
                  background: linear-gradient(to bottom, #8b5cf6, #e5e7eb);
                }

                &.normal-connector {
                  background: #e5e7eb;
                }
              }
            }

            .timeline-info {
              flex: 1;
              min-width: 0;

              .commit-hash-mini {
                font-family: 'SF Mono', Monaco, 'Cascadia Code', 'Roboto Mono', Consolas, 'Courier New', monospace;
                font-size: 12px;
                color: #6b7280;
                margin-bottom: 2px;
                font-weight: 600;
              }

              .commit-time {
                font-size: 12px;
                color: #9ca3af;
                margin-bottom: 4px;
              }

              .commit-message-mini {
                display: none;
                font-size: 12px;
                color: #374151;
                margin-bottom: 4px;
                line-height: 1.3;
              }

              .commit-tags-mini {
                display: flex;
                gap: 4px;
                flex-wrap: wrap;

                .tag-mini {
                  background: #fef3c7;
                  color: #92400e;
                  padding: 2px 6px;
                  border-radius: 4px;
                  font-size: 10px;
                  font-weight: 500;
                }
              }
            }
          }
        }
      }

      .commits-container {
        flex: 1;
        display: flex;
        flex-direction: column;
        overflow: hidden;

        .commits-header {
          display: flex;
          justify-content: space-between;
          align-items: center;
          padding: 16px;
          border-bottom: 1px solid #e5e7eb;
          background: white;

          h3 {
            margin: 0;
            font-size: 16px;
            color: #1f2937;
            font-weight: 600;
          }

          .header-controls {
            display: flex;
            align-items: center;
            gap: 16px;

            .view-options {
              display: flex;
              gap: 4px;

              .view-btn {
                display: flex;
                align-items: center;
                gap: 4px;
                padding: 6px 10px;
                border: 1px solid #d1d5db;
                background: white;
                color: #6b7280;
                font-size: 14px;
                cursor: pointer;
                transition: all 0.2s;

                svg {
                  width: 14px;
                  height: 14px;
                }

                &:first-child {
                  border-radius: 6px 0 0 6px;
                }

                &:last-child {
                  border-radius: 0 6px 6px 0;
                  border-left: none;
                }

                &.active {
                  background: #3b82f6;
                  color: white;
                  border-color: #3b82f6;
                }

                &:hover:not(.active) {
                  background: #f3f4f6;
                }
              }
            }

            .sort-options {
              .sort-select {
                padding: 6px 10px;
                border: 1px solid #d1d5db;
                border-radius: 6px;
                font-size: 14px;
                background: white;
                cursor: pointer;

                &:focus {
                  outline: none;
                  border-color: #3b82f6;
                }
              }
            }
          }
        }

        .commits-list {
          flex: 1;
          overflow-y: auto;
          padding: 8px 16px;

          .commit-item {
            border: 1px solid #e5e7eb;
            border-radius: 12px;
            padding: 16px;
            margin-bottom: 12px;
            background: white;
            cursor: pointer;
            transition: all 0.2s;
            position: relative;

            &:hover {
              border-color: #3b82f6;
              box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
              transform: translateY(-2px);
            }

            &.selected {
              border-color: #3b82f6;
              box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
            }

            &.merge-commit {
              border-left: 4px solid #8b5cf6;
            }

            .commit-header {
              display: flex;
              justify-content: space-between;
              align-items: center;
              margin-bottom: 12px;

              .commit-meta {
                display: flex;
                align-items: center;
                gap: 12px;

                .commit-avatar {
                  width: 36px;
                  height: 36px;
                  border-radius: 50%;
                  border: 2px solid #e5e7eb;
                }

                .commit-author-info {
                  .commit-author {
                    font-weight: 600;
                    color: #1f2937;
                    font-size: 14px;
                  }

                  .commit-date {
                    font-size: 12px;
                    color: #6b7280;
                  }
                }

                .merge-badge {
                  display: flex;
                  align-items: center;
                  gap: 4px;
                  background: #8b5cf6;
                  color: white;
                  padding: 4px 8px;
                  border-radius: 6px;
                  font-size: 12px;
                  font-weight: 500;

                  svg {
                    width: 12px;
                    height: 12px;
                  }
                }
              }

              .commit-actions {
                display: flex;
                align-items: center;
                gap: 8px;

                .commit-hash {
                  font-family: 'SF Mono', Monaco, 'Cascadia Code', 'Roboto Mono', Consolas, 'Courier New', monospace;
                  font-size: 12px;
                  color: #6b7280;
                  background: #f3f4f6;
                  padding: 4px 8px;
                  border-radius: 6px;
                  font-weight: 600;
                }

                .action-buttons {
                  display: flex;
                  gap: 4px;

                  .action-btn {
                    display: flex;
                    align-items: center;
                    justify-content: center;
                    width: 32px;
                    height: 32px;
                    border: 1px solid #d1d5db;
                    border-radius: 6px;
                    background: white;
                    color: #374151;
                    cursor: pointer;
                    transition: all 0.2s;

                    svg {
                      width: 14px;
                      height: 14px;
                    }

                    &.danger {
                      color: #dc2626;
                      border-color: #fca5a5;

                      &:hover {
                        background: #fef2f2;
                        border-color: #dc2626;
                      }
                    }

                    &:hover {
                      background: #f9fafb;
                      border-color: #9ca3af;
                      transform: translateY(-1px);
                    }
                  }
                }
              }
            }

            .commit-message {
              font-size: 14px;
              color: #1f2937;
              margin-bottom: 12px;
              line-height: 1.5;
              display: flex;
              align-items: flex-start;
              gap: 8px;

              .commit-type-prefix {
                font-weight: 600;
                font-size: 12px;
                padding: 2px 6px;
                border-radius: 4px;
                text-transform: uppercase;
                letter-spacing: 0.5px;

                &.feature {
                  background: #dcfce7;
                  color: #166534;
                }

                &.bugfix {
                  background: #fee2e2;
                  color: #991b1b;
                }

                &.docs {
                  background: #f3f4f6;
                  color: #374151;
                }

                &.style {
                  background: #fce7f3;
                  color: #be185d;
                }

                &.refactor {
                  background: #dbeafe;
                  color: #1e40af;
                }

                &.test {
                  background: #ecfccb;
                  color: #365314;
                }

                &.merge {
                  background: #ede9fe;
                  color: #6b21a8;
                }
              }
            }

            .commit-labels {
              display: flex;
              gap: 8px;
              margin-bottom: 8px;
              flex-wrap: wrap;

              .commit-tag {
                display: flex;
                align-items: center;
                gap: 4px;
                background: #fef3c7;
                color: #92400e;
                padding: 4px 8px;
                border-radius: 6px;
                font-size: 12px;
                font-weight: 500;

                svg {
                  width: 12px;
                  height: 12px;
                }
              }

              .commit-branch {
                display: flex;
                align-items: center;
                gap: 4px;
                background: #dbeafe;
                color: #1e40af;
                padding: 4px 8px;
                border-radius: 6px;
                font-size: 12px;
                font-weight: 500;

                svg {
                  width: 12px;
                  height: 12px;
                }
              }
            }

            .commit-details {
              border-top: 1px solid #f3f4f6;
              padding-top: 12px;
              margin-top: 12px;

              .commit-stats {
                display: flex;
                gap: 16px;
                margin-bottom: 12px;

                .stat-item {
                  display: flex;
                  align-items: center;
                  gap: 4px;
                  font-size: 12px;

                  svg {
                    width: 14px;
                    height: 14px;
                    color: #6b7280;
                  }

                  .stat-value {
                    font-weight: 600;
                  }

                  &.additions .stat-value {
                    color: #10b981;
                  }

                  &.deletions .stat-value {
                    color: #ef4444;
                  }
                }
              }

              .commit-files {
                .files-header {
                  font-size: 12px;
                  color: #6b7280;
                  font-weight: 500;
                  margin-bottom: 6px;
                }

                .files-list {
                  .changed-file {
                    display: flex;
                    align-items: center;
                    gap: 6px;
                    font-size: 12px;
                    color: #6b7280;
                    font-family: 'SF Mono', Monaco, 'Cascadia Code', 'Roboto Mono', Consolas, 'Courier New', monospace;
                    margin-bottom: 2px;

                    svg {
                      width: 12px;
                      height: 12px;
                    }
                  }

                  .more-files {
                    font-size: 12px;
                    color: #9ca3af;
                    font-style: italic;
                    margin-top: 4px;
                  }
                }
              }
            }
          }

          .empty-state {
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            padding: 60px 20px;
            text-align: center;
            color: #6b7280;

            svg {
              width: 48px;
              height: 48px;
              margin-bottom: 16px;
              opacity: 0.5;
            }

            h3 {
              margin: 0 0 8px 0;
              font-size: 18px;
              color: #374151;
            }

            p {
              margin: 0 0 16px 0;
              font-size: 14px;
            }

            .clear-btn {
              padding: 8px 16px;
              border: 1px solid #3b82f6;
              border-radius: 6px;
              background: #3b82f6;
              color: white;
              cursor: pointer;
              transition: all 0.2s;

              &:hover {
                background: #2563eb;
              }
            }
          }

          &.detailed .commit-item {
            .commit-details {
              display: block;
            }
          }
        }

        .pagination-container {
          display: flex;
          justify-content: space-between;
          align-items: center;
          padding: 16px;
          border-top: 1px solid #e5e7eb;
          background: white;

          .load-more-btn {
            display: flex;
            align-items: center;
            gap: 8px;
            padding: 10px 20px;
            border: 1px solid #d1d5db;
            border-radius: 8px;
            background: white;
            color: #374151;
            cursor: pointer;
            transition: all 0.2s;
            font-weight: 500;

            .loading-icon {
              width: 16px;
              height: 16px;
              animation: spin 1s linear infinite;
            }

            &:hover:not(:disabled) {
              background: #f9fafb;
              border-color: #9ca3af;
              transform: translateY(-1px);
            }

            &:disabled {
              opacity: 0.5;
              cursor: not-allowed;
            }
          }

          .pagination-info {
            font-size: 14px;
            color: #6b7280;
            font-weight: 500;
          }
        }
      }
    }
  }

  // 对话框样式
  .dialog-overlay {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: rgba(0, 0, 0, 0.5);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 1000;
    backdrop-filter: blur(4px);

    .dialog-content {
      background: white;
      border-radius: 12px;
      width: 90%;
      max-width: 600px;
      max-height: 80vh;
      overflow: hidden;
      display: flex;
      flex-direction: column;
      box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.1), 0 10px 10px -5px rgba(0, 0, 0, 0.04);

      &.small {
        max-width: 400px;
      }

      &.large {
        max-width: 800px;
      }

      .dialog-header {
        display: flex;
        justify-content: space-between;
        align-items: center;
        padding: 20px 24px;
        border-bottom: 1px solid #e5e7eb;
        background: #fafafa;

        h3 {
          margin: 0;
          font-size: 18px;
          color: #1f2937;
          font-weight: 600;
        }

        .close-btn {
          background: none;
          border: none;
          cursor: pointer;
          padding: 6px;
          color: #6b7280;
          border-radius: 6px;
          transition: all 0.2s;

          svg {
            width: 20px;
            height: 20px;
          }

          &:hover {
            color: #374151;
            background: #f3f4f6;
          }
        }
      }

      .dialog-body {
        flex: 1;
        overflow-y: auto;
        padding: 24px;

        .detail-section {
          margin-bottom: 24px;

          &:last-child {
            margin-bottom: 0;
          }

          h4 {
            margin: 0 0 16px 0;
            font-size: 16px;
            color: #1f2937;
            font-weight: 600;
            border-bottom: 2px solid #e5e7eb;
            padding-bottom: 8px;
          }

          .detail-grid {
            display: grid;
            gap: 16px;

            .detail-item {
              display: flex;
              gap: 12px;
              align-items: flex-start;

              &.full-width {
                grid-column: 1 / -1;
                flex-direction: column;
                gap: 8px;
              }

              label {
                font-weight: 600;
                color: #374151;
                min-width: 80px;
                font-size: 14px;
              }

              .author-info {
                display: flex;
                align-items: center;
                gap: 8px;

                .author-avatar {
                  width: 24px;
                  height: 24px;
                  border-radius: 50%;
                }
              }

              .hash-full {
                font-family: 'SF Mono', Monaco, 'Cascadia Code', 'Roboto Mono', Consolas, 'Courier New', monospace;
                font-size: 12px;
                background: #f3f4f6;
                padding: 6px 10px;
                border-radius: 6px;
                word-break: break-all;
                border: 1px solid #e5e7eb;
              }

              .commit-type-badge {
                padding: 4px 8px;
                border-radius: 6px;
                font-size: 12px;
                font-weight: 600;
                text-transform: uppercase;
                letter-spacing: 0.5px;

                &.feature {
                  background: #dcfce7;
                  color: #166534;
                }

                &.bugfix {
                  background: #fee2e2;
                  color: #991b1b;
                }

                &.docs {
                  background: #f3f4f6;
                  color: #374151;
                }

                &.style {
                  background: #fce7f3;
                  color: #be185d;
                }

                &.refactor {
                  background: #dbeafe;
                  color: #1e40af;
                }

                &.test {
                  background: #ecfccb;
                  color: #365314;
                }

                &.merge {
                  background: #ede9fe;
                  color: #6b21a8;
                }
              }

              .commit-message-full {
                background: #f9fafb;
                padding: 12px;
                border-radius: 8px;
                border: 1px solid #e5e7eb;
                line-height: 1.5;
                font-size: 14px;
              }
            }
          }

          .files-list-detailed {
            max-height: 200px;
            overflow-y: auto;
            border: 1px solid #e5e7eb;
            border-radius: 8px;
            background: #fafafa;

            .file-item {
              display: flex;
              align-items: center;
              gap: 8px;
              padding: 10px 12px;
              border-bottom: 1px solid #f3f4f6;
              font-family: 'SF Mono', Monaco, 'Cascadia Code', 'Roboto Mono', Consolas, 'Courier New', monospace;
              font-size: 13px;
              color: #374151;

              svg {
                width: 14px;
                height: 14px;
                color: #6b7280;
              }

              &:last-child {
                border-bottom: none;
              }

              &:hover {
                background: #f3f4f6;
              }
            }
          }

          .labels-container {
            .label-group {
              display: flex;
              align-items: center;
              gap: 8px;
              margin-bottom: 8px;

              .label-type {
                font-weight: 600;
                color: #374151;
                font-size: 14px;
                min-width: 60px;
              }

              .label-item {
                padding: 4px 8px;
                border-radius: 6px;
                font-size: 12px;
                font-weight: 500;

                &.tag {
                  background: #fef3c7;
                  color: #92400e;
                }

                &.branch {
                  background: #dbeafe;
                  color: #1e40af;
                }
              }
            }
          }
        }

        .detail-actions {
          display: flex;
          gap: 12px;
          padding-top: 20px;
          border-top: 1px solid #e5e7eb;
          flex-wrap: wrap;

          .action-btn {
            display: flex;
            align-items: center;
            gap: 6px;
            padding: 10px 16px;
            border: 1px solid #d1d5db;
            border-radius: 8px;
            cursor: pointer;
            transition: all 0.2s;
            font-weight: 500;
            font-size: 14px;

            svg {
              width: 16px;
              height: 16px;
            }

            &.primary {
              background: #3b82f6;
              color: white;
              border-color: #3b82f6;

              &:hover {
                background: #2563eb;
                transform: translateY(-1px);
              }
            }

            &.danger {
              color: #dc2626;
              border-color: #fca5a5;

              &:hover {
                background: #fef2f2;
                border-color: #dc2626;
                transform: translateY(-1px);
              }
            }

            &:not(.primary):not(.danger) {
              background: white;
              color: #374151;

              &:hover {
                background: #f9fafb;
                border-color: #9ca3af;
                transform: translateY(-1px);
              }
            }
          }
        }

        .form-group {
          margin-bottom: 20px;

          label {
            display: block;
            margin-bottom: 6px;
            font-weight: 600;
            color: #374151;
            font-size: 14px;
          }

          .form-input,
          .form-textarea,
          .form-select {
            width: 100%;
            padding: 10px 12px;
            border: 1px solid #d1d5db;
            border-radius: 8px;
            font-size: 14px;
            transition: all 0.2s;

            &:focus {
              outline: none;
              border-color: #3b82f6;
              box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
            }

            &:hover {
              border-color: #9ca3af;
            }
          }

          .form-textarea {
            resize: vertical;
            min-height: 80px;
            font-family: inherit;
          }
        }

        .dialog-actions {
          display: flex;
          justify-content: flex-end;
          gap: 12px;
          margin-top: 24px;
          padding-top: 20px;
          border-top: 1px solid #e5e7eb;

          .action-btn {
            padding: 10px 20px;
            border: 1px solid #d1d5db;
            border-radius: 8px;
            cursor: pointer;
            transition: all 0.2s;
            font-weight: 500;
            font-size: 14px;

            &.secondary {
              background: white;
              color: #374151;

              &:hover {
                background: #f9fafb;
                transform: translateY(-1px);
              }
            }

            &.primary {
              background: #3b82f6;
              color: white;
              border-color: #3b82f6;

              &:hover:not(:disabled) {
                background: #2563eb;
                transform: translateY(-1px);
              }

              &:disabled {
                opacity: 0.5;
                cursor: not-allowed;
              }
            }
          }
        }

        // 比较对话框特殊样式
        .compare-header {
          margin-bottom: 20px;

          .compare-info {
            .compare-item {
              display: flex;
              gap: 12px;
              margin-bottom: 8px;

              label {
                font-weight: 600;
                color: #374151;
                min-width: 80px;
              }

              .version-info {
                font-family: 'SF Mono', Monaco, 'Cascadia Code', 'Roboto Mono', Consolas, 'Courier New', monospace;
                font-size: 13px;
                color: #6b7280;
              }
            }
          }
        }

        .compare-stats {
          display: flex;
          gap: 16px;
          margin-bottom: 24px;

          .stat-card {
            flex: 1;
            background: #f9fafb;
            border: 1px solid #e5e7eb;
            border-radius: 8px;
            padding: 16px;
            text-align: center;

            .stat-number {
              font-size: 24px;
              font-weight: 700;
              color: #374151;
            }

            .stat-label {
              font-size: 12px;
              color: #6b7280;
              margin-top: 4px;
            }

            &.additions {
              border-color: #10b981;
              background: #ecfdf5;

              .stat-number {
                color: #10b981;
              }
            }

            &.deletions {
              border-color: #ef4444;
              background: #fef2f2;

              .stat-number {
                color: #ef4444;
              }
            }
          }
        }

        .compare-files {
          h4 {
            margin-bottom: 12px;
          }

          .files-diff-list {
            .file-diff-item {
              border: 1px solid #e5e7eb;
              border-radius: 6px;
              margin-bottom: 8px;
              background: white;

              .file-diff-header {
                display: flex;
                align-items: center;
                justify-content: space-between;
                padding: 12px;

                .file-name {
                  display: flex;
                  align-items: center;
                  gap: 8px;
                  font-family: 'SF Mono', Monaco, 'Cascadia Code', 'Roboto Mono', Consolas, 'Courier New', monospace;
                  font-size: 13px;
                  color: #374151;

                  svg {
                    width: 14px;
                    height: 14px;
                    color: #6b7280;
                  }
                }

                .file-stats {
                  display: flex;
                  gap: 8px;
                  font-size: 12px;
                  font-weight: 600;

                  .additions {
                    color: #10b981;
                  }

                  .deletions {
                    color: #ef4444;
                  }
                }
              }
            }
          }
        }
      }
    }
  }

  // 动画
  @keyframes spin {
    from {
      transform: rotate(0deg);
    }
    to {
      transform: rotate(360deg);
    }
  }

  // 响应式设计
  @media (max-width: 1200px) {
    width: 60vw;

    .main-content {
      .timeline-container {
        width: 280px;
      }
    }
  }

  @media (max-width: 900px) {
    width: 70vw;

    .version-control-header {
      .header-center {
        .search-container .search-input {
          width: 250px;
        }
      }
    }

    .main-content {
      .timeline-container {
        width: 240px;
      }
    }
  }

  @media (max-width: 768px) {
    width: 90vw;

    .version-control-header {
      flex-direction: column;
      gap: 12px;
      align-items: stretch;

      .header-center {
        justify-content: space-between;

        .search-container .search-input {
          width: 200px;
        }
      }
    }

    .main-content {
      flex-direction: column;

      .timeline-container {
        width: 100%;
        max-height: 200px;
      }
    }

    .filters-container {
      flex-direction: column;
      align-items: stretch;
      gap: 12px;

      .filter-group {
        justify-content: space-between;

        &.stats {
          margin-left: 0;
        }
      }
    }
  }
}
</style>
