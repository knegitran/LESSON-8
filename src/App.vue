<script setup lang="ts">
import { computed, ref } from 'vue'
import { Bar, Line } from 'vue-chartjs'
import {
  BarElement,
  CategoryScale,
  Chart as ChartJS,
  Filler,
  Legend,
  LineElement,
  LinearScale,
  PointElement,
  Title,
  Tooltip,
} from 'chart.js'
import metrics from './data/metrics.json'

ChartJS.register(CategoryScale, LinearScale, BarElement, PointElement, LineElement, Title, Tooltip, Legend, Filler)

type MetricEntry = {
  month: string
  revenue: number
  visitors: number
  conversions: number
  orders: number
}

const data = metrics as MetricEntry[]
const selectedMonth = ref('ALL')

const monthOptions = [
  { title: 'All months', value: 'ALL' },
  ...data.map((item) => ({ title: item.month, value: item.month })),
]

const currentMonthEntry = computed(() => {
  if (selectedMonth.value === 'ALL') return null
  return data.find((item) => item.month === selectedMonth.value) ?? null
})

const previousMonthEntry = computed(() => {
  if (selectedMonth.value === 'ALL') {
    return data[data.length - 2] ?? data[data.length - 1]
  }

  const index = data.findIndex((item) => item.month === selectedMonth.value)
  return data[index - 1] ?? data[0]
})

const getChangePercent = (current: number, previous: number) => {
  if (!previous) return 0
  return ((current - previous) / previous) * 100
}

const formatCurrency = (value: number) =>
  new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: 'USD',
    maximumFractionDigits: 0,
  }).format(value)

const formatCompactCurrency = (value: number) => {
  if (value >= 1000000) return `$${(value / 1000000).toFixed(1)}M`
  if (value >= 1000) return `$${(value / 1000).toFixed(1)}K`
  return `$${value.toFixed(0)}`
}

const formatNumber = (value: number) =>
  new Intl.NumberFormat('en-US', { maximumFractionDigits: 0 }).format(value)

const formatPercent = (value: number) => `${value.toFixed(1)}%`

const summaryCards = computed(() => {
  const yearlyRevenueAverage = data.reduce((sum, item) => sum + item.revenue, 0) / data.length
  const yearlyVisitorsAverage = data.reduce((sum, item) => sum + item.visitors, 0) / data.length
  const yearlyConversionsAverage = data.reduce((sum, item) => sum + item.conversions, 0) / data.length
  const yearlyOrdersAverage = data.reduce((sum, item) => sum + item.orders, 0) / data.length

  const currentRevenue = selectedMonth.value === 'ALL' ? yearlyRevenueAverage : currentMonthEntry.value?.revenue ?? 0
  const currentVisitors = selectedMonth.value === 'ALL' ? yearlyVisitorsAverage : currentMonthEntry.value?.visitors ?? 0
  const currentConversions = selectedMonth.value === 'ALL' ? yearlyConversionsAverage : currentMonthEntry.value?.conversions ?? 0
  const currentOrders = selectedMonth.value === 'ALL' ? yearlyOrdersAverage : currentMonthEntry.value?.orders ?? 0

  return [
    {
      label: 'Revenue',
      value: selectedMonth.value === 'ALL' ? formatCompactCurrency(currentRevenue) : formatCurrency(currentRevenue),
      delta: getChangePercent(
        currentRevenue,
        selectedMonth.value === 'ALL' ? data[data.length - 2]?.revenue ?? currentRevenue : previousMonthEntry.value?.revenue ?? currentRevenue,
      ),
      icon: 'mdi-currency-usd',
      color: 'success',
    },
    {
      label: 'Visitors',
      value: formatNumber(currentVisitors),
      delta: getChangePercent(
        currentVisitors,
        selectedMonth.value === 'ALL' ? data[data.length - 2]?.visitors ?? currentVisitors : previousMonthEntry.value?.visitors ?? currentVisitors,
      ),
      icon: 'mdi-account-group',
      color: 'info',
    },
    {
      label: 'Conversions',
      value: formatPercent(currentConversions),
      delta: getChangePercent(
        currentConversions,
        selectedMonth.value === 'ALL' ? data[data.length - 2]?.conversions ?? currentConversions : previousMonthEntry.value?.conversions ?? currentConversions,
      ),
      icon: 'mdi-trending-up',
      color: 'primary',
    },
    {
      label: 'Orders',
      value: formatNumber(currentOrders),
      delta: getChangePercent(
        currentOrders,
        selectedMonth.value === 'ALL' ? data[data.length - 2]?.orders ?? currentOrders : previousMonthEntry.value?.orders ?? currentOrders,
      ),
      icon: 'mdi-bag-check',
      color: 'warning',
    },
  ]
})

