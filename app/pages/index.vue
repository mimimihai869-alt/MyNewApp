<template>
  <div class="bugetel">
    <header class="header">
      <h1>💰 Bugetel</h1>
      <p class="subtitle">Bugetul tău simplu</p>
    </header>

    <!-- Quick Add Expense -->
    <div class="card add-expense">
      <h2>➕ Adaugă cheltuială</h2>
      <form @submit.prevent="addExpense">
        <input
          v-model="newExpense.description"
          type="text"
          placeholder="Ex: Cafea la birou"
          required
          class="input"
        />
        <div class="input-group">
          <input
            v-model.number="newExpense.amount"
            type="number"
            step="0.01"
            placeholder="Sumă"
            required
            class="input input-small"
          />
          <select v-model="newExpense.category" class="input input-small">
            <option value="mancare">🍕 Mâncare</option>
            <option value="transport">🚗 Transport</option>
            <option value="distractie">🎉 Distracție</option>
            <option value="cumparaturi">🛍️ Cumpărături</option>
            <option value="casa">🏠 Casă</option>
            <option value="altele">📦 Altele</option>
          </select>
        </div>
        <button type="submit" class="btn btn-primary">Adaugă</button>
      </form>
    </div>

    <!-- Summary Cards -->
    <div class="summary-grid">
      <div class="card summary-card">
        <div class="summary-label">Cheltuieli de azi</div>
        <div class="summary-amount">{{ todayExpenses }} lei</div>
      </div>
      <div class="card summary-card">
        <div class="summary-label">Luna asta</div>
        <div class="summary-amount">{{ monthExpenses }} lei</div>
      </div>
    </div>

    <!-- Monthly Spending Chart -->
    <div class="card chart-section">
      <h2>📈 Evoluție lunară</h2>
      <div v-if="expenses.length === 0" class="empty-state">
        Adaugă cheltuieli pentru a vedea graficul!
      </div>
      <div v-else>
        <canvas ref="monthlyChart" class="chart-canvas"></canvas>
        <div class="month-comparison" v-if="monthlyComparison">
          <div class="comparison-card" :class="monthlyComparison.trend">
            <div class="comparison-label">Față de luna trecută</div>
            <div class="comparison-value">
              <span class="comparison-icon">{{ monthlyComparison.icon }}</span>
              {{ monthlyComparison.percentage }}%
              <span class="comparison-text">{{ monthlyComparison.text }}</span>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Savings Goals -->
    <div class="card savings">
      <h2>🎯 Pune deoparte</h2>
      <div v-if="!showSavingsForm" class="savings-list">
        <div v-for="goal in savingsGoals" :key="goal.id" class="savings-item">
          <div class="savings-info">
            <div class="savings-name">{{ goal.name }}</div>
            <div class="savings-progress">
              {{ goal.saved }} / {{ goal.target }} lei
            </div>
          </div>
          <div class="progress-bar">
            <div
              class="progress-fill"
              :style="{ width: (goal.saved / goal.target * 100) + '%' }"
            ></div>
          </div>
          <button @click="addToSavings(goal)" class="btn-small">+ Adaugă</button>
        </div>
        <button @click="showSavingsForm = true" class="btn btn-secondary">
          + Obiectiv nou
        </button>
      </div>
      <form v-else @submit.prevent="addSavingsGoal" class="savings-form">
        <input
          v-model="newGoal.name"
          type="text"
          placeholder="Ex: Weekend la munte"
          required
          class="input"
        />
        <input
          v-model.number="newGoal.target"
          type="number"
          placeholder="Sumă țintă (lei)"
          required
          class="input"
        />
        <div class="btn-group">
          <button type="submit" class="btn btn-primary">Salvează</button>
          <button type="button" @click="showSavingsForm = false" class="btn btn-secondary">
            Anulează
          </button>
        </div>
      </form>
    </div>

    <!-- Spending Analysis - Unde se duc banii -->
    <div class="card spending-analysis">
      <h2>📊 Unde se duc banii</h2>
      <div v-if="groupedExpenses.length === 0" class="empty-state">
        Nicio cheltuială încă. Începe să urmărești banii!
      </div>
      <div v-else>
        <div v-for="group in groupedExpenses" :key="group.name" class="grouped-item">
          <div class="group-header" @click="toggleGroup(group.name)">
            <div class="group-main">
              <div class="group-icon">{{ getCategoryIcon(group.category) }}</div>
              <div class="group-info">
                <div class="group-name">{{ group.name }}</div>
                <div class="group-stats">
                  {{ group.count }}x cumpărat • Total: {{ group.total.toFixed(2) }} lei
                </div>
              </div>
            </div>
            <div class="group-toggle">
              {{ expandedGroups[group.name] ? '▼' : '▶' }}
            </div>
          </div>

          <div v-if="expandedGroups[group.name]" class="group-details">
            <div v-for="transaction in group.transactions" :key="transaction.id" class="transaction-item">
              <div class="transaction-date">{{ formatDate(transaction.date) }}</div>
              <div class="transaction-amount">{{ transaction.amount }} lei</div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Recent Expenses -->
    <div class="card expenses-list">
      <h2>📝 Cheltuieli recente</h2>
      <div v-if="expenses.length === 0" class="empty-state">
        Nicio cheltuială încă. Adaugă prima!
      </div>
      <div v-for="expense in recentExpenses" :key="expense.id" class="expense-item">
        <div class="expense-icon">{{ getCategoryIcon(expense.category) }}</div>
        <div class="expense-details">
          <div class="expense-description">{{ expense.description }}</div>
          <div class="expense-date">{{ formatDate(expense.date) }}</div>
        </div>
        <div class="expense-amount">{{ expense.amount }} lei</div>
        <button @click="deleteExpense(expense.id)" class="btn-delete">×</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, watch, nextTick } from 'vue'
