<script setup lang="ts">
import { ref, computed } from 'vue'
import { useData, withBase } from 'vitepress'

type Category = 'common' | 'ai' | 'agent'
type TabKey = 'all' | Category

interface Item { name: string; icon: string; cat: Category; desc: string; link: string }
interface Tab { k: TabKey; t: string }

const itemsZh: Item[] = [
  { name: 'AI 对话产品', icon: '🤖', cat: 'common', desc: 'LLM 推理 · 流式 · 上下文 · 成本', link: '/templates/ai-chat-product/README' },
  { name: '浏览器插件', icon: '🧩', cat: 'common', desc: '脚本分离 · 注入 · 隐私边界', link: '/templates/browser-extension/README' },
  { name: '普通网站', icon: '🌐', cat: 'common', desc: '三层 · 缓存 · 读写分离', link: '/templates/standard-web-app/README' },
  { name: '移动 App', icon: '📱', cat: 'common', desc: '离线优先 · 同步 · 推送', link: '/templates/mobile-app/README' },
  { name: '电商平台', icon: '🛒', cat: 'common', desc: '库存 · 订单 · 超卖 · 洪峰', link: '/templates/ecommerce-platform/README' },
  { name: '社交信息流', icon: '📰', cat: 'common', desc: 'Feed 推拉 · 关注图 · 扩散', link: '/templates/social-feed/README' },
  { name: '视频流媒体', icon: '🎬', cat: 'common', desc: '转码 · CDN · 自适应码率', link: '/templates/video-streaming/README' },
  { name: '实时通讯', icon: '💬', cat: 'common', desc: '长连接 · 时序 · 离线投递', link: '/templates/realtime-chat/README' },
  { name: '短链接服务', icon: '🔗', cat: 'common', desc: '读多写少 · 重定向 · 唯一 ID', link: '/templates/url-shortener/README' },
  { name: '支付系统', icon: '💳', cat: 'common', desc: '幂等 · 复式记账 · 对账', link: '/templates/payment-system/README' },
  { name: '搜索引擎', icon: '🔍', cat: 'common', desc: '倒排索引 · 召回 + 精排', link: '/templates/search-engine/README' },
  { name: '网约车 / 出行', icon: '🚗', cat: 'common', desc: '地理索引 · 实时匹配', link: '/templates/ride-hailing/README' },
  { name: '实时协同文档', icon: '📝', cat: 'common', desc: 'OT/CRDT · 单 writer 串行', link: '/templates/collaborative-doc/README' },
  { name: '云存储 / 网盘', icon: '☁️', cat: 'common', desc: '分块 · 去重 · 增量同步', link: '/templates/cloud-storage/README' },
  { name: '通知 / 推送', icon: '🔔', cat: 'common', desc: '多渠道扇出 · 去重限频', link: '/templates/notification-system/README' },
  { name: '在线票务 / 抢票', icon: '🎫', cat: 'common', desc: '等候室 · 原子扣减 · 锁座', link: '/templates/online-ticketing/README' },
  { name: 'AI 网关 / 中转站', icon: '🚪', cat: 'ai', desc: '统一接口 · 计费 · 负载均衡', link: '/templates/ai-gateway/README' },
  { name: 'RAG 知识库', icon: '📚', cat: 'ai', desc: '切块 · 向量检索 · 重排', link: '/templates/rag-knowledge-base/README' },
  { name: 'AI Agent / 工作流', icon: '🤹', cat: 'ai', desc: '行动循环 · 工具 · 记忆', link: '/templates/ai-agent-platform/README' },
  { name: '模型推理服务', icon: '⚡', cat: 'ai', desc: '连续批处理 · 分页 KV', link: '/templates/inference-serving/README' },
  { name: '向量数据库', icon: '🧭', cat: 'ai', desc: 'ANN · HNSW · 相似检索', link: '/templates/vector-database/README' },
  { name: 'Claude Code', icon: '🖥️', cat: 'agent', desc: '本地优先编码 · 子代理/钩子/MCP · 双层权限', link: '/templates/claude-code/README' },
  { name: 'OpenAI Codex', icon: '🛰️', cat: 'agent', desc: '本地 CLI + 云端沙箱 · 异步开 PR · 双轴安全', link: '/templates/codex/README' },
  { name: 'OpenClaw(龙虾)', icon: '🦞', cat: 'agent', desc: '自托管网关 · 聊天软件即 UI · 常驻', link: '/templates/openclaw/README' },
  { name: 'Hermes(爱马仕)', icon: '🧠', cat: 'agent', desc: 'FTS5 持久记忆 · 自动沉淀技能 · 常驻', link: '/templates/hermes/README' },
]

