<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'

const API_URL = 'https://script.google.com/macros/s/AKfycbygDzpgv3QonFdi-Gj834SuxfMdTE1sLDJnXplTjLb-FdY0peRxFvjikDeVSsWsJ7Yw/exec'

const creators = [
  'นายวรดร ส่องแสงรักษ์',
  'น.ส.กุลณัฐดา สมมา',
  'นายธนวิชญ์ จันทร์โชติ',
  'นายชุติเดช พนิชธนาดล'
]

const logs = ref([])
const loading = ref(false)
const statusText = ref('')
const lastFetchedTime = ref('-')
let pollTimer = null
let isFetching = false

// แปลงเวลา ISO (.000Z) เป็นวันที่และเวลาไทย
function formatThaiDate(val) {
  if (!val || val === '-') return '-'
  const d = new Date(val)
  if (Number.isNaN(d.getTime())) return String(val)
  return d.toLocaleString('th-TH', {
    timeZone: 'Asia/Bangkok',
    year: 'numeric',
    month: '2-digit',
    day: '2-digit',
    hour: '2-digit',
    minute: '2-digit',
    second: '2-digit'
  })
}

// เวลาไทยแบบ HH:mm:ss สำหรับแสดงตอนที่ดึงข้อมูลล่าสุด
function formatThaiTimeOnly(dateObj = new Date()) {
  return dateObj.toLocaleTimeString('th-TH', {
    timeZone: 'Asia/Bangkok',
    hour: '2-digit',
    minute: '2-digit',
    second: '2-digit'
  })
}

// คำนวณ Grade จากค่าระยะทาง (cm)
function getGrade(dist) {
  const d = Number(dist)
  if (Number.isNaN(d)) return '-'
  if (d <= 30) return 'A'
  if (d <= 70) return 'B'
  if (d <= 120) return 'C'
  return 'D'
}

function getGradeLabel(grade) {
  const map = {
    A: 'ใกล้มาก',
    B: 'ปานกลาง',
    C: 'ไกล',
    D: 'ไกลมาก'
  }
  return map[grade] || '-'
}

const latest = computed(() => {
  if (!logs.value.length) {
    return { Date: '-', Distance: '-', grade: '-' }
  }
  const last = logs.value[logs.value.length - 1]
  return {
    ...last,
    grade: getGrade(last.Distance)
  }
})

// มุมเข็มเกจครึ่งวงกลม (-90deg ซ้ายสุด ถึง +90deg ขวาสุด)
const gaugeAngle = computed(() => {
  const d = Number(latest.value.Distance)
  if (Number.isNaN(d)) return -90
  const maxGauge = 150
  const ratio = Math.max(0, Math.min(d / maxGauge, 1))
  return -90 + ratio * 180
})

// จุดกราฟเส้นด้านบน (15 รายการล่าสุด) ปรับสเกลแกน Y ให้เห็นความเปลี่ยนแปลงชัดขึ้น
const chartPoints = computed(() => {
  const recent = logs.value
    .slice(-15)
    .map((item, idx) => ({
      index: idx,
      date: item.Date,
      distance: Number(item.Distance),
      grade: getGrade(item.Distance)
    }))
    .filter(p => !Number.isNaN(p.distance))

  if (!recent.length) return []

  const rawMin = Math.min(...recent.map(p => p.distance))
  const rawMax = Math.max(...recent.map(p => p.distance))
  const minVal = Math.max(0, Math.floor((rawMin - 8) / 5) * 5)
  const maxVal = Math.max(minVal + 20, Math.ceil((rawMax + 10) / 5) * 5)
  const span = maxVal - minVal || 1

  const width = 680
  const height = 205
  const padX = 48
  const padY = 26
  const plotW = width - padX * 2
  const plotH = height - padY * 2

  return recent.map((p, i) => {
    const x =
      recent.length === 1
        ? width / 2
        : padX + (i / (recent.length - 1)) * plotW
    const y = padY + plotH - ((p.distance - minVal) / span) * plotH
    return { ...p, x, y, minVal, maxVal, baselineY: padY + plotH }
  })
})

const polylinePoints = computed(() =>
  chartPoints.value.map(p => `${p.x},${p.y}`).join(' ')
)

const areaPoints = computed(() => {
  if (!chartPoints.value.length) return ''
  const first = chartPoints.value[0]
  const last = chartPoints.value[chartPoints.value.length - 1]
  return `${first.x},${first.baselineY} ${polylinePoints.value} ${last.x},${first.baselineY}`
})

