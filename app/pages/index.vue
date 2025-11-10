<template>
  <div class="bugetel">
    <header class="header">
      <h1>🛒 Bugetel</h1>
      <p class="subtitle">Tracker prețuri & alimente</p>
    </header>

    <!-- Quick Add Product -->
    <div class="card add-product">
      <h2>➕ Adaugă produs</h2>
      <form @submit.prevent="addProduct">
        <input
          v-model="newProduct.name"
          type="text"
          placeholder="Nume produs (ex: Lapte 1L)"
          required
          class="input"
        />
        <div class="input-group">
          <input
            v-model.number="newProduct.price"
            type="number"
            step="0.01"
            placeholder="Preț (lei)"
            required
            class="input"
          />
          <input
            v-model="newProduct.store"
            type="text"
            placeholder="Magazin (ex: Kaufland)"
            class="input"
          />
        </div>
        <input
          v-model="newProduct.quantity"
          type="text"
          placeholder="Cantitate (opțional, ex: 1kg, 500g)"
          class="input"
        />
        <button type="submit" class="btn btn-primary">Adaugă</button>
      </form>
    </div>

    <!-- Receipt Scanner -->
    <div class="card scanner-section">
      <h2>📸 Scanează bon fiscal</h2>
      <p class="scanner-description">
        Încarcă o poză cu bonul pentru a adăuga automat produsele
      </p>
      <div class="scanner-controls">
        <input
          ref="fileInput"
          type="file"
          accept="image/*"
          @change="handleImageUpload"
          class="file-input"
          id="receipt-upload"
        />
        <label for="receipt-upload" class="btn btn-secondary">
          📷 Alege poză bon
        </label>
        <button
          v-if="isScanning"
          disabled
          class="btn btn-secondary"
        >
          ⏳ Se procesează...
        </button>
      </div>
      <div v-if="scannedImage" class="preview-section">
        <img :src="scannedImage" alt="Receipt preview" class="receipt-preview" />
        <p class="preview-hint">Se scanează bonul...</p>
      </div>
    </div>

    <!-- Summary Cards -->
    <div class="summary-grid">
      <div class="card summary-card">
        <div class="summary-label">Săptămâna asta</div>
        <div class="summary-amount">{{ weekExpenses }} lei</div>
        <div class="summary-count">{{ weekProductsCount }} produse</div>
      </div>
      <div class="card summary-card">
        <div class="summary-label">Luna asta</div>
        <div class="summary-amount">{{ monthExpenses }} lei</div>
        <div class="summary-count">{{ monthProductsCount }} produse</div>
      </div>
    </div>

    <!-- Price Evolution Chart -->
    <div class="card chart-section" v-if="products.length > 0">
      <h2>📈 Evoluție cheltuieli lunare</h2>
      <canvas ref="monthlyChart" class="chart-canvas"></canvas>
    </div>

    <!-- Product Price History - Grouped -->
    <div class="card products-analysis">
      <h2>🏷️ Istoric prețuri produse</h2>
      <div v-if="groupedProducts.length === 0" class="empty-state">
        Niciun produs încă. Adaugă primul!
      </div>
      <div v-else>
        <div v-for="group in groupedProducts" :key="group.name" class="product-group">
          <div class="group-header" @click="toggleGroup(group.name)">
            <div class="group-main">
              <div class="product-icon">🛒</div>
              <div class="group-info">
                <div class="group-name">{{ group.name }}</div>
                <div class="group-stats">
                  Cumpărat {{ group.count }}x •
                  Ultimul preț: {{ group.lastPrice }} lei
                  <span v-if="group.priceChange" :class="'price-trend ' + group.priceChange.trend">
                    {{ group.priceChange.icon }} {{ group.priceChange.text }}
                  </span>
                </div>
              </div>
            </div>
            <div class="group-toggle">
              {{ expandedGroups[group.name] ? '▼' : '▶' }}
            </div>
          </div>

          <div v-if="expandedGroups[group.name]" class="group-details">
            <div class="price-stats">
              <div class="stat-item">
                <span class="stat-label">Preț minim:</span>
                <span class="stat-value">{{ group.minPrice }} lei</span>
              </div>
              <div class="stat-item">
                <span class="stat-label">Preț maxim:</span>
                <span class="stat-value">{{ group.maxPrice }} lei</span>
              </div>
              <div class="stat-item">
                <span class="stat-label">Preț mediu:</span>
                <span class="stat-value">{{ group.avgPrice }} lei</span>
              </div>
            </div>
            <div class="purchase-history">
              <h4>Istoric achiziții:</h4>
              <div v-for="purchase in group.purchases" :key="purchase.id" class="purchase-item">
                <div class="purchase-info">
                  <div class="purchase-date">{{ formatDate(purchase.date) }}</div>
                  <div class="purchase-store" v-if="purchase.store">
                    📍 {{ purchase.store }}
                  </div>
                </div>
                <div class="purchase-price">{{ purchase.price }} lei</div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Recent Products -->
    <div class="card recent-list">
      <h2>📝 Produse recente</h2>
      <div v-if="products.length === 0" class="empty-state">
        Niciun produs încă. Începe să adaugi!
      </div>
      <div v-for="product in recentProducts" :key="product.id" class="product-item">
        <div class="product-icon">🛒</div>
        <div class="product-details">
          <div class="product-name">{{ product.name }}</div>
          <div class="product-meta">
            <span class="product-date">{{ formatDate(product.date) }}</span>
            <span v-if="product.store" class="product-store">• {{ product.store }}</span>
            <span v-if="product.quantity" class="product-quantity">• {{ product.quantity }}</span>
          </div>
        </div>
        <div class="product-price">{{ product.price }} lei</div>
        <button @click="deleteProduct(product.id)" class="btn-delete">×</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, watch, nextTick } from 'vue'
