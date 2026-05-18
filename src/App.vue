<script setup>
import { computed, onMounted, reactive, ref } from 'vue'
import { ElMessage } from 'element-plus'
import {
  Coin,
  CreditCard,
  DataAnalysis,
  House,
  Lock,
  Money,
  Operation,
  Present,
  SwitchButton,
  User,
  Wallet
} from '@element-plus/icons-vue'

const API_BASE = 'https://ddslot777-api.vercel.app'

const isAuthed = ref(localStorage.getItem('dd-admin-authed') === '1')
const activeMenu = ref('dashboard')
const loginForm = reactive({ username: 'admin', password: 'admin123' })
const loginError = ref('')
const tableLoading = ref(false)
const apiStatus = ref('Checking API')
const apiError = ref('')

const createEmptyMetrics = () => [
  { label: '今日充值', value: '$ 0.00', trend: 'API实时', icon: Money },
  { label: '待处理提现', value: '0', trend: 'API实时', icon: CreditCard },
  { label: '新增会员', value: '0', trend: 'API实时', icon: User },
  { label: '活动领取', value: '0', trend: 'API实时', icon: Present }
]

const metrics = ref(createEmptyMetrics())
const members = ref([])
const deposits = ref([])

const activities = ref([
  { name: '首存奖励', rule: '1st Deposit +30%', state: true },
  { name: '每日返水', rule: 'Slot Rebate 2%', state: true },
  { name: '登录奖励', rule: '$28 Free Bonus', state: true },
  { name: '好友邀请', rule: 'Refer & Earn', state: false }
])

const currentTitle = computed(() => {
  const map = {
    dashboard: '运营总览',
    members: '会员管理',
    deposits: '充值审核',
    activities: '活动配置',
    risk: '风控记录'
  }
  return map[activeMenu.value]
})

const apiStatusType = computed(() => {
  if (apiStatus.value === 'Go API Connected') return 'success'
  if (apiStatus.value === 'API Error') return 'danger'
  return 'warning'
})

const formatMoney = (value) => `$ ${Number(value || 0).toLocaleString('en-US', { minimumFractionDigits: 2 })}`

const memberStatusMap = {
  normal: '正常',
  risk_review: '风控观察'
}

const depositStatusMap = {
  pending: '待确认',
  credited: '已上分',
  risk_review: '风控复核'
}

const assertArray = (value, name) => {
  if (!Array.isArray(value)) {
    throw new Error(`${name}_contract_error`)
  }
}

const assertNumber = (value, name) => {
  if (typeof value !== 'number') {
    throw new Error(`${name}_contract_error`)
  }
}

const requestAPI = async (path, options = {}) => {
  const headers = {
    Accept: 'application/json',
    'Content-Type': 'application/json',
    ...(options.headers || {})
  }
  const token = localStorage.getItem('dd-admin-token')
  if (token) headers.Authorization = `Bearer ${token}`

  const response = await fetch(`${API_BASE}${path}`, {
    ...options,
    headers
  })

  const data = await response.json().catch(() => ({}))
  if (!response.ok) {
    throw new Error(data.error || 'request_failed')
  }
  return data
}

const loadAdminData = async () => {
  tableLoading.value = true
  apiError.value = ''
  apiStatus.value = 'Checking API'

  try {
    const health = await requestAPI('/api/health')
    if (!health.ok || health.service !== 'ddslot777-api') {
      throw new Error('health_contract_error')
    }

    const [summary, memberData, depositData] = await Promise.all([
      requestAPI('/api/admin/summary'),
      requestAPI('/api/admin/members'),
      requestAPI('/api/admin/deposits')
    ])

    assertNumber(summary.todayDeposit, 'summary.todayDeposit')
    assertNumber(summary.pendingWithdraws, 'summary.pendingWithdraws')
    assertNumber(summary.newMembers, 'summary.newMembers')
    assertNumber(summary.activityClaims, 'summary.activityClaims')
    assertArray(memberData.items, 'members.items')
    assertArray(depositData.items, 'deposits.items')

    metrics.value = [
      { label: '今日充值', value: formatMoney(summary.todayDeposit), trend: 'API实时', icon: Money },
      { label: '待处理提现', value: String(summary.pendingWithdraws), trend: '需审核', icon: CreditCard },
      { label: '新增会员', value: String(summary.newMembers), trend: 'API实时', icon: User },
      { label: '活动领取', value: String(summary.activityClaims), trend: 'API实时', icon: Present }
    ]

    members.value = memberData.items.map((item) => ({
      id: item.id,
      name: item.name,
      balance: item.balance,
      vip: item.vip,
      status: memberStatusMap[item.status] || item.status,
      lastLogin: item.lastLogin
    }))

    deposits.value = depositData.items.map((item) => ({
      order: item.order,
      user: item.user,
      amount: item.amount,
      channel: item.channel,
      state: depositStatusMap[item.state] || item.state,
      time: item.time
    }))

    apiStatus.value = 'Go API Connected'
  } catch (error) {
    apiStatus.value = 'API Error'
    apiError.value = `API连接或字段结构异常：${error.message}`
    metrics.value = createEmptyMetrics()
    members.value = []
    deposits.value = []
    ElMessage.error('API连接失败，已停止显示演示假数据')
  } finally {
    tableLoading.value = false
  }
}

