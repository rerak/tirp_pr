<template>
  <div class="page-container">
    <div v-if="festival" class="content-wrap">
      
      <section class="hero-section">
        <div class="image-container">
          <div class="bg-image-blur" :style="`background-image: url(${imageUrl})`"></div>
          <img class="main-image" :src="imageUrl" :alt="festival.title" />
        </div>
      </section>

      <main class="main-container">
        
        <header class="festival-header">
          <div class="tags">
            <span class="tag region">{{ festival.region }}</span>
            <span class="tag category">{{ festival.category || '축제' }}</span>
          </div>
          <h1 class="title">{{ festival.title }}</h1>
          <p class="date-range">{{ formatPeriod() }}</p>
        </header>

        <div class="divider"></div>

        <div class="detail-grid">
          
          <div class="left-panel">
            <div class="info-group">
              <h3 class="group-label">위치</h3>
              <p class="group-value">{{ festival.address }}</p>
              <button @click="copyAddress" class="btn-text-action">
                주소 복사
              </button>
            </div>

            <div class="info-group" v-if="festival.phone">
              <h3 class="group-label">문의</h3>
              <p class="group-value">{{ festival.phone }}</p>
            </div>

            <div class="info-group description" v-if="festival.description">
              <h3 class="group-label">상세 소개</h3>
              <p class="group-text">{{ festival.description }}</p>
            </div>
            
            <div class="action-area">
              <button @click="goBack" class="btn-back">
                <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><line x1="19" y1="12" x2="5" y2="12"></line><polyline points="12 19 5 12 12 5"></polyline></svg>
                목록으로
              </button>
            </div>
          </div>

          <div class="right-panel" v-if="festival.latitude && festival.longitude">
            <div class="map-card">
              <div class="map-header">
                <span class="map-label">지도 보기</span>
                <button @click="openKakaoNavi" class="btn-kakao-official">
                  <svg class="kakao-svg" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="#000000">
                    <path d="M12 3c-5.52 0-10 3.68-10 8.21 0 2.89 1.92 5.45 4.89 6.94-.24.88-.87 3.18-.99 3.64-.05.19-.03.37.08.49.1.12.28.18.45.18.1 0 .2-.03.29-.09l4.77-3.23c.17.01.33.03.51.03 5.52 0 10-3.68 10-8.21C22 6.68 17.52 3 12 3z"/>
                  </svg>
                  카카오내비 연결
                </button>
              </div>
              <div id="kakao-map" class="kakao-map"></div>
            </div>
          </div>

        </div>
      </main>
    </div>

    <div v-else class="loading-container">
      <div class="loader"></div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, nextTick } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { getFestivalDetail } from '@/api/festivals'

const route = useRoute()
const router = useRouter()

const festival = ref(null)
const loading = ref(false)
const map = ref(null)
const kakaoSdkLoaded = ref(false)

// 이미지 URL 계산 (플레이스홀더 고해상도로 변경)
const imageUrl = computed(() => {
  return festival.value?.image_url || 'https://via.placeholder.com/1920x800?text=No+Image'
})

// 날짜 포맷팅
const formatPeriod = () => {
  if (!festival.value) return ''
  if (festival.value.event_start_date && festival.value.event_end_date) {
    const start = formatDate(festival.value.event_start_date)
    const end = formatDate(festival.value.event_end_date)
    return `${start} - ${end}`
  }
  return festival.value.start_month ? `${festival.value.start_month}월 예정` : '날짜 미정'
}

const formatDate = (dateStr) => {
  if (!dateStr || dateStr.length < 8) return dateStr
  const y = dateStr.substring(0, 4)
  const m = dateStr.substring(4, 6)
  const d = dateStr.substring(6, 8)
  return `${y}.${m}.${d}`
}

// 지도 로직
const loadKakaoMapSDK = () => {
  return new Promise((resolve, reject) => {
    if (window.kakao && window.kakao.maps) {
      kakaoSdkLoaded.value = true
      resolve()
      return
    }
    const apiKey = import.meta.env.VITE_KAKAO_MAP_KEY
    if (!apiKey) return reject('No API Key')
    
    const script = document.createElement('script')
    script.src = `//dapi.kakao.com/v2/maps/sdk.js?appkey=${apiKey}&libraries=services&autoload=false`
    script.onload = () => window.kakao.maps.load(() => {
      kakaoSdkLoaded.value = true
      resolve()
    })
    document.head.appendChild(script)
  })
}

const fetchFestivalDetail = async () => {
  try {
    loading.value = true
    festival.value = await getFestivalDetail(route.params.id)
    await nextTick()
    if (festival.value?.latitude) {
      await loadKakaoMapSDK()
      initMap()
    }
  } catch (e) {
    console.error(e)
  } finally {
    loading.value = false
  }
}

const initMap = () => {
  const container = document.getElementById('kakao-map')
  if (!container) return
  
  const pos = new window.kakao.maps.LatLng(festival.value.latitude, festival.value.longitude)
  map.value = new window.kakao.maps.Map(container, { center: pos, level: 4 })
  
  const marker = new window.kakao.maps.Marker({ position: pos })
  marker.setMap(map.value)
}

const openKakaoNavi = () => {
  if (!festival.value) return
  const { latitude, longitude, title } = festival.value
  window.open(`https://map.kakao.com/link/to/${encodeURIComponent(title)},${latitude},${longitude}`, '_blank')
}

const copyAddress = async () => {
  try {
    await navigator.clipboard.writeText(festival.value.address)
    alert('주소가 복사되었습니다.')
  } catch { /* ignore */ }
}

const goBack = () => router.push({ name: 'festivals' })

