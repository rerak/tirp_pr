<script setup>
import { onMounted, computed, ref } from 'vue'
import { useRoute } from 'vue-router'
import { useTripStore } from '@/stores/trip'
import { useAuthStore } from '@/stores/auth'

const route = useRoute()
const tripStore = useTripStore()
const authStore = useAuthStore()

const showReviewModal = ref(false)
const review = ref('')
const rating = ref(5.0)
const isSubmitting = ref(false)
const reviewError = ref('')

// 계획 수정 관련
const showModifyModal = ref(false)
const requirements = ref('')
const isModifying = ref(false)
const modifyError = ref('')

// 작성자 닉네임 가져오기
const getWriterNickname = (planUser) => {
  if (!planUser) return '...'
  if (isOwner.value && authStore.user?.nickname) {
    return authStore.user.nickname
  }
  if (typeof planUser === 'object') {
    return planUser.nickname || planUser.username || '익명'
  }
  return planUser
}

// 날짜 포맷팅
const formatDate = (dateString) => {
  if (!dateString) return ''
  const date = new Date(dateString)
  return `${date.getFullYear()}. ${date.getMonth() + 1}. ${date.getDate()}`
}

// 숙박 타입 한글 변환
const getAccommodationTypeLabel = (type) => {
  const labels = {
    'hotel': '호텔',
    'motel': '모텔',
    'pension': '펜션',
    'guesthouse': '게스트하우스',
    'resort': '리조트'
  }
  return labels[type?.toLowerCase()] || type
}

// 교통수단 값 라벨 변환
const getTransportationValueLabel = (value) => {
  if (!value) return ''
  const strValue = String(value)
  const labels = {
    'car': '자동차',
    'public_transport': '대중교통',
    'walk': '도보',
    'walking': '도보',
    'taxi': '택시',
    'bus': '버스',
    'subway': '지하철',
    'train': '기차',
    'airplane': '비행기',
    'bicycle': '자전거'
  }
  return labels[strValue.toLowerCase()] || strValue
}

// 교통수단 키 라벨 변환
const getTransportKeyLabel = (key) => {
  const labelMap = {
    'morning': '오전',
    'afternoon': '오후',
    'evening': '저녁',
    'night': '밤',
    'cost': '비용',
    'duration': '소요 시간',
    'type': '이동 수단',
    'distance': '거리'
  }
  if (labelMap[key]) return labelMap[key]
  if (key.includes('_')) {
    const parts = key.split('_')
    const timeKeys = ['morning', 'afternoon', 'evening', 'night']
    if (timeKeys.includes(parts[1])) return `${labelMap[parts[1]] || parts[1]} ${labelMap[parts[0]] || parts[0]}`
    if (timeKeys.includes(parts[0])) return `${labelMap[parts[0]] || parts[0]} ${labelMap[parts[1]] || parts[1]}`
    return parts.map(part => labelMap[part] || part).join(' ')
  }
  return key.charAt(0).toUpperCase() + key.slice(1)
}

// 총 예상 비용 계산
const totalEstimatedCost = computed(() => {
  if (!tripStore.currentPlan?.itineraries) return 0
  return tripStore.currentPlan.itineraries.reduce((sum, itinerary) => {
    return sum + (itinerary.estimated_cost || 0)
  }, 0)
})

// 소유자 확인
const isOwner = computed(() => {
  if (!tripStore.currentPlan || !authStore.user) {
    return false
  }
  
  // user_id로 비교
  if (tripStore.currentPlan.user_id && authStore.user.id) {
    return tripStore.currentPlan.user_id === authStore.user.id
  }
  
  // username으로 fallback
  const planUser = typeof tripStore.currentPlan.user === 'string' 
    ? tripStore.currentPlan.user 
    : tripStore.currentPlan.user?.username || tripStore.currentPlan.user
  
  const currentUser = authStore.user.username || authStore.user
  
  return planUser === currentUser
})

// [변경됨] 별점 채우기 퍼센티지 계산 (1단위)
const getStarFill = (index) => {
  // 현재 점수가 해당 인덱스보다 크거나 같으면 100%, 아니면 0%
  return rating.value >= index ? '100%' : '0%'
}

// [변경됨] 별점 설정 (1단위)
const setRating = (index) => {
  // 반쪽(0.5) 로직 제거하고 인덱스 그대로 사용
  rating.value = index
}

// 시간대별 색상 클래스 반환
const getTimePeriodClass = (input) => {
  if (!input) return ''
  const str = String(input).toLowerCase()

  // 키워드 매칭
  if (['아침', 'morning', 'breakfast', '오전'].some(k => str.includes(k))) return 'period-morning'
  if (['점심', 'lunch', 'afternoon', '오후'].some(k => str.includes(k))) return 'period-afternoon'
  if (['저녁', 'dinner', 'evening', 'night', '밤'].some(k => str.includes(k))) return 'period-evening'

  // 시간 형식 매칭
  const timeMatch = str.match(/(\d{1,2}):/)
  if (timeMatch) {
    const hour = parseInt(timeMatch[1])
    if (hour < 12) return 'period-morning'
    if (hour < 18) return 'period-afternoon'
    return 'period-evening'
  }

  return ''
}