const doLogin = async () => {
  loginError.value = ''
  try {
    const data = await requestAPI('/api/auth/login', {
      method: 'POST',
      body: JSON.stringify(loginForm)
    })

    if (!data.token) throw new Error('missing_token')

    localStorage.setItem('dd-admin-authed', '1')
    localStorage.setItem('dd-admin-token', data.token)
    isAuthed.value = true
    ElMessage.success('登录成功')
    await loadAdminData()
  } catch (error) {
    loginError.value = '账号或密码不正确，或 API 登录接口不可用'
  }
}

const confirmDeposit = async (row) => {
  if (row.state !== '待确认') return
  try {
    await requestAPI('/api/admin/deposits/confirm', {
      method: 'POST',
      body: JSON.stringify({ order: row.order })
    })
    ElMessage.success('已同步 API 上分状态')
    await loadAdminData()
  } catch (error) {
    ElMessage.error('确认失败，API未同步')
  }
}

const logout = () => {
  localStorage.removeItem('dd-admin-authed')
  localStorage.removeItem('dd-admin-token')
  isAuthed.value = false
}

onMounted(() => {
  if (isAuthed.value) {
    if (!localStorage.getItem('dd-admin-token')) {
      localStorage.removeItem('dd-admin-authed')
      isAuthed.value = false
      return
    }
    loadAdminData()
  }
})
</script>

