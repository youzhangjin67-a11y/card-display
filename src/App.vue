<template>
  <div class="page" :style="pageStyle">
    <div class="settings" :class="{ open: settingsOpen }">
      <div class="settings-hotzone" @click="toggleSettings"></div>
      <div class="settings-bar" @click="toggleSettings">
        <button class="settings-toggle" @click.stop="toggleSettings">⚙ 设置</button>
      </div>
      <div v-if="settingsOpen" class="settings-panel">
        <div class="settings-row">
          <label>背景图片</label>
          <input type="file" accept="image/*" @change="onBgImage" />
        </div>
        <div class="settings-row">
          <label>蒙版颜色</label>
          <input type="color" v-model="bgMaskColor" class="settings-color" />
          <input type="range" min="0" max="100" v-model.number="bgMaskOpacity" class="settings-range" />
          <span class="settings-mask-val">{{ bgMaskOpacity }}%</span>
        </div>
        <div class="settings-row">
          <label>上传视频</label>
          <input type="file" accept="video/*" @change="onVideo" />
        </div>
        <div class="settings-row">
          <label>滚动文字</label>
          <input type="text" v-model="scrollText" class="settings-input" />
        </div>
        <div class="settings-list">
          <div v-for="item in items" :key="item.id" class="settings-item">
            <img :src="item.image" class="settings-thumb" alt="缩略图" />
            <div class="settings-item-fields">
              <input type="file" accept="image/*" @change="updateItemImage(item, $event)" />
              <input type="text" v-model="item.name" class="settings-input" placeholder="名称" />
              <select v-model="item.type" class="settings-input">
                <option value="prop">道具</option>
                <option value="role">角色</option>
              </select>
              <button class="settings-del" @click="removeItem(item.id)">删除</button>
            </div>
          </div>
        </div>
        <div class="settings-actions">
          <button class="settings-apply" @click="addItem">添加物品</button>
        </div>
      </div>
    </div>

    <div class="video-banner">
      <video
        class="banner-video"
        :src="bannerVideo"
        autoplay
        muted
        loop
        playsinline
      ></video>
    </div>
    <div class="scroll-title">
      <div class="scroll-track">
        <span>{{ scrollText }}&nbsp;&nbsp;&nbsp;&nbsp;</span>
        <span>{{ scrollText }}&nbsp;&nbsp;&nbsp;&nbsp;</span>
      </div>
    </div>
    <div class="grid-container">
      <Card
        v-for="(card, i) in cards"
        :key="card.id"
        :card="card"
        :index="i"
        @click="openCard(card)"
      />
    </div>

    <div v-if="selectedCard" class="modal-mask" @click.self="closeModal">
      <div class="modal">
        <button class="modal-close" @click="closeModal">×</button>
        <img :src="selectedCard.image" class="modal-img" alt="卡片图片" />
        <div class="modal-name">{{ selectedCard.name }}</div>
        <input
          v-model="gameId"
          class="modal-input"
          type="text"
          placeholder="请输入游戏id/名称"
        />
        <button class="modal-confirm" @click="submit">确认提交</button>
      </div>
    </div>

    <div v-if="submitted" class="modal-mask" @click.self="closeSubmitted">
      <div class="modal">
        <button class="modal-close" @click="closeSubmitted">×</button>
        <img :src="submitted.image" class="modal-img" alt="道具图片" />
        <div class="modal-game">游戏名称：{{ submitted.gameId }}</div>
        <div class="modal-tip">中途退出直播间或直播中断会失败</div>
        <div class="modal-tip">道具将于 3-7天发送至游戏邮件</div>
        <div class="modal-tip">请注意查收</div>
        <button class="modal-confirm" @click="closeSubmitted">确认</button>
      </div>
    </div>

    <div v-if="submitting" class="modal-mask loading-mask">
      <div class="loading-box">
        <div class="spinner"></div>
        <div class="loading-text">提交中...</div>
        <div class="progress-bar"><div class="progress-fill"></div></div>
      </div>
    </div>

    <div v-if="toast" class="toast">{{ toast }}</div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'
import Card from '@/components/Card.vue'

const bannerVideo = ref('/video_activity.mp4')
const scrollText = ref(
  '欢迎来到卡牌陈列馆，全服限时活动火热进行中，稀有道具限时掉落，敬请关注！'
)

