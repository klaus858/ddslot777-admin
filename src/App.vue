<script setup>
import { computed, onMounted, reactive, ref } from 'vue'
import { ElMessage, ElMessageBox } from 'element-plus'
import {
  Coin,
  CreditCard,
  DataAnalysis,
  House,
  Lock,
  Money,
  Operation,
  SwitchButton,
  User,
  Wallet
} from '@element-plus/icons-vue'

const API_BASE = 'https://ddslot777-api.vercel.app'

const isAuthed = ref(localStorage.getItem('dd-admin-authed') === '1')
const activeMenu = ref('dashboard')
const loginForm = reactive({ username: 'admin', password: 'admin123' })
const loginError = ref('')
const loading = ref(false)
const apiStatus = ref('Checking API')
const apiError = ref('')

const summary = ref({
  todayDeposit: 0,
  pendingDeposits: 0,
  pendingWithdrawals: 0,
  totalUsers: 0
})
const users = ref([])
const deposits = ref([])
const withdrawals = ref([])
const auditLogs = ref([])

const currentTitle = computed(() => {
  const titles = {
    dashboard: '运营总览',
    users: '用户列表',
    deposits: '充值订单',
    withdrawals: '提现订单',
    logs: '操作记录'
  }
  return titles[activeMenu.value] || '后台'
})

const apiStatusType = computed(() => {
  if (apiStatus.value === 'API Connected') return 'success'
  if (apiStatus.value === 'API Error') return 'danger'
  return 'warning'
})

const metrics = computed(() => [
  { label: '今日充值', value: money(summary.value.todayDeposit), trend: 'API 同步', icon: Money },
  { label: '待确认充值', value: String(summary.value.pendingDeposits || 0), trend: '需处理', icon: Wallet },
  { label: '待审核提现', value: String(summary.value.pendingWithdrawals || summary.value.pendingWithdraws || 0), trend: '需审核', icon: CreditCard },
  { label: '用户总数', value: String(summary.value.totalUsers || summary.value.newMembers || 0), trend: 'API 同步', icon: User }
])

const money = (value) => `$ ${Number(value || 0).toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 })}`

const statusText = {
  normal: '正常',
  risk_review: '风控观察',
  disabled: '停用',
  pending: '待处理',
  credited: '已上分',
  approved: '已通过',
  rejected: '已拒绝'
}

const requestAPI = async (path, options = {}) => {
  const headers = {
    Accept: 'application/json',
    'Content-Type': 'application/json',
    ...(options.headers || {})
  }
  const token = localStorage.getItem('dd-admin-token')
  if (token) headers.Authorization = `Bearer ${token}`

  const response = await fetch(`${API_BASE}${path}`, { ...options, headers })
  const data = await response.json().catch(() => ({}))
  if (!response.ok) {
    throw new Error(data.error || `request_failed_${response.status}`)
  }
  return data
}

const assertItems = (payload, name) => {
  if (!payload || !Array.isArray(payload.items)) {
    throw new Error(`${name}_contract_error`)
  }
}

const refreshAll = async () => {
  loading.value = true
  apiError.value = ''
  apiStatus.value = 'Checking API'

  try {
    const health = await requestAPI('/api/health')
    if (!health.ok || health.service !== 'ddslot777-api') {
      throw new Error('health_contract_error')
    }

    const [summaryData, userData, depositData, withdrawalData, logData] = await Promise.all([
      requestAPI('/api/admin/summary'),
      requestAPI('/api/admin/users'),
      requestAPI('/api/admin/deposits'),
      requestAPI('/api/admin/withdrawals'),
      requestAPI('/api/admin/audit-logs')
    ])

    assertItems(userData, 'users')
    assertItems(depositData, 'deposits')
    assertItems(withdrawalData, 'withdrawals')
    assertItems(logData, 'auditLogs')

    summary.value = summaryData
    users.value = userData.items
    deposits.value = depositData.items
    withdrawals.value = withdrawalData.items
    auditLogs.value = logData.items
    apiStatus.value = 'API Connected'
  } catch (error) {
    apiStatus.value = 'API Error'
    apiError.value = `API 数据同步失败：${error.message}`
    users.value = []
    deposits.value = []
    withdrawals.value = []
    auditLogs.value = []
    ElMessage.error('API 同步失败，后台已停止显示本地假数据')
  } finally {
    loading.value = false
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
    await refreshAll()
  } catch (error) {
    loginError.value = '账号密码不正确，或 API 登录接口不可用'
  }
}