import { Chart, registerables } from 'chart.js'
import Tesseract from 'tesseract.js'

// Register Chart.js components
Chart.register(...registerables)

// State
const products = ref([])
const expandedGroups = ref({})
const monthlyChart = ref(null)
const fileInput = ref(null)
const scannedImage = ref(null)
const isScanning = ref(false)
let chartInstance = null

const newProduct = ref({
  name: '',
  price: null,
  store: '',
  quantity: ''
})

// Load data from localStorage
onMounted(() => {
  const savedProducts = localStorage.getItem('bugetel-products')

  if (savedProducts) {
    products.value = JSON.parse(savedProducts)
  }

  // Create chart after data is loaded
  nextTick(() => {
    createChart()
  })
})

// Watch for changes in products to update chart
watch(() => products.value.length, () => {
  nextTick(() => {
    updateChart()
  })
})

// Computed
const weekExpenses = computed(() => {
  const now = new Date()
  const weekAgo = new Date(now.getTime() - 7 * 24 * 60 * 60 * 1000)

  return products.value
    .filter(p => new Date(p.date) >= weekAgo)
    .reduce((sum, p) => sum + p.price, 0)
    .toFixed(2)
})

const weekProductsCount = computed(() => {
  const now = new Date()
  const weekAgo = new Date(now.getTime() - 7 * 24 * 60 * 60 * 1000)

  return products.value.filter(p => new Date(p.date) >= weekAgo).length
})

const monthExpenses = computed(() => {
  const now = new Date()
  const currentMonth = now.getMonth()
  const currentYear = now.getFullYear()

  return products.value
    .filter(p => {
      const d = new Date(p.date)
      return d.getMonth() === currentMonth && d.getFullYear() === currentYear
    })
    .reduce((sum, p) => sum + p.price, 0)
    .toFixed(2)
})

const monthProductsCount = computed(() => {
  const now = new Date()
  const currentMonth = now.getMonth()
  const currentYear = now.getFullYear()

  return products.value.filter(p => {
    const d = new Date(p.date)
    return d.getMonth() === currentMonth && d.getFullYear() === currentYear
  }).length
})

const recentProducts = computed(() => {
  return [...products.value]
    .sort((a, b) => new Date(b.date) - new Date(a.date))
    .slice(0, 15)
})