import { Chart, registerables } from 'chart.js'

// Register Chart.js components
Chart.register(...registerables)

// State
const expenses = ref([])
const savingsGoals = ref([])
const showSavingsForm = ref(false)
const expandedGroups = ref({})
const monthlyChart = ref(null)
let chartInstance = null

const newExpense = ref({
  description: '',
  amount: null,
  category: 'mancare'
})

const newGoal = ref({
  name: '',
  target: null
})

// Load data from localStorage
onMounted(() => {
  const savedExpenses = localStorage.getItem('bugetel-expenses')
  const savedGoals = localStorage.getItem('bugetel-goals')

  if (savedExpenses) {
    expenses.value = JSON.parse(savedExpenses)
  }
  if (savedGoals) {
    savingsGoals.value = JSON.parse(savedGoals)
  }

  // Create chart after data is loaded
  nextTick(() => {
    createChart()
  })
})

// Watch for changes in expenses to update chart
watch(() => expenses.value.length, () => {
  nextTick(() => {
    updateChart()
  })
})

// Create the monthly chart
function createChart() {
  if (!monthlyChart.value) return

  const ctx = monthlyChart.value.getContext('2d')

  // Destroy existing chart if it exists
  if (chartInstance) {
    chartInstance.destroy()
  }

  chartInstance = new Chart(ctx, {
    type: 'bar',
    data: {
      labels: monthlyData.value.map(m => m.label),
      datasets: [{
        label: 'Cheltuieli (lei)',
        data: monthlyData.value.map(m => m.total),
        backgroundColor: 'rgba(102, 126, 234, 0.8)',
        borderColor: 'rgba(102, 126, 234, 1)',
        borderWidth: 2,
        borderRadius: 8,
      }]
    },
    options: {
      responsive: true,
      maintainAspectRatio: true,
      aspectRatio: 2,
      plugins: {
        legend: {
          display: false
        },
        tooltip: {
          backgroundColor: 'rgba(0, 0, 0, 0.8)',
          padding: 12,
          titleFont: {
            size: 14,
            weight: 'bold'
          },
          bodyFont: {
            size: 13
          },
          callbacks: {
            label: function(context) {
              return context.parsed.y.toFixed(2) + ' lei'
            }
          }
        }
      },
      scales: {
        y: {
          beginAtZero: true,
          ticks: {
            callback: function(value) {
              return value + ' lei'
            },
            font: {
              size: 11
            }
          },
          grid: {
            color: 'rgba(0, 0, 0, 0.05)'
          }
        },
        x: {
          ticks: {
            font: {
              size: 11
            }
          },
          grid: {
            display: false
          }
        }
      }
    }
  })
}

// Update chart data
function updateChart() {
  if (!chartInstance) {
    createChart()
    return
  }

  chartInstance.data.labels = monthlyData.value.map(m => m.label)
  chartInstance.data.datasets[0].data = monthlyData.value.map(m => m.total)
  chartInstance.update()
}