const itemsEn: Item[] = [
  { name: 'AI Chat Product', icon: '🤖', cat: 'common', desc: 'LLM inference · streaming · context · cost', link: '/en/templates/ai-chat-product/README' },
  { name: 'Browser Extension', icon: '🧩', cat: 'common', desc: 'Content/background split · injection · privacy', link: '/en/templates/browser-extension/README' },
  { name: 'Standard Web App', icon: '🌐', cat: 'common', desc: 'Three-tier · caching · read/write split', link: '/en/templates/standard-web-app/README' },
  { name: 'Mobile App', icon: '📱', cat: 'common', desc: 'Offline-first · sync · push', link: '/en/templates/mobile-app/README' },
  { name: 'E-commerce Platform', icon: '🛒', cat: 'common', desc: 'Inventory · orders · overselling · spikes', link: '/en/templates/ecommerce-platform/README' },
  { name: 'Social Feed', icon: '📰', cat: 'common', desc: 'Feed push/pull · follow graph · fan-out', link: '/en/templates/social-feed/README' },
  { name: 'Video Streaming', icon: '🎬', cat: 'common', desc: 'Transcoding · CDN · adaptive bitrate', link: '/en/templates/video-streaming/README' },
  { name: 'Realtime Chat', icon: '💬', cat: 'common', desc: 'Long-lived conns · ordering · offline delivery', link: '/en/templates/realtime-chat/README' },
  { name: 'URL Shortener', icon: '🔗', cat: 'common', desc: 'Read-heavy · redirect · unique ID', link: '/en/templates/url-shortener/README' },
  { name: 'Payment System', icon: '💳', cat: 'common', desc: 'Idempotency · double-entry · reconciliation', link: '/en/templates/payment-system/README' },
  { name: 'Search Engine', icon: '🔍', cat: 'common', desc: 'Inverted index · recall + rerank', link: '/en/templates/search-engine/README' },
  { name: 'Ride-Hailing', icon: '🚗', cat: 'common', desc: 'Geo index · realtime matching', link: '/en/templates/ride-hailing/README' },
  { name: 'Collaborative Doc', icon: '📝', cat: 'common', desc: 'OT/CRDT · single-writer serialization', link: '/en/templates/collaborative-doc/README' },
  { name: 'Cloud Storage', icon: '☁️', cat: 'common', desc: 'Chunking · dedup · incremental sync', link: '/en/templates/cloud-storage/README' },
  { name: 'Notification System', icon: '🔔', cat: 'common', desc: 'Multi-channel fan-out · dedup/throttling', link: '/en/templates/notification-system/README' },
  { name: 'Online Ticketing', icon: '🎫', cat: 'common', desc: 'Waiting room · atomic decrement · seat lock', link: '/en/templates/online-ticketing/README' },
  { name: 'AI Gateway / Relay', icon: '🚪', cat: 'ai', desc: 'Unified API · billing · load balancing', link: '/en/templates/ai-gateway/README' },
  { name: 'RAG Knowledge Base', icon: '📚', cat: 'ai', desc: 'Chunking · vector retrieval · rerank', link: '/en/templates/rag-knowledge-base/README' },
  { name: 'AI Agent / Workflow', icon: '🤹', cat: 'ai', desc: 'Action loop · tools · memory', link: '/en/templates/ai-agent-platform/README' },
  { name: 'Inference Serving', icon: '⚡', cat: 'ai', desc: 'Continuous batching · paged KV', link: '/en/templates/inference-serving/README' },
  { name: 'Vector Database', icon: '🧭', cat: 'ai', desc: 'ANN · HNSW · similarity search', link: '/en/templates/vector-database/README' },
  { name: 'Claude Code', icon: '🖥️', cat: 'agent', desc: 'Local-first coding · subagents/hooks/MCP · two-tier perms', link: '/en/templates/claude-code/README' },
  { name: 'OpenAI Codex', icon: '🛰️', cat: 'agent', desc: 'Local CLI + cloud sandbox · async PRs · two-axis safety', link: '/en/templates/codex/README' },
  { name: 'OpenClaw (the lobster)', icon: '🦞', cat: 'agent', desc: 'Self-hosted gateway · chat as UI · always-on', link: '/en/templates/openclaw/README' },
  { name: 'Hermes', icon: '🧠', cat: 'agent', desc: 'FTS5 memory · auto-distilled skills · always-on', link: '/en/templates/hermes/README' },
]

