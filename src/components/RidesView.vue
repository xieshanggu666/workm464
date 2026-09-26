<script setup>
import { ref, computed } from 'vue'
import { useParkStore } from '@/store/park'

const store = useParkStore()
const typeFilter = ref('all')
const zoneFilter = ref(0)
const buildOpen = ref(false)
const build = ref({ type: '过山车', zone_id: 1, name: '' })

const types = ['过山车', '旋转木马', '摩天轮', '跳楼机', '碰碰车', '海盗船', '水上漂流', '云霄飞车']
const typeIcon = t => ({ '过山车':'🎢', '旋转木马':'🎠', '摩天轮':'🎡', '跳楼机':'🪂', '碰碰车':'🚗', '海盗船':'⛵', '水上漂流':'💦', '云霄飞车':'🚀' }[t])

const list = computed(() => store.rides.filter(r =>
  (typeFilter.value === 'all' || r.type === typeFilter.value) &&
  (!zoneFilter.value || r.zone_id === zoneFilter.value)
))

function update(r, payload) { store.updateRide(r.id, payload) }
function submit() {
  store.buildRide({ ...build.value, name: build.value.name || `${build.value.type}·新馆` })
  buildOpen.value = false
  build.value.name = ''
}

// ---------------- 维修工单 ----------------
const ACTIVE = ['queued', 'processing']
const stats = computed(() => store.repairStats)
// 进行中 + 当天完工的工单
const orders = computed(() => store.repairs.filter(o =>
  ACTIVE.includes(o.status) || (o.status === 'done' && o.done_day >= store.clock.day)
))
// 设施 → 进行中工单（表格联动）
const orderByRide = computed(() => {
  const m = {}
  for (const o of store.repairs) if (ACTIVE.includes(o.status)) m[o.ride_id] = o
  return m
})
// 在岗维修员（标注是否手上有活）
const repairStaff = computed(() => store.staff
  .filter(s => s.active && s.role === '维修')
  .map(s => ({ ...s, busy: !!store.repairs.find(o => o.status === 'processing' && o.assignee_id === s.id) })))

const assignSel = ref({})   // orderId -> staffId
const staffText = s => `${s.name} · Lv.${s.skill}${s.busy ? ' · 工单中' : ''}`
const statusMeta = st => ({
  queued: { label: '排队中', cls: 'st-queued' },
  processing: { label: '维修中', cls: 'st-processing' },
  done: { label: '已完工', cls: 'st-done' },
  cancelled: { label: '已取消', cls: 'st-cancel' }
}[st] || { label: st, cls: '' })

function assign(o) {
  const sid = assignSel.value[o.id]
  if (!sid) return
  store.assignRepair(o.id, sid)
  assignSel.value[o.id] = undefined
}

// 工单时间线
const detail = ref(null)
const detailLogs = ref([])
const ACTION_LABEL = {
  create: '生成工单', assign: '指派接单', auto_assign: '自动派单', reassign: '转派',
  unassign: '退回排队', done: '完工结算', cancel: '工单取消'
}
async function openDetail(o) {
  const r = await store.repairDetail(o.id)
  if (r?.order) { detail.value = r.order; detailLogs.value = r.logs || [] }
}
function closeDetail() { detail.value = null }
</script>

