<template>
  <div class="room">
    <div class="container">
      <!-- 顶部栏 -->
      <div class="top-bar animate-fadeIn">
        <button class="btn btn-secondary back-btn" @click="goBack">
          ← 返回
        </button>
        <div class="room-info">
          <span class="room-id">房间号: {{ roomId }}</span>
          <button class="copy-btn" @click="copyRoomId">
            {{ copied ? '已复制!' : '复制' }}
          </button>
        </div>
        <div class="round-info">
          第 <span class="round-number">{{ round }}</span> 轮
        </div>
      </div>

      <!-- 主游戏区域 -->
      <div class="game-area">
        <!-- 左侧：用户列表 + 排行榜 -->
        <div class="sidebar glass-card animate-fadeIn" style="animation-delay: 0.1s">
          <h3>在线玩家 ({{ users.length }})</h3>
          <div class="user-list">
            <div 
              v-for="user in users" 
              :key="user.id" 
              class="user-item"
            >
              <span class="user-avatar">{{ user.nickname.charAt(0) }}</span>
              <span class="user-name">{{ user.nickname }}</span>
              <span class="online-dot"></span>
            </div>
          </div>

          <!-- 🏆 新增：排行榜 -->
          <div class="leaderboard-section">
            <h3>🏆 房间排行榜</h3>
            <div class="leaderboard-list">
              <div 
                v-for="player in leaderboard" 
                :key="player.id" 
                class="leaderboard-item"
                :class="{ 'leaderboard-first': player.rank === 1, 'leaderboard-second': player.rank === 2, 'leaderboard-third': player.rank === 3 }"
              >
                <span class="leaderboard-rank">
                  <span v-if="player.rank === 1">🥇</span>
                  <span v-else-if="player.rank === 2">🥈</span>
                  <span v-else-if="player.rank === 3">🥉</span>
                  <span v-else>{{ player.rank }}</span>
                </span>
                <span class="leaderboard-name">{{ player.nickname }}</span>
                <span class="leaderboard-score">
                  🏆 {{ player.correctCount }}
                </span>
              </div>
              <div v-if="leaderboard.length === 0" class="empty-leaderboard">
                还没有人猜对
              </div>
            </div>
          </div>
        </div>

        <!-- 中间：游戏主区域 -->
        <div class="main-area">
          <!-- 游戏提示卡片 -->
          <div class="hint-card glass-card animate-fadeIn" style="animation-delay: 0.2s">
            <!-- 字数提示 -->
            <div class="word-length-display">
              📝 当前词语：<span class="length-number">{{ wordLength }}</span> 个字
            </div>
            
            <!-- 关联词提示 -->
            <div class="ai-hints" v-if="aiHints.length > 0">
              <div class="ai-hint-item">
                💡 提示：{{ aiHints.join('、') }}
              </div>
            </div>
            
            <div class="category-badge">{{ categoryName }}</div>
            <div class="word-hint">
              <span 
                v-for="(char, index) in wordLength" 
                :key="index" 
                class="hint-char"
              >_</span>
            </div>
            <p class="hint-text">猜词次数：{{ guessCount }} 次</p>

            <!-- 跳过投票按钮 -->
            <div class="skip-vote-section">
              <button 
                class="btn skip-btn" 
                :class="{ voted: hasVotedSkip, disabled: showWinner }"
                @click="voteSkip"
                :disabled="hasVotedSkip || showWinner"
              >
                {{ hasVotedSkip ? '✓ 已投票跳过' : '⏭️ 跳过本轮' }}
              </button>
              <div class="skip-vote-status" v-if="skipVote.total > 0">
                <div class="skip-progress-bar">
                  <div 
                    class="skip-progress-fill" 
                    :style="{ width: skipVote.percent + '%' }"
                  ></div>
                </div>
                <span class="skip-vote-text">
                  跳过投票: {{ skipVote.votes }} / {{ skipVote.total }} ({{ skipVote.percent }}%)
                </span>
                <span v-if="skipVote.canSkip" class="skip-soon">
                  即将跳过...
                </span>
              </div>
            </div>
          </div>

          <!-- 猜词输入区 -->
          <div class="input-area glass-card animate-fadeIn" style="animation-delay: 0.3s">
            <div class="guess-input-wrapper">
              <input 
                v-model="guessText" 
                type="text" 
                class="input guess-input" 
                placeholder="输入你的答案..."
                @keyup.enter="submitGuess"
                :disabled="showWinner"
              />
              <button 
                class="btn btn-primary submit-btn" 
                @click="submitGuess"
                :disabled="showWinner || !guessText.trim()"
              >
                发送
              </button>
            </div>
          </div>

          <!-- 猜词结果列表（按关联度从高到低排序） -->
          <div class="guesses-area glass-card animate-fadeIn" style="animation-delay: 0.4s">
            <h3>猜词记录（按关联度排序）</h3>
            <div class="guesses-list" ref="guessesList">
              <div 
                v-for="(guess, index) in sortedGuesses" 
                :key="index" 
                class="guess-item"
                :style="{ animationDelay: `${index * 0.05}s` }"
              >
                <div class="guess-user">
                  <span class="guess-avatar">{{ guess.nickname.charAt(0) }}</span>
                  <span class="guess-name">{{ guess.nickname }}</span>
                </div>
                <div class="guess-text">{{ guess.text }}</div>
                <div class="similarity" :class="getSimilarityClass(guess.similarity.score)">
                  <div class="similarity-bar">
                    <div 
                      class="similarity-fill" 
                      :style="{ width: guess.similarity.score + '%' }"
                    ></div>
                  </div>
                  <span class="similarity-score">{{ guess.similarity.score }}%</span>
                </div>
              </div>
              <div v-if="guesses.length === 0" class="empty-state">
                还没有人猜词，快来第一个！
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- 胜利弹窗 -->
      <div v-if="showWinner" class="winner-modal animate-fadeIn">
        <div class="winner-content glass-card">
          <div class="winner-icon">🎉</div>
          <h2>{{ winner.nickname }} 猜对了！</h2>
          <p class="winner-word">正确答案: <span>{{ correctWord }}</span></p>
          <p class="countdown">3秒后开始下一轮...</p>
        </div>
      </div>

      <!-- 跳过弹窗 -->
      <div v-if="showSkipped" class="winner-modal animate-fadeIn">
        <div class="winner-content glass-card">
          <div class="winner-icon">⏭️</div>
          <h2>本轮已跳过</h2>
          <p class="winner-word">正确答案: <span>{{ correctWord }}</span></p>
          <p class="countdown">3秒后开始下一轮...</p>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, nextTick, watch } from 'vue'