const handleRecommend = () => {
  if (!tripStore.currentPlan.is_recommended) {
    showReviewModal.value = true
    review.value = tripStore.currentPlan.review || ''
    rating.value = tripStore.currentPlan.rating ? parseFloat(tripStore.currentPlan.rating) : 5.0
  } else {
    handleUnrecommend()
  }
}

const handleUnrecommend = async () => {
  if (!confirm('추천을 취소하시겠습니까?')) return
  
  try {
    console.log('추천 취소 시작 - Plan ID:', tripStore.currentPlan.id)
    await tripStore.unrecommendPlan(tripStore.currentPlan.id)
    await tripStore.fetchPlan(tripStore.currentPlan.id)
    console.log('추천 취소 성공')
    alert('추천이 취소되었습니다.')
  } catch (error) {
    console.error('추천 취소 오류 상세:', error)
    if (error.response?.status === 404) {
      await tripStore.fetchPlan(tripStore.currentPlan.id)
      return
    }
    const errorMessage = error.response?.data?.error || error.response?.data?.detail || error.message || '추천 취소 중 오류가 발생했습니다.'
    alert(errorMessage)
  }
}

const submitReview = async () => {
  if (!review.value.trim()) {
    reviewError.value = '후기를 입력해주세요.'
    return
  }
  
  if (review.value.trim().length > 2000) {
    reviewError.value = '후기는 2000자 이하여야 합니다.'
    return
  }
  
  isSubmitting.value = true
  reviewError.value = ''
  
  try {
    await tripStore.recommendPlan(tripStore.currentPlan.id, {
      review: review.value.trim(),
      rating: parseFloat(rating.value)
    })
    showReviewModal.value = false
    review.value = ''
    rating.value = 5.0
  } catch (error) {
    reviewError.value = error.response?.data?.error || '후기 작성 중 오류가 발생했습니다.'
  } finally {
    isSubmitting.value = false
  }
}

const closeReviewModal = () => {
  showReviewModal.value = false
  review.value = ''
  rating.value = 5.0
  reviewError.value = ''
}

const handleModify = () => {
  showModifyModal.value = true
  requirements.value = ''
  modifyError.value = ''
}

const closeModifyModal = () => {
  showModifyModal.value = false
  requirements.value = ''
  modifyError.value = ''
}

const submitModify = async () => {
  if (!requirements.value.trim()) {
    modifyError.value = '수정 요구사항을 입력해주세요.'
    return
  }
  
  if (requirements.value.trim().length > 2000) {
    modifyError.value = '요구사항은 2000자 이하여야 합니다.'
    return
  }
  
  if (!confirm('계획을 수정하시겠습니까? 기존 일정이 삭제되고 새로운 일정으로 대체됩니다.')) {
    return
  }
  
  isModifying.value = true
  modifyError.value = ''
  
  try {
    console.log('계획 수정 시작 - Plan ID:', tripStore.currentPlan.id)
    await tripStore.modifyPlan(tripStore.currentPlan.id, requirements.value.trim())
    console.log('계획 수정 성공')
    showModifyModal.value = false
    requirements.value = ''
    alert('계획이 성공적으로 수정되었습니다!')
  } catch (error) {
    console.error('계획 수정 오류 상세:', error)
    modifyError.value = error.response?.data?.error || error.message || '계획 수정 중 오류가 발생했습니다.'
  } finally {
    isModifying.value = false
  }
}

const error = ref('')

onMounted(async () => {
  const id = route.params.id
  
  // 사용자 정보가 없으면 로드
  if (!authStore.user && authStore.isAuthenticated) {
    try {
      await authStore.getProfile()
    } catch (error) {
      console.error('프로필 로드 실패:', error)
    }
  }
  
  try {
    await tripStore.fetchPlan(id)
  } catch (err) {
    console.error('여행 계획 로드 실패:', err)
    if (err.response?.status === 404) {
      error.value = '여행 계획을 찾을 수 없습니다.'
    } else if (err.response?.status === 403) {
      error.value = '이 여행 계획에 접근할 수 없습니다.'
    } else {
      error.value = '여행 계획을 불러오는 중 오류가 발생했습니다.'
    }
  }
})
</script>