const bgImage = ref('')
const bgMaskColor = ref('#000000')
const bgMaskOpacity = ref(35)
const bgMask = computed(() => {
  const c = bgMaskColor.value
  const r = parseInt(c.slice(1, 3), 16)
  const g = parseInt(c.slice(3, 5), 16)
  const b = parseInt(c.slice(5, 7), 16)
  return `rgba(${r}, ${g}, ${b}, ${bgMaskOpacity.value / 100})`
})
const pageStyle = computed(() => {
  if (!bgImage.value) return {}
  return {
    backgroundImage: `linear-gradient(${bgMask.value}, ${bgMask.value}), url(${bgImage.value})`,
    backgroundSize: 'cover',
    backgroundPosition: 'center',
    backgroundRepeat: 'no-repeat',
    backgroundAttachment: 'fixed',
  }
})

const settingsOpen = ref(false)

const rarities = ['金', '紫', '橙', '红']

const images = import.meta.glob('./assets/wupin/*.png', { eager: true, import: 'default' })

const wupinImages = Object.keys(images)
  .sort((a, b) => {
    const na = Number(a.match(/(\d+)\.png$/)?.[1] || 0)
    const nb = Number(b.match(/(\d+)\.png$/)?.[1] || 0)
    return na - nb
  })
  .map((key) => images[key])

const items = ref(
  wupinImages.map((image, i) => {
    const type = i % 4 === 1 ? 'role' : 'prop'
    const item = {
      id: i + 1,
      type,
      name: `物品 ${i + 1}`,
      image,
    }
    if (type === 'role') {
      item.rarity = rarities[i % rarities.length]
      item.limited = i % 8 === 1
    }
    return item
  })
)

const offsetMs = ref(0)
const now = ref(new Date())
let timer = null

function formatTime(date) {
  const m = date.getMonth() + 1
  const d = date.getDate()
  const hh = String(date.getHours()).padStart(2, '0')
  const mm = String(date.getMinutes()).padStart(2, '0')
  const ss = String(date.getSeconds()).padStart(2, '0')
  return `${m}月${d}日 ${hh}:${mm}:${ss}`
}

async function syncInternetTime() {
  const sources = [
    async () => {
      const res = await fetch('https://www.timeapi.io/api/Time/current/zone?timeZone=Asia/Shanghai')
      const d = await res.json()
      return new Date(d.year, d.month - 1, d.day, d.hour, d.minute, d.seconds).getTime()
    },
    async () => {
      const res = await fetch('https://worldtimeapi.org/api/timezone/Asia/Shanghai')
      const d = await res.json()
      return new Date(d.datetime).getTime()
    },
  ]
  for (const get of sources) {
    try {
      const t = await get()
      offsetMs.value = t - Date.now()
      console.log('互联网时间同步成功，偏移:', offsetMs.value, 'ms')
      return
    } catch (e) {
      console.warn('时间源同步失败，尝试下一个', e)
    }
  }
  console.warn('所有互联网时间源不可用，使用本地时间')
}

onMounted(async () => {
  await syncInternetTime()
  now.value = new Date(Date.now() + offsetMs.value)
  timer = setInterval(() => {
    now.value = new Date(Date.now() + offsetMs.value)
  }, 1000)
})

onUnmounted(() => {
  if (timer) clearInterval(timer)
})

const selectedCard = ref(null)
const gameId = ref('')
const toast = ref('')
const submitted = ref(null)
const submitting = ref(false)
let toastTimer = null
let submitTimer = null

function openCard(card) {
  selectedCard.value = card
  gameId.value = ''
}

function closeModal() {
  selectedCard.value = null
}

function closeSubmitted() {
  submitted.value = null
}

function showToast(msg) {
  toast.value = msg
  clearTimeout(toastTimer)
  toastTimer = setTimeout(() => {
    toast.value = ''
  }, 3000)
}

function submit() {
  if (!gameId.value.trim()) return
  submitting.value = true
  clearTimeout(submitTimer)
  submitTimer = setTimeout(() => {
    submitting.value = false
    submitted.value = {
      image: selectedCard.value.image,
      gameId: gameId.value.trim(),
    }
    closeModal()
  }, 1500)
}

function toggleSettings() {
  settingsOpen.value = !settingsOpen.value
}

function onVideo(e) {
  const file = e.target.files && e.target.files[0]
  if (file) {
    const validTypes = ['video/mp4', 'video/quicktime', 'video/mpeg', 'video/x-m4v', 'video/3gpp']
    const name = file.name.toLowerCase()
    if (validTypes.includes(file.type) || name.endsWith('.mp4') || name.endsWith('.mov')) {
      URL.revokeObjectURL(bannerVideo.value)
      bannerVideo.value = URL.createObjectURL(file)
      showToast('视频已设置')
    } else {
      alert('仅支持 MP4/MOV 格式的视频文件')
    }
  }
}

