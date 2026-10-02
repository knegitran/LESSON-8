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
  type ScriptableContext,
  Tooltip,
} from 'chart.js'
import logistics from './data/logistics.json'

ChartJS.register(CategoryScale, LinearScale, BarElement, PointElement, LineElement, Tooltip, Legend, Filler)

type RegionName = 'Northeast' | 'Central' | 'South' | 'West'
type RegionMetrics = {
  shipments: number
  delivered: number
  onTime: number
  transitDaysTotal: number
}
type LogisticsMonth = {
  month: string
  year: number
  regions: Record<RegionName, RegionMetrics>
}
type ExceptionRecord = {
  id: string
  shipmentId: string
  region: RegionName
  issueType: string
  priority: 'Critical' | 'High' | 'Medium' | 'Low'
  ageDays: number
  status: 'Open' | 'Investigating' | 'Monitoring' | 'Resolved'
  openedMonth: string
}
type LogisticsData = {
  regions: RegionName[]
  months: LogisticsMonth[]
  exceptions: ExceptionRecord[]
}

const dataset = logistics as LogisticsData
const selectedPeriod = ref('ALL')
const selectedRegion = ref<'ALL' | RegionName>('ALL')
const exceptionSort = ref<'priority' | 'age'>('priority')
const emptyMetrics = (): RegionMetrics => ({ shipments: 0, delivered: 0, onTime: 0, transitDaysTotal: 0 })
const numberFormat = new Intl.NumberFormat('en-US')
const formatNumber = (value: number) => numberFormat.format(value)
const formatPercent = (value: number) => `${value.toFixed(1)}%`
const formatRateDelta = (value: number) => `${value > 0 ? '+' : ''}${value.toFixed(1)} pp`

const periodOptions = [
  { title: 'All periods · 2025', value: 'ALL' },
  ...dataset.months.map((item) => ({ title: `${item.month} ${item.year}`, value: item.month })),
]
const regionOptions = [
  { title: 'All regions', value: 'ALL' },
  ...dataset.regions.map((region) => ({ title: region, value: region })),
]
const sortOptions = [
  { title: 'Priority first', value: 'priority' },
  { title: 'Oldest first', value: 'age' },
]

const visibleMonths = computed(() => selectedPeriod.value === 'ALL'
  ? dataset.months
  : dataset.months.filter((item) => item.month === selectedPeriod.value))

const sumMetrics = (months: LogisticsMonth[], region = selectedRegion.value) => months.reduce((totals, item) => {
  const regionRows = region === 'ALL' ? Object.values(item.regions) : [item.regions[region]]
  for (const row of regionRows) {
    totals.shipments += row.shipments
    totals.delivered += row.delivered
    totals.onTime += row.onTime
    totals.transitDaysTotal += row.transitDaysTotal
  }
  return totals
}, emptyMetrics())

const currentMetrics = computed(() => sumMetrics(visibleMonths.value))
const previousMonth = computed(() => {
  if (selectedPeriod.value === 'ALL') return null
  const index = dataset.months.findIndex((item) => item.month === selectedPeriod.value)
  return index > 0 ? dataset.months[index - 1] : null
})
const previousMetrics = computed(() => previousMonth.value ? sumMetrics([previousMonth.value]) : null)

const filteredExceptions = computed(() => {
  const matches = dataset.exceptions.filter((item) =>
    item.status !== 'Resolved'
    && (selectedPeriod.value === 'ALL' || item.openedMonth === selectedPeriod.value)
    && (selectedRegion.value === 'ALL' || item.region === selectedRegion.value),
  )

  const priorityRank: Record<ExceptionRecord['priority'], number> = { Critical: 4, High: 3, Medium: 2, Low: 1 }
  return [...matches].sort((first, second) => exceptionSort.value === 'priority'
    ? priorityRank[second.priority] - priorityRank[first.priority] || second.ageDays - first.ageDays
    : second.ageDays - first.ageDays)
})

const countOpenExceptions = (month?: string) => dataset.exceptions.filter((item) =>
  item.status !== 'Resolved'
  && (month === undefined || item.openedMonth === month)
  && (selectedRegion.value === 'ALL' || item.region === selectedRegion.value),
).length

const priorityExceptionCount = computed(() => filteredExceptions.value.filter((item) =>
  item.priority === 'Critical' || item.priority === 'High',
).length)

