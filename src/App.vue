<script setup lang="ts">
import { computed, onMounted, ref } from 'vue'

const mobileNavOpen = ref(false)

// 상단 카테고리 칩(추천/인기/편집자 추천/사진/일러스트/벡터/비디오) 상태
type TopCategory =
  | 'recommend'
  | 'popular'
  | 'editor'
  | 'photo'
  | 'illustration'
  | 'vector'
  | 'video'

const activeTopCategory = ref<TopCategory>('recommend')

type Category = 'photo' | 'illustration' | 'vector' | 'video' | 'music'

interface MediaItem {
  id: number
  url: string
  type: 'image' | 'gif' | 'vector'
  audioUrl?: string
}

const currentCategory = ref<Category>('photo')
const categoryLabels: Record<Category, string> = {
  photo: '사진',
  illustration: '일러스트',
  vector: '벡터',
  video: '비디오',
  music: '음악'
}
const items = ref<MediaItem[]>([])
const likes = ref<number[]>([])

// 이미지 그리드용 페이지네이션 (총 4페이지, 한 페이지당 전체의 1/4만 표시)
const TOTAL_PAGES = 4
const currentPage = ref(1)

const itemsPerPage = computed(() => {
  if (!items.value.length) return 1
  return Math.ceil(items.value.length / TOTAL_PAGES)
})

const paginatedItems = computed(() => {
  const start = (currentPage.value - 1) * itemsPerPage.value
  const end = start + itemsPerPage.value
  return items.value.slice(start, end)
})

// 음악용 오디오 인스턴스 캐시
const audioInstances = new Map<number, HTMLAudioElement>()

const resetLikes = (length: number) => {
  likes.value = Array.from({ length }, () => 0)
}

const toggleLike = (index: number) => {
  likes.value[index] = likes.value[index] === 1 ? 0 : 1
}

const getRandomPhotoHeight = (index: number) => {
  const sizes = ['180px', '240px', '300px', '350px', '420px']
  return sizes[index % sizes.length]
}

const getRandomIllustrationHeight = (index: number) => {
  const sizes = ['160px', '220px', '300px', '360px', '420px']
  return sizes[index % sizes.length]
}

const getRandomVectorHeight = (index: number) => {
  const sizes = ['140px', '200px', '260px', '320px', '380px', '440px']
  return sizes[index % sizes.length]
}

const getRandomVideoHeight = (index: number) => {
  const sizes = ['160px', '220px', '280px', '340px', '400px', '460px']
  return sizes[index % sizes.length]
}

const getRandomMusicHeight = (index: number) => {
  const sizes = ['150px', '200px', '260px', '320px', '380px', '450px']
  return sizes[index % sizes.length]
}

// Hero 섹션의 마우스 기반 아주 미세한 패럴랙스 효과
const heroShiftX = ref(0)
const heroShiftY = ref(0)

const handleHeroMouseMove = (event: MouseEvent) => {
  const target = event.currentTarget as HTMLElement
  const rect = target.getBoundingClientRect()
  const relativeX = (event.clientX - (rect.left + rect.width / 2)) / rect.width
  const relativeY = (event.clientY - (rect.top + rect.height / 2)) / rect.height

  // -1 ~ 1 범위를 아주 작은 픽셀 이동으로 변환
  heroShiftX.value = relativeX * 14
  heroShiftY.value = relativeY * 10
}

const resetHeroShift = () => {
  heroShiftX.value = 0
  heroShiftY.value = 0
}

const heroBackgroundStyle = computed(() => ({
  '--hero-shift-x': `${heroShiftX.value}px`,
  '--hero-shift-y': `${heroShiftY.value}px`
}))

// 스크롤 기반 섹션 리빌 애니메이션용 커스텀 디렉티브
const vReveal = {
  mounted(el: HTMLElement) {
    el.classList.add('section-reveal')

    const observer = new IntersectionObserver(
      (entries) => {
        entries.forEach((entry) => {
          if (entry.isIntersecting) {
            el.classList.add('section-reveal-visible')
            observer.unobserve(el)
          }
        })
      },
      {
        threshold: 0.15
      }
    )

    observer.observe(el)
    ;(el as any)._revealObserver = observer
  },
  unmounted(el: HTMLElement) {
    const observer = (el as any)._revealObserver as IntersectionObserver | undefined
    if (observer) {
      observer.unobserve(el)
      observer.disconnect()
    }
  }
}

const goToPrevPage = () => {
  if (currentPage.value > 1) {
    currentPage.value--
  }
}

const goToNextPage = () => {
  if (currentPage.value < TOTAL_PAGES) {
    currentPage.value++
  }
}

const createPhotoItems = (): MediaItem[] =>
  Array.from({ length: 180 }, (_, i) => {
    const id = i + 1
    return {
      id,
      url: `https://picsum.photos/seed/photo-${id}/600/800`,
      type: 'image'
    }
  })

const createIllustrationItems = (): MediaItem[] =>
  Array.from({ length: 180 }, (_, i) => ({
    id: i + 1,
    url: `https://picsum.photos/seed/illustration-${i}/600/800`,
    type: 'image'
  }))

const createVectorItems = (): MediaItem[] =>
  Array.from({ length: 180 }, (_, i) => ({
    id: i + 1,
    url: `https://picsum.photos/seed/vector-${i}/600/800?blur=1`,
    type: 'vector'
  }))

const createGifItems = (): MediaItem[] =>
  Array.from({ length: 180 }, (_, i) => ({
    id: i + 1,
    // picsum 기반 seed URL 사용 — 실제로는 정적인 이미지지만,
    // 나중에 직접 준비한 GIF URL 배열로 교체하기 쉽게 분리
    url: `https://picsum.photos/seed/gif-${i}/600/800`,
    type: 'gif'
  }))

