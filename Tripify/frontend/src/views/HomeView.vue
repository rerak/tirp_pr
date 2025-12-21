<script setup>
import { ref, computed, onMounted, onUnmounted, nextTick } from 'vue'
import { useRouter } from 'vue-router'

const router = useRouter()
const searchQuery = ref('')

// --- 1. 여행지 데이터 및 랜덤 추천 로직 (기존 유지) ---
const allDestinations = [
  { 
    id: 101, 
    name: '서울의 찬란한 야경', 
    image: 'https://images.pexels.com/photos/237211/pexels-photo-237211.jpeg?auto=compress&cs=tinysrgb&w=800' 
  },
  { 
    id: 102, 
    name: '부산의 시원한 바다', 
    image: 'https://images.pexels.com/photos/1005417/pexels-photo-1005417.jpeg?auto=compress&cs=tinysrgb&w=800' 
  },
  { 
    id: 103, 
    name: '제주 유채꽃 필 무렵', 
    image: 'https://images.pexels.com/photos/1083822/pexels-photo-1083822.jpeg?auto=compress&cs=tinysrgb&w=800' 
  },
  { 
    id: 104, 
    name: '동해의 푸른 파도', 
    image: 'https://images.pexels.com/photos/994605/pexels-photo-994605.jpeg?auto=compress&cs=tinysrgb&w=800' 
  },
  { 
    id: 105, 
    name: '청량한 숲속 힐링',
    image: 'https://images.pexels.com/photos/1320684/pexels-photo-1320684.jpeg?auto=compress&cs=tinysrgb&w=800' 
  },
]

const randomPicks = ref([])
const activeBgIndex = ref(0)
let slideInterval = null
const isHovering = ref(false)

const shuffleAndPick = () => {
  randomPicks.value = [...allDestinations]
    .sort(() => 0.5 - Math.random())
    .slice(0, 3)
}

const startSlideShow = () => {
  slideInterval = setInterval(() => {
    if (!isHovering.value && randomPicks.value.length > 0) {
      activeBgIndex.value = (activeBgIndex.value + 1) % randomPicks.value.length
    }
  }, 5000)
}

const stopSlideShow = () => {
  if (slideInterval) clearInterval(slideInterval)
}

const onCardHover = (index) => {
  isHovering.value = true
  activeBgIndex.value = index
}

const onCardLeave = () => {
  isHovering.value = false
}

const goToDetail = (id) => {
  console.log(`여행지 ID ${id} 상세 페이지로 이동`)
  router.push({ name: 'festivals' })
}

const currentBgImage = computed(() => {
  return randomPicks.value[activeBgIndex.value]?.image || ''
})

// --- 2. 달력 및 기타 로직 ---
const today = new Date()
const selectedDate = ref(today)
const dateList = ref([])
const scrollContainer = ref(null)
const displayYear = computed(() => selectedDate.value.getFullYear())

const generateMonths = () => {
  const months = []
  const start = new Date(today.getFullYear(), today.getMonth() - 12, 1)
  for (let i = 0; i < 25; i++) {
    const d = new Date(start)
    d.setMonth(start.getMonth() + i)
    months.push(d)
  }
  dateList.value = months
}

const selectDate = (date) => { selectedDate.value = date }

const isSelected = (date) => {
  return date.getFullYear() === selectedDate.value.getFullYear() &&
         date.getMonth() === selectedDate.value.getMonth()
}

// [추가됨] 이번 달인지 확인하는 함수 (붉은색 강조용)
const isCurrentMonth = (date) => {
  return date.getFullYear() === today.getFullYear() &&
         date.getMonth() === today.getMonth()
}

const scrollToCurrentDate = () => {
  if (scrollContainer.value) {
    const currentIndex = dateList.value.findIndex(d => 
      d.getFullYear() === today.getFullYear() && 
      d.getMonth() === today.getMonth()
    )
    if (currentIndex !== -1) {
      const itemWidth = 60 
      const containerWidth = scrollContainer.value.clientWidth
      const scrollPos = (currentIndex * itemWidth) - (containerWidth / 2) + (itemWidth / 2)
      scrollContainer.value.scrollLeft = scrollPos
    }
  }
}

const scroll = (direction) => {
  if (scrollContainer.value) {
    const scrollAmount = 200
    scrollContainer.value.scrollBy({
      left: direction === 'left' ? -scrollAmount : scrollAmount,
      behavior: 'smooth'
    })
  }
}

const handleSearch = () => {
  if (searchQuery.value) {
    router.push({ name: 'trip-plan', query: { search: searchQuery.value } })
  }
}