<template>
  <div class="rides">
    <div class="bar">
      <div class="filters">
        <select v-model="typeFilter"><option value="all">全部类型</option><option v-for="t in types" :key="t" :value="t">{{ t }}</option></select>
        <select v-model="zoneFilter"><option :value="0">全部区域</option><option v-for="z in store.zones" :key="z.id" :value="z.id">{{ z.name }}</option></select>
      </div>
      <button class="primary" @click="buildOpen = true">＋ 新建设施</button>
    </div>

    <!-- 维修工单 -->
    <div class="card shop">
      <h3>🛠️ 维修工单
        <span class="chip" v-if="stats.downRides">停运 {{ stats.downRides }} 台</span>
        <span class="chip warn" v-if="stats.queued">排队 {{ stats.queued }}</span>
        <span class="chip" v-if="stats.processing">维修中 {{ stats.processing }}</span>
        <span class="chip ok" v-if="stats.doneToday">今日完工 {{ stats.doneToday }}</span>
        <span class="chip muted-chip">累计维修费 ¥{{ stats.totalCost.toLocaleString() }}</span>
      </h3>
      <div class="olist" v-if="orders.length">
        <div class="oitem" v-for="o in orders" :key="o.id" :class="o.status">
          <div class="o-head">
            <span class="big-ic">{{ o.status === 'done' ? '✅' : o.status === 'processing' ? '🔧' : '⏳' }}</span>
            <div class="o-title">
              <b>{{ o.ride_name }}</b>
              <em class="muted">{{ o.code }} · 第{{ o.created_day }}天报修 · 报修时健康度 {{ Math.round(o.health_from) }}</em>
            </div>
            <span class="badge" :class="statusMeta(o.status).cls">{{ statusMeta(o.status).label }}</span>
            <button class="ghost timeline-btn" @click="openDetail(o)">时间线</button>
          </div>

          <!-- 排队中：手动指派 / 等待自动派单 -->
          <div class="o-body" v-if="o.status === 'queued'">
            <span class="muted hint">已排队 {{ o.queued_hours }}h{{ o.queued_hours < 2 ? '，2h 未指派将自动派单' : '，等待空闲维修员' }} · 预计维修费 ¥{{ o.est_cost.toLocaleString() }}</span>
            <div class="o-ops">
              <select v-model.number="assignSel[o.id]">
                <option :value="undefined" disabled>选择维修员…</option>
                <option v-for="s in repairStaff" :key="s.id" :value="s.id" :disabled="s.busy">{{ staffText(s) }}</option>
              </select>
              <button class="succ" :disabled="!assignSel[o.id]" @click="assign(o)">指派接单</button>
            </div>
          </div>

          <!-- 维修中：进度 / 转派 / 退回排队 -->
          <div class="o-body" v-else-if="o.status === 'processing'">
            <div class="o-info">
              <span>👷 {{ o.assignee_name }}（Lv.{{ o.assignee_skill }}）</span>
              <span class="muted">预计还需 {{ o.eta_hours }}h · 维修费 ¥{{ o.est_cost.toLocaleString() }}</span>
            </div>
            <div class="pbar"><i :style="{ width: o.progress + '%' }"></i><b>{{ Math.round(o.progress) }}%</b></div>
            <div class="o-ops">
              <select v-model.number="assignSel[o.id]">
                <option :value="undefined" disabled>转派给…</option>
                <option v-for="s in repairStaff.filter(x => x.id !== o.assignee_id)" :key="s.id" :value="s.id" :disabled="s.busy">{{ staffText(s) }}</option>
              </select>
              <button class="ghost" :disabled="!assignSel[o.id]" @click="assign(o)">转派</button>
              <button class="ghost" @click="store.unassignRepair(o.id)">退回排队</button>
            </div>
          </div>

          <!-- 已完工 -->
          <div class="o-body done-info" v-else-if="o.status === 'done'">
            <span>✅ 第{{ o.done_day }}天完工 · 健康度恢复 100，设施已恢复运营、预约时段重新开放 · 维修费 <b class="money neg">¥{{ o.cost.toLocaleString() }}</b> 已入账</span>
          </div>
        </div>
      </div>
      <div class="muted empty" v-else>暂无维修工单，设施运转良好 🎉（可在下表对设施手动报修）</div>
    </div>

    <div class="table card">
      <div class="thead">
        <span>设施</span><span>类型</span><span>区域</span><span>状态</span><span>健康度</span><span>刺激度</span><span>票价</span><span>累计营收</span><span>操作</span>
      </div>
      <div class="trow" v-for="r in list" :key="r.id">
        <span><b>{{ r.name }}</b><em class="muted">{{ r.type }}</em></span>
        <span>{{ typeIcon(r.type) }}</span>
        <span>{{ store.zones.find(z=>z.id===r.zone_id)?.name }}</span>
        <span>
          <i class="dot" :class="r.status"></i>{{ r.status === 'operating' ? '运营' : r.status === 'maintenance' ? '检修' : '关闭' }}
          <em class="muted sub" v-if="orderByRide[r.id]">{{ orderByRide[r.id].code }} · {{ Math.round(orderByRide[r.id].progress) }}%</em>
        </span>
        <span><div class="hb"><i :style="{width:r.health+'%', background: r.health>60?'var(--green)':r.health>40?'var(--accent2)':'var(--red)'}"></i></div>{{ r.health }}</span>
        <span>{{ r.thrill }}</span>
        <span class="money">{{ r.price }}</span>
        <span class="money">{{ r.rev.toLocaleString() }}</span>
        <span class="ops">
          <button class="ghost" :disabled="!!orderByRide[r.id]" :title="orderByRide[r.id] ? '工单流转中，完工后自动恢复运营' : ''"
                  @click="r.status==='operating'?update(r,{status:'closed'}):update(r,{status:'operating'})">{{ r.status==='operating'?'关闭':'开放' }}</button>
          <button class="ghost" v-if="!orderByRide[r.id] && r.status!=='closed'" @click="store.createRepair(r.id)">报修</button>
          <span class="tag" v-else-if="orderByRide[r.id]">工单流转中</span>
          <button class="ghost" @click="update(r,{upgrade:10})">升级</button>
          <button class="ghost danger" @click="store.delRide(r.id)">拆除</button>
        </span>
      </div>
    </div>

    <div class="modal" v-if="buildOpen">
      <div class="modal-box card">
        <h3>🏗️ 新建设施</h3>
        <div class="form">
          <label>设施类型
            <select v-model="build.type"><option v-for="t in types" :key="t" :value="t">{{ t }}</option></select>
          </label>
          <label>所属区域
            <select v-model.number="build.zone_id"><option v-for="z in store.zones.filter(z=>z.unlocked)" :key="z.id" :value="z.id">{{ z.name }}</option></select>
          </label>
          <label>名称 <input v-model="build.name" placeholder="留空自动命名" /></label>
        </div>
        <div class="acts">
          <button class="primary" @click="submit">确认建造</button>
          <button class="ghost" @click="buildOpen=false">取消</button>
        </div>
      </div>
    </div>

    <!-- 工单时间线 -->
    <div class="modal" v-if="detail" @click.self="closeDetail">
      <div class="modal-box card">
        <h3>🔧 {{ detail.ride_name }} · {{ detail.code }}
          <span class="badge" :class="statusMeta(detail.status).cls">{{ statusMeta(detail.status).label }}</span>
          <button class="ghost x" @click="closeDetail">✕</button>
        </h3>
        <div class="d-meta muted">
          第{{ detail.created_day }}天报修 · 报修时健康度 {{ Math.round(detail.health_from) }}
          <span v-if="detail.assignee_name"> · 当前维修员 {{ detail.assignee_name }}</span>
          <span v-if="detail.status === 'done'"> · 维修费 ¥{{ detail.cost.toLocaleString() }}</span>
          <span v-else> · 预计维修费 ¥{{ detail.est_cost.toLocaleString() }}</span>
        </div>
        <h4>处理时间线</h4>
        <div class="logs">
          <div v-for="l in detailLogs" :key="l.id" class="log">
            <span class="ldot"></span>
            <b>{{ ACTION_LABEL[l.action] || l.action }}</b>
            <em class="muted">第{{ l.day }}天 {{ l.hour }}:00</em>
            <p class="muted">{{ l.note }}</p>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.rides { display: flex; flex-direction: column; gap: 14px; }
