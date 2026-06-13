<template>
  <div>
    <div style="margin-bottom:24px;">
      <h1 style="color:white;margin-bottom:8px;">交换记录</h1>
      <p style="color:rgba(255,255,255,0.8);">查看你完成的所有交换</p>
    </div>

    <div v-if="loading" style="text-align:center;padding:60px;color:white;">
      加载中...
    </div>

    <div v-else-if="exchanges.length === 0" class="empty-state card">
      <h2>暂无交换记录</h2>
      <p style="margin-bottom:20px;">去市场看看，寻找心仪的盲盒吧！</p>
      <router-link to="/">
        <button class="btn btn-primary">浏览盲盒市场</button>
      </router-link>
    </div>

    <div v-else class="grid" style="gap:24px;">
      <div v-for="exc in exchanges" :key="exc.id" class="card">
        <div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:16px;">
          <div class="progress-status-badge" :class="'status-' + exc.progressStatus">
            <span class="status-dot"></span>
            {{ exc.progressText }}
          </div>
          <span style="color:#999;font-size:13px;">{{ formatDate(exc.createdAt) }}</span>
        </div>

        <div style="display:flex;gap:24px;align-items:center;margin-bottom:20px;">
          <div style="flex:1;text-align:center;">
            <p style="color:#667eea;font-weight:600;margin-bottom:8px;">我的物品</p>
            <div v-if="exc.myItem" style="display:flex;gap:12px;align-items:center;justify-content:center;">
              <img :src="appendAuth(exc.myItem.image)"
                   style="width:80px;height:80px;object-fit:cover;border-radius:8px;"/>
              <div style="text-align:left;">
                <p style="font-weight:500;">{{ exc.myItem.realName }}</p>
                <p style="color:#999;font-size:12px;">
                  {{ exc.myItem.mysteryTags ? exc.myItem.mysteryTags.join(', ') : '' }}
                </p>
              </div>
            </div>
          </div>

          <div style="font-size:32px;color:#667eea;">
            &lt;&gt;
          </div>

          <div style="flex:1;text-align:center;">
            <p style="color:#f5576c;font-weight:600;margin-bottom:8px;">对方物品</p>
            <div v-if="exc.otherItem" style="display:flex;gap:12px;align-items:center;justify-content:center;">
              <img :src="appendAuth(exc.otherItem.image)"
                   style="width:80px;height:80px;object-fit:cover;border-radius:8px;"/>
              <div style="text-align:left;">
                <p style="font-weight:500;">{{ exc.otherItem.realName }}</p>
                <p style="color:#999;font-size:12px;">
                  {{ exc.otherOwnerName || exc.otherItem.ownerName }}
                </p>
              </div>
            </div>
          </div>
        </div>

        <div v-if="exc.otherContact"
             style="margin-bottom:20px;padding:16px;background:#eef2ff;border-radius:8px;">
          <p style="font-weight:500;margin-bottom:8px;">对方联系方式</p>
          <p style="color:#667eea;font-size:18px;font-weight:600;">{{ exc.otherContact }}</p>
          <p style="color:#999;font-size:12px;margin-top:8px;">请主动联系对方完成物品交换</p>
        </div>

        <div class="progress-stepper">
          <div v-for="(step, idx) in progressSteps" :key="step.status" class="step-item">
            <div class="step-node"
                 :class="{ 'step-done': isStepDone(exc.progressStatus, step.status),
                            'step-current': isStepCurrent(exc.progressStatus, step.status) }">
              <span v-if="isStepDone(exc.progressStatus, step.status)" class="step-check">✓</span>
              <span v-else class="step-num">{{ idx + 1 }}</span>
            </div>
            <div class="step-label" :class="{ 'label-done': isStepDone(exc.progressStatus, step.status) }">
              {{ step.text }}
            </div>
            <div v-if="idx < progressSteps.length - 1" class="step-line"
                 :class="{ 'line-done': isStepDone(exc.progressStatus, step.status) }"></div>
          </div>
        </div>

        <div v-if="getNextStep(exc.progressStatus)" style="margin-top:20px;text-align:center;">
          <button class="btn btn-primary"
                  style="width:100%;padding:14px;font-size:15px;"
                  :disabled="advancingId === exc.id"
                  @click="handleAdvanceProgress(exc)">
            <span v-if="advancingId === exc.id">处理中...</span>
            <span v-else>推进为：{{ getNextStep(exc.progressStatus).text }}</span>
          </button>
          <p style="color:#999;font-size:12px;margin-top:8px;">
            双方均可推进状态，请在实际完成该步骤后点击
          </p>
        </div>

        <div v-if="!getNextStep(exc.progressStatus)"
             style="margin-top:20px;padding:16px;background:#e8f5e9;border-radius:8px;text-align:center;">
          <p style="color:#2e7d32;font-weight:600;">🎉 交换已全部完成！</p>
        </div>

        <div style="margin-top:24px;">
          <div style="display:flex;align-items:center;margin-bottom:16px;cursor:pointer;"
               @click="toggleTimeline(exc.id)">
            <h4 style="margin:0;color:#333;">进度时间线</h4>
            <span style="margin-left:8px;color:#667eea;font-size:12px;">
              {{ expandedTimeline === exc.id ? '收起 ▲' : '展开 ▼' }}
            </span>
          </div>

          <div v-show="expandedTimeline === exc.id" class="timeline">
            <div v-for="(node, nIdx) in exc.timeline" :key="node.id" class="timeline-item">
              <div class="timeline-marker">
                <div class="timeline-dot" :class="'dot-' + node.status"></div>
                <div v-if="nIdx < exc.timeline.length - 1" class="timeline-connector"></div>
              </div>
              <div class="timeline-content">
                <div class="timeline-header">
                  <span class="timeline-status" :class="'status-' + node.status">{{ node.statusText }}</span>
                  <span class="timeline-time">{{ formatDate(node.createdAt) }}</span>
                </div>
                <div class="timeline-operator">
                  操作人：{{ isMyOperation(node, exc) ? '我' : node.operatorName }}
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'
import { getMyExchanges, advanceExchangeProgress, appendAuth } from '../api/index.js'
import { userStore } from '../store/user.js'