const goToFestivals = () => {
  router.push({ name: 'festivals' })
}

onMounted(async () => {
  shuffleAndPick()
  startSlideShow()
  generateMonths()
  await nextTick()
  scrollToCurrentDate()
})

onUnmounted(() => {
  stopSlideShow()
})
</script>

<template>
  <div class="home">
    <section class="hero-container">
      <transition name="fade" mode="out-in">
         <div 
          :key="currentBgImage" 
          class="hero-bg" 
          :style="{ backgroundImage: `url(${currentBgImage})` }"
        ></div>
      </transition>
      <div class="hero-overlay"></div>

      <div class="hero-content-wrapper">
        <div class="hero-text-area animate-slide-up">
          <h1 class="logo-title">
            <span class="trip">Trip</span><span class="ify">ify</span>
          </h1>
          <p class="hero-subtitle">
            한국에서 만나는,<br>
            마법 같은 여행의 시작
          </p>
          
          <div class="search-box">
            <span class="search-icon">🔍</span>
            <input 
              v-model="searchQuery" 
              type="text" 
              placeholder="어디로 떠나고 싶으세요?" 
              @keyup.enter="handleSearch" 
            />
          </div>
        </div>

        <div class="hero-visual-area animate-slide-up-delay">
          <div 
            v-for="(place, index) in randomPicks" 
            :key="place.id" 
            class="polaroid-card"
            :class="[`pos-${index}`, { 'active-card': index === activeBgIndex }]"
            @mouseover="onCardHover(index)"
            @mouseleave="onCardLeave"
            @click="goToDetail(place.id)"
          >
            <div class="active-indicator" v-if="index === activeBgIndex">Viewing</div>
            <div class="img-box">
              <img :src="place.image" :alt="place.name" />
            </div>
            <div class="img-caption">
              <span>{{ place.name }}</span>
              <span class="click-hint">자세히 보기 &rarr;</span>
            </div>
          </div>
        </div>
      </div>
    </section>

    <section class="features">
       <div class="section-header">
        <h2>Tripify 주요 기능</h2>
        <p class="section-desc">여행의 시작부터 끝까지, 스마트하게 도와드립니다.</p>
      </div>

      <div class="feature-grid">
        <div class="feature-card">
          <div class="icon-circle">
            <span class="icon">🤖</span>
          </div>
          <div class="card-content">
            <h3>Tripify 맞춤 추천</h3>
            <p>예산과 여행 스타일에 딱 맞는 최적의 코스를 AI가 자동으로 생성해드립니다.</p>
          </div>
        </div>
        
        <div class="feature-card clickable" @click="goToFestivals">
          <div class="icon-circle">
            <span class="icon">🎉</span>
          </div>
          <div class="card-content">
            <h3>지역 축제 정보</h3>
            <p>여행지에서 열리는 다양한 축제와 행사 정보를 한눈에 확인하세요.</p>
            <span class="link-text">축제 보러가기 &rarr;</span>
          </div>
        </div>
        
        <div class="feature-card">
          <div class="icon-circle">
            <span class="icon">⭐</span>
          </div>
          <div class="card-content">
            <h3>나만의 북마크</h3>
            <p>마음에 드는 장소를 발견하셨나요? 저장해두고 언제든 확인하세요.</p>
          </div>
        </div>
      </div>
    </section>

    <section class="calendar-section">
       <h2 class="calendar-title">월별 일정 확인</h2>
      <div class="calendar-wrapper">
        <div class="calendar-left">
          <span class="year-text">YEAR</span>
          <span class="month-text">{{ displayYear }}</span>
        </div>
        <div class="divider"></div>
        <button class="nav-btn prev" @click="scroll('left')">←</button>
        <div class="date-scroll-area" ref="scrollContainer">
          <div 
            v-for="date in dateList" 
            :key="date" 
            class="date-item" 
            :class="{ active: isSelected(date), current: isCurrentMonth(date) }"
            @click="selectDate(date)"
          >
            <span class="day-num">{{ date.getMonth() + 1 }}</span>
            <span class="day-name">월</span>
          </div>
        </div>
        <button class="nav-btn next" @click="scroll('right')">→</button>
      </div>
    </section>
  </div>
</template>