const createMusicItems = (): MediaItem[] =>
  Array.from({ length: 180 }, (_, i) => {
    const id = i + 1
    return {
      id,
      url: `https://picsum.photos/seed/music-${id}/600/800`,
      type: 'image',
      audioUrl: `https://www.soundhelix.com/examples/mp3/SoundHelix-Song-${id}.mp3`
    }
  })

const loadCategory = (category: Category) => {
  currentCategory.value = category

  let data: MediaItem[] = []
  switch (category) {
    case 'photo':
      data = createPhotoItems()
      break
    case 'illustration':
      data = createIllustrationItems()
      break
    case 'vector':
      data = createVectorItems()
      break
    case 'video':
      data = createGifItems()
      break
    case 'music':
      data = createMusicItems()
      break
  }

  items.value = data
  resetLikes(data.length)
  currentPage.value = 1
}

const handleMediaMouseEnter = (item: MediaItem) => {
  if (!item.audioUrl) return
  let audio = audioInstances.get(item.id)
  if (!audio) {
    audio = new Audio(item.audioUrl)
    audioInstances.set(item.id, audio)
  }
  audio.currentTime = 0
  audio.play().catch(() => {
    // autoplay 차단 등으로 인한 에러는 무시
  })
}

const handleMediaMouseLeave = (item: MediaItem) => {
  if (!item.audioUrl) return
  const audio = audioInstances.get(item.id)
  if (audio) {
    audio.pause()
    audio.currentTime = 0
  }
}

onMounted(() => {
  loadCategory('photo')
})
</script>