const groupedProducts = computed(() => {
  const groups = {}

  products.value.forEach(product => {
    const key = product.name.toLowerCase().trim()

    if (!groups[key]) {
      groups[key] = {
        name: product.name,
        count: 0,
        purchases: [],
        prices: []
      }
    }

    groups[key].count += 1
    groups[key].purchases.push({
      id: product.id,
      date: product.date,
      price: product.price,
      store: product.store,
      quantity: product.quantity
    })
    groups[key].prices.push(product.price)
  })

  // Calculate stats for each product
  return Object.values(groups)
    .map(group => {
      const sortedPurchases = group.purchases.sort((a, b) => new Date(b.date) - new Date(a.date))
      const lastPrice = sortedPurchases[0].price
      const previousPrice = sortedPurchases.length > 1 ? sortedPurchases[1].price : null

      let priceChange = null
      if (previousPrice) {
        const diff = lastPrice - previousPrice
        const percentage = Math.abs((diff / previousPrice) * 100).toFixed(1)

        if (diff > 0) {
          priceChange = {
            trend: 'increase',
            icon: '📈',
            text: `+${percentage}%`
          }
        } else if (diff < 0) {
          priceChange = {
            trend: 'decrease',
            icon: '📉',
            text: `-${percentage}%`
          }
        }
      }

      return {
        ...group,
        purchases: sortedPurchases,
        lastPrice: lastPrice.toFixed(2),
        minPrice: Math.min(...group.prices).toFixed(2),
        maxPrice: Math.max(...group.prices).toFixed(2),
        avgPrice: (group.prices.reduce((a, b) => a + b, 0) / group.prices.length).toFixed(2),
        priceChange
      }
    })
    .sort((a, b) => b.count - a.count)
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
  products.value.forEach(product => {
    const productDate = new Date(product.date)
    const monthKey = `${productDate.getFullYear()}-${String(productDate.getMonth() + 1).padStart(2, '0')}`

    const monthData = months.find(m => m.key === monthKey)
    if (monthData) {
      monthData.total += product.price
    }
  })

  return months
})

// Methods
function addProduct() {
  const product = {
    id: Date.now(),
    name: newProduct.value.name,
    price: newProduct.value.price,
    store: newProduct.value.store,
    quantity: newProduct.value.quantity,
    date: new Date().toISOString()
  }

  products.value.push(product)
  localStorage.setItem('bugetel-products', JSON.stringify(products.value))

  // Reset form
  newProduct.value = {
    name: '',
    price: null,
    store: '',
    quantity: ''
  }
}

function deleteProduct(id) {
  products.value = products.value.filter(p => p.id !== id)
  localStorage.setItem('bugetel-products', JSON.stringify(products.value))
}

async function handleImageUpload(event) {
  const file = event.target.files[0]
  if (!file) return

  isScanning.value = true

  // Create preview
  const reader = new FileReader()
  reader.onload = (e) => {
    scannedImage.value = e.target.result
  }
  reader.readAsDataURL(file)

  try {
    // Perform OCR
    const result = await Tesseract.recognize(file, 'ron', {
      logger: (m) => console.log(m)
    })

    const text = result.data.text
    console.log('OCR Result:', text)

    // Parse the receipt text
    parseReceipt(text)

    // Clear preview after 3 seconds
    setTimeout(() => {
      scannedImage.value = null
      isScanning.value = false
    }, 3000)
  } catch (error) {
    console.error('OCR Error:', error)
    alert('Eroare la scanarea bonului. Încearcă din nou sau adaugă manual.')
    isScanning.value = false
    scannedImage.value = null
  }
}

function parseReceipt(text) {
  // Simple parser for Romanian receipts
  // This is a basic implementation - can be improved based on actual receipt formats
  const lines = text.split('\n')
  let detectedStore = ''
  const parsedProducts = []

  // Try to detect store name (usually in first few lines)
  const storePatterns = ['KAUFLAND', 'CARREFOUR', 'LIDL', 'MEGA IMAGE', 'PENNY', 'AUCHAN', 'PROFI']
  for (const line of lines.slice(0, 5)) {
    const upperLine = line.toUpperCase()
    for (const store of storePatterns) {
      if (upperLine.includes(store)) {
        detectedStore = store
        break
      }
    }
    if (detectedStore) break
  }

  // Parse product lines - look for price patterns
  const pricePattern = /(\d+[.,]\d{2})\s*(?:LEI|RON)?/i

  for (const line of lines) {
    const priceMatch = line.match(pricePattern)
    if (priceMatch) {
      const price = parseFloat(priceMatch[1].replace(',', '.'))

      // Extract product name (text before price)
      const productName = line.substring(0, line.indexOf(priceMatch[0])).trim()

      if (productName && price > 0 && price < 1000) { // Sanity check
        parsedProducts.push({
          name: productName,
          price: price,
          store: detectedStore
        })
      }
    }
  }

  // Add parsed products
  if (parsedProducts.length > 0) {
    const now = new Date().toISOString()
    parsedProducts.forEach(prod => {
      products.value.push({
        id: Date.now() + Math.random(),
        name: prod.name,
        price: prod.price,
        store: prod.store,
        quantity: '',
        date: now
      })
    })
    localStorage.setItem('bugetel-products', JSON.stringify(products.value))
    alert(`✅ Am adăugat ${parsedProducts.length} produse din bon!`)
  } else {
    alert('Nu am putut detecta produse în bon. Încearcă să adaugi manual.')
  }
}

