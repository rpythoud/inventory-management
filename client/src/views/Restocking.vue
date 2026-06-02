<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Recommendations</h2>
      <p>Budget-based restocking plan derived from demand forecasts</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <!-- Budget Slider -->
      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">Budget</h3>
          <span class="budget-display">${{ budget.toLocaleString() }}</span>
        </div>
        <div class="slider-wrapper">
          <span class="slider-label">$0</span>
          <input
            type="range"
            class="budget-slider"
            min="0"
            max="50000"
            step="500"
            v-model.number="budget"
          />
          <span class="slider-label">$50,000</span>
        </div>
      </div>

      <!-- Stat Cards -->
      <div class="stats-grid">
        <div class="stat-card info">
          <div class="stat-label">Budget</div>
          <div class="stat-value">${{ budget.toLocaleString() }}</div>
        </div>
        <div class="stat-card success">
          <div class="stat-label">Items Recommended</div>
          <div class="stat-value">{{ recommendedItems.length }}</div>
        </div>
        <div class="stat-card warning">
          <div class="stat-label">Total Cost</div>
          <div class="stat-value">${{ totalCost.toLocaleString() }}</div>
        </div>
        <div class="stat-card info">
          <div class="stat-label">Budget Remaining</div>
          <div class="stat-value">${{ budgetRemaining.toLocaleString() }}</div>
        </div>
      </div>

      <!-- Success Message -->
      <div v-if="orderSuccess" class="success-message">
        <div class="success-icon">&#10003;</div>
        <div class="success-content">
          <strong>Order placed successfully.</strong>
          Order ID: <code>{{ placedOrderId }}</code>
          <span class="view-orders-hint">View in Orders tab</span>
        </div>
      </div>

      <!-- Recommendations Table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommendations ({{ sortedForecasts.length }} items)</h3>
          <button
            class="place-order-btn"
            :disabled="recommendedItems.length === 0 || orderSuccess || submitting"
            @click="placeOrder"
          >
            {{ submitting ? 'Placing Order...' : 'Place Order' }}
          </button>
        </div>
        <div class="table-container">
          <table class="restocking-table">
            <thead>
              <tr>
                <th>SKU</th>
                <th>Item Name</th>
                <th>Forecasted Demand</th>
                <th>Unit Cost</th>
                <th>Recommended Qty</th>
                <th>Estimated Cost</th>
                <th>Status</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in sortedForecasts"
                :key="item.id"
                :class="{ 'row-over-budget': isOverBudget(item) }"
              >
                <td><strong>{{ item.item_sku }}</strong></td>
                <td :class="{ 'text-muted': isOverBudget(item) }">{{ item.item_name }}</td>
                <td :class="{ 'text-muted': isOverBudget(item) }">{{ item.forecasted_demand }}</td>
                <td :class="{ 'text-muted': isOverBudget(item) }">${{ item.unit_cost.toLocaleString() }}</td>
                <td :class="{ 'text-muted': isOverBudget(item) }">{{ item.forecasted_demand }}</td>
                <td :class="{ 'text-muted': isOverBudget(item) }">${{ (item.unit_cost * item.forecasted_demand).toLocaleString() }}</td>
                <td>
                  <span v-if="isOverBudget(item)" class="badge danger over-budget-badge">Over budget</span>
                  <span v-else class="badge success">Included</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>

        <!-- Summary Bar -->
        <div class="summary-bar">
          <span>Total: <strong>${{ totalCost.toLocaleString() }}</strong></span>
          <span class="summary-divider">|</span>
          <span>Budget remaining: <strong>${{ budgetRemaining.toLocaleString() }}</strong></span>
        </div>
      </div>

    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const loading = ref(true)
    const error = ref(null)
    const forecasts = ref([])
    const budget = ref(25000)
    const submitting = ref(false)
    const orderSuccess = ref(false)
    const placedOrderId = ref(null)

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const data = await api.getDemandForecasts()
        forecasts.value = data
      } catch (err) {
        error.value = 'Failed to load demand forecasts: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    // Sorted by forecasted_demand descending
    const sortedForecasts = computed(() => {
      return [...forecasts.value].sort((a, b) => b.forecasted_demand - a.forecasted_demand)
    })

    // Greedy budget allocation
    const recommendedItems = computed(() => {
      let budgetUsed = 0
      const result = []
      for (const item of sortedForecasts.value) {
        const itemCost = item.forecasted_demand * item.unit_cost
        if (budgetUsed + itemCost <= budget.value) {
          budgetUsed += itemCost
          result.push(item)
        }
      }
      return result
    })

    const recommendedSet = computed(() => new Set(recommendedItems.value.map(i => i.id)))

    const isOverBudget = (item) => !recommendedSet.value.has(item.id)

    const totalCost = computed(() =>
      recommendedItems.value.reduce((sum, item) => sum + item.forecasted_demand * item.unit_cost, 0)
    )

    const budgetRemaining = computed(() => budget.value - totalCost.value)

    const placeOrder = async () => {
      if (recommendedItems.value.length === 0 || orderSuccess.value || submitting.value) return
      submitting.value = true
      error.value = null
      try {
        const payload = {
          items: recommendedItems.value.map(item => ({
            sku: item.item_sku,
            name: item.item_name,
            quantity: item.forecasted_demand,
            unit_cost: item.unit_cost,
            estimated_cost: item.forecasted_demand * item.unit_cost
          })),
          total_cost: totalCost.value,
          budget: budget.value
        }
        const response = await api.submitRestockingOrder(payload)
        placedOrderId.value = response.id || response.order_id || 'RST-' + Date.now()
        orderSuccess.value = true
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadData)

    return {
      loading,
      error,
      budget,
      submitting,
      orderSuccess,
      placedOrderId,
      sortedForecasts,
      recommendedItems,
      totalCost,
      budgetRemaining,
      isOverBudget,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-card .card-header {
  align-items: center;
}

.budget-display {
  font-size: 1.5rem;
  font-weight: 700;
  color: #2563eb;
  letter-spacing: -0.025em;
}

.slider-wrapper {
  display: flex;
  align-items: center;
  gap: 1rem;
  padding: 0.5rem 0;
}

.slider-label {
  font-size: 0.813rem;
  color: #64748b;
  font-weight: 500;
  white-space: nowrap;
}

.budget-slider {
  flex: 1;
  height: 6px;
  accent-color: #2563eb;
  cursor: pointer;
}

/* Table */
.restocking-table {
  table-layout: fixed;
  width: 100%;
}

.row-over-budget {
  opacity: 0.45;
  background: #f8fafc;
}

.text-muted {
  color: #94a3b8;
}

.over-budget-badge {
  white-space: nowrap;
}

/* Summary bar */
.summary-bar {
  display: flex;
  align-items: center;
  gap: 1rem;
  padding: 0.875rem 0.75rem;
  border-top: 1px solid #e2e8f0;
  font-size: 0.938rem;
  color: #334155;
  background: #f8fafc;
  border-radius: 0 0 6px 6px;
}

.summary-divider {
  color: #cbd5e1;
}

/* Place Order button */
.place-order-btn {
  padding: 0.5rem 1.25rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

/* Success message */
.success-message {
  display: flex;
  align-items: flex-start;
  gap: 0.875rem;
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  border-radius: 8px;
  padding: 1rem 1.25rem;
  margin-bottom: 1.25rem;
  color: #065f46;
}

.success-icon {
  font-size: 1.125rem;
  font-weight: 700;
  flex-shrink: 0;
  margin-top: 0.1rem;
}

.success-content {
  font-size: 0.938rem;
  line-height: 1.5;
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.success-content code {
  font-family: 'Menlo', 'Monaco', 'Consolas', monospace;
  font-size: 0.875rem;
  background: #a7f3d0;
  padding: 0.1rem 0.4rem;
  border-radius: 4px;
}

.view-orders-hint {
  font-size: 0.813rem;
  color: #047857;
  font-style: italic;
}
</style>