<template>
  <div class="itinerary-view">
    <div v-if="tripStore.loading" class="loading-container">
      <div class="loading-spinner"></div>
      <p>여행 계획을 불러오는 중입니다...</p>
    </div>

    <div v-else-if="error" class="error-message-container">
      <div class="error-message-box content-card">
        <svg class="error-icon-svg" xmlns="http://www.w3.org/2000/svg" width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
          <circle cx="12" cy="12" r="10"></circle>
          <line x1="12" y1="8" x2="12" y2="12"></line>
          <line x1="12" y1="16" x2="12.01" y2="16"></line>
        </svg>
        <h2>오류 발생</h2>
        <p>{{ error }}</p>
        <button @click="$router.push('/recommended')" class="btn-back">목록으로 돌아가기</button>
      </div>
    </div>

    <div v-else-if="tripStore.currentPlan" class="itinerary-content">
      
      <div class="header-section content-card main-header">
        <div class="header-top">
          <span class="location-badge">{{ tripStore.currentPlan.region }}</span>
          <div class="plan-meta">
            <span>{{ getWriterNickname(tripStore.currentPlan.user) }}</span>
            <span class="divider"></span>
            <span>{{ formatDate(tripStore.currentPlan.created_at) }}</span>
          </div>
        </div>
        <h1>{{ tripStore.currentPlan.title }}</h1>
        
        <div class="header-bottom">
          <div class="trip-summary-bar">
            <div class="summary-item">
              <span class="label">기간</span>
              <span class="value">{{ tripStore.currentPlan.start_date }} ~ {{ tripStore.currentPlan.end_date }}</span>
            </div>
            <div class="summary-item">
              <span class="label">인원</span>
              <span class="value">{{ tripStore.currentPlan.people_count }}명</span>
            </div>
            <div class="summary-item">
              <span class="label">스타일</span>
              <span class="value tag">{{ tripStore.currentPlan.travel_style }}</span>
            </div>
          </div>

          <div v-if="isOwner" class="action-buttons">
            <button 
              @click="handleModify" 
              class="btn-modify"
              :disabled="tripStore.loading || isModifying"
            >
              <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                <path d="M11 4H4a2 2 0 0 0-2 2v14a2 2 0 0 0 2 2h14a2 2 0 0 0 2-2v-7"></path>
                <path d="M18.5 2.5a2.121 2.121 0 0 1 3 3L12 15l-4 1 1-4 9.5-9.5z"></path>
              </svg>
              계획 수정
            </button>
            <button 
              @click="handleRecommend"
              :class="['btn-recommend-action', { 'active': tripStore.currentPlan.is_recommended }]"
            >
              <svg class="icon-svg heart" xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                <path d="M20.84 4.61a5.5 5.5 0 0 0-7.78 0L12 5.67l-1.06-1.06a5.5 5.5 0 0 0-7.78 7.78l1.06 1.06L12 21.23l7.78-7.78 1.06-1.06a5.5 5.5 0 0 0 0-7.78z"></path>
              </svg>
              {{ tripStore.currentPlan.is_recommended ? '추천됨' : '추천하기' }}
            </button>
          </div>
          
          <div v-else-if="authStore.isAuthenticated" class="recommend-section">
            <button 
              @click="handleRecommend"
              :class="['btn-recommend-action', { 'active': tripStore.currentPlan.is_recommended }]"
            >
              <svg class="icon-svg heart" xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                <path d="M20.84 4.61a5.5 5.5 0 0 0-7.78 0L12 5.67l-1.06-1.06a5.5 5.5 0 0 0-7.78 7.78l1.06 1.06L12 21.23l7.78-7.78 1.06-1.06a5.5 5.5 0 0 0 0-7.78z"></path>
              </svg>
              {{ tripStore.currentPlan.is_recommended ? '추천됨' : '추천하기' }}
            </button>
          </div>
        </div>
      </div>

      <div class="info-bar-container">
        <div class="budget-overview content-card">
          <span class="budget-label">총 예산</span>
          <span class="budget-value">{{ tripStore.currentPlan.budget.toLocaleString() }}원</span>
          <span class="budget-sub">(1인당 약 {{ Math.floor(tripStore.currentPlan.budget / tripStore.currentPlan.people_count).toLocaleString() }}원)</span>
        </div>
        <div class="price-notice content-card">
          <svg class="notice-icon-svg" xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
            <circle cx="12" cy="12" r="10"></circle>
            <line x1="12" y1="16" x2="12" y2="12"></line>
            <line x1="12" y1="8" x2="12.01" y2="8"></line>
          </svg>
          <p>표시된 예상 금액은 참고용이며, 실제 예약 시 가격이 상이할 수 있습니다.</p>
        </div>
      </div>

      <div v-if="tripStore.currentPlan.itineraries && tripStore.currentPlan.itineraries.length > 0" class="itineraries">
        <h2 class="section-title">상세 일정표</h2>
        
        <div v-for="itinerary in tripStore.currentPlan.itineraries" :key="itinerary.id" class="itinerary-day content-card">
          <div class="day-sidebar">
            <div class="day-badge">DAY {{ itinerary.day_number }}</div>
            <div class="day-date">{{ formatDate(itinerary.date) }}</div>
          </div>
          
          <div class="day-content">
            <div class="day-description-box">
              <p>{{ itinerary.description }}</p>
            </div>

            <div class="schedule-container">
              <div class="timeline-section">
                
                <template v-if="itinerary.attractions && itinerary.attractions.length > 0">
                  <h4 class="schedule-title">
                    <svg class="section-icon" xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                      <path d="M21 10c0 7-9 13-9 13s-9-6-9-13a9 9 0 0 1 18 0z"></path>
                      <circle cx="12" cy="10" r="3"></circle>
                    </svg>
                    주요 관광
                  </h4>
                  <div class="timeline-list">
                    <div 
                      v-for="(attraction, index) in itinerary.attractions" 
                      :key="'att'+index" 
                      class="timeline-item"
                      :class="getTimePeriodClass(attraction.time)"
                    >
                      <div class="time-marker">{{ attraction.time || '-' }}</div>
                      <div class="place-info">
                        <div class="place-name">{{ attraction.name }}</div>
                        <p v-if="attraction.description" class="place-desc">{{ attraction.description }}</p>
                      </div>
                    </div>
                  </div>
                </template>

                <template v-if="itinerary.meals_info && Object.keys(itinerary.meals_info).length > 0">
                  <h4 class="schedule-title" style="margin-top: 2rem;">
                    <svg class="section-icon" xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                      <path d="M3 2v7c0 1.1.9 2 2 2h4a2 2 0 0 0 2-2V2"></path>
                      <path d="M7 2v20"></path>
                      <path d="M21 15V2v0a5 5 0 0 0-5 5v6c0 1.1.9 2 2 2h3Zm0 0v7"></path>
                    </svg>
                    식사
                  </h4>
                  <div class="timeline-list">
                    <div 
                      v-for="(meal, time) in itinerary.meals_info" 
                      :key="'meal'+time" 
                      class="timeline-item meal"
                      :class="getTimePeriodClass(time)"
                    >
                      <div class="time-marker">{{ time }}</div>
                      <div class="place-info">
                        <div class="place-name">{{ typeof meal === 'string' ? meal : meal.restaurant }}</div>
                        <span v-if="typeof meal === 'object' && meal.cost" class="cost-tag">
                          예상 {{ meal.cost.toLocaleString() }}원
                        </span>
                      </div>
                    </div>
                  </div>
                </template>

                <template v-if="itinerary.events_info && itinerary.events_info.length > 0">
                  <h4 class="schedule-title" style="margin-top: 2rem;">
                    <svg class="section-icon" xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                      <path d="M8 2v4"></path>
                      <path d="M16 2v4"></path>
                      <rect x="3" y="4" width="18" height="18" rx="2" ry="2"></rect>
                      <path d="M3 10h18"></path>
                    </svg>
                    축제/행사
                  </h4>
                  <div class="timeline-list">
                    <div 
                      v-for="(event, index) in itinerary.events_info" 
                      :key="'event'+index" 
                      class="timeline-item"
                      :class="getTimePeriodClass(event.time)"
                    >
                      <div class="time-marker">{{ event.time || '-' }}</div>
                      <div class="place-info">
                        <div class="place-name">{{ event.name }}</div>
                        <p v-if="event.description" class="place-desc">{{ event.description }}</p>
                        <p v-if="event.location" class="place-desc">📍 {{ event.location }}</p>
                      </div>
                    </div>
                  </div>
                </template>
              </div>

              <div class="logistics-section">
                <div v-if="itinerary.accommodation_info?.name" class="logistics-card">
                  <h4>
                    <svg class="section-icon" xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                      <path d="M2 4v16"></path>
                      <path d="M2 8h18a2 2 0 0 1 2 2v10"></path>
                      <path d="M2 17h20"></path>
                      <path d="M6 8v9"></path>
                    </svg>
                    숙소
                  </h4>
                  <p class="main-text">{{ itinerary.accommodation_info.name }}</p>
                  <p class="sub-text" v-if="itinerary.accommodation_info.cost">₩{{ itinerary.accommodation_info.cost.toLocaleString() }}</p>
                </div>

                <div v-if="itinerary.transportation_info && Object.keys(itinerary.transportation_info).length > 0" class="logistics-card">
                  <h4>
                    <svg class="section-icon" xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                      <rect x="1" y="3" width="15" height="13"></rect>
                      <polygon points="16 8 20 8 23 11 23 16 16 16 16 8"></polygon>
                      <circle cx="5.5" cy="18.5" r="2.5"></circle>
                      <circle cx="18.5" cy="18.5" r="2.5"></circle>
                    </svg>
                    교통
                  </h4>
                  <div class="transport-list">
                    <div v-for="(info, key) in itinerary.transportation_info" :key="key" class="transport-item">
                      <span class="transport-label">{{ getTransportKeyLabel(key) }}</span>
                      <span class="transport-value">{{ getTransportationValueLabel(info) }}</span>
                    </div>
                  </div>
                </div>

                <div v-if="itinerary.estimated_cost" class="daily-cost-summary">
                  <span>일일 예상 비용</span>
                  <strong>{{ itinerary.estimated_cost.toLocaleString() }}원</strong>
                </div>
              </div>
            </div>
          </div>
        </div>

        <div v-if="totalEstimatedCost > 0" class="total-cost-summary content-card">
          <h3>전체 예상 여행 경비</h3>
          <div class="cost-comparison">
            <div class="cost-box plan">
              <span>계획 예산</span>
              <strong>{{ tripStore.currentPlan.budget.toLocaleString() }}원</strong>
            </div>
            <div class="cost-divider">/</div>
            <div class="cost-box estimate">
              <span>추정 비용</span>
              <strong>{{ totalEstimatedCost.toLocaleString() }}원</strong>
            </div>
          </div>
          <div class="cost-result">
            <span class="result-label">결과</span>
            <span class="result-value" :class="totalEstimatedCost > tripStore.currentPlan.budget ? 'over' : 'save'">
              {{ totalEstimatedCost > tripStore.currentPlan.budget ? '예산 초과' : '예산 절약' }}
              ({{ Math.abs(totalEstimatedCost - tripStore.currentPlan.budget).toLocaleString() }}원)
            </span>
          </div>
        </div>
      </div>

      <div v-else class="empty-itinerary content-card">
        <p>표시할 일정 정보가 없습니다.</p>
      </div>
    </div>

    <transition name="modal">
      <div v-if="showModifyModal" class="modal-overlay" @click.self="closeModifyModal">
        <div class="modal-content content-card modify-modal">
          <div class="modal-header">
            <h2>여행 계획 수정</h2>
            <button class="btn-close" @click="closeModifyModal">
              <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                <line x1="18" y1="6" x2="6" y2="18"></line>
                <line x1="6" y1="6" x2="18" y2="18"></line>
              </svg>
            </button>
          </div>
          <p class="modal-subtitle">수정하고 싶은 요구사항을 자세히 작성해주세요.</p>
          
          <div class="modify-form">
            <div class="form-group">
              <textarea
                v-model="requirements"
                placeholder="예: 2일차에 해변 관광지를 추가하고 싶어요. 저녁 식사는 해산물 요리로 변경해주세요."
                rows="8"
                maxlength="2000"
                class="requirements-textarea"
              ></textarea>
              <div class="char-count">{{ requirements.length }} / 2000</div>
            </div>
            
            <div v-if="modifyError" class="error-message">{{ modifyError }}</div>
            
            <button @click="submitModify" class="btn-submit" :disabled="isModifying">
              {{ isModifying ? '수정 중...' : '수정하기' }}
            </button>
          </div>
        </div>
      </div>
    </transition>

    <transition name="modal">
      <div v-if="showReviewModal" class="modal-overlay" @click.self="closeReviewModal">
        <div class="modal-content content-card review-modal">
          <div class="modal-header">
            <h2>추천하기</h2>
            <button class="btn-close" @click="closeReviewModal">
              <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
                <line x1="18" y1="6" x2="6" y2="18"></line>
                <line x1="6" y1="6" x2="18" y2="18"></line>
              </svg>
            </button>
          </div>
          <p class="modal-subtitle">이 여행 계획이 도움이 되셨나요?</p>
          
          <div class="review-form">
            <div class="form-group rating-group">
              <div class="rating-input">
                <div 
                  v-for="index in 5" 
                  :key="index" 
                  class="star-container"
                >
                  <svg class="star-icon bg" viewBox="0 0 24 24">
                    <path d="M12 17.27L18.18 21l-1.64-7.03L22 9.24l-7.19-.61L12 2 9.19 8.63 2 9.24l5.46 4.73L5.82 21z"/>
                  </svg>
                  <div class="star-fill-wrapper" :style="{ width: getStarFill(index) }">
                    <svg class="star-icon fill" viewBox="0 0 24 24">
                      <path d="M12 17.27L18.18 21l-1.64-7.03L22 9.24l-7.19-.61L12 2 9.19 8.63 2 9.24l5.46 4.73L5.82 21z"/>
                    </svg>
                  </div>
                  <div class="click-area" style="width: 100%; left: 0;" @click="setRating(index)"></div>
                </div>
                <span class="rating-text">{{ rating.toFixed(1) }}</span>
              </div>
            </div>
            
            <div class="form-group">
              <textarea
                v-model="review"
                placeholder="자유롭게 후기를 남겨주세요."
                rows="5"
                maxlength="2000"
                class="review-textarea"
              ></textarea>
              <div class="char-count">{{ review.length }} / 2000</div>
            </div>
            
            <div v-if="reviewError" class="error-message">{{ reviewError }}</div>
            
            <button @click="submitReview" class="btn-submit" :disabled="isSubmitting">
              {{ isSubmitting ? '처리 중...' : '작성 완료' }}
            </button>
          </div>
        </div>
      </div>
    </transition>
  </div>
