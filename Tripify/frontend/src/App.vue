<script setup>
import { RouterView, RouterLink } from 'vue-router'
import { useAuthStore } from '@/stores/auth'
import { computed } from 'vue'

const authStore = useAuthStore()
const isAuthenticated = computed(() => authStore.isAuthenticated)

const handleLogout = async () => {
  await authStore.logout()
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
          <RouterLink to="/trips">내 여행</RouterLink>
          <RouterLink to="/trip/new">여행 계획</RouterLink>
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
/* 전역 폰트 및 초기화 */
@import url('https://fonts.googleapis.com/css2?family=Noto+Sans+KR:wght@400;500;700;900&display=swap');

* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

#app {
  font-family: 'Noto Sans KR', -apple-system, BlinkMacSystemFont, sans-serif;
  color: #333;
  min-height: 100vh;
  background-color: #fff; 
}

/* 네비게이션 바 스타일 */
.navbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 1rem 3rem;
  background-color: #e0efff; 
  color: #333;
}

.nav-brand a {
  font-size: 1.5rem;
  font-weight: 900;
  color: #4285f4; 
  text-decoration: none;
}

.nav-links {
  display: flex;
  gap: 2rem;
  align-items: center;
}

/* 링크 텍스트 스타일 수정 */
.nav-links a,
.btn-link {
  color: #333; 
  text-decoration: none;
  font-weight: 500;
  font-size: 1rem;
  transition: color 0.3s;
}

.nav-links a:hover,
.btn-link:hover {
  color: #4285f4;
}

.btn-link {
  background: none;
  border: none;
  cursor: pointer;
  padding: 0;
  font-family: inherit;
  color: #333;
}

/* 메인 컨텐츠 영역 */
.main-content {
  width: 100%;
  margin: 0 auto;
}
</style>