async function loadLogs(manual = false) {
  if (isFetching) return
  isFetching = true
  if (manual) loading.value = true

  try {
    const res = await fetch(API_URL)
    const data = await res.json()
    if (data.status === 'ok') {
      logs.value = data.data
      lastFetchedTime.value = formatThaiTimeOnly(new Date())
      statusText.value = ''
    }
  } catch (err) {
    statusText.value = 'ดึงข้อมูลไม่สำเร็จ: ' + err.message
  } finally {
    isFetching = false
    if (manual) loading.value = false
  }
}

onMounted(() => {
  loadLogs(true)
  pollTimer = setInterval(() => loadLogs(false), 1000)
})

onUnmounted(() => {
  if (pollTimer) clearInterval(pollTimer)
})
</script>

<template>
  <div class="page-shell">
    <!-- แถบส่วนหัวโทนสีส้ม -->
    <header class="top-banner">
      <div class="banner-inner">
        <div class="college-brand">
          <div class="logo-ring">
            <img
              src="https://www.sbtvc.ac.th/wp-content/uploads/2024/08/logo_sbtvc.png"
              alt="โลโก้วิทยาลัยอาชีวศึกษาเทคโนโลยีฐานวิทยาศาสตร์ (ชลบุรี)"
              class="college-logo"
            />
          </div>
          <div class="brand-text">
            <span class="brand-kicker">SCIENCE-BASED TECHNOLOGICAL VOCATIONAL COLLEGE</span>
            <h1 class="college-title">
              วิทยาลัยอาชีวศึกษาเทคโนโลยีฐานวิทยาศาสตร์ (ชลบุรี)
            </h1>
          </div>
        </div>

        <button type="button" class="btn-refresh" @click="loadLogs(true)" :disabled="loading">
          <span class="refresh-dot"></span>
          {{ loading ? 'กำลังโหลด...' : 'รีเฟรชข้อมูล (1s)' }}
        </button>
      </div>
    </header>

    <div class="wrapper">
      <p v-if="statusText" class="status-msg">{{ statusText }}</p>

      <!-- กล่องคณะผู้จัดทำ วางก่อนกล่องกราฟ -->
      <section class="creators-panel">
        <div class="panel-top">
          <div class="title-group">
            <span class="dept-tag">PROJECT TEAM</span>
            <h2>คณะผู้จัดทำ</h2>
          </div>
          <span class="dept-badge">
            <span class="dept-badge-dot"></span>
            แผนกวิชาเทคโนโลยีสารสนเทศ
          </span>
        </div>

        <div class="creators-grid">
          <div v-for="(person, i) in creators" :key="i" class="creator-card">
            <span class="creator-num">{{ i + 1 }}</span>
            <span class="creator-name">{{ person }}</span>
          </div>
        </div>
      </section>

      <!-- กล่องหลัก Sonar & Fan (มีกล่องกราฟอยู่ด้านใน) -->
      <section class="main-panel">
        <div class="panel-top">
          <div class="title-group">
            <span class="dept-tag">COMPUTER DEPARTMENT · SONAR × FAN</span>
            <h2>Sonar &amp; Fan Monitor</h2>
          </div>
          <div class="update-pill">
            ข้อมูลล่าสุด: <strong>{{ formatThaiDate(latest.Date) }}</strong>
          </div>
        </div>

        <!-- 1. กล่องกราฟเส้น -->
        <div class="block-box">
          <div class="block-header">
            <span class="block-title">แนวโน้มระยะทาง (cm)</span>
            <span class="block-hint">แสดง {{ chartPoints.length }} รายการล่าสุด</span>
          </div>

          <div v-if="chartPoints.length" class="chart-wrap">
            <svg viewBox="0 0 680 205" class="chart-svg">
              <defs>
                <linearGradient id="orangeArea" x1="0" y1="0" x2="0" y2="1">
                  <stop offset="0%" stop-color="#f97316" stop-opacity="0.30" />
                  <stop offset="100%" stop-color="#f97316" stop-opacity="0.02" />
                </linearGradient>
              </defs>

              <!-- เส้นกริดแนวนอน -->
              <line x1="48" y1="26" x2="632" y2="26" stroke="#ffedd5" stroke-dasharray="4 4" stroke-width="1" />
              <line x1="48" y1="103" x2="632" y2="103" stroke="#ffedd5" stroke-dasharray="4 4" stroke-width="1" />
              <line x1="48" y1="179" x2="632" y2="179" stroke="#fdba74" stroke-width="1.5" />
              <line x1="48" y1="26" x2="48" y2="179" stroke="#fdba74" stroke-width="1.5" />

              <text x="40" y="30" text-anchor="end" font-size="11" font-weight="600" fill="#9a3412">
                {{ chartPoints[0].maxVal }}
              </text>
              <text x="40" y="107" text-anchor="end" font-size="11" fill="#9a3412">
                {{ Math.round((chartPoints[0].minVal + chartPoints[0].maxVal) / 2) }}
              </text>
              <text x="40" y="179" text-anchor="end" font-size="11" font-weight="600" fill="#9a3412">
                {{ chartPoints[0].minVal }}
              </text>

              <polygon :points="areaPoints" fill="url(#orangeArea)" />
              <polyline
                fill="none"
                stroke="#ea580c"
                stroke-width="2.75"
                stroke-linecap="round"
                stroke-linejoin="round"
                :points="polylinePoints"
              />

              <g v-for="pt in chartPoints" :key="pt.index">
                <circle :cx="pt.x" :cy="pt.y" r="4.8" fill="#ea580c" stroke="#ffffff" stroke-width="1.8" />
                <text
                  :x="pt.x"
                  :y="pt.y - 10"
                  text-anchor="middle"
                  font-size="10.5"
                  font-weight="700"
                  fill="#431407"
                >
                  {{ pt.distance }}
                </text>
              </g>
            </svg>
          </div>
          <p v-else class="empty-text">ยังไม่มีข้อมูลกราฟ</p>

          <!-- แถบบอกเวลาใต้กราฟ ว่าดึงตอนกี่โมง -->
          <div class="chart-footer">
            <span>ดึงข้อมูลล่าสุดเมื่อ: <strong>{{ lastFetchedTime }} น.</strong> (ทุก 1 วินาที)</span>
            <span>เวลาข้อมูลในชีต: <strong>{{ formatThaiDate(latest.Date) }}</strong></span>
          </div>
        </div>

        <!-- 2. เกจครึ่งวงกลม + การ์ด Distance & Grade -->
        <div class="gauge-section">
          <div class="gauge-box">
            <svg viewBox="0 0 280 165" class="gauge-svg">
              <defs>
                <linearGradient id="gaugeGrad" x1="0%" y1="0%" x2="100%" y2="0%">
                  <stop offset="0%" stop-color="#f97316" />
                  <stop offset="100%" stop-color="#ea580c" />
                </linearGradient>
              </defs>

              <path
                d="M 35 140 A 105 105 0 0 1 245 140"
                fill="none"
                stroke="#ffedd5"
                stroke-width="20"
                stroke-linecap="round"
              />
              <path
                d="M 35 140 A 105 105 0 0 1 245 140"
                fill="none"
                stroke="url(#gaugeGrad)"
                stroke-width="20"
                stroke-linecap="round"
                stroke-dasharray="330"
                :stroke-dashoffset="330 - (330 * (gaugeAngle + 90)) / 180"
              />
              <g :transform="`translate(140, 140) rotate(${gaugeAngle})`">
                <line x1="0" y1="0" x2="0" y2="-82" stroke="#431407" stroke-width="4" stroke-linecap="round" />
              </g>
              <circle cx="140" cy="140" r="10" fill="#ea580c" stroke="#ffffff" stroke-width="2.5" />
            </svg>
            <div class="gauge-caption">
              <span>ระดับระยะตรวจจับ</span>
              <strong>Grade {{ latest.grade }} · {{ getGradeLabel(latest.grade) }}</strong>
            </div>
          </div>

          <div class="metrics-side">
            <div class="metric-card">
              <span class="m-label">ระยะทางล่าสุด</span>
              <strong class="m-value">{{ latest.Distance }} <span>cm</span></strong>
            </div>

            <div class="metric-card">
              <span class="m-label">Grade</span>
              <div class="grade-flex">
                <span class="grade-chip" :class="'grade-' + latest.grade.toLowerCase()">
                  {{ latest.grade }}
                </span>
                <div>
                  <div class="grade-desc">{{ getGradeLabel(latest.grade) }}</div>
                  <div class="grade-rule">A ≤30 · B ≤70 · C ≤120 · D &gt;120 cm</div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </section>
    </div>
  </div>