import { useRouter, useRoute } from 'vue-router'
import { useWebSocket } from '../composables/useWebSocket'

const router = useRouter()
const route = useRoute()
const { connect, send, on, disconnect } = useWebSocket()

const roomId = ref(route.params.roomId)
const nickname = ref(localStorage.getItem('nickname') || '匿名玩家')
const userId = ref(localStorage.getItem('userId'))
const users = ref([])
const round = ref(1)
const wordLength = ref(0)
const category = ref('')
const guessText = ref('')
const guesses = ref([])
const showWinner = ref(false)
const showSkipped = ref(false)
const winner = ref(null)
const correctWord = ref('')
const copied = ref(false)
const guessesList = ref(null)
const guessCount = ref(0)
const aiHints = ref([])
const skipVote = ref({ votes: 0, total: 0, percent: 0, canSkip: false })
const hasVotedSkip = ref(false)

// 🏆 新增：排行榜数据
const leaderboard = ref([])

const sortedGuesses = computed(() => {
  return [...guesses.value].sort((a, b) => {
    if (b.similarity.score !== a.similarity.score) {
      return b.similarity.score - a.similarity.score
    }
    return a.timestamp - b.timestamp
  })
})

const categoryNames = {
  animals: '🐾 动物',
  food: '🍔 食物',
  profession: '💼 职业',
  transportation: '🚗 交通工具',
  electronics: '📱 电子产品',
  sports: '⚽ 运动',
  nature: '🌿 自然景观',
  daily: '🏠 日常用品',
  places: '🏛️ 建筑场所',
  abstract: '💭 抽象概念',
  idiom: '📜 成语',
  anime: '🎬 动漫',
  game: '🎮 游戏',
  movie: '🎥 影视',
  brand: '🏷️ 品牌'
}

const categoryName = ref('')

watch(category, (newVal) => {
  categoryName.value = categoryNames[newVal] || newVal
})

function voteSkip() {
  if (hasVotedSkip.value || showWinner.value) return
  
  send('VOTE_SKIP', {
    roomId: roomId.value
  })
  
  hasVotedSkip.value = true
}

