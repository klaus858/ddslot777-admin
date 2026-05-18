<script setup>
import { computed, reactive, ref } from 'vue'
import {
  Coin,
  CreditCard,
  DataAnalysis,
  Gift,
  House,
  Lock,
  Money,
  Operation,
  SwitchButton,
  User,
  Wallet
} from '@element-plus/icons-vue'

const isAuthed = ref(localStorage.getItem('dd-admin-authed') === '1')
const activeMenu = ref('dashboard')
const loginForm = reactive({ username: 'admin', password: 'admin123' })
const loginError = ref('')

const metrics = [
  { label: '今日充值', value: '$ 12,480.00', trend: '+18%', icon: Money },
  { label: '待处理提现', value: '7', trend: '需审核', icon: CreditCard },
  { label: '新增会员', value: '126', trend: '+9.4%', icon: User },
  { label: '活动领取', value: '384', trend: '实时', icon: Gift }
]

const members = [
  { id: 'U10021', name: 'Player123', balance: '$ 860.50', vip: 'VIP 2', status: '正常', lastLogin: '2026-05-19 12:40' },
  { id: 'U10022', name: 'LuckyJoe', balance: '$ 2,410.00', vip: 'VIP 4', status: '正常', lastLogin: '2026-05-19 11:18' },
  { id: 'U10023', name: 'HighRoller', balance: '$ 9,800.00', vip: 'VIP 6', status: '风控观察', lastLogin: '2026-05-19 10:06' },
  { id: 'U10024', name: 'PlayWin88', balance: '$ 320.75', vip: 'VIP 1', status: '正常', lastLogin: '2026-05-19 09:54' }
]

const deposits = [
  { order: 'D20260519001', user: 'Player123', amount: '$ 100.00', channel: 'USDT', state: '已上分', time: '12:34' },
  { order: 'D20260519002', user: 'LuckyJoe', amount: '$ 300.00', channel: 'CashApp', state: '待确认', time: '12:22' },
  { order: 'D20260519003', user: 'PlayWin88', amount: '$ 50.00', channel: 'PayPal', state: '已上分', time: '11:48' },
  { order: 'D20260519004', user: 'HighRoller', amount: '$ 1,000.00', channel: 'Bank', state: '风控复核', time: '11:05' }
]

const activities = [
  { name: '首存奖励', rule: '1st Deposit +30%', state: true },
  { name: '每日返水', rule: 'Slot Rebate 2%', state: true },
  { name: '登录奖励', rule: '$28 Free Bonus', state: true },
  { name: '好友邀请', rule: 'Refer & Earn', state: false }
]

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

const doLogin = () => {
  if (loginForm.username === 'admin' && loginForm.password === 'admin123') {
    localStorage.setItem('dd-admin-authed', '1')
    isAuthed.value = true
    loginError.value = ''
    return
  }
  loginError.value = '账号或密码不正确'
}

const logout = () => {
  localStorage.removeItem('dd-admin-authed')
  isAuthed.value = false
}
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
        <el-menu-item index="activities" @click="activeMenu = 'activities'"><el-icon><Gift /></el-icon>活动配置</el-menu-item>
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
          <el-tag effect="dark" type="success">Demo Server</el-tag>
          <el-button :icon="SwitchButton" circle @click="logout" />
        </div>
      </el-header>

      <el-main class="admin-main">
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
                <el-button size="small" type="primary" plain>{{ scope.row.state === '待确认' ? '确认上分' : '查看' }}</el-button>
              </template>
            </el-table-column>
          </el-table>
        </section>

        <section v-if="activeMenu === 'activities'" class="panel">
          <div class="panel-title"><Gift /> 活动配置</div>
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