onMounted(() => fetchFestivalDetail())
</script>

<style scoped>
/* 기본 설정 */
.page-container {
  min-height: 100vh;
  background-color: #ffffff;
  font-family: 'Pretendard', -apple-system, BlinkMacSystemFont, system-ui, Roboto, sans-serif;
  color: #191f28;
}

/* 1. 히어로 섹션 (이미지 품질 개선 CSS) */
.hero-section {
  width: 100%;
  height: 50vh;
  min-height: 400px;
  background-color: #111;
}

.image-container {
  width: 100%;
  height: 100%;
  position: relative;
  overflow: hidden;
  display: flex;
  justify-content: center;
  align-items: center;
}

.bg-image-blur {
  position: absolute;
  top: 0; left: 0; right: 0; bottom: 0;
  background-size: cover;
  background-position: center;
  filter: blur(30px) brightness(0.7);
  transform: scale(1.1);
  z-index: 1;
}

.main-image {
  position: relative;
  z-index: 2;
  max-width: 100%;
  max-height: 100%;
  object-fit: contain; 
  box-shadow: 0 20px 40px rgba(0,0,0,0.3);
}


/* 2. 메인 컨테이너 */
.main-container {
  max-width: 1100px;
  margin: 0 auto;
  padding: 4rem 1.5rem 6rem;
}

/* 헤더 스타일 */
.festival-header {
  margin-bottom: 2rem;
}

.tags {
  display: flex;
  gap: 8px;
  margin-bottom: 1rem;
}

.tag {
  font-size: 0.85rem;
  font-weight: 700;
  padding: 6px 10px;
  border-radius: 6px;
}

.tag.region {
  color: #3182f6;
  background-color: rgba(49, 130, 246, 0.1);
}

.tag.category {
  color: #4e5968;
  background-color: #f2f4f6;
}

.title {
  font-size: 2.8rem;
  font-weight: 800;
  line-height: 1.25;
  margin: 0 0 1rem;
  letter-spacing: -0.02em;
  word-break: keep-all;
}

.date-range {
  font-size: 1.25rem;
  color: #4e5968;
  font-weight: 500;
}

.divider {
  width: 100%;
  height: 1px;
  background-color: #e5e8eb;
  margin-bottom: 3rem;
}

/* 3. 그리드 레이아웃 */
.detail-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 5rem;
}

/* 좌측 패널 */
.left-panel {
  display: flex;
  flex-direction: column;
  gap: 3rem;
}

.info-group .group-label {
  font-size: 0.95rem;
  color: #8b95a1;
  font-weight: 600;
  margin-bottom: 0.8rem;
}

.info-group .group-value {
  font-size: 1.2rem;
  font-weight: 500;
  color: #191f28;
  line-height: 1.5;
}

.info-group .group-text {
  font-size: 1.05rem;
  line-height: 1.75;
  color: #333;
  white-space: pre-wrap;
}

.btn-text-action {
  margin-top: 0.8rem;
  font-size: 0.9rem;
  color: #8b95a1;
  background: none;
  border: none;
  text-decoration: underline;
  cursor: pointer;
  padding: 0;
}

.btn-text-action:hover {
  color: #333;
}

.btn-back {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 14px 28px;
  border-radius: 10px;
  border: 1px solid #d1d6db;
  background-color: white;
  font-size: 1rem;
  font-weight: 600;
  color: #333;
  cursor: pointer;
  transition: all 0.2s;
  margin-top: 1rem;
}

.btn-back:hover {
  background-color: #f9fafb;
  border-color: #b0b8c1;
}

/* 우측 패널 (Sticky 지도) */
.right-panel {
  position: sticky;
  top: 2rem;
  height: fit-content;
}

.map-card {
  border-radius: 16px;
  overflow: hidden;
  box-shadow: 0 0 0 1px rgba(0,0,0,0.08);
}

.map-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1.2rem 1.5rem;
  background-color: white;
  border-bottom: 1px solid #f2f4f6;
}

.map-label {
  font-size: 1rem;
  font-weight: 800;
  color: #191f28;
}

/* --- 새로운 카카오내비 버튼 스타일 --- */
.btn-kakao-official {
  display: flex;
  align-items: center;
  gap: 8px;
  background-color: #FEE500; /* 카카오 공식 노란색 */
  border: none;
  padding: 8px 16px 8px 12px;
  border-radius: 20px; /* 둥근 뱃지 형태 */
  font-size: 0.9rem;
  font-weight: 700;
  color: #000000; /* 카카오 공식 블랙 */
  cursor: pointer;
  transition: all 0.2s ease;
  box-shadow: 0 2px 4px rgba(0,0,0,0.05);
}

.btn-kakao-official:hover {
  background-color: #fdd835;
  transform: translateY(-1px);
  box-shadow: 0 4px 8px rgba(0,0,0,0.1);
}

.btn-kakao-official:active {
  transform: translateY(0);
}

.kakao-svg {
  width: 18px;
  height: 18px;
}
/* ------------------------------------ */

.kakao-map {
  width: 100%;
  height: 420px;
  background-color: #f2f4f6;
}

/* 로딩 */
.loading-container {
  height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
}

.loader {
  width: 36px;
  height: 36px;
  border: 4px solid #e5e8eb;
  border-top-color: #3182f6;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

@keyframes spin { to { transform: rotate(360deg); } }

/* 반응형 */
@media (max-width: 960px) {
  .detail-grid {
    grid-template-columns: 1fr;
    gap: 4rem;
  }
  
  .right-panel {
    position: static;
  }

  .title {
    font-size: 2.2rem;
  }
  
  .hero-section {
    height: 40vh;
  }
}
</style>