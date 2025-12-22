<script setup>
import { onMounted, ref, computed, watch } from 'vue' // watch 추가
import { useTripStore } from '@/stores/trip'
import { useRouter } from 'vue-router'

const tripStore = useTripStore()
const router = useRouter()

// --- 색상 팔레트 ---
const colorPalette = [
  { bg: '#e3f2fd', text: '#1565c0' }, // Blue
  { bg: '#e8f5e9', text: '#2e7d32' }, // Green
  { bg: '#f3e5f5', text: '#7b1fa2' }, // Purple
  { bg: '#fff3e0', text: '#ef6c00' }, // Orange
  { bg: '#ffebee', text: '#c62828' }, // Red
  { bg: '#e0f7fa', text: '#006064' }, // Cyan
  { bg: '#fff8e1', text: '#ff8f00' }, // Amber
  { bg: '#fce4ec', text: '#c2185b' }, // Pink
  { bg: '#f9fbe7', text: '#827717' }, // Lime (라임)
  { bg: '#e8eaf6', text: '#283593' }, // Indigo (인디고)
  { bg: '#efebe9', text: '#4e342e' }, // Brown (브라운)
  { bg: '#eceff1', text: '#37474f' }, // Blue Grey (차분한 블루그레이)
  { bg: '#f3e5f5', text: '#4a148c' }, // Deep Purple (진한 보라)
  { bg: '#e0f2f1', text: '#004d40' }, // Teal (청록)
  { bg: '#fafafa', text: '#212121' }, // Dark Grey (진한 회색/검정 텍스트)
  { bg: '#fbe9e7', text: '#bf360c' }, // Deep Orange (진한 주황)
]

const getPlanStyle = (id) => {
  const numId = typeof id === 'number' ? id : String(id).split('').reduce((acc, char) => acc + char.charCodeAt(0), 0)
  const color = colorPalette[numId % colorPalette.length]
  return { '--plan-bg': color.bg, '--plan-text': color.text }
}

// --- 달력 상태 ---
const currentDate = ref(new Date())
const weekDays = ['일', '월', '화', '수', '목', '금', '토']

const currentYear = computed(() => currentDate.value.getFullYear())
const currentMonth = computed(() => currentDate.value.getMonth())
const daysInMonth = computed(() => new Date(currentYear.value, currentMonth.value + 1, 0).getDate())
const startDay = computed(() => new Date(currentYear.value, currentMonth.value, 1).getDay())

const changeMonth = (diff) => {
  currentDate.value = new Date(currentYear.value, currentMonth.value + diff, 1)
}

// --- 일정 정렬 ---
const sortedPlans = computed(() => {
  return [...tripStore.plans].sort((a, b) => {
    const startA = new Date(a.start_date)
    const startB = new Date(b.start_date)
    if (startA - startB !== 0) return startA - startB
    const endA = new Date(a.end_date)
    const endB = new Date(b.end_date)
    return endB - endA
  })
})

const getPlansForDate = (day) => {
  if (!sortedPlans.value.length) return []
  const targetDate = new Date(currentYear.value, currentMonth.value, day)
  targetDate.setHours(0, 0, 0, 0)

  return sortedPlans.value.filter(plan => {
    const start = new Date(plan.start_date)
    const end = new Date(plan.end_date)
    start.setHours(0, 0, 0, 0)
    end.setHours(0, 0, 0, 0)
    return targetDate >= start && targetDate <= end
  })
}

const getPlanClass = (day, plan) => {
  const targetDate = new Date(currentYear.value, currentMonth.value, day)
  const targetTime = targetDate.setHours(0,0,0,0)
  const dayOfWeek = targetDate.getDay()

  const start = new Date(plan.start_date).setHours(0,0,0,0)
  const end = new Date(plan.end_date).setHours(0,0,0,0)

  const classes = []
  const isVisualStart = (targetTime === start) || (dayOfWeek === 0)
  const isVisualEnd = (targetTime === end) || (dayOfWeek === 6)

  if (isVisualStart) classes.push('is-start')
  if (isVisualEnd) classes.push('is-end')
  if (!isVisualStart && !isVisualEnd) classes.push('is-middle')

  return classes.join(' ')
}