// Computed
const todayExpenses = computed(() => {
  const today = new Date().toDateString()
  return expenses.value
    .filter(e => new Date(e.date).toDateString() === today)
    .reduce((sum, e) => sum + e.amount, 0)
    .toFixed(2)
})

const monthExpenses = computed(() => {
  const now = new Date()
  const currentMonth = now.getMonth()
  const currentYear = now.getFullYear()

  return expenses.value
    .filter(e => {
      const d = new Date(e.date)
      return d.getMonth() === currentMonth && d.getFullYear() === currentYear
    })
    .reduce((sum, e) => sum + e.amount, 0)
    .toFixed(2)
})

const recentExpenses = computed(() => {
  return [...expenses.value]
    .sort((a, b) => new Date(b.date) - new Date(a.date))
    .slice(0, 10)
})

const groupedExpenses = computed(() => {
  // Group expenses by description (case-insensitive)
  const groups = {}

  expenses.value.forEach(expense => {
    const key = expense.description.toLowerCase().trim()

    if (!groups[key]) {
      groups[key] = {
        name: expense.description,
        category: expense.category,
        total: 0,
        count: 0,
        transactions: []
      }
    }

    groups[key].total += expense.amount
    groups[key].count += 1
    groups[key].transactions.push({
      id: expense.id,
      date: expense.date,
      amount: expense.amount
    })
  })

  // Convert to array and sort by total spent (descending)
  return Object.values(groups)
    .map(group => ({
      ...group,
      transactions: group.transactions.sort((a, b) => new Date(b.date) - new Date(a.date))
    }))
    .sort((a, b) => b.total - a.total)
})

// Calculate monthly expenses for the last 6 months
const monthlyData = computed(() => {
  const months = []
  const now = new Date()

  // Get last 6 months
  for (let i = 5; i >= 0; i--) {
    const date = new Date(now.getFullYear(), now.getMonth() - i, 1)
    const monthKey = `${date.getFullYear()}-${String(date.getMonth() + 1).padStart(2, '0')}`

    months.push({
      key: monthKey,
      label: date.toLocaleDateString('ro-RO', { month: 'short', year: 'numeric' }),
      total: 0
    })
  }

  // Calculate totals for each month
  expenses.value.forEach(expense => {
    const expenseDate = new Date(expense.date)
    const monthKey = `${expenseDate.getFullYear()}-${String(expenseDate.getMonth() + 1).padStart(2, '0')}`

    const monthData = months.find(m => m.key === monthKey)
    if (monthData) {
      monthData.total += expense.amount
    }
  })

  return months
})

// Calculate comparison with previous month
const monthlyComparison = computed(() => {
  if (monthlyData.value.length < 2) return null

  const currentMonth = monthlyData.value[monthlyData.value.length - 1]
  const previousMonth = monthlyData.value[monthlyData.value.length - 2]

  if (previousMonth.total === 0) return null

  const difference = currentMonth.total - previousMonth.total
  const percentage = Math.abs((difference / previousMonth.total) * 100).toFixed(1)

  if (difference > 0) {
    return {
      trend: 'increase',
      icon: '📈',
      percentage: percentage,
      text: 'mai mult',
      amount: difference.toFixed(2)
    }
  } else if (difference < 0) {
    return {
      trend: 'decrease',
      icon: '📉',
      percentage: percentage,
      text: 'mai puțin',
      amount: Math.abs(difference).toFixed(2)
    }
  } else {
    return {
      trend: 'same',
      icon: '➖',
      percentage: '0',
      text: 'la fel',
      amount: '0'
    }
  }
})

// Methods
function addExpense() {
  const expense = {
    id: Date.now(),
    description: newExpense.value.description,
    amount: newExpense.value.amount,
    category: newExpense.value.category,
    date: new Date().toISOString()
  }

  expenses.value.push(expense)
  localStorage.setItem('bugetel-expenses', JSON.stringify(expenses.value))

  // Reset form
  newExpense.value = {
    description: '',
    amount: null,
    category: 'mancare'
  }
}

function deleteExpense(id) {
  expenses.value = expenses.value.filter(e => e.id !== id)
  localStorage.setItem('bugetel-expenses', JSON.stringify(expenses.value))
}

