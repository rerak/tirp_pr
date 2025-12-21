<script setup>
import { ref } from 'vue'
import axios from 'axios'

// --- 상태 변수 ---
const formData = ref({
  email: '',
})

const error = ref('')
const success = ref('')
const loading = ref(false)

// 환경변수 설정
const API_URL = import.meta.env.VITE_API_URL || 'http://localhost:8000/api'

const handleSubmit = async () => {
  try {
    loading.value = true
    error.value = ''
    success.value = ''

    // API 호출
    const response = await axios.post(`${API_URL}/auth/recover-username/`, formData.value)

    success.value = response.data.message
    formData.value.email = '' // 성공 후 입력창 초기화
  } catch (err) {
    error.value = err.response?.data?.error || '일치하는 정보가 없습니다.'
  } finally {
    loading.value = false
  }
}
</script>

<template>
  <div class="page-container">
    <main class="main-content">
      <div class="find-card">
        
        <div class="header-section">
          <h2>계정을 잊으셨나요? 🤔</h2>
          <h1>아이디 찾기</h1>
          <p>가입 시 등록한 이메일 주소를 입력하시면,<br>아이디를 이메일로 보내드립니다.</p>
        </div>

        <transition name="fade">
          <div v-if="error" class="message error">
            <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" class="icon">
              <path fill-rule="evenodd" d="M18 10a8 8 0 11-16 0 8 8 0 0116 0zm-7 4a1 1 0 11-2 0 1 1 0 012 0zm-1-9a1 1 0 00-1 1v4a1 1 0 102 0V6a1 1 0 00-1-1z" clip-rule="evenodd" />
            </svg>
            <span>{{ error }}</span>
          </div>
        </transition>

        <transition name="fade">
          <div v-if="success" class="message success">
            <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor" class="icon">
              <path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zm3.707-9.293a1 1 0 00-1.414-1.414L9 10.586 7.707 9.293a1 1 0 00-1.414 1.414l2 2a1 1 0 001.414 0l4-4z" clip-rule="evenodd" />
            </svg>
            <span>{{ success }}</span>
          </div>
        </transition>

        <form @submit.prevent="handleSubmit" class="form-wrapper">
          <div class="form-group">
            <label for="email">이메일</label>
            <input 
              id="email"
              v-model="formData.email" 
              type="email" 
              required 
              placeholder="example@tripify.com" 
              :disabled="loading"
            />
          </div>

          <button type="submit" class="btn-submit" :disabled="loading">
            {{ loading ? '전송 중...' : '아이디 찾기' }}
          </button>
        </form>

        <div class="footer-links">
          <router-link to="/login" class="back-link">
            로그인으로 돌아가기
          </router-link>
        </div>

      </div>
    </main>
  </div>
</template>

<style scoped>
/* 1. 폰트 로드 (Pretendard) - 로그인 페이지와 동일 */
@import url("https://cdn.jsdelivr.net/gh/orioncactus/pretendard@v1.3.9/dist/web/static/pretendard.min.css");

/* 2. 기본 레이아웃 */
.page-container {
  min-height: 80vh;
  display: flex;
  flex-direction: column;
  background-color: transparent;
  font-family: "Pretendard", -apple-system, BlinkMacSystemFont, system-ui, Roboto, sans-serif;
  color: #333;
}

.main-content {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 1.5rem;
}

/* 3. 카드 디자인 (로그인 페이지와 통일) */
.find-card {
  width: 100%;
  max-width: 460px; /* 로그인창보다 살짝 넓게 잡거나 동일하게 */
  padding: 3.5rem 3rem;
  background: #ffffff;
  border-radius: 24px;
  box-shadow: 0 10px 40px rgba(0, 0, 0, 0.04), 0 2px 10px rgba(0, 0, 0, 0.02);
  border: 1px solid rgba(255, 255, 255, 0.8);
  transition: transform 0.3s ease;
}

/* 헤더 텍스트 */
.header-section {
  text-align: left;
  margin-bottom: 2.5rem;
}
.header-section h2 {
  font-size: 1.1rem;
  color: #64748b;
  margin: 0 0 0.5rem 0;
  font-weight: 500;
}
.header-section h1 {
  font-size: 2rem;
  font-weight: 800;
  color: #1e293b;
  margin: 0 0 1rem 0;
  letter-spacing: -0.5px;
}
.header-section p {
  color: #94a3b8;
  font-size: 0.95rem;
  line-height: 1.5;
  margin: 0;
}

/* 4. 폼 스타일 */
.form-wrapper {
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.form-group label {
  font-size: 0.9rem;
  font-weight: 600;
  color: #475569;
  margin-left: 4px;
}

.form-group input {
  width: 100%;
  padding: 0.95rem 1.25rem;
  background-color: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 12px;
  font-size: 1rem;
  outline: none;
  transition: all 0.2s ease;
  box-sizing: border-box;
  color: #1e293b;
}

.form-group input:focus {
  background-color: #fff;
  border-color: #3b82f6;
  box-shadow: 0 0 0 4px rgba(59, 130, 246, 0.1);
}

.form-group input:disabled {
  background-color: #e2e8f0;
  cursor: not-allowed;
  color: #94a3b8;
}

/* 5. 버튼 스타일 */
.btn-submit {
  background-color: #1e293b;
  color: #fff;
  padding: 1.1rem;
  border-radius: 12px;
  border: none;
  font-weight: 700;
  font-size: 1.05rem;
  cursor: pointer;
  margin-top: 0.5rem;
  transition: all 0.2s;
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
}

.btn-submit:hover:not(:disabled) {
  background-color: #334155;
  transform: translateY(-2px);
  box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1);
}

.btn-submit:disabled {
  background-color: #94a3b8;
  cursor: not-allowed;
  transform: none;
  box-shadow: none;
}

/* 6. 메시지 박스 (에러/성공) */
.message {
  padding: 0.9rem;
  border-radius: 10px;
  margin-bottom: 1.5rem;
  font-size: 0.9rem;
  display: flex;
  align-items: flex-start;
  gap: 0.6rem;
  line-height: 1.4;
}
.icon {
  width: 20px;
  height: 20px;
  flex-shrink: 0;
  margin-top: 1px;
}

/* 에러 스타일 */
.message.error {
  background-color: #fef2f2;
  color: #dc2626;
  border: 1px solid #fecaca;
}

/* 성공 스타일 */
.message.success {
  background-color: #f0fdf4;
  color: #166534;
  border: 1px solid #bbf7d0;
}

/* 애니메이션 */
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s ease;
}
.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

/* 7. 하단 링크 */
.footer-links {
  margin-top: 2rem;
  text-align: center;
}

.back-link {
  color: #64748b;
  font-weight: 600;
  text-decoration: none;
  font-size: 0.95rem;
  transition: color 0.2s;
}

.back-link:hover {
  color: #1e293b;
  text-decoration: underline;
}

/* 모바일 반응형 */
@media (max-width: 640px) {
  .find-card {
    padding: 2.5rem 1.5rem;
    box-shadow: none;
    border: none;
  }
}
</style>