const getSegmentWidth = (day, plan) => {
  const targetDate = new Date(currentYear.value, currentMonth.value, day)
  const dayOfWeek = targetDate.getDay()

  const endDate = new Date(plan.end_date)
  endDate.setHours(0,0,0,0)

  const daysLeftInWeek = 7 - dayOfWeek
  const msPerDay = 1000 * 60 * 60 * 24
  const daysLeftInPlan = Math.floor((endDate - targetDate) / msPerDay) + 1

  const span = Math.min(daysLeftInWeek, daysLeftInPlan)
  return `calc(100% * ${span} + ${span - 1}px - 6px)`
}

// --- 위시리스트 기능 ---
const newWish = ref('')
const savedWishlist = localStorage.getItem('my-trip-wishlist')
const wishlist = ref(savedWishlist ? JSON.parse(savedWishlist) : [
  { id: 1, text: '제주도 한라산 등반', checked: false },
  { id: 2, text: '부산 해운대 요트 투어', checked: false },
])

watch(wishlist, (newVal) => {
  localStorage.setItem('my-trip-wishlist', JSON.stringify(newVal))
}, { deep: true })

const addWish = () => {
  if (newWish.value.trim()) {
    wishlist.value.push({
      id: Date.now(),
      text: newWish.value,
      checked: false
    })
    newWish.value = ''
  }
}

const removeWish = (id) => {
  wishlist.value = wishlist.value.filter(item => item.id !== id)
}

// --- 추천 여행지 ---
const recommendations = ref([
  { id: 1, title: '강원도 평창', tag: '#겨울왕국 #스키', color: '#E3F2FD', icon: '❄️' },
  { id: 2, title: '여수 밤바다', tag: '#야경 #낭만포차', color: '#E8EAF6', icon: '🌊' },
  { id: 3, title: '전주 한옥마을', tag: '#먹방 #한복체험', color: '#F3E5F5', icon: '🏯' },
])

onMounted(async () => {
  await tripStore.fetchPlans()
})

const goToTrip = (id) => {
  router.push({ name: 'itinerary', params: { id } })
}

const goToCreate = () => {
  router.push('/trip/new')
}
</script>

<template>
  <div class="my-trips-view">
    <div class="static-bg-wrapper"></div>

    <div v-if="tripStore.loading" class="loading">로딩 중...</div>

    <div v-else class="content-wrapper">
      <div class="calendar-section glass-card">
        <div class="calendar-header">
          <div class="month-nav">
            <button @click="changeMonth(-1)">&lt;</button>
            <h2>{{ currentYear }}. {{ currentMonth + 1 }}</h2>
            <button @click="changeMonth(1)">&gt;</button>
          </div>
          <button class="btn-create" @click="goToCreate">여행 추가 +</button>
        </div>

        <div class="calendar-board">
          <div v-for="day in weekDays" :key="day" class="weekday">{{ day }}</div>
          <div v-for="n in startDay" :key="'blank-' + n" class="day blank"></div>

          <div v-for="day in daysInMonth" :key="day" class="day">
            <span class="day-number">{{ day }}</span>
            <div class="plan-bars">
              <div
                v-for="plan in getPlansForDate(day)"
                :key="plan.id"
                class="plan-bar"
                :class="getPlanClass(day, plan)"
                :style="getPlanStyle(plan.id)"
                @click.stop="goToTrip(plan.id)"
              >
                <span
                  v-if="getPlanClass(day, plan).includes('is-start')"
                  class="plan-title"
                  :style="{ width: getSegmentWidth(day, plan) }"
                >
                  {{ plan.title }}
                </span>
                <span v-else class="plan-title-hidden">&nbsp;</span>
              </div>
            </div>
          </div>
        </div>
      </div>

      <div class="bottom-section">
        
        <div class="card wishlist-card glass-card">
          <div class="card-header">
            <h3>여행 위시리스트</h3>
            <span class="subtitle">언젠가 떠나고 싶은 곳을 적어보세요</span>
          </div>
          
          <div class="wish-input-area">
            <input 
              v-model="newWish" 
              @keyup.enter="addWish" 
              placeholder="가고 싶은 곳 입력 (Enter)" 
              type="text"
            />
            <button @click="addWish" class="btn-add">+</button>
          </div>

          <ul class="wish-list">
            <li v-for="item in wishlist" :key="item.id" class="wish-item">
              <label :class="{ checked: item.checked }">
                <input type="checkbox" v-model="item.checked" />
                <span class="wish-text">{{ item.text }}</span>
              </label>
              <button @click="removeWish(item.id)" class="btn-del">×</button>
            </li>
            <li v-if="wishlist.length === 0" class="empty-msg">위시리스트가 비어있습니다.</li>
          </ul>
        </div>

        <div class="card recommend-card glass-card">
          <div class="card-header">
            <h3>☃️ {{ currentMonth + 1 }}월의 추천 여행지</h3>
            <span class="subtitle">지금 떠나기 딱 좋은 곳들</span>
          </div>
          
          <div class="recommend-grid">
            <div 
              v-for="place in recommendations" 
              :key="place.id" 
              class="recommend-item"
              :style="{ backgroundColor: place.color }"
            >
              <div class="place-icon">{{ place.icon }}</div>
              <div class="place-info">
                <span class="place-tag">{{ place.tag }}</span>
                <strong class="place-title">{{ place.title }}</strong>
              </div>
            </div>
          </div>
          <div class="more-link">
            <span>더 많은 여행지 찾아보기 &rarr;</span>
          </div>
        </div>

      </div>
    </div>
  </div>
