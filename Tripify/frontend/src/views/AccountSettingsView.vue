<template>
  <div class="mypage-container">
    <div v-if="showPasswordVerification" class="modal-overlay">
      <div class="modal-content verification-modal">
        <h3 class="modal-title">본인 확인</h3>
        <p class="modal-info">
          개인정보 보호를 위해 비밀번호를 다시 한번 입력해주세요.
        </p>

        <div class="input-group">
          <label for="verify-password">비밀번호</label>
          <input
            type="password"
            id="verify-password"
            v-model="verifyPassword"
            placeholder="비밀번호 입력"
            class="styled-input"
            @keyup.enter="handlePasswordVerification"
            autofocus
          />
        </div>

        <div v-if="verifyError" class="error-message">
          {{ verifyError }}
        </div>

        <div class="modal-buttons">
          <button
            @click="handleCancelVerification"
            class="btn btn-secondary"
            :disabled="isVerifying"
          >
            취소
          </button>
          <button
            @click="handlePasswordVerification"
            class="btn btn-primary"
            :disabled="isVerifying || !verifyPassword"
          >
            {{ isVerifying ? '확인 중...' : '확인' }}
          </button>
        </div>
      </div>
    </div>

    <div v-if="isVerified" class="content-wrapper">
      
      <section class="section-card">
        <div class="grid-container" v-if="user">
          <div class="grid-item">
            <label>아이디</label>
            <div class="value-box">{{ user.username }}</div>
          </div>
          <div class="grid-item">
            <label>이메일</label>
            <div class="value-box">{{ user.email }}</div>
          </div>
          <div class="grid-item">
            <label>로그인 유형</label>
            <div class="value-box">{{ getLoginTypeLabel(user.login_type) }}</div>
          </div>
          <div class="grid-item">
            <label>가입일</label>
            <div class="value-box">{{ formatDate(user.created_at) }}</div>
          </div>
          <div class="grid-item" v-if="user.preferred_region">
            <label>선호 지역</label>
            <div class="value-box">{{ user.preferred_region }}</div>
          </div>
          <div class="grid-item" v-if="user.travel_style">
            <label>여행 스타일</label>
            <div class="value-box">{{ user.travel_style }}</div>
          </div>
        </div>
      </section>

      <section class="section-card" v-if="isNormalLogin">
        <h3 class="section-title">비밀번호 변경</h3>
        
        <form @submit.prevent="handlePasswordChange" class="password-grid">
          <div class="grid-full">
            <label for="current-password">현재 비밀번호</label>
            <input
              type="password"
              id="current-password"
              v-model="passwordForm.currentPassword"
              placeholder="현재 사용 중인 비밀번호"
              class="styled-input"
            />
          </div>

          <div class="grid-item">
            <label for="new-password">새 비밀번호</label>
            <input
              type="password"
              id="new-password"
              v-model="passwordForm.newPassword"
              placeholder="변경할 비밀번호 (8자 이상)"
              class="styled-input"
            />
          </div>

          <div class="grid-item">
            <label for="new-password-confirm">새 비밀번호 확인</label>
            <input
              type="password"
              id="new-password-confirm"
              v-model="passwordForm.newPasswordConfirm"
              placeholder="비밀번호 재입력"
              class="styled-input"
            />
          </div>
        </form>

        <div v-if="passwordError" class="status-message error">
          {{ passwordError }}
        </div>
        <div v-if="passwordSuccess" class="status-message success">
          {{ passwordSuccess }}
        </div>

        <div class="action-row">
           <button
            @click="handlePasswordChange"
            class="btn btn-primary"
            :disabled="isChangingPassword || !isPasswordFormValid"
          >
            {{ isChangingPassword ? '변경 중...' : '비밀번호 변경' }}
          </button>
        </div>
      </section>

      <div class="delete-zone">
        <button
          @click="showDeleteConfirmation = true"
          class="btn btn-danger-soft"
          :disabled="isDeleting"
        >
          회원탈퇴
        </button>
      </div>

    </div>

    <div v-if="showDeleteConfirmation" class="modal-overlay" @click.self="closeDeleteModal">
      <div class="modal-content">
        <h3 class="modal-title">정말 떠나시나요?</h3>
        <p class="modal-warning">
          탈퇴 시 모든 여행 기록과 데이터가 삭제되며<br>복구할 수 없습니다.
        </p>

        <div v-if="isNormalLogin" class="input-group">
          <label for="delete-password">비밀번호 확인</label>
          <input
            type="password"
            id="delete-password"
            v-model="deletePassword"
            placeholder="비밀번호를 입력해주세요"
            class="styled-input"
            @keyup.enter="handleDeleteAccount"
          />
        </div>

        <div v-if="deleteError" class="error-message">
          {{ deleteError }}
        </div>

        <div class="modal-buttons">
          <button
            @click="closeDeleteModal"
            class="btn btn-secondary"
            :disabled="isDeleting"
          >
            취소
          </button>
          <button
            @click="handleDeleteAccount"
            class="btn btn-danger"
            :disabled="isDeleting || (isNormalLogin && !deletePassword)"
          >
            {{ isDeleting ? '처리 중...' : '탈퇴하기' }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue'
import { useRouter } from 'vue-router'
import { useAuthStore } from '@/stores/auth'
import { authAPI } from '@/api/auth'

const router = useRouter()
const authStore = useAuthStore()

const user = ref(null)
const showDeleteConfirmation = ref(false)
const deletePassword = ref('')
const isDeleting = ref(false)
const deleteError = ref('')

// 비밀번호 확인 관련
const showPasswordVerification = ref(false)
const verifyPassword = ref('')
const isVerifying = ref(false)
const verifyError = ref('')
const isVerified = ref(false)

// 비밀번호 변경 관련
const passwordForm = ref({
  currentPassword: '',
  newPassword: '',
  newPasswordConfirm: '',
})
const isChangingPassword = ref(false)
const passwordError = ref('')
const passwordSuccess = ref('')

// 일반 로그인 사용자인지 확인
const isNormalLogin = computed(() => {
  return user.value && (!user.value.login_type || user.value.login_type === 'normal')
})

// 비밀번호 폼 유효성 검사
const isPasswordFormValid = computed(() => {
  return (
    passwordForm.value.currentPassword &&
    passwordForm.value.newPassword &&
    passwordForm.value.newPasswordConfirm
  )
})

onMounted(async () => {
  try {
    user.value = await authStore.getProfile()

    // 일반 로그인 사용자는 비밀번호 확인 필요
    if (isNormalLogin.value) {
      showPasswordVerification.value = true
    } else {
      // 소셜 로그인 사용자는 바로 접근
      isVerified.value = true
    }
  } catch (error) {
    console.error('프로필 로드 실패:', error)
    router.push('/login')
  }
})

const handlePasswordVerification = async () => {
  if (isVerifying.value || !verifyPassword.value) return

  try {
    isVerifying.value = true
    verifyError.value = ''

    await authAPI.verifyPassword(verifyPassword.value)

    showPasswordVerification.value = false
    isVerified.value = true
  } catch (error) {
    console.error('비밀번호 확인 실패:', error)
    if (error.response?.data?.password) {
      verifyError.value = error.response.data.password[0]
    } else {
      verifyError.value = '비밀번호가 올바르지 않습니다.'
    }
  } finally {
    isVerifying.value = false
  }
}

const handleCancelVerification = () => {
  router.push('/')
}

const getLoginTypeLabel = (type) => {
  const labels = {
    normal: '일반 로그인',
    kakao: '카카오 로그인',
    google: '구글 로그인',
  }
  return labels[type] || '일반 로그인'
}

const formatDate = (dateString) => {
  if (!dateString) return '-'
  const date = new Date(dateString)
  return date.toLocaleDateString('ko-KR', {
    year: 'numeric',
    month: 'long',
    day: 'numeric',
  })
}

const handlePasswordChange = async () => {
  if (isChangingPassword.value) return

  passwordError.value = ''
  passwordSuccess.value = ''

  if (passwordForm.value.newPassword !== passwordForm.value.newPasswordConfirm) {
    passwordError.value = '새 비밀번호가 일치하지 않습니다.'
    return
  }

  if (passwordForm.value.newPassword.length < 8) {
    passwordError.value = '새 비밀번호는 최소 8자 이상이어야 합니다.'
    return
  }

  try {
    isChangingPassword.value = true

    await authStore.changePassword(
      passwordForm.value.currentPassword,
      passwordForm.value.newPassword,
      passwordForm.value.newPasswordConfirm
    )

    passwordSuccess.value = '비밀번호가 성공적으로 변경되었습니다.'
    passwordForm.value = {
      currentPassword: '',
      newPassword: '',
      newPasswordConfirm: '',
    }

    setTimeout(() => {
      passwordSuccess.value = ''
    }, 3000)
  } catch (error) {
    console.error('비밀번호 변경 실패:', error)
    if (error.response?.data?.error) {
      passwordError.value = error.response.data.error
    } else if (error.response?.data?.current_password) {
      passwordError.value = error.response.data.current_password[0]
    } else if (error.response?.data?.new_password) {
      passwordError.value = error.response.data.new_password[0]
    } else {
      passwordError.value = '비밀번호 변경 중 오류가 발생했습니다.'
    }
  } finally {
    isChangingPassword.value = false
  }
}

const closeDeleteModal = () => {
  if (!isDeleting.value) {
    showDeleteConfirmation.value = false
    deletePassword.value = ''
    deleteError.value = ''
  }
}

const handleDeleteAccount = async () => {
  if (isDeleting.value) return

  if (isNormalLogin.value && !deletePassword.value) {
    deleteError.value = '비밀번호를 입력해주세요.'
    return
  }

  try {
    isDeleting.value = true
    deleteError.value = ''

    const passwordToSend = isNormalLogin.value ? deletePassword.value : null
    await authStore.deleteAccount(passwordToSend)

    alert('회원탈퇴가 완료되었습니다.')
    router.push('/')
  } catch (error) {
    console.error('회원탈퇴 실패:', error)
    if (error.response?.data?.error) {
      deleteError.value = error.response.data.error
    } else {
      deleteError.value = '회원탈퇴 처리 중 오류가 발생했습니다.'
    }
  } finally {
    isDeleting.value = false
  }
}
</script>

<style scoped>
/* 전체 컨테이너 및 초기화 */
.mypage-container {
  max-width: 1000px;
  margin: 4rem auto;
  padding: 0 1.5rem;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
  color: #333;
}

/* 카드 공통 스타일 (이미지의 흰색 둥근 배경 느낌) */
.content-wrapper {
  background: white;
  border-radius: 24px;
  border: 1px solid #eaeaea;
  padding: 3rem 4rem;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.04); 
}