<template>
  <div class="page">
    <!-- Header -->
    <header class="header">
      <div class="header-inner">
        <!-- Left: 로고 + 텍스트 -->
        <div class="header-left">
          <a href="/" class="logo">
            <img src="/queen.png" alt="하트셉수트 로고" class="logo-image" />
            <span class="logo-text">하트셉수트</span>
            <span class="logo-tagline">감성적인 무료 이미지 스튜디오</span>
          </a>
        </div>

        <!-- Center: (비워 두고, 메뉴는 이미지 바로 위로 이동) -->

        <!-- Right: 검색, 업로드, 로그인 / 가입 -->
        <div class="header-right">
          <button class="header-search" type="button">
            <svg
              xmlns="http://www.w3.org/2000/svg"
              viewBox="0 0 24 24"
              fill="none"
              stroke="currentColor"
              stroke-width="1.5"
              class="header-search-icon"
              aria-hidden="true"
            >
              <path
                stroke-linecap="round"
                stroke-linejoin="round"
                d="m21 21-5.197-5.197m0 0A7.5 7.5 0 1 0 5.196 5.196a7.5 7.5 0 0 0 10.607 10.607Z"
              />
            </svg>
            <span class="header-search-label">검색</span>
          </button>
          <button class="btn btn-outline small">업로드</button>
          <button class="header-auth-link" type="button">로그인</button>
          <button class="btn btn-primary small">가입</button>

          <button
            class="mobile-menu-button"
            type="button"
            @click="mobileNavOpen = !mobileNavOpen"
          >
            <span></span>
            <span></span>
            <span></span>
          </button>
        </div>
      </div>

      <!-- Mobile Nav -->
      <transition name="fade">
        <nav v-if="mobileNavOpen" class="nav mobile-nav">
          <a href="#" class="nav-link active">
            <span>사진</span>
          </a>
          <a href="#" class="nav-link">
            <span>일러스트</span>
          </a>
          <a href="#" class="nav-link">
            <span>벡터</span>
          </a>
          <a href="#" class="nav-link">
            <span>비디오</span>
          </a>
          <a href="#" class="nav-link">
            <span>음악</span>
          </a>

          <hr />

          <button class="btn btn-outline full-width">로그인</button>
          <button class="btn btn-primary full-width">가입</button>
        </nav>
      </transition>
    </header>

    <!-- Hero with search -->
    <main class="main">
      <section
        class="hero"
        @mousemove="handleHeroMouseMove"
        @mouseleave="resetHeroShift"
        :style="heroBackgroundStyle"
      >
        <div class="hero-content">
          <p class="hero-eyebrow">무료 이미지 플랫폼</p>
          <h1 class="hero-title">아이디어는 자유롭게. 이미지는 무료로</h1>
          <p class="hero-subtitle">
            상업적으로 사용할 수 있는 사진, 일러스트, 벡터, 비디오, 음악을 한 번에 검색하세요.
          </p>

          <div class="search-shell hero-search-shell">
            <div class="search-box">
              <input
                class="search-input"
                type="text"
                placeholder="원하는 이미지를 키워드로 검색해 보세요."
              />
              <button class="search-button" type="button" aria-label="검색">
                <svg
                  xmlns="http://www.w3.org/2000/svg"
                  viewBox="0 0 24 24"
                  fill="none"
                  stroke="currentColor"
                  stroke-width="1.5"
                  class="search-icon-svg"
                >
                  <path
                    stroke-linecap="round"
                    stroke-linejoin="round"
                    d="m21 21-5.197-5.197m0 0A7.5 7.5 0 1 0 5.196 5.196a7.5 7.5 0 0 0 10.607 10.607Z"
                  />
                </svg>
              </button>
            </div>
          </div>

          <div class="hero-ctas">
            <button class="btn btn-primary hero-cta-primary" type="button">
              무료로 시작하기
            </button>
            <button class="btn btn-outline hero-cta-secondary" type="button">
              아티스트 업로드
            </button>
          </div>

          <div class="hero-tags">
            <span class="hero-tags-label">인기 검색어:</span>
            <button class="tag">배경</button>
            <button class="tag">자연</button>
            <button class="tag">비즈니스</button>
            <button class="tag">도시</button>
          </div>

          <p class="hero-legal">
            이미지는 `상업적 사용자 사용 가능 · 표시해야 할` 권한을 부여합니다.
          </p>

          <!-- Category chips: 히어로 하단에 바로 배치 -->
      <section class="categories" v-reveal>
            <div class="categories-inner container">
              <button
                class="chip"
                :class="{ 'chip-active': activeTopCategory === 'recommend' }"
                @click="activeTopCategory = 'recommend'"
              >
                추천
              </button>
              <button
                class="chip"
                :class="{ 'chip-active': activeTopCategory === 'popular' }"
                @click="activeTopCategory = 'popular'"
              >
                인기
              </button>
              <button
                class="chip"
                :class="{ 'chip-active': activeTopCategory === 'editor' }"
                @click="activeTopCategory = 'editor'"
              >
                편집자 추천
              </button>
              <button
                class="chip"
                :class="{ 'chip-active': activeTopCategory === 'photo' }"
                @click="activeTopCategory = 'photo'"
              >
                사진
              </button>
              <button
                class="chip"
                :class="{ 'chip-active': activeTopCategory === 'illustration' }"
                @click="activeTopCategory = 'illustration'"
              >
                일러스트
              </button>
              <button
                class="chip"
                :class="{ 'chip-active': activeTopCategory === 'vector' }"
                @click="activeTopCategory = 'vector'"
              >
                벡터
              </button>
              <button
                class="chip"
                :class="{ 'chip-active': activeTopCategory === 'video' }"
                @click="activeTopCategory = 'video'"
              >
                비디오
              </button>
            </div>
          </section>
        </div>
      </section>

      <!-- Image grid -->
      <section class="grid-section" v-reveal>
        <!-- 사진/일러스트/벡터/비디오/음악 메뉴 -->
        <nav class="nav media-nav">
          <button
            type="button"
            class="nav-link"
            :class="{ active: currentCategory === 'photo' }"
            @click="loadCategory('photo')"
          >
            <span>사진</span>
          </button>
          <button
            type="button"
            class="nav-link"
            :class="{ active: currentCategory === 'illustration' }"
            @click="loadCategory('illustration')"
          >
            <span>일러스트</span>
          </button>
          <button
            type="button"
            class="nav-link"
            :class="{ active: currentCategory === 'vector' }"
            @click="loadCategory('vector')"
          >
            <span>벡터</span>
          </button>
          <button
            type="button"
            class="nav-link"
            :class="{ active: currentCategory === 'video' }"
            @click="loadCategory('video')"
          >
            <span>비디오</span>
          </button>
          <button
            type="button"
            class="nav-link"
            :class="{ active: currentCategory === 'music' }"
            @click="loadCategory('music')"
          >
            <span>음악</span>
          </button>
        </nav>

        <div class="masonry">
          <div
            v-for="(item, index) in paginatedItems"
            :key="item.id"
            class="masonry-item"
            :class="{
              'photo-random-height': currentCategory === 'photo',
              'illustration-random-height': currentCategory === 'illustration',
              'vector-random-height': currentCategory === 'vector',
              'video-random-height': currentCategory === 'video',
              'music-random-height': currentCategory === 'music'
            }"
            :style="
              currentCategory === 'photo'
                ? { height: getRandomPhotoHeight(index) }
                : currentCategory === 'illustration'
                ? { height: getRandomIllustrationHeight(index) }
                : currentCategory === 'vector'
                ? { height: getRandomVectorHeight(index) }
                : currentCategory === 'video'
                ? { height: getRandomVideoHeight(index) }
                : currentCategory === 'music'
                ? { height: getRandomMusicHeight(index) }
                : {}
            "
          >
            <div
              class="grid-media"
              @mouseenter="handleMediaMouseEnter(item)"
              @mouseleave="handleMediaMouseLeave(item)"
            >
              <div class="grid-image-wrapper">
                <img
                  class="grid-image"
                  loading="lazy"
                  :src="item.url"
                  alt="랜덤 스톡 이미지 예시"
                />
              </div>
              <div class="grid-overlay">
                <div class="grid-overlay-top">
                  <div class="grid-icon" aria-hidden="true">
                    <svg
                      xmlns="http://www.w3.org/2000/svg"
                      viewBox="0 0 24 24"
                      fill="none"
                      stroke="currentColor"
                      stroke-width="1.5"
                      class="grid-icon-svg"
                    >
                      <circle cx="12" cy="12" r="4.25" />
                      <path
                        d="M2.5 12s2.75-5 9.5-5 9.5 5 9.5 5-2.75 5-9.5 5-9.5-5-9.5-5Z"
                      />
                    </svg>
                  </div>
                  <div class="grid-icon" aria-hidden="true">
                    <svg
                      xmlns="http://www.w3.org/2000/svg"
                      viewBox="0 0 24 24"
                      fill="none"
                      stroke="currentColor"
                      stroke-width="1.5"
                      class="grid-icon-svg"
                    >
                      <path
                        d="M12 20s-6-3.35-6-8.25A3.75 3.75 0 0 1 12 9a3.75 3.75 0 0 1 6 2.75C18 16.65 12 20 12 20Z"
                      />
                    </svg>
                  </div>
                </div>
                <div class="grid-overlay-inner">
                  <span class="grid-overlay-label">자세히 보기</span>
                </div>
              </div>
              <div class="grid-caption">
                <span class="grid-caption-category">
                  {{ categoryLabels[currentCategory] }}
                </span>
                <span class="grid-caption-meta">Hatshepsut Studio</span>
              </div>
            </div>
          </div>
        </div>

        <div class="pagination">
          <button
            type="button"
            class="pagination-button"
            :disabled="currentPage === 1"
            @click="goToPrevPage"
          >
            &lt;
          </button>
          <span class="pagination-info">{{ currentPage }} / {{ TOTAL_PAGES }}</span>
          <button
            type="button"
            class="pagination-button"
            :disabled="currentPage === TOTAL_PAGES"
            @click="goToNextPage"
          >
            &gt;
          </button>
        </div>
      </section>

      <!-- Info strip -->
      <section class="info-strip" v-reveal>
        <div class="info-strip-inner container">
          <div class="info-item">
            <h2>Hatshepsut 커뮤니티에 참여하세요</h2>
            <p>
              Hatshepsut 커뮤니티의 일원이 되어, 여러분의 사진과 영상, 일러스트를 전 세계와 공유해
              보세요.
            </p>
          </div>
          <div class="info-actions">
            <button class="btn btn-primary">무료로 가입하기</button>
            <button class="btn btn-outline">작품 업로드</button>
          </div>
        </div>
      </section>
    </main>

    <!-- Footer -->
    <footer class="footer" v-reveal>
      <div class="footer-inner container">
        <div class="footer-column">
          <h3>Hatshepsut</h3>
          <p>
            Hatshepsut는 크리에이터들을 위한 무료 이미지 및 영상 공유 커뮤니티입니다.
          </p>
        </div>
        <div class="footer-column">
          <h4>탐색</h4>
          <a href="#">사진</a>
          <a href="#">일러스트</a>
          <a href="#">벡터</a>
          <a href="#">비디오</a>
          <a href="#">음악</a>
        </div>
        <div class="footer-column">
          <h4>정보</h4>
          <a href="#">이용 약관</a>
          <a href="#">개인정보 보호정책</a>
          <a href="#">FAQ</a>
          <a href="#">회사 소개</a>
        </div>
        <div class="footer-column footer-follow">
          <h4>팔로우</h4>
          <div class="footer-follow-links">
            <a href="#" class="footer-follow-icon" aria-label="인스타그램">
              <img src="/찐인스타로고.jpg" alt="인스타그램" class="footer-follow-img" />
            </a>
            <a href="#" class="footer-follow-icon" aria-label="유튜브">
              <img src="/Youtube_logo.png" alt="유튜브" class="footer-follow-img" />
            </a>
            <a href="#" class="footer-follow-icon" aria-label="틱톡">
              <img src="/real_tiktok.png" alt="틱톡" class="footer-follow-img" />
            </a>
          </div>
        </div>
      </div>
      <div class="footer-bottom">
        <span>© {{ new Date().getFullYear() }} Hatshepsut</span>
      </div>
    </footer>
  </div>