onMounted(async () => {
  await connect()
  
  const savedToken = localStorage.getItem('userToken')
  const savedUserId = localStorage.getItem('userId')
  userId.value = savedUserId
  
  send('JOIN_ROOM', {
    roomId: roomId.value,
    nickname: nickname.value,
    token: savedToken,
    userId: savedUserId
  })

  on('ROOM_JOINED', (data) => {
    const state = data.gameState || data.roomInfo
    users.value = state.users
    round.value = state.round
    wordLength.value = state.wordLength || 0
    category.value = state.currentCategory
    guesses.value = state.guesses || []
    guessCount.value = state.guessCount || 0
    aiHints.value = state.aiHints || []
    skipVote.value = state.skipVote || { votes: 0, total: 0, percent: 0, canSkip: false }
    leaderboard.value = state.leaderboard || [] // 🏆 初始化排行榜
  })

  on('USER_JOINED', (data) => {
    const state = data.gameState || data.roomInfo
    users.value = state.users
    skipVote.value = state.skipVote || skipVote.value
    leaderboard.value = state.leaderboard || leaderboard.value
  })

  on('USER_LEFT', (data) => {
    const state = data.gameState || data.roomInfo
    users.value = state.users
    skipVote.value = state.skipVote || skipVote.value
    leaderboard.value = state.leaderboard || leaderboard.value
  })

  on('USER_OFFLINE', (data) => {
    const state = data.gameState || data.roomInfo
    users.value = state.users
    skipVote.value = state.skipVote || skipVote.value
    leaderboard.value = state.leaderboard || leaderboard.value
  })

  on('NEW_ROUND', (data) => {
    round.value = data.round
    wordLength.value = data.wordLength
    category.value = data.category
    guesses.value = []
    showWinner.value = false
    showSkipped.value = false
    winner.value = null
    correctWord.value = ''
    guessCount.value = 0
    aiHints.value = []
    skipVote.value = data.skipVote || { votes: 0, total: 0, percent: 0, canSkip: false }
    hasVotedSkip.value = false
  })

  on('NEW_GUESS', (data) => {
    guesses.value = data.allGuesses
    guessCount.value = data.guessCount
    aiHints.value = data.aiHints || []
    skipVote.value = data.skipVote || skipVote.value
    nextTick(() => {
      if (guessesList.value) {
        guessesList.value.scrollTop = guessesList.value.scrollHeight
      }
    })
  })

  on('SKIP_VOTE_UPDATE', (data) => {
    skipVote.value = data
  })

  on('ROUND_SKIPPED', (data) => {
    showSkipped.value = true
    correctWord.value = data.correctWord
  })

  // 🏆 新增：监听排行榜更新
  on('LEADERBOARD_UPDATE', (data) => {
    leaderboard.value = data.leaderboard
  })

  on('ROUND_WON', (data) => {
    showWinner.value = true
    winner.value = data.winner
    correctWord.value = data.correctWord
  })

  on('ERROR', (data) => {
    alert(data.message)
    if (data.message.includes('房间不存在')) {
      router.push('/')
    }
  })
})

function goBack() {
  send('LEAVE_ROOM', { roomId: roomId.value })
  disconnect()
  router.push('/')
}

function copyRoomId() {
  navigator.clipboard.writeText(roomId.value)
  copied.value = true
  setTimeout(() => copied.value = false, 2000)
}

function submitGuess() {
  if (!guessText.value.trim() || showWinner.value) return
  
  send('MAKE_GUESS', {
    roomId: roomId.value,
    text: guessText.value.trim()
  })
  
  guessText.value = ''
}

function getSimilarityClass(score) {
  if (score >= 80) return 'high'
  if (score >= 50) return 'medium'
  return 'low'
}
</script>

<style scoped>
.room {
  min-height: 100vh;
  padding: 20px;
}

.container {
  max-width: 1200px;
  margin: 0 auto;
  position: relative;
  z-index: 1;
}

.top-bar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 20px;
  flex-wrap: wrap;
  gap: 12px;
}

.back-btn {
  padding: 10px 20px;
}

.room-info {
  display: flex;
  align-items: center;
  gap: 12px;
}

.room-id {
  font-size: 18px;
  font-weight: 600;
  color: var(--primary-light);
}

.copy-btn {
  padding: 6px 12px;
  background: var(--bg-input);
  border: 1px solid var(--border);
  border-radius: 8px;
  color: var(--text-primary);
  cursor: pointer;
  font-size: 13px;
  transition: all 0.3s;
}

.copy-btn:hover {
  border-color: var(--primary);
}

.round-info {
  font-size: 16px;
  color: var(--text-secondary);
}