</template>

<style scoped>
/* --- [신규] 배경 스타일 (고정형, 최적화, 연한 핑크) --- */
.my-trips-view {
  position: relative;
  min-height: 100vh;
  width: 100%;
  overflow-x: hidden;
}

.static-bg-wrapper {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  z-index: -2;
  background-color: #f5f7fa;
  /* 우측 상단의 핑크색을 연하게 수정함 (rgba(255, 182, 193, 0.3)) */
  background-image: 
    radial-gradient(at 0% 0%, rgba(161, 196, 253, 0.5) 0px, transparent 50%),
    radial-gradient(at 100% 0%, rgba(255, 182, 193, 0.3) 0px, transparent 50%),
    radial-gradient(at 100% 100%, rgba(132, 250, 176, 0.4) 0px, transparent 50%),
    radial-gradient(at 0% 100%, rgba(194, 233, 251, 0.5) 0px, transparent 50%);
  background-attachment: fixed;
  background-size: cover;
  pointer-events: none;
}

/* --- [신규] 유리 질감 클래스 --- */
.glass-card {
  background: rgba(255, 255, 255, 0.75) !important;
  border: 1px solid rgba(255, 255, 255, 0.6) !important;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.05) !important;
}

/* --- 기본 레이아웃 --- */
.content-wrapper {
  display: flex;
  flex-direction: column;
  gap: 20px;
  max-width: 1200px;
  margin: 0 auto;
  padding: 2rem 1rem;
  position: relative;
  z-index: 1;
}
.loading { text-align: center; padding: 2rem; color: #555; }

/* --- 캘린더 스타일 --- */
.calendar-section {
  padding: 24px;
  border-radius: 16px;
  width: 100%;
  box-sizing: border-box;
}

.calendar-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 24px;
}