function onBgImage(e) {
  const file = e.target.files && e.target.files[0]
  if (file) {
    if (file.type.startsWith('image/')) {
      URL.revokeObjectURL(bgImage.value)
      bgImage.value = URL.createObjectURL(file)
      showToast('背景图片已设置')
    } else {
      alert('仅支持图片文件')
    }
  }
}

function updateItemImage(item, e) {
  const file = e.target.files && e.target.files[0]
  if (file) item.image = URL.createObjectURL(file)
}

function removeItem(id) {
  items.value = items.value.filter((it) => it.id !== id)
}

function addItem() {
  const nextId = items.value.reduce((m, it) => Math.max(m, it.id), 0) + 1
  items.value.push({
    id: nextId,
    type: 'prop',
    name: `新物品 ${nextId}`,
    image: wupinImages[0],
  })
}

const cards = computed(() =>
  items.value.map((it, i) => {
    const ts = new Date(now.value.getTime() - i * 5000)
    return {
      id: it.id,
      type: it.type,
      timestamp: formatTime(ts),
      image: it.image,
      name: it.name,
      rarity: it.rarity,
      limited: it.limited,
    }
  })
)
</script>

<style scoped>
.page {
  background-color: #f5f5f5;
  min-height: 100vh;
  padding: 0;
}

.settings-color {
  width: 40px;
  height: 28px;
  border: 1px solid #ddd;
  border-radius: 6px;
  padding: 0;
  background: #fff;
  cursor: pointer;
}

.settings-range {
  flex: 1;
  min-width: 80px;
}

.settings-mask-val {
  font-size: 12px;
  color: #666;
  width: 38px;
  text-align: right;
}

.settings {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  z-index: 1500;
  background: transparent;
  transition: background 0.2s ease, box-shadow 0.2s ease;
}

.settings:hover,
.settings.open {
  background: rgba(255, 255, 255, 0.95);
  backdrop-filter: blur(4px);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.12);
}

.settings-hotzone {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  height: 12px;
  cursor: pointer;
}

.settings-bar {
  height: 44px;
  display: flex;
  align-items: center;
  justify-content: flex-end;
  padding: 0 12px;
  cursor: pointer;
  /* Hidden by default, revealed when hovering/clicking the hotzone or when open */
  opacity: 0;
  pointer-events: none;
  transition: opacity 0.2s ease;
}

.settings:hover .settings-bar,
.settings.open .settings-bar {
  opacity: 1;
  pointer-events: auto;
}