</template>

<style scoped>
@import url("https://cdn.jsdelivr.net/gh/orioncactus/pretendard@v1.3.9/dist/web/static/pretendard.min.css");

.itinerary-view {
  background-color: #f9f9fb;
  min-height: 100vh;
  padding: 3rem 1rem;
  font-family: 'Pretendard', -apple-system, BlinkMacSystemFont, system-ui, Roboto, sans-serif;
  color: #333;
}

.itinerary-content {
  max-width: 1024px;
  margin: 0 auto;
}

/* SVG 아이콘 */
.icon-svg {
  width: 1.2rem;
  height: 1.2rem;
  vertical-align: middle;
}

.section-icon {
  width: 1.1rem;
  height: 1.1rem;
  margin-right: 0.4rem;
  color: #6a11cb;
  vertical-align: text-bottom;
}

.error-icon-svg {
  color: #e03131;
  margin-bottom: 1rem;
}

.notice-icon-svg {
  color: #ffd700;
  flex-shrink: 0;
}

/* 카드 스타일 */
.content-card {
  background: #ffffff;
  border-radius: 12px;
  box-shadow: 0 4px 16px rgba(106, 17, 203, 0.06);
  border: 1px solid #f0f0f0;
  padding: 2rem;
  margin-bottom: 1.5rem;
}