const comparisonText = computed<string[]>(() => {
  if (selectedPeriod.value === 'ALL') return ['', '', '', '']
  if (!previousMetrics.value) return Array(4).fill('No previous month in sample')

  const previous = previousMetrics.value
  const current = currentMetrics.value
  const previousRate = previous.delivered ? (previous.onTime / previous.delivered) * 100 : 0
  const currentRate = current.delivered ? (current.onTime / current.delivered) * 100 : 0
  const previousTransit = previous.delivered ? previous.transitDaysTotal / previous.delivered : 0
  const currentTransit = current.delivered ? current.transitDaysTotal / current.delivered : 0
  const shipmentDelta = previous.shipments ? ((current.shipments - previous.shipments) / previous.shipments) * 100 : 0
  const transitDelta = currentTransit - previousTransit

  return [
    `${shipmentDelta > 0 ? '+' : ''}${shipmentDelta.toFixed(1)}% shipments vs prior month`,
    `${formatRateDelta(currentRate - previousRate)} vs prior month`,
    `${countOpenExceptions(selectedPeriod.value)} currently unresolved · opened this month`,
    `${transitDelta > 0 ? '+' : ''}${transitDelta.toFixed(1)} days vs prior month`,
  ]
})

const summaryCards = computed(() => {
  const totals = currentMetrics.value
  const onTimeRate = totals.delivered ? (totals.onTime / totals.delivered) * 100 : 0
  const averageTransit = totals.delivered ? totals.transitDaysTotal / totals.delivered : 0
  const exceptionCount = filteredExceptions.value.length
  const allPeriodDetails = [
    '12-month network total',
    `${formatNumber(totals.delivered)} deliveries counted`,
    `${priorityExceptionCount.value} high-priority items · active snapshot`,
    `${formatNumber(totals.delivered)} completed shipments`,
  ]

  return [
    { label: 'Shipments', value: formatNumber(totals.shipments), detail: selectedPeriod.value === 'ALL' ? allPeriodDetails[0] : comparisonText.value[0], icon: 'mdi-truck-fast-outline', accent: 'pink' },
    { label: 'On-time delivery', value: formatPercent(onTimeRate), detail: selectedPeriod.value === 'ALL' ? allPeriodDetails[1] : comparisonText.value[1], icon: 'mdi-clock-check-outline', accent: 'green' },
    { label: 'Open exceptions', value: formatNumber(exceptionCount), detail: selectedPeriod.value === 'ALL' ? allPeriodDetails[2] : `${priorityExceptionCount.value} high priority · opened ${selectedPeriod.value}`, icon: 'mdi-alert-circle-outline', accent: 'orange' },
    { label: 'Average transit', value: `${averageTransit.toFixed(1)} days`, detail: selectedPeriod.value === 'ALL' ? allPeriodDetails[3] : comparisonText.value[3], icon: 'mdi-timer-sand', accent: 'violet' },
  ]
})

const monthTotals = computed(() => visibleMonths.value.map((item) => ({
  month: item.month,
  ...sumMetrics([item]),
})))

const shipmentChartData = computed(() => ({
  labels: monthTotals.value.map((item) => item.month),
  datasets: [{
    label: 'Shipments',
    data: monthTotals.value.map((item) => item.shipments),
    backgroundColor: ['#8f005d', '#ff2bd6', '#c000a8', '#ff70e8', '#a60069', '#e600b8', '#ff9af0', '#b0008b', '#f51ccf', '#d40072', '#ff4dce', '#7a005f'],
    borderColor: '#170014',
    borderWidth: 1,
    borderRadius: 5,
    maxBarThickness: 34,
  }],
}))

const onTimeChartData = computed(() => ({
  labels: monthTotals.value.map((item) => item.month),
  datasets: [{
    label: 'On-time delivery',
    data: monthTotals.value.map((item) => item.delivered ? (item.onTime / item.delivered) * 100 : 0),
    borderColor: '#ff2bd6',
    backgroundColor: (context: ScriptableContext<'line'>) => {
      const { chart } = context
      if (!chart.chartArea) return 'rgba(255, 43, 214, 0.12)'
      const gradient = chart.ctx.createLinearGradient(0, chart.chartArea.bottom, 0, chart.chartArea.top)
      gradient.addColorStop(0, 'rgba(255, 43, 214, 0.01)')
      gradient.addColorStop(1, 'rgba(255, 43, 214, 0.2)')
      return gradient
    },
    fill: true,
    tension: 0.35,
    pointRadius: 3,
    pointHoverRadius: 5,
    pointBackgroundColor: '#ff9af0',
    pointBorderColor: '#08090d',
    pointBorderWidth: 2,
  }],
}))