</template>

<style scoped>
.page {
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  background-color: var(--color-bg-page);
  color: var(--color-text);
}

.container {
  max-width: var(--page-width);
  margin: 0 auto;
  padding-inline: var(--space-page-x);
}

.header {
  position: sticky;
  top: 0;
  z-index: 20;
  background-color: rgba(255, 255, 255, 0.9);
  border-bottom: 1px solid rgba(229, 231, 235, 0.7);
  backdrop-filter: blur(18px);
}

.header-inner {
  max-width: var(--page-width);
  margin: 0 auto;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding-block: 0.75rem;
  padding-inline: var(--space-page-x);
}

.header-left {
  display: flex;
  align-items: center;
  gap: var(--space-lg);
}

.header-nav {
  flex: 1;
  display: flex;
  justify-content: center;
}

.logo {
  display: inline-flex;
  align-items: center;
  gap: 0.6rem;
  padding: 0.3rem 0.9rem;
  border-radius: var(--radius-pill);
  color: var(--color-text-strong);
  font-weight: 700;
  font-size: 1.1rem;
  letter-spacing: 0.03em;
}

.logo-image {
  width: 24px;
  height: 24px;
  object-fit: contain;
  display: block;
}

.logo-icon {
  width: 30px;
  height: 30px;
  border-radius: 999px;
  background: radial-gradient(circle at 30% 20%, #ffffff, #a8ffdf 40%, #58c8ff 100%);
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 800;
  font-size: 0.95rem;
  box-shadow: 0 0 0 2px rgba(243, 244, 246, 0.9), 0 6px 18px rgba(15, 23, 42, 0.12);
}

.logo-text {
  letter-spacing: 0.03em;
}

.logo-tagline {
  font-size: 0.75rem;
  color: #6b7280;
  margin-left: 0.5rem;
  padding-left: 0.75rem;
  border-left: 1px solid rgba(148, 163, 184, 0.8);
  font-weight: 400;
}

.nav {
  display: flex;
  align-items: center;
  gap: 0.6rem;
}

.nav.desktop-nav {
  justify-content: center;
  gap: 1.6rem;
}

.media-nav {
  justify-content: center;
  align-items: center;
  margin-bottom: 0.75rem;
  gap: 0.75rem;
}

.media-nav .nav-link {
  background-color: transparent;
  color: #6b7280;
  border-radius: var(--radius-pill);
  padding-inline: 0.85rem;
  transition: background-color 0.16s ease, color 0.16s ease, box-shadow 0.16s ease,
    transform 0.12s ease;
}

.nav-link {
  position: relative;
  display: inline-flex;
  align-items: center;
  gap: 0.35rem;
  font-size: 0.88rem;
  color: #000000;
  padding: 0.35rem 0.9rem 0.35rem 0.75rem;
  border-radius: var(--radius-pill);
  transition: background-color 0.18s ease, color 0.18s ease, box-shadow 0.18s ease,
    transform 0.12s ease;
}

.nav-icon {
  display: inline-flex;
  align-items: center;
  justify-content: center;
}

.nav-icon-svg {
  width: 14px;
  height: 14px;
}

.nav-link:hover {
  background-color: #f3f4f6;
  color: #111827;
  transform: none;
}

.nav-link.active {
  /* 헤더/모바일 내비게이션 활성 상태는 브라운 포인트 컬러 사용 */
  background-color: var(--color-accent-earth);
  color: #ffffff;
}

/* 이미지 카테고리 탭 전용: 활성 탭은 진한 텍스트와 얇은 보더로만 강조 */
.media-nav .nav-link.active {
  background-color: #000000;
  color: #ffffff;
  box-shadow: 0 6px 16px rgba(15, 23, 42, 0.16);
}

.nav-link.small {
  font-size: 0.82rem;
  padding-inline: 0.75rem;
  color: var(--color-text-muted);
}

.header-right {
  display: flex;
  align-items: center;
  gap: 0.6rem;
}

.header-search {
  display: inline-flex;
  align-items: center;
  gap: 0.25rem;
  border-radius: 999px;
  border: 1px solid rgba(209, 213, 219, 0.9);
  padding: 0.25rem 0.75rem;
  background-color: rgba(255, 255, 255, 0.7);
  cursor: pointer;
  transition: all 0.25s ease;
}

.header-search-icon {
  width: 16px;
  height: 16px;
  color: #9ca3af;
}

.header-search-label {
  font-size: 0.85rem;
  color: #6b7280;
}

.header-search:hover {
  transform: translateY(-1px) scaleX(1.06);
  box-shadow: 0 10px 24px rgba(15, 23, 42, 0.12);
}

.header-auth-link {
  border: none;
  background: none;
  padding: 0.35rem 0.5rem;
  font-size: 0.85rem;
  color: #6b7280;
  cursor: pointer;
  transition: color 0.2s ease;
}

.header-auth-link:hover {
  color: #111111;
}

/* 헤더 우측 '가입' 기본 버튼 hover 시 텍스트를 검정색으로 */
.header-right .btn-primary:hover {
  color: #ffffff;
}

.btn {
  border-radius: var(--radius-pill);
  border: 1px solid #000000;
  padding: 0.5rem 1.2rem;
  font-size: 0.9rem;
  font-weight: 600;
  cursor: pointer;
  transition: background-color 0.18s ease, color 0.18s ease, border-color 0.18s ease,
    transform 0.12s ease;
  font-family: inherit;
}

.btn.small {
  padding: 0.35rem 1rem;
  font-size: 0.82rem;
}

.btn-primary {
  background-color: #000000;
  color: #ffffff;
}

.btn-primary:hover {
  background-color: #000000;
  transform: translateY(-1px);
}

.btn-outline {
  background-color: transparent;
  border-color: #000000;
  color: #000000;
}

.btn-outline:hover {
  background-color: #000000;
  color: #ffffff;
}

.btn.full-width {
  width: 100%;
}

.mobile-menu-button {
  display: none;
  flex-direction: column;
  gap: 4px;
  width: 34px;
  height: 34px;
  justify-content: center;
  align-items: center;
  border-radius: var(--radius-pill);
  border: 1px solid rgba(148, 163, 184, 0.7);
  background: rgba(15, 23, 42, 0.9);
  cursor: pointer;
}

.mobile-menu-button span {
  width: 16px;
  height: 2px;
  background-color: #795c34;
  border-radius: 999px;
}

.mobile-nav {
  display: none;
  flex-direction: column;
  gap: 0.25rem;
  padding: 0.5rem var(--space-page-x) 1rem;
  background-color: var(--color-bg-page);
  border-bottom: 1px solid var(--color-border-subtle);
}

.mobile-nav hr {
  border: none;
  border-top: 1px solid rgba(30, 64, 175, 0.6);
  margin: 0.4rem 0 0.6rem;
}

.main {
  flex: 1;
  display: flex;
  flex-direction: column;
}

.hero {
  position: relative;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: #ffe6f0;
  color: #000000;
  padding-block: var(--space-section-y);
  overflow: hidden;
}

.hero::before {
  content: '';
  position: absolute;
  inset: -40px;
  background:
    radial-gradient(circle at 0% 0%, rgba(255, 255, 255, 0.7), transparent 55%),
    radial-gradient(circle at 100% 100%, rgba(255, 255, 255, 0.6), transparent 55%),
    linear-gradient(135deg, #ffe6f0 0%, #ffeef8 40%, #ffe6f0 100%);
  opacity: 0.95;
  transform: translate3d(var(--hero-shift-x, 0), var(--hero-shift-y, 0), 0);
  transition: transform 0.4s ease-out;
  pointer-events: none;
}

/* Hero → 콘텐츠 영역이 자연스럽게 이어지도록 하단을 화이트로 페이드 아웃 */
.hero::after {
  content: '';
  position: absolute;
  left: 0;
  right: 0;
  bottom: -1px;
  height: 120px;
  background: linear-gradient(to bottom, rgba(255, 230, 240, 0), #ffffff 100%);
  pointer-events: none;
}

.hero-content {
  position: relative;
  max-width: 1200px; /* 히어로 컨텐츠 영역 너비를 더 넓게 */
  width: 100%;
  padding-inline: var(--space-page-x);
  text-align: center;
}

.hero-eyebrow {
  font-size: 0.82rem;
  text-transform: uppercase;
  letter-spacing: 0.22em;
  color: rgba(31, 41, 55, 0.7);
  margin-bottom: 0.4rem;
  font-weight: 500;
}

.hero-title {
  font-size: 2.4rem;
  font-weight: 600;
  margin: 0 0 1rem;
  letter-spacing: 0.02em;
  line-height: 1.4;
}

.hero-subtitle {
  font-size: 0.95rem;
  color: var(--color-text-secondary);
  margin-bottom: 1.75rem;
  max-width: 640px;
  margin-inline: auto;
  font-weight: 400;
  line-height: 1.8;
}

.hero-ctas {
  display: flex;
  justify-content: center;
  gap: var(--space-component);
  margin-top: 1.5rem;
  margin-bottom: 1.5rem;
}

.hero-cta-primary {
  padding-inline: 1.6rem;
}

.hero-cta-secondary {
  padding-inline: 1.4rem;
  background-color: transparent;
  border-color: #000000;
  color: #000000;
}

.hero-cta-secondary:hover {
  background-color: #000000;
  border-color: #000000;
  color: #ffffff;
}

.search-shell {
  max-width: 720px;
  margin: 0 auto;
  padding: 0;
}

.hero-search-shell {
  margin-top: 1.75rem;
}

.search-box {
  display: flex;
  align-items: stretch;
  /* 히어로 검색창: 살짝 각진 모서리 + 은은한 그라디언트 */
  background: linear-gradient(
    135deg,
    rgba(255, 255, 255, 0.98),
    rgba(255, 255, 255, 0.9)
  );
  border-radius: 50px;
  padding: 0.25rem 0.25rem;
  border: 1px solid var(--color-border-subtle);
  overflow: hidden;
  transition: border-color 0.25s ease, box-shadow 0.25s ease, transform 0.25s ease;
}

.search-shell:focus-within .search-box {
  border-color: var(--color-accent);
  box-shadow: 0 14px 30px rgba(15, 23, 42, 0.12);
  transform: scale(1.02);
}

.search-type {
  display: flex;
  align-items: center;
  padding-inline: 1.2rem;
  border-radius: var(--radius-pill);
  background-color: #fcd12a;
  gap: 0.5rem;
  font-size: 0.95rem;
  white-space: nowrap;
  color: var(--color-text-muted);
}

.search-type-selected {
  font-weight: 600;
  color: #111827;
}

.search-type-divider {
  width: 1px;
  height: 18px;
  background-color: rgba(209, 213, 219, 0.9);
}

.search-type-button {
  border: none;
  background: none;
  font-size: 0.82rem;
  cursor: pointer;
  color: var(--color-text-muted);
}

.search-input {
  flex: 1;
  border: none;
  padding: 0 1.4rem;
  font-size: 0.95rem;
  outline: none;
  background-color: transparent;
  color: var(--color-text-primary);
}

.search-input::placeholder {
  color: rgba(156, 163, 175, 0.9);
}

.search-button {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  /* 검색창 컨테이너와 동일한 모서리 반경 사용 */
  border-radius: 50px;
  border: none;
  padding: 0.4rem 1.2rem;
  font-size: 0.9rem;
  font-weight: 500;
  background-color: #000000;
  color: #ffffff;
  cursor: pointer;
  transition: background-color 0.2s ease, transform 0.2s ease, box-shadow 0.2s ease;
}

.search-icon-svg {
  width: 22px;
  height: 22px;
  transition: transform 0.2s ease;
}

.search-button:hover {
  background-color: #000000;
  box-shadow: 0 10px 24px rgba(15, 23, 42, 0.22);
  transform: translateY(-1px);
}

.search-button:hover .search-icon-svg {
  transform: translateX(1px);
}

.hero-tags {
  margin-top: var(--space-md);
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  align-items: center;
  gap: 0.45rem;
  font-size: 0.84rem;
}

.hero-tags-label {
  color: #000000;
  opacity: 1;
  font-weight: 500;
}

.tag {
  border-radius: var(--radius-pill);
  border: 1px solid transparent;
  padding: 0.22rem 0.9rem;
  font-size: 0.82rem;
  background-color: transparent;
  color: var(--color-text-secondary);
  cursor: pointer;
  transition: background-color 0.15s ease, border-color 0.15s ease, color 0.15s ease;
}

.tag:hover {
  background-color: #000000;
  color: #ffffff;
  border-color: #000000;
}

.hero-legal {
  margin-top: 0.9rem;
  font-size: 0.76rem;
  color: #4b5563;
  background-color: #f3f4f6;
  display: inline-block;
  padding: 0.45rem 0.9rem;
  border-radius: 999px;
  font-weight: 300;
}

.categories {
  background-color: transparent;
  border-top: none;
  border-bottom: none;
}

.categories-inner {
  display: flex;
  flex-wrap: wrap;
  gap: 0.3rem;             /* 카테고리 칩 간 간격을 더 촘촘하게 축소 */
  padding-block: 0.4rem;   /* 위·아래 여백도 함께 축소 */
  justify-content: center; /* 좌우 여백을 둔 채 중앙 정렬 */
}

.chip {
  border-radius: var(--radius-pill);
  border: 1px solid var(--color-border-subtle);
  padding: 0.4rem 1.4rem; /* 더 크고 클릭하기 쉬운 영역으로 확장 */
  font-size: 0.88rem;
  font-weight: 500;
  background: #ffffff;
  cursor: pointer;
  color: #4b5563;
  transition: background-color 0.18s ease, border-color 0.18s ease, color 0.18s ease,
    box-shadow 0.18s ease, transform 0.16s ease;
}

.chip:hover {
  border-color: #000000;
  background: #000000;
  color: #ffffff;
  box-shadow: 0 6px 16px rgba(15, 23, 42, 0.16);
  transform: translateY(-1px);
}

.chip-active {
  /* 선택된 카테고리는 검정 배경 + 흰 텍스트로 확실히 구분 */
  background: #000000;
  color: #ffffff;
  border-color: #000000;
}

/* chip-active 상태도 hover 시 동일하게 유지 */
.chip-active:hover {
  background: #000000;
  color: #ffffff;
  border-color: #000000;
}

.grid-section {
  padding-top: 3rem;
  padding-bottom: var(--space-section-y);
  background-color: #ffffff;
  padding-inline: 1rem;
}

/* 스크롤 기반 리빌 애니메이션 (Hero 제외 섹션용) */
.section-reveal {
  opacity: 0;
  transform: translateY(56px);
  transition: opacity 0.7s ease-out, transform 0.7s ease-out;
}

.section-reveal-visible {
  opacity: 1;
  transform: translateY(0);
}

.pagination {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 0.5rem;
  margin-top: 1.5rem;
}

.pagination-button {
  width: 32px;
  height: 32px;
  border-radius: var(--radius-pill);
  border: 1px solid #000000;
  background-color: #ffffff;
  color: #000000;
  cursor: pointer;
  font-size: 0.9rem;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  transition: background-color 0.16s ease, color 0.16s ease, transform 0.12s ease,
    opacity 0.16s ease;
}

.pagination-button:hover:not(:disabled) {
  background-color: #000000;
  color: #ffffff;
  transform: translateY(-1px);
}

.pagination-button:disabled {
  opacity: 0.35;
  cursor: default;
  transform: none;
}

.pagination-info {
  font-size: 0.8rem;
  color: #4b5563;
}

.masonry {
  column-count: 5;
  column-gap: var(--space-grid); /* Masonry 레이아웃 컬럼 간 간격 ~24px */
}

.masonry-item {
  break-inside: avoid;
  margin-bottom: var(--space-grid); /* Masonry 행 간 간격도 ~24px로 통일 */
  width: 100%; /* 각 카드가 컬럼 너비를 꽉 채워 규칙적인 폭을 유지하도록 */
  border-radius: 12px;
  overflow: hidden;
}

.masonry-item img {
  width: 100%;
  height: auto;
  display: block;
}

.photo-random-height .grid-image-wrapper {
  height: 100%;
}

.photo-random-height .grid-image {
  height: 100%;
  object-fit: cover;
}

.illustration-random-height .grid-image-wrapper {
  height: 100%;
}

.illustration-random-height .grid-image {
  height: 100%;
  object-fit: cover;
}

.vector-random-height .grid-image-wrapper {
  height: 100%;
}

.vector-random-height .grid-image {
  height: 100%;
  object-fit: cover;
}

.video-random-height .grid-image-wrapper {
  height: 100%;
}

.video-random-height .grid-image {
  height: 100%;
  object-fit: cover;
}

.music-random-height .grid-image-wrapper {
  height: 100%;
}

.music-random-height .grid-image {
  height: 100%;
  object-fit: cover;
}

@media (max-width: 1280px) {
  .masonry {
    column-count: 4;
  }
}

@media (max-width: 1024px) {
  .masonry {
    column-count: 3;
  }
}

@media (max-width: 768px) {
  .masonry {
    column-count: 2;
  }
}

.grid-media {
  position: relative;
  border-radius: var(--radius-card);
  overflow: hidden;
  background-color: var(--color-surface-card);
  border: 1px solid var(--color-border-subtle);
  transition: background-color 0.2s ease, opacity 0.2s ease;
}

.masonry-item:hover .grid-media {
  opacity: 0.97;
  background-color: #ffffff;
}

.grid-image-wrapper {
  overflow: hidden;
}

.grid-image {
  width: 100%;
  height: auto;
  object-fit: cover;
  display: block;
  transition: transform 0.25s ease-out;
}

.grid-overlay {
  position: absolute;
  inset: 0;
  background: linear-gradient(to bottom, rgba(15, 23, 42, 0.06), rgba(15, 23, 42, 0.2));
  opacity: 0;
  display: flex;
  align-items: flex-end;
  justify-content: center;
  transition: opacity 0.25s ease;
  pointer-events: none;
}

.grid-overlay-top {
  position: absolute;
  top: 0.55rem;
  right: 0.55rem;
  display: flex;
  gap: 0.35rem;
}

.grid-icon {
  width: 26px;
  height: 26px;
  border-radius: 999px;
  background-color: rgba(15, 23, 42, 0.78);
  display: inline-flex;
  align-items: center;
  justify-content: center;
  opacity: 0;
  transform: translateY(-4px);
  transition: opacity 0.2s ease, transform 0.2s ease;
}

.grid-icon-svg {
  width: 14px;
  height: 14px;
  color: #f9fafb;
}

.grid-overlay-inner {
  width: 100%;
  padding: 0.6rem 0.85rem;
  text-align: left;
}

.grid-overlay-label {
  display: inline-flex;
  align-items: center;
  padding: 0.18rem 0.7rem;
  border-radius: 999px;
  font-size: 0.75rem;
  letter-spacing: 0.06em;
  text-transform: uppercase;
  color: #f9fafb;
  background-color: rgba(15, 23, 42, 0.75);
}

.masonry-item:hover .grid-overlay {
  opacity: 1;
}

.masonry-item:hover .grid-icon {
  opacity: 1;
  transform: translateY(0);
}

/* 이미지 카드 hover 시 이미지가 살짝 확대되도록 */
.masonry-item:hover .grid-image {
  transform: scale(1.04);
}

.grid-caption {
  padding: 0.6rem 0.75rem 0.8rem;
  display: flex;
  flex-direction: column;
  gap: 0.35rem;
}

.grid-caption-category {
  font-size: 0.78rem;
  letter-spacing: 0.08em;
  text-transform: uppercase;
  color: var(--color-text-secondary);
}

.grid-caption-meta {
  font-size: 0.8rem;
  color: var(--color-text-secondary);
}

.info-strip {
  border-top: none;
  background: linear-gradient(135deg, #f9fafb 0%, #fff4fb 40%, #f9fafb 100%);
  padding-block: 4rem;
}

.info-strip-inner {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: var(--space-component);
  background-color: #ffffff;
  border-radius: 18px;
  padding: 2.25rem var(--space-page-x);
  box-shadow: 0 20px 45px rgba(15, 23, 42, 0.08);
}

.info-strip .info-item {
  flex: 1;
}

.info-strip h2 {
  font-size: 1.5rem;
  margin: 0 0 0.4rem;
  color: var(--color-text-strong);
  font-weight: 700;
}

.info-strip p {
  margin: 0;
  font-size: 0.95rem;
  color: var(--color-text-muted);
}

.info-actions {
  display: flex;
  gap: var(--space-component);
}

.info-actions .btn-primary,
.info-actions .btn-outline {
  box-shadow: 0 0 0 rgba(15, 23, 42, 0);
  transition: box-shadow 0.2s ease, transform 0.16s ease;
}

.info-actions .btn-primary:hover,
.info-actions .btn-outline:hover {
  box-shadow: 0 14px 30px rgba(15, 23, 42, 0.14);
  transform: translateY(-1px);
}

/* 커뮤니티 CTA의 '무료로 가입하기' 버튼 hover 시 텍스트를 검정색으로 */
.info-actions .btn-primary:hover {
  color: #ffffff;
}

/* Hero 섹션의 '무료로 시작하기' 버튼 hover 시 텍스트를 검정색으로 */
.hero-ctas .btn-primary:hover {
  color: #ffffff;
}

.footer {
  background-color: #ffffff;
  color: #000000;
  padding-block: 3.5rem; /* 상단 여백만 조금 넉넉하게 */
  border-top: 1px solid #dddddd;
}

.footer-inner {
  display: grid;
  grid-template-columns: 1.8fr repeat(3, 1fr);
  column-gap: var(--space-component); /* 컴포넌트 간 간격 시스템에 맞춤 (~24px) */
  row-gap: var(--space-component);
}

.footer-column h3,
.footer-column h4 {
  margin-top: 0;
  margin-bottom: 0.5rem;
  color: #000000;
  font-weight: 700;
}

.footer-column p {
  margin: 0;
  font-size: 0.92rem;
  color: #333333;
  font-weight: 300;
  line-height: 1.8; /* 텍스트 줄 간격을 더 넉넉하게 */
}

.footer-column a {
  display: block;
  margin-bottom: 0.35rem; /* 항목 간 세로 간격 증가 */
  color: #333333;
  font-size: 0.88rem;
  font-weight: 400;
  line-height: 1.8; /* 링크 목록 줄 간격도 확대 */
}

.footer-follow a {
  font-weight: 500;
}

.footer-follow-links {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  margin-top: 0.4rem;
}

.footer-follow-icon {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 32px;
  height: 32px;
  border-radius: 999px;
  border: 1px solid #000000;
  color: #000000;
  cursor: pointer;
  transition: background-color 0.18s ease, color 0.18s ease, transform 0.16s ease,
    box-shadow 0.18s ease;
}

.footer-follow-icon-svg {
  width: 18px;
  height: 18px;
}

.footer-follow-icon:hover {
  box-shadow: 0 8px 18px rgba(15, 23, 42, 0.22);
  transform: scale(1.05);
}

.footer-follow-img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  border-radius: 999px;
  display: block;
}

/* 제목(h4)과 항목 리스트 사이의 시각적 구분 강화 (탐색/정보/팔로우 영역) */
.footer-column h4 + a {
  margin-top: 0.4rem;
  padding-top: 0.35rem;
  border-top: 1px solid #dddddd;
}

.footer-column a:hover {
  color: #000000;
}

.footer-bottom {
  max-width: var(--page-width);
  margin: 1.1rem auto 0;
  padding-top: 0.4rem;
  border-top: 1px solid #dddddd;
  font-size: 0.75rem;
  color: #555555;
  padding-inline: var(--space-page-x);
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.16s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

@media (max-width: 1024px) {
  .container {
    padding-inline: 2.5rem;
  }

  .header-inner,
  .mobile-nav {
    padding-inline: 2.5rem;
  }

  .hero-content {
    padding-inline: 2.5rem;
  }
}

@media (max-width: 900px) {
  .hero {
    padding-block: var(--space-2xl) var(--space-xl);
  }

  .hero-title {
    font-size: 2.3rem;
  }

  .hero-content {
    padding-inline: 2rem;
  }

  .search-box {
    flex-direction: column;
    padding: 0.6rem;
    gap: 0.45rem;
  }

  .search-type {
    align-self: stretch;
    justify-content: flex-start;
  }

  .search-input {
    padding-block: 0.42rem;
  }

  .search-button {
    padding-block: 0.6rem;
  }

  .info-strip-inner {
    flex-direction: column;
    align-items: flex-start;
  }
}

@media (max-width: 768px) {
  .container {
    padding-inline: 1.5rem;
  }

  .header-inner,
  .mobile-nav {
    padding-inline: 1.5rem;
  }

  .desktop-nav {
    display: none;
  }

  .mobile-menu-button {
    display: flex;
  }

  .mobile-nav {
    display: flex;
  }

  .footer-inner {
    grid-template-columns: 1.4fr 1fr;
  }
}

@media (max-width: 600px) {
  .container {
    padding-inline: 1.25rem;
  }

  .hero-content {
    padding-inline: 1.25rem;
  }

  .hero-title {
    font-size: 2rem;
  }
  .footer-inner {
    grid-template-columns: 1fr;
  }
}
</style>
