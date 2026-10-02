<script setup lang="ts">
import RevenueChart from '../components/RevenueChart.vue'
import TrafficChart from '../components/TrafficChart.vue'

const statCards = [
  { label: 'Total Revenue', value: '$84.2K', delta: '+12.4%', icon: 'mdi-currency-usd', color: 'success' },
  { label: 'Active Users', value: '24.8K', delta: '+8.1%', icon: 'mdi-account-group', color: 'info' },
  { label: 'Conversion', value: '6.34%', delta: '+1.2%', icon: 'mdi-trending-up', color: 'primary' },
  { label: 'Retention', value: '91.8%', delta: '+3.4%', icon: 'mdi-chart-timeline-variant', color: 'warning' },
]

const revenueData = {
  labels: ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun'],
  datasets: [
    {
      label: 'Revenue',
      data: [12000, 15000, 13500, 18500, 21000, 24000],
      backgroundColor: ['#5B7CFA', '#5B7CFA', '#7C3AED', '#5B7CFA', '#3B82F6', '#22C55E'],
      borderRadius: 10,
    },
  ],
}

const trafficData = {
  labels: ['Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat', 'Sun'],
  datasets: [
    {
      label: 'Visitors',
      data: [3200, 4100, 3900, 4700, 5200, 6100, 5800],
      borderColor: '#34d399',
      backgroundColor: 'rgba(52, 211, 153, 0.12)',
      fill: true,
      tension: 0.35,
    },
  ],
}
</script>

<template>
  <v-row class="mb-5">
    <v-col v-for="card in statCards" :key="card.label" cols="12" sm="6" md="3">
      <v-card class="pa-4 rounded-xl" elevation="2">
        <div class="d-flex justify-space-between align-center mb-4">
          <div class="text-medium-emphasis text-subtitle-2">{{ card.label }}</div>
          <v-icon :icon="card.icon" :color="card.color" size="28" />
        </div>
        <div class="kpi-number">{{ card.value }}</div>
        <div class="kpi-trend text-success mt-2">
          <v-icon icon="mdi-arrow-up" size="14" />
          {{ card.delta }} vs last period
        </div>
      </v-card>
    </v-col>
  </v-row>

  <v-row>
    <v-col cols="12" lg="8">
      <v-card class="chart-card rounded-xl pa-4" elevation="2">
        <v-card-title class="px-0 pb-4">Revenue Overview</v-card-title>
        <RevenueChart :data="revenueData" />
      </v-card>
    </v-col>

    <v-col cols="12" lg="4">
      <v-card class="chart-card rounded-xl pa-4" elevation="2">
        <v-card-title class="px-0 pb-4">Traffic Growth</v-card-title>
        <TrafficChart :data="trafficData" />
      </v-card>
    </v-col>
  </v-row>
</template>