/* 섹션 스타일 */
.section-card {
  margin-bottom: 3rem;
}

.section-title {
  font-size: 1.1rem;
  font-weight: 700;
  color: #2c3e50;
  margin-bottom: 1.5rem;
  padding-left: 4px;
}

.grid-container, .password-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1.5rem 2.5rem; /* 세로 가로 간격 */
}

.grid-full {
  grid-column: 1 / -1;
}

.grid-item, .input-group, .grid-full {
  display: flex;
  flex-direction: column;
}

label {
  font-size: 0.85rem;
  font-weight: 600;
  color: #666; 
  margin-bottom: 0.6rem;
  padding-left: 4px;
}

/* 읽기 전용 정보 박스 */
.value-box {
  background-color: #F8F9FB; 
  padding: 14px 18px;
  border-radius: 12px;
  font-size: 0.95rem;
  color: #333;
  font-weight: 500;
  min-height: 50px; 
  display: flex;
  align-items: center;
}

/* 입력 필드 (비밀번호 변경 등) */
.styled-input {
  background-color: #F8F9FB;
  border: 1px solid transparent;
  padding: 14px 18px;
  border-radius: 12px;
  font-size: 0.95rem;
  color: #333;
  width: 100%;
  transition: all 0.2s ease;
  box-sizing: border-box;
}