</template>

<style scoped>
.page-shell {
  min-height: 100vh;
  background-color: #fff7ed;
  background-image: radial-gradient(#fdba74 0.85px, transparent 0.85px);
  background-size: 22px 22px;
  padding-bottom: 3rem;
  font-family: 'Segoe UI', system-ui, -apple-system, sans-serif;
  color: #1c1917;
}

/* ส่วนหัวโทนสีส้มอบอุ่นเข้ากับทั้งหน้า */
.top-banner {
  width: 100%;
  background: linear-gradient(90deg, #ffffff 0%, #fff7ed 60%, #ffedd5 100%);
  border-bottom: 3px solid #f97316;
  padding: 0.95rem 1.5rem;
  margin-bottom: 1.75rem;
  box-shadow: 0 6px 24px rgba(234, 88, 12, 0.08);
}
.banner-inner {
  max-width: 960px;
  margin: 0 auto;
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: 1rem;
}
.college-brand {
  display: flex;
  align-items: center;
  gap: 1.1rem;
  flex-wrap: wrap;
}
.logo-ring {
  width: 72px;
  height: 72px;
  border-radius: 999px;
  background: #ffffff;
  border: 2px solid #fdba74;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 4px 12px rgba(234, 88, 12, 0.12);
  flex-shrink: 0;
}
.college-logo {
  height: 60px;
  width: auto;
  object-fit: contain;
  display: block;
}
.brand-kicker {
  display: block;
  font-size: 0.68rem;
  font-weight: 700;
  letter-spacing: 0.08em;
  color: #c2410c;
  margin-bottom: 0.2rem;
}
.college-title {
  margin: 0;
  font-size: clamp(1.15rem, 2.3vw, 1.65rem);
  font-weight: 700;
  color: #431407;
  line-height: 1.25;
}

.wrapper {
  max-width: 920px;
  margin: 0 auto;
  padding: 0 1.25rem;
}
.dept-tag {
  display: inline-block;
  font-size: 0.68rem;
  font-weight: 700;
  letter-spacing: 0.07em;
  color: #c2410c;
  background: #ffedd5;
  padding: 0.22rem 0.65rem;
  border-radius: 999px;
  margin-bottom: 0.35rem;
}
h2 {
  margin: 0;
  font-size: 1.22rem;
  font-weight: 700;
  color: #1c1917;
}
.main-panel,
.creators-panel {
  background: #ffffff;
  border: 1px solid #fed7aa;
  border-top: 4px solid #f97316;
  border-radius: 18px;
  padding: 1.45rem;
  box-shadow: 0 8px 24px rgba(234, 88, 12, 0.07);
  margin-bottom: 1.35rem;
}
.panel-top {
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: 0.65rem;
  margin-bottom: 1.1rem;
}
.update-pill {
  font-size: 0.84rem;
  color: #7c2d12;
  background: #fff7ed;
  border: 1px solid #fed7aa;
  padding: 0.38rem 0.75rem;
  border-radius: 999px;
}

/* ป้ายแผนกวิชาเด่นๆ */
.dept-badge {
  display: inline-flex;
  align-items: center;
  gap: 0.45rem;
  font-size: 0.92rem;
  font-weight: 700;
  color: #9a3412;
  background: linear-gradient(135deg, #ffedd5 0%, #fed7aa 100%);
  border: 1.5px solid #fb923c;
  padding: 0.42rem 0.95rem;
  border-radius: 999px;
  box-shadow: 0 3px 10px rgba(234, 88, 12, 0.12);
}
.dept-badge-dot {
  width: 8px;
  height: 8px;
  border-radius: 999px;
  background: #ea580c;
  flex-shrink: 0;
}

.block-box {
  border: 1px solid #ffedd5;
  background: linear-gradient(180deg, #fffaf5 0%, #ffffff 100%);
  border-radius: 14px;
  padding: 1rem 1.1rem;
  margin-bottom: 1.35rem;
}
.block-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: 0.5rem;
  margin-bottom: 0.65rem;
}
.block-title {
  font-size: 0.92rem;
  font-weight: 700;
  color: #9a3412;
}
.block-hint {
  font-size: 0.78rem;
  color: #9a3412;
  background: #ffedd5;
  padding: 0.18rem 0.55rem;
  border-radius: 999px;
}
.chart-wrap {
  width: 100%;
  overflow-x: auto;
}
.chart-svg {
  width: 100%;
  height: auto;
  display: block;
}
.chart-footer {
  margin-top: 0.7rem;
  padding-top: 0.65rem;
  border-top: 1px dashed #fed7aa;
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: 0.5rem;
  font-size: 0.82rem;
  color: #78716c;
}
.chart-footer strong {
  color: #9a3412;
}
.gauge-section {
  display: grid;
  grid-template-columns: 1.15fr 1fr;
  gap: 1.25rem;
  align-items: stretch;
}
@media (max-width: 720px) {
  .gauge-section {
    grid-template-columns: 1fr;
  }
}
.gauge-box {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  background: linear-gradient(180deg, #fffaf5 0%, #ffffff 100%);
  border: 1px solid #ffedd5;
  border-radius: 14px;
  padding: 1rem;
}
.gauge-svg {
  width: 100%;
  max-width: 265px;
  height: auto;
}
.gauge-caption {
  margin-top: 0.35rem;
  text-align: center;
  display: flex;
  flex-direction: column;
  gap: 0.15rem;
}
.gauge-caption span {
  font-size: 0.78rem;
  color: #78716c;
}
.gauge-caption strong {
  font-size: 0.98rem;
  color: #c2410c;
}
.metrics-side {
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  gap: 0.95rem;
}
.metric-card {
  flex: 1;
  display: flex;
  flex-direction: column;
  justify-content: center;
  border: 1px solid #fed7aa;
  border-radius: 14px;
  padding: 1rem 1.2rem;
  background: linear-gradient(180deg, #fffaf5 0%, #ffffff 100%);
}
.m-label {
  display: block;
  font-size: 0.82rem;
  font-weight: 600;
  color: #78716c;
  margin-bottom: 0.35rem;
}
.m-value {
  font-size: 1.85rem;
  font-weight: 800;
  color: #c2410c;
  line-height: 1.1;
}
.m-value span {
  font-size: 1rem;
  font-weight: 600;
  color: #9a3412;
}
.grade-flex {
  display: flex;
  align-items: center;
  gap: 0.8rem;
}
.grade-desc {
  font-size: 1rem;
  font-weight: 700;
  color: #1c1917;
}
.grade-rule {
  font-size: 0.76rem;
  color: #78716c;
  margin-top: 0.15rem;
}
.grade-chip {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 42px;
  height: 42px;
  border-radius: 11px;
  font-weight: 800;
  font-size: 1.1rem;
  background: #ffedd5;
  color: #9a3412;
  flex-shrink: 0;
}
.grade-a {
  background: #dcfce7;
  color: #15803d;
}
.grade-b {
  background: #ffedd5;
  color: #c2410c;
}
.grade-c {
  background: #fef9c3;
  color: #a16207;
}
.grade-d {
  background: #fee2e2;
  color: #b91c1c;
}

.btn-refresh {
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.6rem 1.05rem;
  border-radius: 10px;
  font-weight: 700;
  font-size: 0.9rem;
  cursor: pointer;
  border: 1.5px solid #fb923c;
  background: #ffffff;
  color: #c2410c;
  box-shadow: 0 2px 8px rgba(234, 88, 12, 0.08);
  transition: all 0.15s ease;
}
.btn-refresh:hover:not(:disabled) {
  background: #ea580c;
  border-color: #ea580c;
  color: #ffffff;
}
.btn-refresh:disabled {
  opacity: 0.65;
  cursor: not-allowed;
}
.refresh-dot {
  width: 8px;
  height: 8px;
  border-radius: 999px;
  background: #22c55e;
  animation: pulseDot 1s infinite;
}
@keyframes pulseDot {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.35; }
}

.status-msg {
  margin: 0 0 1rem;
  color: #b91c1c;
  font-weight: 600;
}
.empty-text {
  text-align: center;
  color: #78716c;
  padding: 1.1rem 0;
}

/* กล่องการ์ดแสดงรายชื่อผู้จัดทำก่อนกล่องกราฟ */
.creators-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(240px, 1fr));
  gap: 0.8rem;
}
.creator-card {
  display: flex;
  align-items: center;
  gap: 0.8rem;
  padding: 0.8rem 1rem;
  background: linear-gradient(180deg, #fffaf5 0%, #ffffff 100%);
  border: 1px solid #fed7aa;
  border-radius: 12px;
  transition: transform 0.15s ease, box-shadow 0.15s ease;
}
.creator-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 14px rgba(234, 88, 12, 0.09);
}
.creator-num {
  width: 28px;
  height: 28px;
  border-radius: 999px;
  background: #ea580c;
  color: #ffffff;
  font-size: 0.85rem;
  font-weight: 700;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}
.creator-name {
  font-weight: 600;
  font-size: 0.95rem;
  color: #1c1917;
}
</style>