.bar { display: flex; justify-content: space-between; align-items: center; flex-wrap: wrap; gap: 10px; }
.filters { display: flex; gap: 8px; }
.table { padding: 6px; overflow-x: auto; }
.thead, .trow { display: grid; grid-template-columns: 1.6fr .5fr .8fr .9fr 1fr .6fr .6fr .9fr 2fr; gap: 8px; align-items: center; padding: 10px 12px; font-size: 13px; min-width: 900px; }
.thead { color: var(--muted); border-bottom: 1px solid var(--border); font-size: 12px; }
.trow { border-bottom: 1px solid var(--border); }
.trow:last-child { border-bottom: none; }
.trow b { display: block; }
.trow em { font-style: normal; font-size: 11px; }
.trow .sub { display: block; }
.dot { display: inline-block; width: 8px; height: 8px; border-radius: 50%; margin-right: 5px; }
.dot.operating { background: var(--green); }
.dot.maintenance { background: var(--accent2); }
.dot.closed { background: var(--red); }
.hb { width: 90px; height: 7px; background: var(--panel2); border-radius: 4px; overflow: hidden; display: inline-block; margin-right: 6px; vertical-align: middle; }
.hb i { display: block; height: 100%; }
.ops { display: flex; gap: 4px; flex-wrap: wrap; align-items: center; }
.ops button { font-size: 11px; padding: 4px 8px; }
.ops button:disabled { opacity: .45; cursor: not-allowed; }
.tag { font-size: 11px; color: var(--accent2); border: 1px solid rgba(255,209,102,.4); border-radius: 12px; padding: 2px 8px; }
.modal { position: fixed; inset: 0; background: rgba(0,0,0,.55); display: flex; align-items: center; justify-content: center; z-index: 50; }
.modal-box { width: min(460px, 92vw); max-height: 86vh; overflow-y: auto; }
.form { display: flex; flex-direction: column; gap: 10px; margin: 14px 0; }
.form label { display: flex; flex-direction: column; gap: 5px; font-size: 13px; color: var(--muted); }
.acts { display: flex; gap: 8px; }