/* 헤더 */
.header-section.main-header {
  padding: 2.5rem 2rem;
}

.header-top {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
}

.location-badge {
  background: #f3e8ff;
  color: #6a11cb;
  padding: 0.4rem 0.8rem;
  border-radius: 4px;
  font-size: 0.85rem;
  font-weight: 600;
  letter-spacing: -0.5px;
}

.plan-meta {
  font-size: 0.9rem;
  color: #888;
  display: flex;
  align-items: center;
}

.divider {
  display: inline-block;
  width: 1px;
  height: 10px;
  background: #ddd;
  margin: 0 0.8rem;
}

.header-section h1 {
  font-size: 2.2rem;
  font-weight: 800;
  color: #1a1a1a;
  margin: 0 0 2rem 0;
  line-height: 1.3;
  letter-spacing: -0.02em;
}

.header-bottom {
  display: flex;
  justify-content: space-between;
  align-items: flex-end;
  gap: 1.5rem;
  flex-wrap: wrap;
}

.trip-summary-bar {
  display: flex;
  gap: 2.5rem;
  flex-wrap: wrap;
}

.summary-item {
  display: flex;
  flex-direction: column;
  gap: 0.3rem;
}

.summary-item .label {
  font-size: 0.8rem;
  color: #999;
  font-weight: 500;
}