.settings-toggle {
  border: none;
  background: linear-gradient(135deg, #ff9a9e, #ff3b3b);
  color: #fff;
  font-size: 13px;
  font-weight: 600;
  padding: 6px 14px;
  border-radius: 8px;
  cursor: pointer;
}

.settings-panel {
  padding: 12px;
  border-top: 1px solid #eee;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.settings-row {
  display: flex;
  align-items: center;
  gap: 10px;
  flex-wrap: wrap;
}

.settings-row label {
  font-size: 13px;
  color: #333;
  min-width: 96px;
  font-weight: 500;
}

.settings-input {
  flex: 1;
  min-width: 220px;
  padding: 8px 10px;
  border: 1px solid #ddd;
  border-radius: 8px;
  font-size: 13px;
  outline: none;
}

.settings-input:focus {
  border-color: #ff6b6b;
}

.settings-actions {
  display: flex;
  justify-content: flex-end;
}

.settings-list {
  display: flex;
  flex-direction: column;
  gap: 10px;
  max-height: 50vh;
  overflow-y: auto;
}

.settings-item {
  display: flex;
  gap: 10px;
  align-items: center;
  padding: 8px;
  border: 1px solid #eee;
  border-radius: 8px;
  background: #fafafa;
}

.settings-thumb {
  width: 48px;
  height: 48px;
  object-fit: contain;
  border-radius: 6px;
  background: #fff;
  border: 1px solid #eee;
  flex-shrink: 0;
}

.settings-item-fields {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  align-items: center;
  flex: 1;
}

.settings-item-fields .settings-input {
  min-width: 120px;
  flex: 1;
}

.settings-del {
  border: none;
  background: #ff5f5f;
  color: #fff;
  font-size: 12px;
  padding: 6px 12px;
  border-radius: 6px;
  cursor: pointer;
}

.settings-apply {
  border: none;
  background: linear-gradient(135deg, #ff9a9e, #ff3b3b);
  color: #fff;
  font-size: 13px;
  font-weight: 600;
  padding: 8px 18px;
  border-radius: 8px;
  cursor: pointer;
}

.video-banner {
  margin: 0 0 20px;
}

.banner-video {
  width: 100%;
  aspect-ratio: 16 / 9;
  object-fit: cover;
  border-radius: 12px;
  display: block;
  background: #000;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

.scroll-title {
  margin: 0 0 16px;
  overflow: hidden;
  white-space: nowrap;
  background: linear-gradient(90deg, #fff3d6, #ffe3ef);
  border: 1px solid #ffd6e7;
  border-radius: 8px;
}

.scroll-track {
  display: flex;
  width: max-content;
  animation: scroll-left 20s linear infinite;
}

.scroll-track span {
  font-size: 14px;
  color: #c0392b;
  font-weight: 500;
  padding: 8px 0;
}

@keyframes scroll-left {
  from {
    transform: translateX(0);
  }
  to {
    transform: translateX(-50%);
  }
}

.grid-container {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 8px;
  margin: 0;
}

.modal-mask {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.55);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal {
  position: relative;
  width: 320px;
  max-width: 90vw;
  background: #fff;
  border-radius: 14px;
  padding: 24px 20px 20px;
  text-align: center;
  box-shadow: 0 12px 40px rgba(0, 0, 0, 0.3);
}

.modal-close {
  position: absolute;
  top: 8px;
  right: 12px;
  border: none;
  background: transparent;
  font-size: 22px;
  line-height: 1;
  color: #999;
  cursor: pointer;
}

.modal-img {
  width: 140px;
  height: 140px;
  object-fit: contain;
  margin: 0 auto 12px;
}

.modal-name {
  font-size: 16px;
  font-weight: 600;
  color: #333;
  margin-bottom: 4px;
}

.modal-time {
  font-size: 12px;
  color: #999;
  margin-bottom: 16px;
}

.modal-input {
  width: 100%;
  box-sizing: border-box;
  padding: 10px 12px;
  border: 1px solid #ddd;
  border-radius: 8px;
  font-size: 14px;
  outline: none;
}

.modal-input:focus {
  border-color: #ff6b6b;
}

.modal-confirm {
  width: 100%;
  margin-top: 12px;
  padding: 10px 0;
  border: none;
  border-radius: 8px;
  background: linear-gradient(135deg, #ff9a9e, #ff3b3b);
  color: #fff;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
}

.modal-game {
  font-size: 15px;
  font-weight: 600;
  color: #333;
  margin: 8px 0 12px;
}

.modal-tip {
  font-size: 13px;
  color: #e04848;
  margin-bottom: 6px;
  line-height: 1.5;
}

.modal-confirm:active {
  opacity: 0.85;
}

.modal-tip {
  margin-top: 14px;
  font-size: 13px;
  color: #c0392b;
  font-weight: 500;
}

.toast {
  position: fixed;
  top: 45%;
  left: 50%;
  transform: translate(-50%, -50%);
  z-index: 2000;
  background: rgba(0, 0, 0, 0.78);
  color: #fff;
  padding: 14px 28px;
  border-radius: 10px;
  font-size: 15px;
  font-weight: 500;
  line-height: 1.5;
  text-align: center;
  max-width: 80vw;
  box-shadow: 0 6px 20px rgba(0, 0, 0, 0.35);
  pointer-events: none;
  animation: toast-in 0.25s ease;
}

@keyframes toast-in {
  from {
    opacity: 0;
    transform: translate(-50%, -50%) scale(0.9);
  }
  to {
    opacity: 1;
    transform: translate(-50%, -50%) scale(1);
  }
}

.loading-mask {
  display: flex;
  align-items: center;
  justify-content: center;
}
.loading-box {
  background: #fff;
  border-radius: 16px;
  padding: 32px 40px;
  text-align: center;
  box-shadow: 0 8px 30px rgba(0, 0, 0, 0.2);
}
.spinner {
  width: 40px;
  height: 40px;
  margin: 0 auto 16px;
  border: 4px solid #eee;
  border-top-color: #ff4d4f;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}
@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}
.loading-text {
  font-size: 15px;
  color: #333;
  margin-bottom: 14px;
}
.progress-bar {
  width: 160px;
  height: 4px;
  background: #eee;
  border-radius: 2px;
  overflow: hidden;
  margin: 0 auto;
}
.progress-fill {
  height: 100%;
  width: 0;
  background: linear-gradient(90deg, #ff4d4f, #ff7a45);
  animation: fill 1.5s ease forwards;
}
@keyframes fill {
  0% {
    width: 0;
  }
  80% {
    width: 85%;
  }
  100% {
    width: 100%;
  }
}
</style>

<style>
html,
body {
  margin: 0;
  padding: 0;
}
#app {
  min-height: 100vh;
}
</style>