/* 维修工单 */
.shop h3 { display: flex; align-items: center; gap: 8px; flex-wrap: wrap; }
.chip { font-size: 11px; padding: 2px 10px; border-radius: 20px; border: 1px solid var(--border); background: var(--panel2); color: var(--muted); font-weight: 400; }
.chip.warn { color: var(--accent2); border-color: rgba(255,209,102,.45); }
.chip.ok { color: var(--green); border-color: rgba(109,213,160,.45); }
.muted-chip { margin-left: auto; }
.olist { display: flex; flex-direction: column; gap: 10px; margin-top: 12px; }
.oitem { border: 1px solid var(--border); border-left-width: 3px; border-radius: 10px; padding: 12px; background: rgba(255,255,255,.02); }
.oitem.queued { border-left-color: var(--accent2); }
.oitem.processing { border-left-color: var(--blue); }
.oitem.done { border-left-color: var(--green); opacity: .85; }
.o-head { display: flex; align-items: center; gap: 10px; }
.big-ic { font-size: 20px; }
.o-title { flex: 1; min-width: 0; }
.o-title b { display: block; font-size: 14px; }
.o-title em { font-style: normal; font-size: 11px; }
.badge { font-size: 11px; padding: 2px 8px; border-radius: 20px; border: 1px solid var(--border); background: var(--panel2); color: var(--muted); white-space: nowrap; }
.st-queued { color: var(--accent2) !important; border-color: rgba(255,209,102,.5); }
.st-processing { color: var(--blue) !important; border-color: rgba(102,166,255,.5); }
.st-done { color: var(--green) !important; border-color: rgba(109,213,160,.5); }
.st-cancel { color: var(--muted) !important; }
.timeline-btn { font-size: 12px; padding: 4px 10px; }
.o-body { margin-top: 8px; display: flex; flex-direction: column; gap: 8px; }
.o-body .hint { font-size: 12px; }
.o-info { display: flex; justify-content: space-between; font-size: 12px; flex-wrap: wrap; gap: 6px; }
.pbar { position: relative; height: 18px; background: var(--panel2); border-radius: 9px; overflow: hidden; }
.pbar i { display: block; height: 100%; background: linear-gradient(90deg, var(--blue), var(--purple)); border-radius: 9px; transition: .4s; }
.pbar b { position: absolute; inset: 0; display: flex; align-items: center; justify-content: center; font-size: 11px; font-weight: 700; }
.o-ops { display: flex; gap: 6px; align-items: center; flex-wrap: wrap; }
.o-ops select { font-size: 12px; max-width: 220px; }
.o-ops button { font-size: 12px; padding: 5px 12px; }
.done-info { font-size: 12.5px; }
.empty { padding: 14px 4px 4px; }
.x { margin-left: auto; }
.d-meta { font-size: 12px; margin-top: 8px; }
h4 { margin: 14px 0 10px; font-size: 13px; }
.logs { display: flex; flex-direction: column; }
.log { position: relative; padding: 0 0 14px 20px; border-left: 2px solid var(--border); margin-left: 5px; }
.log:last-child { border-left-color: transparent; padding-bottom: 0; }
.ldot { position: absolute; left: -7px; top: 2px; width: 12px; height: 12px; border-radius: 50%; background: var(--accent2); border: 2px solid var(--bg); }
.log b { font-size: 13px; margin-right: 8px; }
.log em { font-size: 11px; }
.log p { font-size: 12px; margin-top: 3px; }
</style>