.styled-input:focus {
  outline: none;
  background-color: #fff;
  border-color: #4A90E2;
  box-shadow: 0 0 0 3px rgba(74, 144, 226, 0.1);
}

.styled-input::placeholder {
  color: #adb5bd;
}

/* 버튼 스타일 */
.modal-buttons, .action-row {
  display: flex;
  gap: 10px;
  margin-top: 1.5rem;
  justify-content: flex-end;
}

.btn {
  padding: 12px 24px;
  border-radius: 10px;
  font-size: 0.95rem;
  font-weight: 600;
  cursor: pointer;
  border: none;
  transition: all 0.2s;
}

.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn-primary {
  background-color: #333;
  color: white;
}
.btn-primary:hover:not(:disabled) {
  background-color: #000;
}

.btn-secondary {
  background-color: #f1f3f5;
  color: #495057;
}
.btn-secondary:hover:not(:disabled) {
  background-color: #e9ecef;
}

/* 탈퇴 버튼 구역 (중앙 하단) */
.delete-zone {
  margin-top: 4rem;
  display: flex;
  justify-content: center; /* 중앙 정렬 */
  border-top: 1px solid #f1f3f5;
  padding-top: 2rem;
}

.btn-danger-soft {
  background-color: #FF5B5C; /* 이미지의 붉은 버튼 색상 */
  color: white;
  width: 200px;
  padding: 14px 0;
  border-radius: 12px;
  font-size: 1rem;
  box-shadow: 0 4px 12px rgba(255, 91, 92, 0.2);
}
.btn-danger-soft:hover:not(:disabled) {
  background-color: #ff4042;
  transform: translateY(-1px);
}