const regionPerformance = computed(() => dataset.regions
  .filter((region) => selectedRegion.value === 'ALL' || region === selectedRegion.value)
  .map((region) => {
    const totals = sumMetrics(visibleMonths.value, region)
    return {
      name: region,
      ...totals,
      onTimeRate: totals.delivered ? (totals.onTime / totals.delivered) * 100 : 0,
    }
  }))
const maxRegionShipments = computed(() => Math.max(...regionPerformance.value.map((item) => item.shipments), 1))

const createChartOptions = (kind: 'number' | 'percent') => ({
  responsive: true,
  maintainAspectRatio: false,
  plugins: {
    legend: { display: false },
    tooltip: { displayColors: false },
  },
  layout: { padding: { top: 6, right: 8, bottom: 0, left: 0 } },
  scales: {
    x: {
      grid: { display: false },
      ticks: { color: '#a8a3b2', maxTicksLimit: 12, maxRotation: 0 },
    },
    y: {
      grid: { color: 'rgba(220, 210, 230, 0.09)' },
      ticks: {
        color: '#a8a3b2',
        maxTicksLimit: 5,
        callback: (value: string | number) => kind === 'percent' ? `${Number(value).toFixed(0)}%` : formatNumber(Number(value)),
      },
      beginAtZero: kind === 'number',
      ...(kind === 'percent' ? { suggestedMin: 85, suggestedMax: 100 } : {}),
    },
  },
})

const shipmentChartOptions = createChartOptions('number')
const onTimeChartOptions = createChartOptions('percent')
const selectedLabel = computed(() => selectedPeriod.value === 'ALL' ? 'Jan–Dec 2025' : `${selectedPeriod.value} 2025`)
</script>