.month-nav {
  display: flex;
  align-items: center;
  gap: 20px;
  flex: 1;
  justify-content: center;
}
.month-nav h2 { margin: 0; font-size: 1.6rem; font-weight: 700; color: #333; }
.month-nav button { 
  padding: 8px 12px; border: 1px solid rgba(0,0,0,0.1); background: rgba(255,255,255,0.5); 
  border-radius: 8px; cursor: pointer; transition: all 0.2s;
}
.month-nav button:hover { background: rgba(255,255,255,0.9); }

.btn-create {
  padding: 10px 20px;
  background-color: #3498db;
  color: white;
  border: none;
  border-radius: 8px;
  font-weight: 600;
  cursor: pointer;
  transition: background-color 0.2s;
  font-size: 0.95rem;
  box-shadow: 0 2px 4px rgba(52, 152, 219, 0.3);
}

.calendar-board {
  display: grid;
  grid-template-columns: repeat(7, 1fr);
  width: 100%;
  border: 1px solid rgba(0,0,0,0.05);
  border-radius: 8px;
  overflow: hidden;
  table-layout: fixed;
  background-color: rgba(255,255,255,0.4);
}
.weekday {
  text-align: center; font-weight: 600; padding: 12px 0;
  background: rgba(248, 249, 250, 0.7); border-bottom: 1px solid rgba(0,0,0,0.05);
  font-size: 0.9rem; color: #555;
}
.weekday:not(:last-child) { border-right: 1px solid rgba(0,0,0,0.05); }

.day {
  min-height: 110px; padding: 34px 0 4px 0;
  border-bottom: 1px solid rgba(0,0,0,0.05); border-right: 1px solid rgba(0,0,0,0.05);
  position: relative; background-color: transparent; z-index: 1;
}
.day:nth-child(7n) { border-right: none; } 
.day-number {
  position: absolute; top: 10px; left: 12px;
  font-size: 0.95rem; font-weight: 600; color: #444;
}
.blank { background: rgba(250, 250, 250, 0.5); }

.plan-bars { display: flex; flex-direction: column; gap: 3px; width: 100%; }
.plan-bar {
  background-color: var(--plan-bg); color: var(--plan-text);
  font-size: 0.75rem; height: 22px; line-height: 22px;
  cursor: pointer; white-space: nowrap; position: relative; margin-bottom: 2px;
  opacity: 0.95;
}
.plan-bar:hover { filter: brightness(0.95); opacity: 1; }
.plan-bar.is-start {
  border-top-left-radius: 4px; border-bottom-left-radius: 4px;
  margin-left: 6px; border-left: 3px solid var(--plan-text); z-index: 10;
}
.plan-bar.is-end { border-top-right-radius: 4px; border-bottom-right-radius: 4px; margin-right: 6px; }

.plan-title {
  display: block; position: absolute; top: 0; left: 0; height: 100%;
  box-sizing: border-box; padding-left: 6px; padding-right: 4px;
  white-space: nowrap; overflow: hidden; text-overflow: ellipsis;
  font-weight: 600; pointer-events: none;
}
.plan-title-hidden { visibility: hidden; }

/* --- 하단 섹션 스타일 --- */
.bottom-section {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 20px;
}

.card {
  padding: 24px;
  border-radius: 16px;
  display: flex;
  flex-direction: column;
}

.card-header { margin-bottom: 20px; }
.card h3 { margin: 0 0 5px 0; font-size: 1.1rem; color: #333; font-weight: 700; }
.subtitle { font-size: 0.85rem; color: #666; }

.wish-input-area { display: flex; gap: 10px; margin-bottom: 15px; }
.wish-input-area input {
  flex: 1; padding: 10px; 
  border: 1px solid rgba(0,0,0,0.1); 
  background: rgba(255,255,255,0.6);
  border-radius: 8px; outline: none; transition: border-color 0.2s;
}
.wish-input-area input:focus { border-color: #3498db; background: #fff; }
.btn-add {
  background: #3498db; color: white; border: none; width: 40px; border-radius: 8px; cursor: pointer; font-size: 1.2rem;
}

.wish-list { list-style: none; padding: 0; margin: 0; flex: 1; overflow-y: auto; max-height: 200px; }
.wish-item {
  display: flex; justify-content: space-between; align-items: center;
  padding: 10px 0; border-bottom: 1px solid rgba(0,0,0,0.05);
}
.wish-item label { display: flex; align-items: center; gap: 10px; cursor: pointer; flex: 1; }
.wish-item input[type="checkbox"] { cursor: pointer; }
.wish-text { transition: color 0.2s; font-size: 0.95rem; }
.checked .wish-text { text-decoration: line-through; color: #999; }
.btn-del { border: none; background: none; color: #ff6b6b; cursor: pointer; font-size: 1.2rem; padding: 0 5px; }
.empty-msg { text-align: center; color: #888; padding: 20px; font-size: 0.9rem; }

.recommend-grid {
  display: grid;
  grid-template-columns: 1fr 1fr 1fr;
  gap: 15px;
  margin-bottom: 15px;
}
.recommend-item {
  border-radius: 12px;
  padding: 15px;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  text-align: center;
  cursor: pointer;
  transition: transform 0.2s;
  height: 100px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.05);
}
.recommend-item:hover { transform: translateY(-3px); }

.place-icon { font-size: 2rem; margin-bottom: 8px; }
.place-tag { font-size: 0.7rem; color: #555; background: rgba(255,255,255,0.6); padding: 2px 6px; border-radius: 4px; margin-bottom: 4px; }
.place-title { font-size: 0.9rem; color: #333; }

.more-link {
  text-align: right; font-size: 0.85rem; color: #3498db; cursor: pointer; margin-top: auto;
}
.more-link span:hover { text-decoration: underline; }

@media (max-width: 768px) {
  .bottom-section { grid-template-columns: 1fr; }
  .day { min-height: 70px; padding-top: 25px; }
}
</style>