.round-number {
  color: var(--primary-light);
  font-weight: 700;
  font-size: 20px;
}

.game-area {
  display: grid;
  grid-template-columns: 280px 1fr;
  gap: 20px;
}

.sidebar {
  padding: 24px;
  height: fit-content;
  position: sticky;
  top: 20px;
}

.sidebar h3 {
  margin-bottom: 16px;
  font-size: 16px;
  color: var(--text-secondary);
}

.user-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
  margin-bottom: 24px;
}

.user-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 10px;
  background: var(--bg-input);
  border-radius: 10px;
}

.user-avatar {
  width: 36px;
  height: 36px;
  border-radius: 50%;
  background: var(--gradient);
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 600;
  font-size: 16px;
}

.user-name {
  flex: 1;
  font-weight: 500;
}

.online-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: var(--success);
  animation: pulse 2s infinite;
}

/* 🏆 新增：排行榜样式 */
.leaderboard-section {
  border-top: 1px solid var(--border);
  padding-top: 20px;
}

.leaderboard-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.leaderboard-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px 12px;
  background: var(--bg-input);
  border-radius: 10px;
  transition: all 0.3s;
}

.leaderboard-item.leaderboard-first {
  background: linear-gradient(135deg, rgba(251, 191, 36, 0.15), rgba(245, 158, 11, 0.1));
  border: 1px solid rgba(251, 191, 36, 0.3);
}

.leaderboard-item.leaderboard-second {
  background: linear-gradient(135deg, rgba(156, 163, 175, 0.15), rgba(107, 114, 128, 0.1));
  border: 1px solid rgba(156, 163, 175, 0.3);
}

.leaderboard-item.leaderboard-third {
  background: linear-gradient(135deg, rgba(217, 119, 6, 0.15), rgba(180, 83, 9, 0.1));
  border: 1px solid rgba(217, 119, 6, 0.3);
}

.leaderboard-rank {
  width: 28px;
  text-align: center;
  font-size: 16px;
  font-weight: 700;
}

