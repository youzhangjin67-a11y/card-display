<template>
  <div class="card" :class="[type, card.limited ? 'limited' : '']">
    <span class="badge">{{ index + 1 }}</span>
    <div class="card-top">{{ card.timestamp }}</div>
    <div class="card-body">
      <template v-if="type === 'prop'">
        <img :src="card.image" alt="道具图片" class="prop-image" />
      </template>
      <template v-else-if="type === 'role'">
        <img :src="card.image" alt="角色立绘" class="role-image" />
      </template>
    </div>
    <div class="card-footer">{{ card.name }}</div>
  </div>
</template>

<script setup>
import { computed } from 'vue'

const props = defineProps({
  card: {
    type: Object,
    required: true,
  },
  index: {
    type: Number,
    default: 0,
  },
})

const type = computed(() => props.card.type || 'prop')
</script>

<style scoped>
.card {
  position: relative;
  display: flex;
  flex-direction: column;
  border: 2px solid transparent;
  border-radius: 12px;
  background:
    linear-gradient(#fff, #fff) padding-box,
    linear-gradient(135deg, var(--g1), var(--g2)) border-box;
  box-shadow:
    0 4px 12px rgba(0, 0, 0, 0.15),
    0 0 10px var(--glow);
  transition: transform 0.25s ease, box-shadow 0.25s ease;
  cursor: pointer;
  overflow: hidden;
}

.card:hover {
  transform: translateY(-8px);
  box-shadow:
    0 12px 24px rgba(0, 0, 0, 0.2),
    0 0 20px var(--glow);
}

/* 类型主题：渐变描边颜色 + 发光强度 */
.card.prop {
  --g1: #ff9a9e;
  --g2: #ff3b3b;
  --glow: rgba(255, 59, 59, 0.35);
  --strength: 0.45;
}

.card.role {
  --g1: #ffe98a;
  --g2: #ffb300;
  --glow: rgba(255, 184, 0, 0.6);
  --strength: 1;
}

.card.limited {
  --g1: #ffd1f0;
  --g2: #ff5fc8;
  --glow: rgba(255, 95, 200, 0.65);
  --strength: 1;
}

/* 内部斜向流光扫过 */
.card::after {
  content: '';
  position: absolute;
  top: -50%;
  left: -45%;
  width: 40%;
  height: 200%;
  background: linear-gradient(
    100deg,
    transparent 0%,
    rgba(255, 255, 255, 0.5) 45%,
    rgba(255, 224, 130, 0.35) 55%,
    transparent 100%
  );
  transform: rotate(18deg) translateX(-20%);
  animation: card-sweep 2.6s ease-in-out infinite;
  pointer-events: none;
  opacity: var(--strength);
  z-index: 1;
}

@keyframes card-sweep {
  0% {
    transform: rotate(18deg) translateX(-20%);
  }
  100% {
    transform: rotate(18deg) translateX(360%);
  }
}

.card:hover::after {
  opacity: calc(var(--strength) + 0.25);
}

.badge {
  position: absolute;
  top: 4px;
  left: 2px;
  z-index: 10;
  background: #ff3b3b;
  color: white;
  width: 24px;
  height: 24px;
  border-radius: 50%;
  text-align: center;
  line-height: 24px;
  font-size: 12px;
  font-weight: bold;
}

/* 内容层位于流光之上 */
.card-top,
.card-body,
.card-footer {
  position: relative;
  z-index: 2;
}

.card-top {
  padding: 8px 6px;
  text-align: right;
  font-size: 11px;
  color: #666;
  border-bottom: 1px solid #eee;
}

.card-body {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 3px;
  min-height: 90px;
}

.prop-image,
.role-image {
  max-width: 100%;
  max-height: 160px;
  object-fit: contain;
}

.card-footer {
  padding: 10px 15px;
  font-size: 13px;
  font-weight: 500;
  color: #333;
  text-align: center;
  border-top: 1px solid #eee;
}
</style>