.summary-item .value {
  font-size: 1.05rem;
  font-weight: 600;
  color: #333;
}

.summary-item .value.tag {
  color: #6a11cb;
}

/* 액션 버튼들 */
.action-buttons {
  display: flex;
  gap: 0.75rem;
  flex-wrap: wrap;
}

.btn-modify {
  padding: 0.7rem 1.4rem;
  border-radius: 8px;
  font-weight: 600;
  font-size: 0.95rem;
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 0.5rem;
  transition: all 0.2s ease;
  background: #1e90ff;
  border: none;
  color: white;
}

.btn-modify:hover:not(:disabled) {
  background: #1873cc;
  transform: translateY(-1px);
}

.btn-modify:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

/* 추천 버튼 */
.btn-recommend-action {
  padding: 0.7rem 1.4rem;
  border-radius: 50px;
  font-weight: 600;
  font-size: 0.95rem;
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 0.5rem;
  transition: all 0.2s ease;
  background: white;
  border: 1px solid #ddd;
  color: #555;
}

.btn-recommend-action:hover {
  background: #f8f9fa;
  border-color: #ccc;
}

.btn-recommend-action.active {
  border-color: #ffd700;
  color: #333;
  background: #fffdf0;
}

.btn-recommend-action .heart {
  transition: fill 0.2s ease;
}

.btn-recommend-action.active .heart {
  fill: #ffd700;
  stroke: #ffd700;
}

/* 예산 바 */
.info-bar-container {
  display: flex;
  gap: 1rem;
  margin-bottom: 2rem;
}

.budget-overview {
  flex: 1;
  display: flex;
  align-items: baseline;
  gap: 0.8rem;
  padding: 1.5rem 2rem;
  background: #f8f9fa;
  border: none;
  flex-wrap: wrap;
}

.budget-label {
  font-weight: 600;
  color: #555;
  font-size: 0.95rem;
}

.budget-value {
  font-size: 1.5rem;
  font-weight: 800;
  color: #6a11cb;
}

.budget-sub {
  font-size: 0.85rem;
  color: #999;
}

.price-notice {
  flex: 1.5;
  display: flex;
  align-items: flex-start;
  gap: 0.8rem;
  padding: 1rem 1.5rem;
  background: #fff9db;
  border: 1px solid #ffec99;
  color: #664d03;
  margin-bottom: 0;
}

.price-notice p {
  margin: 0;
  font-size: 0.9rem;
  line-height: 1.5;
}

/* 상세 일정 */
.section-title {
  font-size: 1.4rem;
  font-weight: 700;
  color: #222;
  margin: 3rem 0 1.2rem;
  display: flex;
  align-items: center;
  border-left: 4px solid #6a11cb;
  padding-left: 0.8rem;
}

.itinerary-day {
  display: flex;
  padding: 0;
  overflow: hidden;
  align-items: stretch;
}

.day-sidebar {
  background: #f8f9fa;
  padding: 2rem 1.5rem;
  display: flex;
  flex-direction: column;
  align-items: center;
  min-width: 120px;
  border-right: 1px solid #eaeaea;
  text-align: center;
}

.day-badge {
  font-size: 1.1rem;
  font-weight: 800;
  color: #6a11cb;
  margin-bottom: 0.4rem;
}

.day-date {
  font-size: 0.85rem;
  color: #999;
  font-weight: 500;
}

.day-content {
  flex: 1;
  padding: 2.5rem 2rem;
}