const runAction = async (successText, action) => {
  try {
    await action()
    ElMessage.success(successText)
    await refreshAll()
  } catch (error) {
    ElMessage.error(`操作失败：${error.message}`)
  }
}

const confirmDeposit = (row) => {
  if (row.state === 'credited') return
  runAction('充值已确认，数据已从 API 刷新', () =>
    requestAPI('/api/admin/deposits/confirm', {
      method: 'POST',
      body: JSON.stringify({ order: row.order })
    })
  )
}

const approveWithdrawal = (row) => {
  if (row.state !== 'pending') return
  runAction('提现已通过，数据已从 API 刷新', () =>
    requestAPI('/api/admin/withdrawals/approve', {
      method: 'POST',
      body: JSON.stringify({ order: row.order, reason: 'manual approve' })
    })
  )
}

const rejectWithdrawal = async (row) => {
  if (row.state !== 'pending') return
  try {
    const { value } = await ElMessageBox.prompt('请输入拒绝原因', '拒绝提现', {
      confirmButtonText: '拒绝',
      cancelButtonText: '取消',
      inputValue: '资料需复核'
    })
    await runAction('提现已拒绝，数据已从 API 刷新', () =>
      requestAPI('/api/admin/withdrawals/reject', {
        method: 'POST',
        body: JSON.stringify({ order: row.order, reason: value })
      })
    )
  } catch (error) {
    if (error !== 'cancel') ElMessage.error('拒绝提现失败')
  }
}

