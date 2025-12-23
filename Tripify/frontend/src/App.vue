<script setup>
import { RouterView, RouterLink, useRouter } from 'vue-router'
import { useAuthStore } from '@/stores/auth'
import { computed, onMounted } from 'vue'

const router = useRouter()
const authStore = useAuthStore()
const isAuthenticated = computed(() => authStore.isAuthenticated)

onMounted(async () => {
  if (isAuthenticated.value && !authStore.user) {
    try {
      await authStore.getProfile()
    } catch (error) {
      console.error('프로필 로드 실패:', error)
    }
  }
})

const handleLogout = async () => {
  await authStore.logout()
  router.push('/')
}
</script>

<template>
  <div id="app">
    <nav class="navbar">
      <div class="nav-brand">
        <RouterLink to="/">Tripify</RouterLink>
      </div>
      <div class="nav-links">
        <RouterLink to="/">홈</RouterLink>
        <template v-if="isAuthenticated">
          <div class="user-greeting" v-if="authStore.user?.nickname">
            <span class="greeting-text"><strong>{{ authStore.user.nickname }}</strong>님 안녕하세요</span>
          </div>
          <RouterLink to="/trips">내 여행</RouterLink>
          <RouterLink to="/trip/new">여행 계획</RouterLink>
          <RouterLink to="/recommended">추천 여행지</RouterLink>
          <RouterLink to="/settings">마이페이지</RouterLink>
          <button @click="handleLogout" class="btn-link">로그아웃</button>
        </template>
        <template v-else>
          <RouterLink to="/login">로그인</RouterLink>
          <RouterLink to="/signup">회원가입</RouterLink>
        </template>
      </div>
    </nav>

    <main class="main-content">
      <RouterView />
    </main>
  </div>
</template>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

#app {
  font-family: 'Noto Sans KR', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  color: #2c3e50;
  min-height: 100vh;
}

.navbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem 2rem;
  /* 배경색을 흰색으로 변경하고 그라데이션 제거 */
  background-color: #ffffff;
  color: #333333;
  /* 하단 구분선 추가 */
  border-bottom: 1px solid #e1e4e8;
  /* 그림자는 제거하거나 아주 연하게 유지 */
  box-shadow: none;
}

.nav-brand a {
  font-size: 1.5rem;
  font-weight: 800;
  /* 로고 색상을 보라색 포인트로 변경 */
  color: #6a11cb;
  text-decoration: none;
  letter-spacing: -0.5px;
}

.nav-links {
  display: flex;
  gap: 1.5rem;
  align-items: center;
}

.user-greeting {
  margin-right: 0.5rem;
  padding: 0.4rem 0.8rem;
  /* 배지 배경색을 연한 회색으로 변경 */
  background-color: #f1f3f5;
  border-radius: 50px; 
  font-size: 0.85rem;
}

.greeting-text {
  color: #495057;
  font-weight: 400;
}

.greeting-text strong {
  color: #6a11cb;
  font-weight: 600;
}

.nav-links a,
.btn-link {
  /* 링크 글자색을 진한 회색으로 변경 */
  color: #495057;
  text-decoration: none;
  padding: 0.5rem 0.8rem;
  border-radius: 6px;
  transition: all 0.2s ease;
  font-weight: 500;
}

.nav-links a:hover,
.btn-link:hover {
  /* 호버 시 배경색과 글자색 변경 */
  background-color: #f8f9fa;
  color: #6a11cb; /* 브랜드 컬러 */
  transform: translateY(-1px);
}

.nav-links a.router-link-active {
  /* 현재 활성화된 메뉴 강조 */
  color: #6a11cb;
  font-weight: 600;
}

.btn-link {
  background: none;
  border: none;
  cursor: pointer;
  font-size: 1rem;
}

.main-content {
  width: 100%;
}
</style>