.day-description-box {
  background: #fff;
  border-left: 3px solid #e0e0e0;
  padding: 0.8rem 1.2rem;
  margin-bottom: 2.5rem;
  color: #555;
  line-height: 1.6;
  font-size: 0.95rem;
}

.schedule-container {
  display: flex;
  gap: 3rem;
}

.timeline-section {
  flex: 2;
}

.schedule-title {
  font-size: 1.05rem;
  color: #444;
  margin-bottom: 1.2rem;
  font-weight: 700;
  display: flex;
  align-items: center;
}

.timeline-list {
  position: relative;
  padding-left: 0.8rem;
  border-left: 2px solid #f1f3f5;
  margin-left: 0.6rem;
}

/* 타임라인 아이템 */
.timeline-item {
  position: relative;
  display: flex;
  gap: 1.2rem;
  margin-bottom: 1.8rem;
  padding-left: 1rem;
}

.timeline-item::after {
  content: '';
  position: absolute;
  left: -0.38rem;
  top: 0.4rem;
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: #fff;
  border: 2px solid #ddd;
}

/* 시간대별 색상 */
.timeline-item.period-morning::after {
  border-color: #3b82f6;
  background: #3b82f6;
}

.timeline-item.period-morning .time-marker {
  color: #3b82f6;
}

.timeline-item.period-afternoon::after {
  border-color: #f97316;
  background: #f97316;
}

.timeline-item.period-afternoon .time-marker {
  color: #f97316;
}

.timeline-item.period-evening::after {
  border-color: #8b5cf6;
  background: #8b5cf6;
}

.timeline-item.period-evening .time-marker {
  color: #8b5cf6;
}

.time-marker {
  font-size: 0.85rem;
  font-weight: 700;
  color: #adb5bd;
  min-width: 50px;
  padding-top: 0.1rem;
  transition: color 0.2s;
}

.place-info {
  flex: 1;
}

.place-name {
  font-weight: 700;
  color: #333;
  font-size: 1rem;
  margin-bottom: 0.2rem;
}

.place-desc {
  font-size: 0.9rem;
  color: #777;
  margin: 0;
  line-height: 1.4;
}

.cost-tag {
  font-size: 0.8rem;
  color: #666;
  background: #f1f3f5;
  padding: 0.1rem 0.4rem;
  border-radius: 4px;
  display: inline-block;
  margin-top: 0.3rem;
}

/* 물류 섹션 */
.logistics-section {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 1.2rem;
}

.logistics-card {
  background: #f8f9fa;
  padding: 1.2rem;
  border-radius: 8px;
  border: 1px solid #f1f3f5;
}

.logistics-card h4 {
  margin: 0 0 0.8rem 0;
  font-size: 0.95rem;
  color: #666;
  font-weight: 600;
  display: flex;
  align-items: center;
}

.main-text {
  font-weight: 600;
  margin: 0 0 0.2rem 0;
  font-size: 0.95rem;
}

.sub-text {
  font-size: 0.85rem;
  color: #888;
  margin: 0;
}