.btn-danger {
  background-color: #FF5B5C;
  color: white;
}

/* 상태 메시지 */
.status-message {
  margin-top: 1rem;
  padding: 12px;
  border-radius: 8px;
  font-size: 0.9rem;
  text-align: center;
}
.status-message.error {
  background-color: #FFF5F5;
  color: #E03131;
}
.status-message.success {
  background-color: #F0FDF4;
  color: #166534;
}

/* 모달 스타일 */
.modal-overlay {
  position: fixed;
  top: 0; left: 0; right: 0; bottom: 0;
  background-color: rgba(0, 0, 0, 0.4);
  backdrop-filter: blur(4px);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal-content {
  background: white;
  border-radius: 20px;
  padding: 2.5rem;
  width: 100%;
  max-width: 440px;
  box-shadow: 0 20px 40px rgba(0,0,0,0.1);
  animation: modalPop 0.3s cubic-bezier(0.16, 1, 0.3, 1);
}

@keyframes modalPop {
  0% { transform: scale(0.95); opacity: 0; }
  100% { transform: scale(1); opacity: 1; }
}

.modal-title {
  font-size: 1.25rem;
  font-weight: 700;
  margin-bottom: 0.5rem;
  color: #333;
}

.modal-info, .modal-warning {
  color: #666;
  font-size: 0.95rem;
  line-height: 1.5;
  margin-bottom: 1.5rem;
}

.error-message {
  color: #E03131;
  font-size: 0.85rem;
  margin-top: 0.5rem;
  margin-bottom: 1rem;
}

/* 반응형 */
@media (max-width: 768px) {
  .mypage-container {
    padding: 0 1rem;
    margin: 2rem auto;
  }
  .content-wrapper {
    padding: 2rem 1.5rem;
  }
  .grid-container, .password-grid {
    grid-template-columns: 1fr; /* 모바일에서는 1열 */
    gap: 1.2rem;
  }
  .btn-danger-soft {
    width: 100%;
  }
}
</style>