const tabsZh: Tab[] = [
  { k: 'all', t: '全部 25' },
  { k: 'common', t: '🗺️ 经典 / 通用' },
  { k: 'ai', t: '🤖 AI 原生' },
  { k: 'agent', t: '🦾 编码 / 自治 Agent' },
]

const tabsEn: Tab[] = [
  { k: 'all', t: 'All 25' },
  { k: 'common', t: '🗺️ Classic / General' },
  { k: 'ai', t: '🤖 AI-native' },
  { k: 'agent', t: '🦾 Coding / Autonomous Agents' },
]

const { lang } = useData()
const isEn = computed(() => lang.value === 'en')
const items = computed(() => (isEn.value ? itemsEn : itemsZh))
const tabs = computed(() => (isEn.value ? tabsEn : tabsZh))
const generalLabel = computed(() => (isEn.value ? 'General' : '通用'))

const filter = ref<TabKey>('all')
const filtered = computed(() => (filter.value === 'all' ? items.value : items.value.filter((i) => i.cat === filter.value)))
</script>

<template>
  <div class="ax">
    <div class="ax-tabs">
      <button
        v-for="tb in tabs"
        :key="tb.k"
        :class="['ax-tab', filter === tb.k ? 'active' : '']"
        @click="filter = tb.k"
      >{{ tb.t }}</button>
    </div>
    <div class="ax-grid">
      <a v-for="it in filtered" :key="it.link" class="ax-card" :href="withBase(it.link)">
        <div class="ax-ic">{{ it.icon }}</div>
        <div class="ax-name">{{ it.name }}</div>
        <div class="ax-desc">{{ it.desc }}</div>
        <span class="ax-tag" :class="it.cat">{{ it.cat === 'ai' ? 'AI' : it.cat === 'agent' ? 'Agent' : generalLabel }}</span>
      </a>
    </div>
  </div>
</template>

<style scoped>
.ax { max-width: 1152px; margin: 0 auto; padding: 0 24px; }
.ax-tabs { display: flex; gap: 10px; justify-content: center; flex-wrap: wrap; margin: 8px 0 28px; }
.ax-tab {
  padding: 7px 20px; border: 1px solid var(--vp-c-divider); border-radius: 20px;
  background: var(--vp-c-bg); cursor: pointer; font-size: 14px; color: var(--vp-c-text-1); transition: 0.2s;
}
.ax-tab:hover { border-color: var(--vp-c-brand-1); }
.ax-tab.active { background: var(--vp-c-brand-1); color: #fff; border-color: var(--vp-c-brand-1); }
.ax-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(220px, 1fr)); gap: 16px; }
.ax-card {
  position: relative; display: block; padding: 18px;
  border: 1px solid var(--vp-c-divider); border-radius: 12px;
  background: var(--vp-c-bg-soft); text-decoration: none; color: inherit; transition: 0.2s;
}
.ax-card:hover { border-color: var(--vp-c-brand-1); transform: translateY(-3px); box-shadow: 0 8px 24px rgba(0, 0, 0, 0.08); }
.ax-ic { font-size: 28px; margin-bottom: 8px; }
.ax-name { font-weight: 600; color: var(--vp-c-text-1); margin-bottom: 6px; }
.ax-desc { font-size: 13px; color: var(--vp-c-text-2); line-height: 1.5; }
.ax-tag {
  position: absolute; top: 14px; right: 14px; font-size: 11px;
  padding: 2px 8px; border-radius: 10px; background: var(--vp-c-default-soft); color: var(--vp-c-text-2);
}
.ax-tag.ai { background: rgba(60, 135, 114, 0.16); color: var(--vp-c-brand-1); }
.ax-tag.agent { background: rgba(200, 134, 13, 0.16); color: #b9770e; }
</style>