.transport-list {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.transport-item {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  font-size: 0.85rem;
  color: #555;
  line-height: 1.6;
}

.transport-label {
  color: #999;
  min-width: 40px;
  flex-shrink: 0;
}

.transport-value {
  font-weight: 600;
  color: #444;
  text-align: right;
  flex: 1;
  word-break: keep-all;
  margin-left: 0.5rem;
}

.daily-cost-summary {
  margin-top: auto;
  text-align: right;
  padding-top: 1rem;
  border-top: 1px dashed #e0e0e0;
  color: #666;
  font-size: 0.9rem;
}

.daily-cost-summary strong {
  display: block;
  font-size: 1.1rem;
  color: #6a11cb;
  margin-top: 0.3rem;
}

/* 최종 비용 */
.total-cost-summary {
  background: #fff;
  border: 1px solid #6a11cb;
  text-align: center;
  padding: 3rem;
}

.total-cost-summary h3 {
  margin-bottom: 2rem;
  color: #333;
  font-size: 1.3rem;
}

.cost-comparison {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 3rem;
  margin-bottom: 2rem;
}

.cost-box {
  text-align: center;
}

.cost-box span {
  display: block;
  font-size: 0.85rem;
  color: #999;
  margin-bottom: 0.5rem;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.cost-box strong {
  font-size: 1.4rem;
  color: #333;
}

.cost-divider {
  font-size: 1.5rem;
  color: #e0e0e0;
  font-weight: 300;
}

.cost-result {
  display: inline-block;
  padding: 0.6rem 1.5rem;
  background: #f8f9fa;
  border-radius: 50px;
  font-size: 1rem;
}

.result-label {
  color: #666;
  margin-right: 0.5rem;
}

.result-value {
  font-weight: 700;
}

.result-value.save {
  color: #27ae60;
}

.result-value.over {
  color: #e03131;
}

/* 모달 */
.modal-overlay {
  background: rgba(0, 0, 0, 0.4);
  backdrop-filter: blur(2px);
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 999;
}

.modal-content {
  width: 90%;
  max-width: 460px;
  padding: 2rem;
  border: none;
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.1);
  max-height: 90vh;
  overflow-y: auto;
}

.modify-modal {
  max-width: 600px;
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.5rem;
}

.modal-header h2 {
  font-size: 1.3rem;
  margin: 0;
  color: #222;
}

.btn-close {
  background: none;
  border: none;
  color: #999;
  cursor: pointer;
  padding: 0;
  display: flex;
}

.btn-close:hover {
  color: #333;
}

.modal-subtitle {
  color: #888;
  margin-bottom: 2rem;
  font-size: 0.9rem;
}

.form-group {
  margin-bottom: 1.5rem;
}

.rating-input {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.4rem;
  margin-bottom: 1.5rem;
}

.star-container {
  position: relative;
  width: 32px;
  height: 32px;
  cursor: pointer;
}

.star-icon {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
}

.star-icon.bg {
  fill: #f1f3f5;
}

.star-fill-wrapper {
  position: absolute;
  top: 0;
  left: 0;
  height: 100%;
  overflow: hidden;
}

.star-icon.fill {
  fill: #ffd700;
}

.click-area {
  position: absolute;
  top: 0;
  width: 50%;
  height: 100%;
  z-index: 1;
}

.click-area.left {
  left: 0;
}

.click-area.right {
  right: 0;
}

.rating-text {
  font-size: 1.4rem;
  font-weight: 800;
  color: #6a11cb;
  margin-left: 0.8rem;
}

.review-textarea,
.requirements-textarea {
  width: 100%;
  background: #f8f9fa;
  border: 1px solid #e0e0e0;
  padding: 1rem;
  border-radius: 8px;
  resize: none;
  font-family: inherit;
  font-size: 0.95rem;
  transition: all 0.2s;
}

.review-textarea:focus,
.requirements-textarea:focus {
  outline: none;
  border-color: #6a11cb;
  background: #fff;
}

.char-count {
  text-align: right;
  color: #999;
  font-size: 0.85rem;
  margin-top: 0.5rem;
}

.error-message {
  padding: 0.75rem;
  background: #ffebee;
  color: #c62828;
  border-radius: 8px;
  font-size: 0.9rem;
  margin-bottom: 1rem;
}

.btn-submit {
  width: 100%;
  padding: 1rem;
  background: #6a11cb;
  color: white;
  border: none;
  border-radius: 8px;
  font-weight: 600;
  font-size: 1rem;
  cursor: pointer;
  margin-top: 1rem;
  transition: background 0.2s;
}

.btn-submit:hover:not(:disabled) {
  background: #5a0eb3;
}

.btn-submit:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

/* 로딩/에러 */
.loading-container {
  height: 50vh;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  color: #888;
}

.loading-spinner {
  border: 3px solid #f1f3f5;
  border-top-color: #6a11cb;
  border-radius: 50%;
  width: 36px;
  height: 36px;
  animation: spin 1s linear infinite;
  margin-bottom: 1rem;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

.error-message-container {
  display: flex;
  justify-content: center;
  align-items: center;
  min-height: 60vh;
  padding: 2rem;
}

.error-message-box {
  text-align: center;
  padding: 4rem 2rem;
  max-width: 500px;
}

.error-message-box h2 {
  color: #e03131;
  margin: 1rem 0;
}

.error-message-box p {
  color: #666;
  margin-bottom: 2rem;
}

.btn-back {
  background: #555;
  color: white;
  border: none;
  padding: 0.7rem 1.4rem;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.9rem;
  transition: background 0.2s;
}

.btn-back:hover {
  background: #333;
}

.empty-itinerary {
  text-align: center;
  padding: 4rem 2rem;
  color: #888;
}

/* 모달 트랜지션 */
.modal-enter-active,
.modal-leave-active {
  transition: opacity 0.3s ease;
}

.modal-enter-from,
.modal-leave-to {
  opacity: 0;
}

/* 반응형 */
@media (max-width: 768px) {
  .header-section.main-header {
    padding: 1.5rem;
  }

  .header-bottom {
    flex-direction: column;
    align-items: flex-start;
    gap: 1.5rem;
  }

  .trip-summary-bar {
    width: 100%;
    justify-content: space-between;
    flex-wrap: wrap;
    gap: 1rem;
  }

  .action-buttons {
    width: 100%;
    flex-direction: column;
  }

  .btn-modify,
  .btn-recommend-action {
    width: 100%;
    justify-content: center;
  }

  .info-bar-container {
    flex-direction: column;
    gap: 1rem;
  }

  .budget-overview,
  .price-notice {
    width: 100%;
  }

  .itinerary-day {
    flex-direction: column;
  }

  .day-sidebar {
    width: 100%;
    flex-direction: row;
    justify-content: space-between;
    padding: 1rem;
    border-right: none;
    border-bottom: 1px solid #f1f3f5;
  }

  .day-content {
    padding: 1.5rem;
  }

  .schedule-container {
    flex-direction: column;
    gap: 2.5rem;
  }

  .timeline-section,
  .logistics-section {
    flex: auto;
  }

  .cost-comparison {
    gap: 1.5rem;
  }

  .modal-content {
    padding: 1.5rem;
  }
}
</style>