const editBalance = async (row) => {
  try {
    const { value } = await ElMessageBox.prompt(`当前余额：${money(row.balance)}`, `修改 ${row.name} 余额`, {
      confirmButtonText: '保存',
      cancelButtonText: '取消',
      inputValue: String(row.balance),
      inputPattern: /^\d+(\.\d{1,2})?$/,
      inputErrorMessage: '请输入正确金额'
    })
    await runAction('用户余额已更新，数据已从 API 刷新', () =>
      requestAPI('/api/admin/users/balance', {
        method: 'POST',
        body: JSON.stringify({ userId: row.id, balance: Number(value), reason: 'manual adjustment' })
      })
    )
  } catch (error) {
    if (error !== 'cancel') ElMessage.error('修改余额失败')
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
    refreshAll()
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
      <el-menu :default-active="activeMenu" class="admin-menu" background-color="transparent" text-color="#9fb5dc" active-text-color="#24d6ff">
        <el-menu-item index="dashboard" @click="activeMenu = 'dashboard'"><el-icon><House /></el-icon>运营总览</el-menu-item>
        <el-menu-item index="users" @click="activeMenu = 'users'"><el-icon><User /></el-icon>用户列表</el-menu-item>
        <el-menu-item index="deposits" @click="activeMenu = 'deposits'"><el-icon><Wallet /></el-icon>充值订单</el-menu-item>
        <el-menu-item index="withdrawals" @click="activeMenu = 'withdrawals'"><el-icon><CreditCard /></el-icon>提现订单</el-menu-item>
        <el-menu-item index="logs" @click="activeMenu = 'logs'"><el-icon><Operation /></el-icon>操作记录</el-menu-item>
      </el-menu>
    </el-aside>

    <el-container>
      <el-header class="admin-header">
        <div>
          <p>DD Slot Admin</p>
          <h2>{{ currentTitle }}</h2>
        </div>
        <div class="header-actions">
          <el-button size="small" type="primary" plain @click="refreshAll">同步 API</el-button>
          <el-tag effect="dark" :type="apiStatusType">{{ apiStatus }}</el-tag>
          <el-button :icon="SwitchButton" circle @click="logout" />
        </div>
      </el-header>

      <el-main class="admin-main" v-loading="loading">
        <el-alert v-if="apiError" class="api-alert" :title="apiError" type="error" show-icon :closable="false" />

        <section v-if="activeMenu === 'dashboard'" class="dashboard-grid">
          <article v-for="item in metrics" :key="item.label" class="metric-card">
            <el-icon><component :is="item.icon" /></el-icon>
            <span>{{ item.label }}</span>
            <strong>{{ item.value }}</strong>
            <small>{{ item.trend }}</small>
          </article>
          <article class="panel wide-panel">
            <div class="panel-title"><DataAnalysis /> 后台闭环状态</div>
            <div class="sync-flow">
              <div>登录</div>
              <div>拉取 API</div>
              <div>处理订单</div>
              <div>刷新数据</div>
              <div>记录日志</div>
            </div>
          </article>
          <article class="panel">
            <div class="panel-title"><Coin /> 资金快照</div>
            <el-statistic title="已确认充值" :value="summary.todayDeposit || 0" prefix="$" />
            <el-statistic title="待处理订单" :value="(summary.pendingDeposits || 0) + (summary.pendingWithdrawals || 0)" />
          </article>
        </section>

        <section v-if="activeMenu === 'users'" class="panel">
          <div class="panel-title"><User /> 用户列表</div>
          <el-table :data="users" stripe>
            <el-table-column prop="id" label="ID" width="110" />
            <el-table-column prop="name" label="用户" />
            <el-table-column label="余额">
              <template #default="scope">{{ money(scope.row.balance) }}</template>
            </el-table-column>
            <el-table-column prop="vip" label="等级" width="110" />
            <el-table-column label="状态" width="120">
              <template #default="scope">{{ statusText[scope.row.status] || scope.row.status }}</template>
            </el-table-column>
            <el-table-column prop="lastLogin" label="最后登录" />
            <el-table-column label="操作" width="140">
              <template #default="scope">
                <el-button size="small" type="primary" plain @click="editBalance(scope.row)">改余额</el-button>
              </template>
            </el-table-column>
          </el-table>
        </section>

        <section v-if="activeMenu === 'deposits'" class="panel">
          <div class="panel-title"><Wallet /> 充值订单</div>
          <el-table :data="deposits" stripe>
            <el-table-column prop="order" label="订单号" width="170" />
            <el-table-column prop="user" label="用户" />
            <el-table-column label="金额">
              <template #default="scope">{{ money(scope.row.amount) }}</template>
            </el-table-column>
            <el-table-column prop="channel" label="通道" />
            <el-table-column label="状态">
              <template #default="scope">{{ statusText[scope.row.state] || scope.row.state }}</template>
            </el-table-column>
            <el-table-column prop="time" label="时间" />
            <el-table-column label="操作" width="150">
              <template #default="scope">
                <el-button size="small" type="primary" :disabled="scope.row.state === 'credited'" @click="confirmDeposit(scope.row)">确认充值</el-button>
              </template>
            </el-table-column>
          </el-table>
        </section>

        <section v-if="activeMenu === 'withdrawals'" class="panel">
          <div class="panel-title"><CreditCard /> 提现订单</div>
          <el-table :data="withdrawals" stripe>
            <el-table-column prop="order" label="订单号" width="170" />
            <el-table-column prop="user" label="用户" />
            <el-table-column label="金额">
              <template #default="scope">{{ money(scope.row.amount) }}</template>
            </el-table-column>
            <el-table-column prop="channel" label="通道" />
            <el-table-column prop="address" label="账户" />
            <el-table-column label="状态">
              <template #default="scope">{{ statusText[scope.row.state] || scope.row.state }}</template>
            </el-table-column>
            <el-table-column label="操作" width="190">
              <template #default="scope">
                <el-button size="small" type="success" :disabled="scope.row.state !== 'pending'" @click="approveWithdrawal(scope.row)">通过</el-button>
                <el-button size="small" type="danger" plain :disabled="scope.row.state !== 'pending'" @click="rejectWithdrawal(scope.row)">拒绝</el-button>
              </template>
            </el-table-column>
          </el-table>
        </section>

        <section v-if="activeMenu === 'logs'" class="panel">
          <div class="panel-title"><Operation /> 操作记录</div>
          <el-table :data="auditLogs" stripe>
            <el-table-column prop="time" label="时间" width="180" />
            <el-table-column prop="actor" label="操作人" width="110" />
            <el-table-column prop="action" label="动作" width="170" />
            <el-table-column prop="target" label="对象" width="160" />
            <el-table-column prop="detail" label="详情" />
          </el-table>
        </section>
      </el-main>
    </el-container>
  </el-container>
</template>