const exchanges = ref([])
const loading = ref(true)
const advancingId = ref(null)
const expandedTimeline = ref(null)

const progressSteps = [
  { status: 'created', text: '交换成功', order: 0 },
  { status: 'contacted', text: '已联系', order: 1 },
  { status: 'appointed', text: '已约定', order: 2 },
  { status: 'shipped', text: '已寄出', order: 3 },
  { status: 'received', text: '已收到', order: 4 },
  { status: 'finished', text: '已完成', order: 5 }
]

const statusOrderMap = {}
progressSteps.forEach(s => { statusOrderMap[s.status] = s.order })

function formatDate(dateStr) {
  const date = new Date(dateStr)
  const y = date.getFullYear()
  const m = String(date.getMonth() + 1).padStart(2, '0')
  const d = String(date.getDate()).padStart(2, '0')
  const h = String(date.getHours()).padStart(2, '0')
  const min = String(date.getMinutes()).padStart(2, '0')
  return y + '-' + m + '-' + d + ' ' + h + ':' + min
}

function isStepDone(currentStatus, stepStatus) {
  return statusOrderMap[currentStatus] > statusOrderMap[stepStatus]
}

function isStepCurrent(currentStatus, stepStatus) {
  return currentStatus === stepStatus
}

function getNextStep(currentStatus) {
  const idx = progressSteps.findIndex(s => s.status === currentStatus)
  if (idx === -1 || idx >= progressSteps.length - 1) return null
  return progressSteps[idx + 1]
}

function toggleTimeline(id) {
  expandedTimeline.value = expandedTimeline.value === id ? null : id
}

function isMyOperation(node, exc) {
  return node.operatorId === userStore.user.id
}

async function handleAdvanceProgress(exc) {
  const nextStep = getNextStep(exc.progressStatus)
  if (!nextStep) return
  if (!confirm(`确定要将状态推进为「${nextStep.text}」吗？请确保该步骤已实际完成。`)) {
    return
  }
  advancingId.value = exc.id
  try {
    const result = await advanceExchangeProgress(exc.id, userStore.user.id, userStore.user.name)
    const targetIdx = exchanges.value.findIndex(e => e.id === exc.id)
    if (targetIdx !== -1) {
      exchanges.value[targetIdx].progressStatus = result.progressStatus
      exchanges.value[targetIdx].progressText = result.progressText
      exchanges.value[targetIdx].timeline = result.timeline
    }
    alert('状态已更新为「' + nextStep.text + '」')
  } catch (e) {
    alert('操作失败：' + (e.response && e.response.data ? e.response.data.error : e.message))
  } finally {
    advancingId.value = null
  }
}

