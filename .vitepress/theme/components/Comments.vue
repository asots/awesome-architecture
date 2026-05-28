<script setup lang="ts">
import { useData, useRoute } from 'vitepress'
import { onMounted, watch, nextTick, ref } from 'vue'

// Giscus 配置(由仓库 study8677/awesome-architecture 的 GitHub Discussions 提供)
const REPO = 'study8677/awesome-architecture'
const REPO_ID = 'R_kgDOSlbKGA'
const CATEGORY = 'Announcements'
const CATEGORY_ID = 'DIC_kwDOSlbKGM4C9yTw'
const GISCUS_ORIGIN = 'https://giscus.app'

const { isDark, lang } = useData()
const route = useRoute()
const container = ref<HTMLElement | null>(null)

const giscusTheme = () => (isDark.value ? 'dark' : 'light')
const giscusLang = () => (lang.value?.startsWith('en') ? 'en' : 'zh-CN')
const isEn = () => giscusLang() === 'en'

// 每次进入新页面都重新挂载脚本,这样每篇文档对应自己的 Discussion(按 pathname 映射)
function load() {
  const el = container.value
  if (!el) return
  el.innerHTML = ''
  const s = document.createElement('script')
  s.src = `${GISCUS_ORIGIN}/client.js`
  const attrs: Record<string, string> = {
    'data-repo': REPO,
    'data-repo-id': REPO_ID,
    'data-category': CATEGORY,
    'data-category-id': CATEGORY_ID,
    'data-mapping': 'pathname',
    'data-strict': '0',
    'data-reactions-enabled': '1',
    'data-emit-metadata': '0',
    'data-input-position': 'top',
    'data-theme': giscusTheme(),
    'data-lang': giscusLang(),
    'data-loading': 'lazy',
  }
  for (const [k, v] of Object.entries(attrs)) s.setAttribute(k, v)
  s.crossOrigin = 'anonymous'
  s.async = true
  el.appendChild(s)
}

// 切换深/浅色时,不重载整个 iframe,直接通过 postMessage 通知 giscus 换主题
function syncTheme() {
  const iframe = document.querySelector<HTMLIFrameElement>('iframe.giscus-frame')
  iframe?.contentWindow?.postMessage(
    { giscus: { setConfig: { theme: giscusTheme() } } },
    GISCUS_ORIGIN,
  )
}

// ─────────────── Hypothesis 划词标注:默认关闭、按需开启 ───────────────
// 历史上 config.mts 里曾全站默认挂载 hypothes.is/embed.js,会让侧边栏在
// 每个页面都默认露面。现改为:默认不加载,用户在评论区顶部按钮里手动启用,
// 偏好存 localStorage 跨页持久。
const annotationsEnabled = ref(false)
const STORAGE_KEY = 'awesome-arch-annotations-enabled'
const HYPOTHESIS_SCRIPT_ID = 'hypothesis-embed-script'

function loadHypothesis() {
  if (typeof document === 'undefined') return
  if (document.getElementById(HYPOTHESIS_SCRIPT_ID)) return // 已加载,避免重复注入
  const s = document.createElement('script')
  s.id = HYPOTHESIS_SCRIPT_ID
  s.src = 'https://hypothes.is/embed.js'
  s.async = true
  document.head.appendChild(s)
}

function toggleAnnotations() {
  if (typeof window === 'undefined') return
  if (annotationsEnabled.value) {
    // 已开启 → 关闭:清偏好,刷新页面让 Hypothesis 完全离场
    localStorage.removeItem(STORAGE_KEY)
    location.reload()
  } else {
    // 关闭 → 开启:存偏好,动态注入脚本
    localStorage.setItem(STORAGE_KEY, '1')
    annotationsEnabled.value = true
    loadHypothesis()
  }
}

onMounted(() => {
  // 恢复划词标注偏好(若用户之前开启过)
  if (typeof localStorage !== 'undefined' && localStorage.getItem(STORAGE_KEY) === '1') {
    annotationsEnabled.value = true
    loadHypothesis()
  }
  load()
})
watch(() => route.path, () => nextTick(load))
watch(isDark, syncTheme)
</script>

<template>
  <div class="comments">
    <h2 class="comments__title">{{ isEn() ? '💬 Comments' : '💬 评论' }}</h2>

    <button
      class="comments__annotate"
      :class="{ on: annotationsEnabled }"
      @click="toggleAnnotations"
      :title="isEn()
        ? 'Powered by Hypothesis (hypothes.is). When ON, select text in the article to highlight & comment; a sidebar appears on the right.'
        : '由 Hypothesis (hypothes.is) 提供。开启后,选中正文文字即可高亮 + 评论;右侧会出现侧边栏。'"
    >
      <template v-if="annotationsEnabled">
        {{ isEn() ? '🖍️ Inline annotations: ON · click to disable' : '🖍️ 划词标注已开启 · 点击关闭' }}
      </template>
      <template v-else>
        {{ isEn() ? '🖍️ Enable inline annotations (Hypothesis)' : '🖍️ 启用划词标注(Hypothesis)' }}
      </template>
    </button>

    <div ref="container" class="giscus" />
  </div>
</template>

<style scoped>
.comments {
  margin-top: 48px;
  padding-top: 24px;
  border-top: 1px solid var(--vp-c-divider);
}
.comments__title {
  font-size: 1.25rem;
  font-weight: 600;
  margin-bottom: 16px;
  color: var(--vp-c-text-1);
}
.comments__annotate {
  display: inline-flex;
  align-items: center;
  margin-bottom: 18px;
  padding: 6px 14px;
  border: 1px solid var(--vp-c-divider);
  border-radius: 18px;
  background: var(--vp-c-bg);
  color: var(--vp-c-text-2);
  cursor: pointer;
  font-size: 13px;
  line-height: 1.5;
  transition: 0.18s;
}
.comments__annotate:hover {
  border-color: var(--vp-c-brand-1);
  color: var(--vp-c-text-1);
}
.comments__annotate.on {
  background: rgba(60, 135, 114, 0.16);
  color: var(--vp-c-brand-1);
  border-color: var(--vp-c-brand-1);
}
</style>