<template>
  <v-app>
    <v-app-bar color="#ff2bd6" flat class="border-b dashboard-app-bar" height="82">
      <v-container class="app-bar-content" fluid>
        <div class="brand-lockup">
          <div class="brand-mark"><v-icon icon="mdi-truck-fast" size="22" /></div>
          <div>
            <div class="brand-name">FastForward Logistics</div>
            <div class="brand-caption">OPERATIONS / LEADERSHIP REVIEW</div>
          </div>
        </div>

        <div class="filters">
          <div class="filter-box">
            <span id="period-filter-label" class="filter-label">Reporting period</span>
            <v-select
              id="period-filter"
              v-model="selectedPeriod"
              :items="periodOptions"
              item-title="title"
              item-value="value"
              aria-labelledby="period-filter-label"
              variant="outlined"
              density="compact"
              hide-details
              class="filter-select period-select"
              bg-color="#111018"
              color="primary"
            />
          </div>
          <div class="filter-box">
            <span id="region-filter-label" class="filter-label">Destination region</span>
            <v-select
              id="region-filter"
              v-model="selectedRegion"
              :items="regionOptions"
              item-title="title"
              item-value="value"
              aria-labelledby="region-filter-label"
              variant="outlined"
              density="compact"
              hide-details
              class="filter-select region-select"
              bg-color="#111018"
              color="primary"
            />
          </div>
        </div>
      </v-container>
    </v-app-bar>

    <v-main>
      <v-container fluid class="dashboard-shell">
        <header class="page-heading">
          <div>
            <div class="eyebrow"><span class="status-dot" /> NETWORK OPERATIONS <span class="eyebrow-divider">/</span> {{ selectedLabel }}</div>
            <h1>Operations overview</h1>
            <p>Shipment health, delivery performance, and active service risks.</p>
          </div>
          <div class="snapshot-label"><span class="snapshot-icon">2025</span><span>Sample performance review</span></div>
        </header>

        <section class="kpi-grid" aria-label="Operations summary">
          <v-card v-for="card in summaryCards" :key="card.label" class="metric-card" elevation="0" :class="`accent-${card.accent}`">
            <div class="metric-topline">
              <span>{{ card.label }}</span>
              <v-icon :icon="card.icon" size="20" />
            </div>
            <div class="metric-value">{{ card.value }}</div>
            <div class="metric-detail">{{ card.detail }}</div>
          </v-card>
        </section>

        <section class="trend-grid" aria-label="Performance trends">
          <v-card class="panel chart-panel" elevation="0">
            <div class="panel-heading">
              <div><h2>Shipment volume</h2><p>Shipments created · {{ selectedLabel }}</p></div>
              <span class="chart-unit">LOADS</span>
            </div>
            <div class="chart-canvas"><Bar :data="shipmentChartData" :options="shipmentChartOptions" /></div>
          </v-card>

          <v-card class="panel chart-panel" elevation="0">
            <div class="panel-heading">
              <div><h2>On-time delivery</h2><p>Delivered by promised date · {{ selectedLabel }}</p></div>
              <span class="chart-unit">RATE</span>
            </div>
            <div class="chart-canvas"><Line :data="onTimeChartData" :options="onTimeChartOptions" /></div>
          </v-card>
        </section>

        <section class="detail-grid" aria-label="Regional performance and open exceptions">
          <v-card class="panel region-panel" elevation="0">
            <div class="panel-heading">
              <div><h2>Regional performance</h2><p>Volume and on-time delivery by destination</p></div>
              <span class="region-count">{{ regionPerformance.length }} REGIONS</span>
            </div>

            <div class="region-list">
              <div v-for="region in regionPerformance" :key="region.name" class="region-row">
                <div class="region-main">
                  <div class="region-name">{{ region.name }}</div>
                  <div class="region-stat"><strong>{{ formatNumber(region.shipments) }}</strong><span>shipments</span></div>
                </div>
                <div class="volume-track" role="img" :aria-label="`${region.name}: ${formatNumber(region.shipments)} shipments`">
                  <span :style="{ width: `${(region.shipments / maxRegionShipments) * 100}%` }" />
                </div>
                <div class="region-rate-block">
                  <strong>{{ formatPercent(region.onTimeRate) }}</strong>
                  <span>on time</span>
                </div>
                <span class="rate-status" :class="region.onTimeRate >= 93 ? 'status-good' : region.onTimeRate >= 91 ? 'status-watch' : 'status-low'">
                  {{ region.onTimeRate >= 93 ? 'At target' : region.onTimeRate >= 91 ? 'Watch' : 'Below target' }}
                </span>
              </div>
            </div>
            <div class="region-footnote">On-time target: 93% · rate uses delivered shipments as denominator</div>
          </v-card>

          <v-card class="panel exceptions-panel" elevation="0">
            <div class="panel-heading exceptions-heading">
              <div><h2>Open exceptions <span class="count-badge">{{ filteredExceptions.length }}</span></h2><p>Unresolved issues opened in {{ selectedLabel }}</p></div>
              <v-select
                v-model="exceptionSort"
                :items="sortOptions"
                item-title="title"
                item-value="value"
                aria-label="Sort exceptions"
                variant="outlined"
                density="compact"
                hide-details
                class="sort-select"
              />
            </div>

            <div v-if="filteredExceptions.length" class="exception-table-wrap">
              <table class="exception-table">
                <thead>
                  <tr><th>Shipment / issue</th><th>Region</th><th>Priority</th><th>Age</th><th>Status</th></tr>
                </thead>
                <tbody>
                  <tr v-for="item in filteredExceptions" :key="item.id">
                    <td>
                      <div class="shipment-id">{{ item.shipmentId }}</div>
                      <div class="issue-name">{{ item.issueType }}</div>
                    </td>
                    <td class="table-region">{{ item.region }}</td>
                    <td><span class="priority-pill" :class="`priority-${item.priority.toLowerCase()}`">{{ item.priority }}</span></td>
                    <td class="age-cell">{{ item.ageDays }}d</td>
                    <td><span class="exception-status">{{ item.status }}</span></td>
                  </tr>
                </tbody>
              </table>
            </div>
            <div v-else class="empty-state">
              <v-icon icon="mdi-check-circle-outline" size="24" />
              <strong>No open exceptions in this view</strong>
              <span>Try another period or region.</span>
            </div>
          </v-card>
        </section>

        <footer class="dashboard-footer">
          <span>FASTFORWARD LOGISTICS <span class="footer-separator">/</span> INTERNAL OPERATIONS</span>
          <span>Fictional sample data · 2025</span>
        </footer>
      </v-container>
    </v-main>
  </v-app>
</template>