const revenueChartData = computed(() => {
  const labels = selectedMonth.value === 'ALL' ? data.map((item) => item.month) : [currentMonthEntry.value?.month ?? '']
  const values = selectedMonth.value === 'ALL'
    ? data.map((item) => item.revenue)
    : [currentMonthEntry.value?.revenue ?? 0]

  return {
    labels,
    datasets: [
      {
        label: 'Revenue',
        data: values,
        backgroundColor: selectedMonth.value === 'ALL'
          ? ['#2dd4bf', '#34d399', '#5eead4', '#86efac', '#a7f3d0', '#d1fae5', '#34d399', '#2dd4bf', '#6ee7b7', '#4ade80', '#10b981', '#14b8a6']
          : ['#2dd4bf'],
        borderRadius: 10,
      },
    ],
  }
})

const visitorsChartData = computed(() => {
  const labels = selectedMonth.value === 'ALL' ? data.map((item) => item.month) : [currentMonthEntry.value?.month ?? '']
  const values = selectedMonth.value === 'ALL'
    ? data.map((item) => item.visitors)
    : [currentMonthEntry.value?.visitors ?? 0]

  return {
    labels,
    datasets: [
      {
        label: 'Visitors',
        data: values,
        borderColor: '#8b5cf6',
        backgroundColor: 'rgba(139, 92, 246, 0.12)',
        fill: true,
        tension: 0.4,
      },
    ],
  }
})

const conversionsChartData = computed(() => {
  const labels = data.map((item) => item.month)
  const values = data.map((item) => item.conversions)

  return {
    labels,
    datasets: [
      {
        label: 'Conversions',
        data: selectedMonth.value === 'ALL' ? values : [(currentMonthEntry.value?.conversions ?? 0)],
        borderColor: '#fbbf24',
        backgroundColor: 'rgba(251, 191, 36, 0.18)',
        fill: true,
        tension: 0.45,
      },
    ],
  }
})

const chartOptions = {
  responsive: true,
  maintainAspectRatio: false,
  plugins: {
    legend: { display: false },
  },
  layout: {
    padding: {
      top: 8,
      right: 8,
      bottom: 0,
      left: 0,
    },
  },
  scales: {
    x: {
      grid: { display: false },
      ticks: { color: '#cbd5e1', maxTicksLimit: 6 },
    },
    y: {
      grid: { color: 'rgba(148, 163, 184, 0.14)' },
      ticks: { color: '#cbd5e1', maxTicksLimit: 5 },
      beginAtZero: false,
    },
  },
}

const selectedLabel = computed(() => selectedMonth.value === 'ALL' ? 'All months' : selectedMonth.value)
</script>

<template>
  <v-app>
    <v-app-bar color="#0f172a" flat class="border-b" height="80">
      <v-container class="d-flex align-center px-4" fluid>
        <div class="text-h5 font-weight-bold">My Dashboard</div>
        <v-spacer />
        <v-select
          v-model="selectedMonth"
          :items="monthOptions"
          item-title="title"
          item-value="value"
          variant="outlined"
          hide-details
          density="comfortable"
          class="month-picker"
          bg-color="#111827"
          color="primary"
        />
      </v-container>
    </v-app-bar>

    <v-main>
      <v-container fluid class="dashboard-shell">
        <div class="mb-4 text-body-1 text-medium-emphasis">Showing: {{ selectedLabel }}</div>

        <v-row class="mb-5">
          <v-col v-for="card in summaryCards" :key="card.label" cols="12" sm="6" md="3">
            <v-card class="rounded-xl pa-3 metric-card" elevation="0">
              <div class="d-flex justify-space-between align-center mb-2">
                <span class="text-medium-emphasis text-subtitle-2">{{ card.label }}</span>
                <v-icon :icon="card.icon" :color="card.color" size="22" />
              </div>

              <div class="metric-value mb-1">{{ card.value }}</div>

              <div :class="['metric-trend', card.delta >= 0 ? 'positive' : 'negative']">
                <v-icon :icon="card.delta >= 0 ? 'mdi-arrow-up' : 'mdi-arrow-down'" size="15" />
                {{ Math.abs(card.delta).toFixed(1) }}% vs prior period
              </div>
            </v-card>
          </v-col>
        </v-row>

        <v-row class="mb-5">
          <v-col cols="12" md="6">
            <v-card class="rounded-xl pa-3 chart-card" elevation="0">
              <v-card-title class="px-0 pb-2">Revenue</v-card-title>
              <Bar :data="revenueChartData" :options="chartOptions" />
            </v-card>
          </v-col>

          <v-col cols="12" md="6">
            <v-card class="rounded-xl pa-3 chart-card" elevation="0">
              <v-card-title class="px-0 pb-2">Visitors</v-card-title>
              <Line :data="visitorsChartData" :options="chartOptions" />
            </v-card>
          </v-col>
        </v-row>

        <v-row>
          <v-col cols="12">
            <v-card class="rounded-xl pa-3 chart-card large-chart" elevation="0">
              <v-card-title class="px-0 pb-2">Conversions</v-card-title>
              <Line :data="conversionsChartData" :options="chartOptions" />
            </v-card>
          </v-col>
        </v-row>
      </v-container>
    </v-main>
  </v-app>
</template>