<style scoped>
/* --- Hero 섹션 스타일 (기존 유지) --- */
.hero-container { position: relative; width: 100%; height: 650px; overflow: hidden; display: flex; align-items: center; justify-content: center; margin-bottom: 2rem; color: #fff; background-color: #1f2937; }
.hero-bg { position: absolute; top: 0; left: 0; width: 100%; height: 100%; background-size: cover; background-position: center; filter: blur(3px) brightness(0.8); z-index: 0; transform: scale(1.05); transition: transform 6s linear; }
.fade-enter-active, .fade-leave-active { transition: opacity 0.8s ease-in-out; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
.hero-overlay { position: absolute; top: 0; left: 0; width: 100%; height: 100%; background: linear-gradient(to right, rgba(0,0,0,0.6) 0%, rgba(0,0,0,0.3) 100%); z-index: 1; }
.hero-content-wrapper { position: relative; z-index: 2; width: 100%; max-width: 1200px; display: flex; justify-content: space-between; align-items: center; padding: 0 2rem; }
.animate-slide-up { animation: slideUp 1s ease-out forwards; opacity: 0; }
.animate-slide-up-delay { animation: slideUp 1s ease-out 0.3s forwards; opacity: 0; }
@keyframes slideUp { from { opacity: 0; transform: translateY(30px); } to { opacity: 1; transform: translateY(0); } }
.hero-text-area { flex: 1; text-align: left; }
.logo-title { font-size: 4.5rem; font-weight: 900; margin-bottom: 1rem; text-shadow: 2px 2px 4px rgba(0,0,0,0.5); }
.trip { color: #4285f4; }
.ify { color: #fff; }
.hero-subtitle { font-size: 2rem; margin-bottom: 2.5rem; font-weight: 700; color: #f3f4f6; line-height: 1.3; text-shadow: 1px 1px 3px rgba(0,0,0,0.5); }
.search-box { display: flex; align-items: center; background-color: rgba(255, 255, 255, 0.95); padding: 1.2rem 1.5rem; border-radius: 50px; box-shadow: 0 8px 20px rgba(0,0,0,0.3); max-width: 500px; transition: transform 0.3s ease, box-shadow 0.3s ease; }
.search-box:focus-within { transform: translateY(-3px); box-shadow: 0 12px 25px rgba(66, 133, 244, 0.4); }
.search-icon { margin-right: 15px; font-size: 1.3rem; color: #4285f4; }
.search-box input { flex: 1; border: none; font-size: 1.1rem; outline: none; color: #333; background: transparent; }
.hero-visual-area { flex: 1; position: relative; height: 450px; display: flex; justify-content: center; align-items: center; }
.polaroid-card { position: absolute; background: #fff; padding: 12px 12px 35px 12px; box-shadow: 0 15px 35px rgba(0,0,0,0.4); transition: all 0.4s cubic-bezier(0.175, 0.885, 0.32, 1.275); width: 260px; cursor: pointer; border-radius: 4px; }
.polaroid-card:hover, .polaroid-card.active-card { z-index: 20 !important; transform: scale(1.1) rotate(0deg) !important; box-shadow: 0 25px 50px rgba(0,0,0,0.5); }
.active-indicator { position: absolute; top: -10px; right: -10px; background: #4285f4; color: white; padding: 4px 10px; border-radius: 20px; font-size: 0.8rem; font-weight: bold; box-shadow: 0 4px 10px rgba(0,0,0,0.2); z-index: 2; }
.img-box { width: 100%; height: 220px; overflow: hidden; background: #eee; border-radius: 2px; }
.img-box img { width: 100%; height: 100%; object-fit: cover; transition: transform 0.5s ease; }
.polaroid-card:hover .img-box img { transform: scale(1.1); }
.img-caption { margin-top: 15px; text-align: center; color: #333; font-weight: bold; font-size: 1.2rem; display: flex; flex-direction: column; align-items: center; }
.click-hint { font-size: 0.9rem; color: #4285f4; margin-top: 5px; opacity: 0; transition: opacity 0.3s ease; }
.polaroid-card:hover .click-hint { opacity: 1; }
.pos-0 { transform: rotate(-6deg) translateX(-100px) translateY(-30px); z-index: 5; }
.pos-1 { transform: rotate(4deg) translateX(0px) translateY(40px); z-index: 4; }
.pos-2 { transform: rotate(9deg) translateX(100px) translateY(-20px); z-index: 3; }

/* --- 기타 섹션 스타일 --- */
.features { padding: 4rem 2rem; max-width: 1200px; margin: 0 auto; }
.section-header { text-align: center; margin-bottom: 3rem; }
.features h2 { font-size: 2.2rem; color: #1f2937; font-weight: 800; margin-bottom: 0.5rem; }
.section-desc { color: #6b7280; font-size: 1.1rem; }
.feature-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(300px, 1fr)); gap: 2.5rem; }
.feature-card { background: white; border-radius: 20px; padding: 2.5rem 2rem; box-shadow: 0 10px 30px rgba(0, 0, 0, 0.04); border: 1px solid #f3f4f6; transition: all 0.3s ease; display: flex; flex-direction: column; align-items: flex-start; position: relative; overflow: hidden; }
.feature-card:hover { transform: translateY(-10px); box-shadow: 0 20px 40px rgba(66, 133, 244, 0.1); border-color: #bfdbfe; }
.icon-circle { width: 60px; height: 60px; background-color: #eff6ff; border-radius: 50%; display: flex; align-items: center; justify-content: center; margin-bottom: 1.5rem; transition: transform 0.3s ease; }
.feature-card:hover .icon-circle { transform: scale(1.1) rotate(5deg); background-color: #dbeafe; }
.icon { font-size: 1.8rem; }
.card-content h3 { font-size: 1.4rem; font-weight: 700; color: #111827; margin-bottom: 0.8rem; }
.card-content p { color: #4b5563; line-height: 1.6; font-size: 1rem; margin-bottom: 1rem; }
.feature-card.clickable { cursor: pointer; }
.link-text { font-size: 0.95rem; font-weight: 600; color: #4285f4; opacity: 0; transform: translateX(-10px); transition: all 0.3s ease; display: inline-block; }
.feature-card.clickable:hover .link-text { opacity: 1; transform: translateX(0); }
.calendar-section { max-width: 900px; margin: 0 auto 4rem; padding: 0 2rem; }
.calendar-title { text-align: center; font-size: 1.8rem; margin-bottom: 1.5rem; color: #333; }
.calendar-wrapper { display: flex; align-items: center; background: white; padding: 15px 20px; border-radius: 12px; box-shadow: 0 4px 15px rgba(0,0,0,0.08); height: 100px; border: 1px solid #eee; }
.calendar-left { display: flex; flex-direction: column; align-items: center; min-width: 80px; }
.year-text { font-size: 0.8rem; color: #666; font-weight: bold; }
.month-text { font-size: 2rem; font-weight: bold; color: #333; line-height: 1; }
.divider { width: 1px; height: 60%; background-color: #e0e0e0; margin: 0 20px; }
.nav-btn { background: none; border: none; font-size: 1.2rem; color: #999; cursor: pointer; padding: 0 10px; }
.date-scroll-area { flex: 1; display: flex; overflow-x: auto; gap: 10px; padding: 5px 0; scrollbar-width: none; }
.date-scroll-area::-webkit-scrollbar { display: none; }
.date-item { display: flex; flex-direction: column; align-items: center; justify-content: center; min-width: 50px; height: 70px; cursor: pointer; border-radius: 8px; color: #666; position: relative; }
.date-item:hover { background-color: #f5f7fa; }
.day-num { font-size: 1.4rem; font-weight: 500; margin-bottom: 4px; }
.day-name { font-size: 0.85rem; }

/* [수정됨] 이번 달 강조 스타일 */
.date-item.current .day-num,
.date-item.current .day-name {
  color: #e11d48; /* 눈에 띄는 붉은색 */
  font-weight: 800;
}

/* 선택된 상태 (파란색 + 밑줄) */
.date-item.active { color: #4285f4; font-weight: bold; }

/* 선택된 상태일 때 색상 덮어쓰기 (선택이 우선) */
.date-item.active .day-num,
.date-item.active .day-name {
  color: #4285f4;
}

.date-item.active::after { content: ''; position: absolute; bottom: 0; left: 50%; transform: translateX(-50%); width: 80%; height: 3px; background-color: #4285f4; border-radius: 2px; }

@media (max-width: 768px) {
  .hero-container { height: auto; padding: 4rem 0; }
  .hero-content-wrapper { flex-direction: column; text-align: center; }
  .hero-text-area { text-align: center; margin-bottom: 3rem; padding: 0 1rem; }
  .logo-title { font-size: 3rem; }
  .hero-subtitle { font-size: 1.5rem; }
  .hero-visual-area { height: 350px; width: 100%; }
  .polaroid-card { width: 200px; padding: 8px 8px 25px 8px; }
  .img-box { height: 160px; }
  .pos-0 { transform: rotate(-6deg) translateX(-60px) translateY(-20px); }
  .pos-2 { transform: rotate(9deg) translateX(60px) translateY(-10px); }
}
</style>