function addSavingsGoal() {
  const goal = {
    id: Date.now(),
    name: newGoal.value.name,
    target: newGoal.value.target,
    saved: 0
  }

  savingsGoals.value.push(goal)
  localStorage.setItem('bugetel-goals', JSON.stringify(savingsGoals.value))

  // Reset form
  newGoal.value = { name: '', target: null }
  showSavingsForm.value = false
}

function addToSavings(goal) {
  const amount = prompt(`Cât vrei să adaugi la "${goal.name}"?`)
  if (amount && !isNaN(amount)) {
    goal.saved = Math.min(goal.saved + parseFloat(amount), goal.target)
    localStorage.setItem('bugetel-goals', JSON.stringify(savingsGoals.value))
  }
}

function getCategoryIcon(category) {
  const icons = {
    mancare: '🍕',
    transport: '🚗',
    distractie: '🎉',
    cumparaturi: '🛍️',
    casa: '🏠',
    altele: '📦'
  }
  return icons[category] || '📦'
}

function formatDate(dateString) {
  const date = new Date(dateString)
  const today = new Date()
  const yesterday = new Date(today)
  yesterday.setDate(yesterday.getDate() - 1)

  if (date.toDateString() === today.toDateString()) {
    return 'Azi, ' + date.toLocaleTimeString('ro-RO', { hour: '2-digit', minute: '2-digit' })
  } else if (date.toDateString() === yesterday.toDateString()) {
    return 'Ieri, ' + date.toLocaleTimeString('ro-RO', { hour: '2-digit', minute: '2-digit' })
  } else {
    return date.toLocaleDateString('ro-RO', { day: 'numeric', month: 'short' })
  }
}

function toggleGroup(groupName) {
  expandedGroups.value[groupName] = !expandedGroups.value[groupName]
}
</script>

<style scoped>
.bugetel {
  padding-bottom: 2rem;
}

.header {
  text-align: center;
  color: white;
  margin-bottom: 2rem;
}

.header h1 {
  font-size: 2.5rem;
  margin-bottom: 0.5rem;
  text-shadow: 2px 2px 4px rgba(0,0,0,0.2);
}

.subtitle {
  font-size: 1.1rem;
  opacity: 0.9;
}

.card {
  background: white;
  border-radius: 16px;
  padding: 1.5rem;
  margin-bottom: 1rem;
  box-shadow: 0 4px 6px rgba(0,0,0,0.1);
}

.card h2 {
  font-size: 1.3rem;
  margin-bottom: 1rem;
  color: #333;
}

/* Add Expense Form */
.add-expense form {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.input {
  padding: 0.75rem;
  border: 2px solid #e0e0e0;
  border-radius: 8px;
  font-size: 1rem;
  transition: border-color 0.2s;
}

.input:focus {
  outline: none;
  border-color: #667eea;
}

.input-group {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 0.75rem;
}

.btn {
  padding: 0.75rem 1.5rem;
  border: none;
  border-radius: 8px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: transform 0.1s, box-shadow 0.2s;
}

.btn:active {
  transform: scale(0.98);
}

.btn-primary {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.4);
}

.btn-primary:hover {
  box-shadow: 0 6px 16px rgba(102, 126, 234, 0.5);
}

.btn-secondary {
  background: #f0f0f0;
  color: #333;
}

.btn-secondary:hover {
  background: #e0e0e0;
}

/* Summary Cards */
.summary-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1rem;
  margin-bottom: 1rem;
}

.summary-card {
  text-align: center;
  padding: 1.25rem;
}

.summary-label {
  font-size: 0.9rem;
  color: #666;
  margin-bottom: 0.5rem;
}

.summary-amount {
  font-size: 1.8rem;
  font-weight: bold;
  color: #667eea;
}

/* Monthly Chart Section */
.chart-section {
  margin-bottom: 1rem;
}

.chart-canvas {
  max-width: 100%;
  height: auto;
  margin-bottom: 1rem;
}

.month-comparison {
  margin-top: 1.5rem;
}

.comparison-card {
  padding: 1rem;
  border-radius: 12px;
  text-align: center;
  background: #f8f9fa;
}

