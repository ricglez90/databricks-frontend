<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'

const telemetryData = ref([])
const loading = ref(true)
const error = ref(null)
const lastUpdated = ref(null)
const isRefreshing = ref(false)
let pollingInterval = null

const fetchTelemetry = async () => {
  isRefreshing.value = true
  try {
    const apiUrl = import.meta.env.VITE_API_URL || 'http://localhost:3000'
    const response = await fetch(`${apiUrl}/api/telemetry`)
    
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: Failed to fetch data`)
    }
    
    telemetryData.value = await response.json()
    lastUpdated.value = new Date().toLocaleTimeString()
    error.value = null
  } catch (err) {
    error.value = err.message
  } finally {
    loading.value = false
    setTimeout(() => { isRefreshing.value = false }, 400)
  }
}

// KPI Metrics
const totalUnits = computed(() => telemetryData.value.length)

const maxTemp = computed(() => {
  if (!telemetryData.value.length) return 0
  return Math.max(...telemetryData.value.map(d => parseFloat(d.avg_engine_temperature_celsius) || 0)).toFixed(1)
})

const alertCount = computed(() => {
  return telemetryData.value.filter(
    d => parseFloat(d.avg_engine_temperature_celsius) > 90 || parseFloat(d.avg_vibration_hz) > 40
  ).length
})

const getHealthStatus = (temp, vib) => {
  const t = parseFloat(temp)
  const v = parseFloat(vib)
  if (t > 95 || v > 45) return { label: 'Critical', class: 'badge-critical' }
  if (t > 88 || v > 38) return { label: 'Elevated', class: 'badge-warning' }
  return { label: 'Optimal', class: 'badge-optimal' }
}

onMounted(() => {
  fetchTelemetry()
  pollingInterval = setInterval(fetchTelemetry, 6000)
})

onUnmounted(() => {
  clearInterval(pollingInterval)
})
</script>

<template>
  <div class="dashboard-shell">
    <!-- Top Navigation Bar -->
    <header class="topbar">
      <div class="brand">
        <div class="brand-icon">
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <path d="M12 2L2 7l10 5 10-5-10-5zM2 17l10 5 10-5M2 12l10 5 10-5"/>
          </svg>
        </div>
        <div>
          <h2>Telemetry Command Center</h2>
          <p class="subtitle">Databricks Unity Catalog · Medallion Gold Layer</p>
        </div>
      </div>

      <div class="system-status">
        <div class="status-chip">
          <span class="pulse-indicator"></span>
          <span>Streaming Active</span>
        </div>
        <span class="timestamp" v-if="lastUpdated">Updated: {{ lastUpdated }}</span>
        <button class="refresh-btn" :class="{ spinning: isRefreshing }" @click="fetchTelemetry" title="Refresh Now">
          <svg viewBox="0 0 24 24" width="16" height="16" fill="none" stroke="currentColor" stroke-width="2">
            <path d="M21.5 2v6h-6M2.5 22v-6h6M2 11.5a10 10 0 0 1 18.8-4.3M22 12.5a10 10 0 0 1-18.8 4.2"/>
          </svg>
        </button>
      </div>
    </header>

    <!-- Stat KPI Cards -->
    <section class="metrics-grid">
      <div class="card metric-card">
        <span class="metric-title">Monitored Assets</span>
        <div class="metric-body">
          <span class="metric-value">{{ totalUnits }}</span>
          <span class="metric-tag">Heavy Equipment</span>
        </div>
      </div>

      <div class="card metric-card">
        <span class="metric-title">Peak Engine Temp</span>
        <div class="metric-body">
          <span class="metric-value" :class="{ 'text-danger': maxTemp > 90 }">{{ maxTemp }}°C</span>
          <span class="metric-tag">Fleet Max</span>
        </div>
      </div>

      <div class="card metric-card">
        <span class="metric-title">Active Threshold Warnings</span>
        <div class="metric-body">
          <span class="metric-value" :class="{ 'text-warning': alertCount > 0 }">{{ alertCount }}</span>
          <span class="metric-tag">Requires Review</span>
        </div>
      </div>
    </section>

    <!-- Main Content Area -->
    <main class="card main-table-card">
      <div class="table-header">
        <h3>Asset Telemetry Performance</h3>
        <span class="caption">Lifetime averages computed via continuous Delta stream aggregation</span>
      </div>

      <div v-if="loading" class="state-container">
        <div class="loader"></div>
        <p>Connecting to Databricks SQL Warehouse...</p>
      </div>

      <div v-else-if="error" class="state-container error-state">
        <p class="error-title">Connection Interrupted</p>
        <p class="error-desc">{{ error }}</p>
        <button class="retry-btn" @click="fetchTelemetry">Retry Connection</button>
      </div>

      <div v-else class="table-wrapper">
        <table class="data-table">
          <thead>
            <tr>
              <th>Asset Identifier</th>
              <th>Health Status</th>
              <th>Avg Temperature</th>
              <th>Avg Vibration</th>
              <th>Operating Profile</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="row in telemetryData" :key="row.equipment_id">
              <td class="cell-equipment">
                <span class="equipment-pill">{{ row.equipment_id }}</span>
              </td>
              <td>
                <span :class="['status-badge', getHealthStatus(row.avg_engine_temperature_celsius, row.avg_vibration_hz).class]">
                  {{ getHealthStatus(row.avg_engine_temperature_celsius, row.avg_vibration_hz).label }}
                </span>
              </td>
              <td class="cell-metric">
                <strong>{{ parseFloat(row.avg_engine_temperature_celsius).toFixed(1) }}</strong>
                <span class="unit">°C</span>
              </td>
              <td class="cell-metric">
                <strong>{{ parseFloat(row.avg_vibration_hz).toFixed(1) }}</strong>
                <span class="unit">Hz</span>
              </td>
              <td class="cell-bar">
                <div class="meter-track">
                  <div 
                    class="meter-fill" 
                    :style="{ 
                      width: `${Math.min((parseFloat(row.avg_engine_temperature_celsius) / 110) * 100, 100)}%`,
                      backgroundColor: parseFloat(row.avg_engine_temperature_celsius) > 90 ? '#ef4444' : '#3b82f6'
                    }"
                  ></div>
                </div>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </main>
  </div>
</template>

<style scoped>
.dashboard-shell {
  max-width: 1180px;
  margin: 0 auto;
  padding: 2.5rem 1.5rem;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, sans-serif;
  color: #0f172a;
  background-color: #f8fafc;
  min-height: 100vh;
}

.topbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 2rem;
  flex-wrap: wrap;
  gap: 1rem;
}

.brand {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.brand-icon {
  width: 44px;
  height: 44px;
  background: linear-gradient(135deg, #1e293b, #0f172a);
  color: #38bdf8;
  border-radius: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.brand-icon svg {
  width: 24px;
  height: 24px;
}

.brand h2 {
  margin: 0;
  font-size: 1.35rem;
  font-weight: 700;
  letter-spacing: -0.02em;
}

.subtitle {
  margin: 0.2rem 0 0 0;
  font-size: 0.825rem;
  color: #64748b;
  font-weight: 500;
}

.system-status {
  display: flex;
  align-items: center;
  gap: 1rem;
  background: white;
  padding: 0.5rem 0.875rem;
  border-radius: 9999px;
  border: 1px solid #e2e8f0;
  box-shadow: 0 1px 2px rgba(0, 0, 0, 0.04);
}

.status-chip {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-size: 0.8rem;
  font-weight: 600;
  color: #059669;
}

.pulse-indicator {
  width: 8px;
  height: 8px;
  background-color: #10b981;
  border-radius: 50%;
  box-shadow: 0 0 0 0 rgba(16, 185, 129, 0.7);
  animation: pulse 2s infinite;
}

@keyframes pulse {
  0% { transform: scale(0.95); box-shadow: 0 0 0 0 rgba(16, 185, 129, 0.7); }
  70% { transform: scale(1); box-shadow: 0 0 0 6px rgba(16, 185, 129, 0); }
  100% { transform: scale(0.95); box-shadow: 0 0 0 0 rgba(16, 185, 129, 0); }
}

.timestamp {
  font-size: 0.75rem;
  color: #94a3b8;
  border-left: 1px solid #cbd5e1;
  padding-left: 0.75rem;
}

.refresh-btn {
  background: none;
  border: none;
  cursor: pointer;
  color: #64748b;
  padding: 4px;
  border-radius: 6px;
  display: flex;
  align-items: center;
  transition: all 0.2s ease;
}

.refresh-btn:hover {
  background-color: #f1f5f9;
  color: #0f172a;
}

.refresh-btn.spinning svg {
  animation: spin 0.8s linear infinite;
}

@keyframes spin {
  from { transform: rotate(0deg); }
  to { transform: rotate(360deg); }
}

.card {
  background: white;
  border: 1px solid #e2e8f0;
  border-radius: 16px;
  box-shadow: 0 1px 3px 0 rgba(0, 0, 0, 0.05);
}

.metrics-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(240px, 1fr));
  gap: 1.25rem;
  margin-bottom: 1.5rem;
}

.metric-card {
  padding: 1.25rem 1.5rem;
}

.metric-title {
  font-size: 0.775rem;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  font-weight: 700;
  color: #64748b;
}

.metric-body {
  display: flex;
  align-items: baseline;
  justify-content: space-between;
  margin-top: 0.5rem;
}

.metric-value {
  font-size: 1.9rem;
  font-weight: 800;
  letter-spacing: -0.03em;
  color: #0f172a;
}

.metric-tag {
  font-size: 0.75rem;
  color: #94a3b8;
  font-weight: 500;
}

.text-danger { color: #dc2626 !important; }
.text-warning { color: #d97706 !important; }

.main-table-card {
  overflow: hidden;
}

.table-header {
  padding: 1.5rem 1.75rem 1rem;
  border-bottom: 1px solid #f1f5f9;
}

.table-header h3 {
  margin: 0;
  font-size: 1.1rem;
  font-weight: 700;
}

.caption {
  font-size: 0.825rem;
  color: #64748b;
  display: block;
  margin-top: 0.25rem;
}

.table-wrapper {
  overflow-x: auto;
}

.data-table {
  width: 100%;
  border-collapse: collapse;
  text-align: left;
}

.data-table th {
  padding: 0.875rem 1.75rem;
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #475569;
  background: #f8fafc;
  font-weight: 700;
  border-bottom: 1px solid #e2e8f0;
}

.data-table td {
  padding: 1.15rem 1.75rem;
  border-bottom: 1px solid #f1f5f9;
  font-size: 0.925rem;
}

.data-table tbody tr:hover {
  background-color: #f8fafc;
}

.equipment-pill {
  font-weight: 700;
  color: #1e293b;
  background: #e2e8f0;
  padding: 0.35rem 0.65rem;
  border-radius: 6px;
  font-size: 0.825rem;
  letter-spacing: -0.01em;
}

.status-badge {
  display: inline-block;
  padding: 0.25rem 0.6rem;
  font-size: 0.75rem;
  font-weight: 700;
  border-radius: 9999px;
  text-transform: uppercase;
  letter-spacing: 0.03em;
}

.badge-optimal { background: #dcfce7; color: #15803d; }
.badge-warning { background: #fef3c7; color: #b45309; }
.badge-critical { background: #fee2e2; color: #b91c1c; }

.cell-metric strong {
  font-size: 1.05rem;
  font-weight: 700;
}

.unit {
  font-size: 0.775rem;
  color: #94a3b8;
  margin-left: 0.25rem;
}

.cell-bar {
  width: 25%;
}

.meter-track {
  height: 8px;
  background: #e2e8f0;
  border-radius: 9999px;
  overflow: hidden;
}

.meter-fill {
  height: 100%;
  border-radius: 9999px;
  transition: width 0.4s ease;
}

.state-container {
  padding: 4rem 2rem;
  text-align: center;
  color: #64748b;
}

.loader {
  width: 32px;
  height: 32px;
  border: 3px solid #e2e8f0;
  border-top-color: #2563eb;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
  margin: 0 auto 1rem;
}

.error-title {
  color: #dc2626;
  font-weight: 700;
  font-size: 1rem;
  margin-bottom: 0.25rem;
}

.error-desc {
  font-size: 0.85rem;
  color: #64748b;
  margin-bottom: 1rem;
}

.retry-btn {
  background: #0f172a;
  color: white;
  border: none;
  padding: 0.5rem 1.25rem;
  border-radius: 8px;
  font-size: 0.85rem;
  font-weight: 600;
  cursor: pointer;
}
</style>