async function loadExchanges() {
  loading.value = true
  try {
    exchanges.value = await getMyExchanges(userStore.user.id)
  } catch (e) {
    alert('加载失败')
  } finally {
    loading.value = false
  }
}

onMounted(loadExchanges)
</script>

<style scoped>
.progress-status-badge {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 6px 14px;
  border-radius: 20px;
  font-size: 13px;
  font-weight: 600;
}

.status-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: currentColor;
}

.status-created { background: #e3f2fd; color: #1976d2; }
.status-contacted { background: #fff3e0; color: #f57c00; }
.status-appointed { background: #fce4ec; color: #c2185b; }
.status-shipped { background: #e8eaf6; color: #303f9f; }
.status-received { background: #e0f2f1; color: #00796b; }
.status-finished { background: #e8f5e9; color: #388e3c; }

.progress-stepper {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  padding: 20px 0 0;
}

.step-item {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  position: relative;
}

.step-node {
  width: 36px;
  height: 36px;
  border-radius: 50%;
  background: #e0e0e0;
  color: #999;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 14px;
  font-weight: 600;
  border: 3px solid #f5f5f5;
  transition: all 0.3s;
  z-index: 2;
}

.step-node.step-current {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  box-shadow: 0 0 0 4px rgba(102, 126, 234, 0.2);
}

.step-node.step-done {
  background: linear-gradient(135deg, #4caf50 0%, #66bb6a 100%);
  color: white;
}

.step-check {
  font-size: 16px;
  font-weight: bold;
}

.step-label {
  margin-top: 8px;
  font-size: 12px;
  color: #999;
  font-weight: 500;
  text-align: center;
  white-space: nowrap;
}

.label-done {
  color: #4caf50;
}

.step-line {
  position: absolute;
  top: 18px;
  left: 50%;
  width: 100%;
  height: 3px;
  background: #e0e0e0;
  z-index: 1;
}

.line-done {
  background: linear-gradient(90deg, #4caf50, #66bb6a);
}

.timeline {
  padding: 8px 0 0 4px;
}

.timeline-item {
  display: flex;
  gap: 16px;
  position: relative;
}

.timeline-marker {
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 20px;
}

.timeline-dot {
  width: 14px;
  height: 14px;
  border-radius: 50%;
  border: 3px solid white;
  box-shadow: 0 0 0 2px #e0e0e0;
  flex-shrink: 0;
  margin-top: 4px;
}

.dot-created { background: #1976d2; box-shadow: 0 0 0 2px #bbdefb; }
.dot-contacted { background: #f57c00; box-shadow: 0 0 0 2px #ffe0b2; }
.dot-appointed { background: #c2185b; box-shadow: 0 0 0 2px #f8bbd0; }
.dot-shipped { background: #303f9f; box-shadow: 0 0 0 2px #c5cae9; }
.dot-received { background: #00796b; box-shadow: 0 0 0 2px #b2dfdb; }
.dot-finished { background: #388e3c; box-shadow: 0 0 0 2px #c8e6c9; }

.timeline-connector {
  width: 2px;
  flex: 1;
  min-height: 24px;
  background: #e0e0e0;
  margin: 4px 0;
}

.timeline-content {
  flex: 1;
  padding-bottom: 20px;
}

.timeline-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 6px;
}

.timeline-status {
  font-size: 14px;
  font-weight: 600;
  padding: 3px 10px;
  border-radius: 12px;
}

.timeline-status.status-created { background: #e3f2fd; color: #1976d2; }
.timeline-status.status-contacted { background: #fff3e0; color: #f57c00; }
.timeline-status.status-appointed { background: #fce4ec; color: #c2185b; }
.timeline-status.status-shipped { background: #e8eaf6; color: #303f9f; }
.timeline-status.status-received { background: #e0f2f1; color: #00796b; }
.timeline-status.status-finished { background: #e8f5e9; color: #388e3c; }

.timeline-time {
  font-size: 12px;
  color: #999;
}

.timeline-operator {
  font-size: 13px;
  color: #666;
}
</style>