<template>
  <main v-if="!isAuthed" class="login-page">
    <section class="login-card">
      <div class="brand-mark">DD</div>
      <h1>大东 Slot 后台</h1>
      <p>运营管理中心</p>
      <el-form class="login-form" @submit.prevent="doLogin">
        <el-form-item>
          <el-input v-model="loginForm.username" size="large" placeholder="Username" :prefix-icon="User" />
        </el-form-item>
        <el-form-item>
          <el-input v-model="loginForm.password" size="large" show-password placeholder="Password" :prefix-icon="Lock" />
        </el-form-item>
        <el-alert v-if="loginError" :title="loginError" type="error" show-icon :closable="false" />
        <el-button class="login-button" size="large" type="primary" @click="doLogin">Login</el-button>
      </el-form>
      <div class="demo-note">Demo: admin / admin123</div>
    </section>
  </main>

  <el-container v-else class="admin-shell">
    <el-aside width="248px" class="admin-sidebar">
      <div class="sidebar-logo">
        <span>DD</span>
        <strong>大东 Slot</strong>
      </div>
      <el-menu v-model="activeMenu" class="admin-menu" background-color="transparent" text-color="#9fb5dc" active-text-color="#24d6ff">
        <el-menu-item index="dashboard" @click="activeMenu = 'dashboard'"><el-icon><House /></el-icon>运营总览</el-menu-item>
        <el-menu-item index="members" @click="activeMenu = 'members'"><el-icon><User /></el-icon>会员管理</el-menu-item>
        <el-menu-item index="deposits" @click="activeMenu = 'deposits'"><el-icon><Wallet /></el-icon>充值审核</el-menu-item>
        <el-menu-item index="activities" @click="activeMenu = 'activities'"><el-icon><Present /></el-icon>活动配置</el-menu-item>
        <el-menu-item index="risk" @click="activeMenu = 'risk'"><el-icon><Operation /></el-icon>风控记录</el-menu-item>
      </el-menu>
    </el-aside>

    <el-container>
      <el-header class="admin-header">
        <div>
          <p>DD Slot Admin</p>
          <h2>{{ currentTitle }}</h2>
        </div>
        <div class="header-actions">
          <el-button size="small" type="primary" plain @click="loadAdminData">同步API</el-button>
          <el-tag effect="dark" :type="apiStatusType">{{ apiStatus }}</el-tag>
          <el-button :icon="SwitchButton" circle @click="logout" />
        </div>
      </el-header>

      <el-main class="admin-main" v-loading="tableLoading">
        <el-alert v-if="apiError" class="api-alert" :title="apiError" type="error" show-icon :closable="false" />

        <section v-if="activeMenu === 'dashboard'" class="dashboard-grid">
          <article v-for="item in metrics" :key="item.label" class="metric-card">
            <el-icon><component :is="item.icon" /></el-icon>
            <span>{{ item.label }}</span>
            <strong>{{ item.value }}</strong>
            <small>{{ item.trend }}</small>
          </article>
          <article class="panel wide-panel">
            <div class="panel-title"><DataAnalysis /> 今日流水趋势</div>
            <div class="fake-chart">
              <i v-for="height in [32, 54, 46, 72, 88, 64, 96, 76, 112, 92, 124, 108]" :key="height" :style="{ height: height + 'px' }" />
            </div>
          </article>
          <article class="panel">
            <div class="panel-title"><Coin /> 资金快照</div>
            <el-statistic title="平台余额" value="284,510" prefix="$" />
            <el-statistic title="今日派奖" value="37,820" prefix="$" />
          </article>
        </section>

        <section v-if="activeMenu === 'members'" class="panel">
          <div class="panel-title"><User /> 会员列表</div>
          <el-table :data="members" stripe>
            <el-table-column prop="id" label="ID" width="110" />
            <el-table-column prop="name" label="会员" />
            <el-table-column prop="balance" label="余额" />
            <el-table-column prop="vip" label="等级" />
            <el-table-column prop="status" label="状态" />
            <el-table-column prop="lastLogin" label="最后登录" />
          </el-table>
        </section>

        <section v-if="activeMenu === 'deposits'" class="panel">
          <div class="panel-title"><Wallet /> 充值记录</div>
          <el-table :data="deposits" stripe>
            <el-table-column prop="order" label="订单号" width="160" />
            <el-table-column prop="user" label="会员" />
            <el-table-column prop="amount" label="金额" />
            <el-table-column prop="channel" label="通道" />
            <el-table-column prop="state" label="状态" />
            <el-table-column prop="time" label="时间" />
            <el-table-column label="操作" width="150">
              <template #default="scope">
                <el-button size="small" type="primary" plain @click="confirmDeposit(scope.row)">
                  {{ scope.row.state === '待确认' ? '确认上分' : '查看' }}
                </el-button>
              </template>
            </el-table-column>
          </el-table>
        </section>

        <section v-if="activeMenu === 'activities'" class="panel">
          <div class="panel-title"><Present /> 活动配置</div>
          <el-alert class="api-alert" title="活动配置暂时是前端本地开关，下一步会接入 Go API 和数据库保存。" type="warning" show-icon :closable="false" />
          <div class="activity-list">
            <div v-for="activity in activities" :key="activity.name" class="activity-row">
              <div>
                <strong>{{ activity.name }}</strong>
                <span>{{ activity.rule }}</span>
              </div>
              <el-switch v-model="activity.state" active-text="开启" inactive-text="关闭" />
            </div>
          </div>
        </section>

        <section v-if="activeMenu === 'risk'" class="panel">
          <div class="panel-title"><Operation /> 风控提醒</div>
          <el-timeline>
            <el-timeline-item timestamp="12:18" type="warning">HighRoller 单笔充值超过 $1,000，进入人工复核。</el-timeline-item>
            <el-timeline-item timestamp="11:42" type="success">Player123 完成 KYC 信息检查。</el-timeline-item>
            <el-timeline-item timestamp="10:30" type="info">系统自动同步 24 条登录日志。</el-timeline-item>
          </el-timeline>
        </section>
      </el-main>
    </el-container>
  </el-container>
</template>