function toggleGroup(groupName) {
  expandedGroups.value[groupName] = !expandedGroups.value[groupName]
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
        label: 'Cheltuieli alimente (lei)',
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

/* Add Product Form */
.add-product form {
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

.btn-secondary:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

/* Scanner Section */
.scanner-section {
  background: linear-gradient(135deg, #f5f7fa 0%, #e8ecf3 100%);
}

.scanner-description {
  color: #666;
  margin-bottom: 1rem;
  font-size: 0.95rem;
}

.scanner-controls {
  display: flex;
  gap: 1rem;
  flex-wrap: wrap;
}

.file-input {
  display: none;
}

label[for="receipt-upload"] {
  display: inline-block;
}

.preview-section {
  margin-top: 1rem;
  text-align: center;
}

.receipt-preview {
  max-width: 100%;
  max-height: 300px;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.preview-hint {
  margin-top: 0.5rem;
  color: #667eea;
  font-weight: 600;
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

.summary-count {
  font-size: 0.85rem;
  color: #999;
  margin-top: 0.25rem;
}

/* Chart Section */
.chart-section {
  margin-bottom: 1rem;
}

.chart-canvas {
  max-width: 100%;
  height: auto;
  margin-bottom: 1rem;
}

/* Products Analysis - Grouped */
.products-analysis {
  margin-bottom: 1rem;
}

.product-group {
  border-bottom: 1px solid #f0f0f0;
  padding: 1rem 0;
}

.product-group:last-child {
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

.product-icon {
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

.price-trend {
  margin-left: 0.5rem;
  font-weight: 600;
}

.price-trend.increase {
  color: #dc3545;
}

.price-trend.decrease {
  color: #28a745;
}

.group-toggle {
  color: #667eea;
  font-size: 1.2rem;
  font-weight: bold;
  padding: 0.5rem;
}

.group-details {
  margin-top: 1rem;
  margin-left: 3.5rem;
  padding: 1rem;
  background: #f8f9fa;
  border-radius: 8px;
}

.price-stats {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(150px, 1fr));
  gap: 1rem;
  margin-bottom: 1rem;
  padding-bottom: 1rem;
  border-bottom: 1px solid #e0e0e0;
}

.stat-item {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.stat-label {
  font-size: 0.85rem;
  color: #666;
}

.stat-value {
  font-size: 1.1rem;
  font-weight: 600;
  color: #667eea;
}

.purchase-history h4 {
  font-size: 0.9rem;
  color: #666;
  margin-bottom: 0.75rem;
}

.purchase-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.75rem;
  border-bottom: 1px solid #e0e0e0;
  background: white;
  border-radius: 6px;
  margin-bottom: 0.5rem;
}

.purchase-item:last-child {
  margin-bottom: 0;
}

.purchase-info {
  flex: 1;
}

.purchase-date {
  font-size: 0.9rem;
  color: #333;
  margin-bottom: 0.25rem;
}

.purchase-store {
  font-size: 0.85rem;
  color: #666;
}

.purchase-price {
  font-weight: 600;
  color: #667eea;
  font-size: 1.05rem;
}

/* Recent Products */
.empty-state {
  text-align: center;
  color: #999;
  padding: 2rem;
  font-style: italic;
}

.product-item {
  display: flex;
  align-items: center;
  padding: 1rem;
  border-bottom: 1px solid #f0f0f0;
  gap: 1rem;
}

.product-item:last-child {
  border-bottom: none;
}

.product-details {
  flex: 1;
}

.product-name {
  font-weight: 500;
  color: #333;
  margin-bottom: 0.25rem;
}

.product-meta {
  font-size: 0.85rem;
  color: #999;
}

.product-price {
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

  .price-stats {
    grid-template-columns: 1fr;
  }
}
</style>