.leaderboard-name {
  flex: 1;
  font-size: 14px;
  font-weight: 500;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.leaderboard-score {
  font-size: 13px;
  font-weight: 600;
  color: var(--primary-light);
}

.empty-leaderboard {
  text-align: center;
  padding: 20px;
  color: var(--text-secondary);
  font-size: 13px;
}

.main-area {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.hint-card {
  padding: 32px;
  text-align: center;
}

.word-length-display {
  font-size: 24px;
  font-weight: 600;
  margin-bottom: 16px;
  color: var(--text-primary);
}

.length-number {
  color: var(--primary-light);
  font-size: 32px;
  font-weight: 700;
}

.ai-hints {
  margin-bottom: 20px;
}

.ai-hint-item {
  padding: 12px 20px;
  background: rgba(16, 185, 129, 0.15);
  border: 1px solid rgba(16, 185, 129, 0.3);
  border-radius: 12px;
  color: #34d399;
  font-size: 16px;
  font-weight: 600;
  animation: fadeIn 0.3s ease;
}

.category-badge {
  display: inline-block;
  padding: 8px 20px;
  background: var(--gradient);
  border-radius: 20px;
  font-size: 14px;
  font-weight: 600;
  margin-bottom: 24px;
}

.word-hint {
  display: flex;
  justify-content: center;
  gap: 12px;
  margin-bottom: 16px;
}

.hint-char {
  width: 40px;
  height: 50px;
  display: flex;
  align-items: flex-end;
  justify-content: center;
  font-size: 32px;
  font-weight: 700;
  color: var(--primary-light);
  border-bottom: 3px solid var(--primary);
}

.hint-text {
  color: var(--text-secondary);
  font-size: 14px;
  margin-bottom: 20px;
}

.skip-vote-section {
  margin-top: 20px;
  padding-top: 20px;
  border-top: 1px solid var(--border);
}

.skip-btn {
  padding: 10px 24px;
  background: var(--bg-input);
  border: 1px solid var(--border);
  border-radius: 10px;
  color: var(--text-primary);
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s;
  margin-bottom: 12px;
}

.skip-btn:hover:not(.voted):not(.disabled) {
  border-color: var(--primary);
  background: rgba(167, 139, 250, 0.1);
}

.skip-btn.voted {
  background: rgba(16, 185, 129, 0.15);
  border-color: #34d399;
  color: #34d399;
  cursor: default;
}

.skip-btn.disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.skip-vote-status {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
}

.skip-progress-bar {
  width: 200px;
  height: 8px;
  background: var(--bg-dark);
  border-radius: 4px;
  overflow: hidden;
}

.skip-progress-fill {
  height: 100%;
  background: linear-gradient(90deg, #f59e0b, #fbbf24);
  border-radius: 4px;
  transition: width 0.3s ease;
}

.skip-vote-text {
  font-size: 13px;
  color: var(--text-secondary);
}

.skip-soon {
  font-size: 13px;
  color: #fbbf24;
  font-weight: 600;
  animation: pulse 1s infinite;
}

.input-area {
  padding: 20px;
}

.guess-input-wrapper {
  display: flex;
  gap: 12px;
}

.guess-input {
  flex: 1;
}

.submit-btn {
  padding: 14px 28px;
}

.guesses-area {
  padding: 24px;
  flex: 1;
  min-height: 300px;
  display: flex;
  flex-direction: column;
}

.guesses-area h3 {
  margin-bottom: 16px;
  font-size: 16px;
  color: var(--text-secondary);
}

.guesses-list {
  flex: 1;
  overflow-y: auto;
  max-height: 400px;
  display: flex;
  flex-direction: column;
  gap: 10px;
  padding-right: 8px;
}

.guess-item {
  display: grid;
  grid-template-columns: 120px 1fr 100px;
  align-items: center;
  gap: 16px;
  padding: 12px 16px;
  background: var(--bg-input);
  border-radius: 12px;
  animation: slideIn 0.3s ease forwards;
}

.guess-user {
  display: flex;
  align-items: center;
  gap: 8px;
}

.guess-avatar {
  width: 28px;
  height: 28px;
  border-radius: 50%;
  background: var(--gradient);
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 600;
  font-size: 12px;
}

.guess-name {
  font-size: 13px;
  font-weight: 500;
  max-width: 70px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.guess-text {
  font-size: 15px;
  font-weight: 500;
}

.similarity {
  display: flex;
  align-items: center;
  gap: 8px;
}

.similarity-bar {
  flex: 1;
  height: 8px;
  background: var(--bg-dark);
  border-radius: 4px;
  overflow: hidden;
}

.similarity-fill {
  height: 100%;
  border-radius: 4px;
  transition: width 0.5s ease;
}

.similarity.high .similarity-fill {
  background: linear-gradient(90deg, #10b981, #34d399);
}

.similarity.medium .similarity-fill {
  background: linear-gradient(90deg, #f59e0b, #fbbf24);
}

.similarity.low .similarity-fill {
  background: linear-gradient(90deg, #64748b, #94a3b8);
}

.similarity-score {
  font-size: 14px;
  font-weight: 700;
  min-width: 40px;
  text-align: right;
}

.similarity.high .similarity-score {
  color: #34d399;
}

.similarity.medium .similarity-score {
  color: #fbbf24;
}

.similarity.low .similarity-score {
  color: #94a3b8;
}

.empty-state {
  text-align: center;
  padding: 40px 20px;
  color: var(--text-secondary);
}

.winner-modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.7);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 100;
  backdrop-filter: blur(8px);
}

.winner-content {
  padding: 48px;
  text-align: center;
  max-width: 400px;
}

.winner-icon {
  font-size: 64px;
  margin-bottom: 16px;
  animation: bounce 1s infinite;
}

.winner-content h2 {
  font-size: 28px;
  margin-bottom: 16px;
  background: var(--gradient);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}

.winner-word {
  font-size: 18px;
  margin-bottom: 12px;
}

.winner-word span {
  color: var(--success);
  font-weight: 700;
  font-size: 24px;
}

.countdown {
  color: var(--text-secondary);
  font-size: 14px;
}

@media (max-width: 900px) {
  .game-area {
    grid-template-columns: 1fr;
  }
  
  .sidebar {
    position: static;
    order: 2;
  }
  
  .main-area {
    order: 1;
  }
  
  .guess-item {
    grid-template-columns: 100px 1fr 80px;
    gap: 10px;
  }
}

@media (max-width: 600px) {
  .top-bar {
    flex-direction: column;
    align-items: stretch;
  }
  
  .room-info {
    justify-content: center;
  }
  
  .hint-card {
    padding: 24px 16px;
  }
  
  .hint-char {
    width: 30px;
    height: 40px;
    font-size: 24px;
  }
  
  .guess-item {
    grid-template-columns: 1fr;
    gap: 8px;
  }
  
  .guess-user {
    justify-content: flex-start;
  }
}
</style>