.comparison-card.increase {
  background: linear-gradient(135deg, #ffe0e0 0%, #ffcccb 100%);
}

.comparison-card.decrease {
  background: linear-gradient(135deg, #d4edda 0%, #c3e6cb 100%);
}

.comparison-card.same {
  background: #f8f9fa;
}

.comparison-label {
  font-size: 0.9rem;
  color: #666;
  margin-bottom: 0.5rem;
  font-weight: 500;
}

.comparison-value {
  font-size: 1.5rem;
  font-weight: bold;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;
}

.comparison-card.increase .comparison-value {
  color: #dc3545;
}

.comparison-card.decrease .comparison-value {
  color: #28a745;
}

.comparison-card.same .comparison-value {
  color: #6c757d;
}

.comparison-icon {
  font-size: 1.8rem;
}

.comparison-text {
  font-size: 1rem;
  font-weight: 500;
  margin-left: 0.25rem;
}

/* Savings */
.savings-list {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.savings-item {
  padding: 1rem;
  background: #f8f9fa;
  border-radius: 12px;
}

.savings-info {
  display: flex;
  justify-content: space-between;
  margin-bottom: 0.5rem;
}

.savings-name {
  font-weight: 600;
  color: #333;
}

.savings-progress {
  color: #666;
  font-size: 0.9rem;
}

.progress-bar {
  height: 8px;
  background: #e0e0e0;
  border-radius: 4px;
  overflow: hidden;
  margin-bottom: 0.75rem;
}

.progress-fill {
  height: 100%;
  background: linear-gradient(90deg, #667eea 0%, #764ba2 100%);
  transition: width 0.3s;
}

.btn-small {
  padding: 0.5rem 1rem;
  background: #667eea;
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 0.9rem;
  cursor: pointer;
  font-weight: 600;
}

.btn-small:hover {
  background: #5568d3;
}

.savings-form {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.btn-group {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 0.75rem;
}

/* Expenses List */
.empty-state {
  text-align: center;
  color: #999;
  padding: 2rem;
  font-style: italic;
}

.expense-item {
  display: flex;
  align-items: center;
  padding: 1rem;
  border-bottom: 1px solid #f0f0f0;
  gap: 1rem;
}

.expense-item:last-child {
  border-bottom: none;
}

.expense-icon {
  font-size: 1.5rem;
}

.expense-details {
  flex: 1;
}

.expense-description {
  font-weight: 500;
  color: #333;
  margin-bottom: 0.25rem;
}

.expense-date {
  font-size: 0.85rem;
  color: #999;
}

.expense-amount {
  font-weight: 600;
  color: #667eea;
  font-size: 1.1rem;
}

.btn-delete {
  background: #ff4444;
  color: white;
  border: none;
  width: 28px;
  height: 28px;
  border-radius: 50%;
  font-size: 1.5rem;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  line-height: 1;
  padding: 0;
}

.btn-delete:hover {
  background: #cc0000;
}

/* Spending Analysis - Grouped Expenses */
.spending-analysis {
  margin-bottom: 1rem;
}

.grouped-item {
  border-bottom: 1px solid #f0f0f0;
  padding: 1rem 0;
}

.grouped-item:last-child {
  border-bottom: none;
}

.group-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  cursor: pointer;
  padding: 0.5rem;
  border-radius: 8px;
  transition: background-color 0.2s;
}

.group-header:hover {
  background-color: #f8f9fa;
}

.group-main {
  display: flex;
  align-items: center;
  gap: 1rem;
  flex: 1;
}

.group-icon {
  font-size: 1.5rem;
}

.group-info {
  flex: 1;
}

.group-name {
  font-weight: 600;
  color: #333;
  font-size: 1.05rem;
  margin-bottom: 0.25rem;
}

.group-stats {
  font-size: 0.85rem;
  color: #666;
}

.group-toggle {
  color: #667eea;
  font-size: 1.2rem;
  font-weight: bold;
  padding: 0.5rem;
}

.group-details {
  margin-top: 0.75rem;
  margin-left: 3.5rem;
  padding: 1rem;
  background: #f8f9fa;
  border-radius: 8px;
}

.transaction-item {
  display: flex;
  justify-content: space-between;
  padding: 0.5rem 0;
  border-bottom: 1px solid #e0e0e0;
}

.transaction-item:last-child {
  border-bottom: none;
}

.transaction-date {
  color: #666;
  font-size: 0.9rem;
}

.transaction-amount {
  color: #667eea;
  font-weight: 600;
}

/* Mobile responsiveness */
@media (max-width: 640px) {
  .summary-grid {
    grid-template-columns: 1fr;
  }

  .input-group {
    grid-template-columns: 1fr;
  }

  .group-details {
    margin-left: 1rem;
  }
}
</style>
