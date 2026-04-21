<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Budget-aware restocking recommendations based on demand forecasts</p>
    </div>

    <div v-if="orderSuccess" class="success-banner">
      Order placed successfully. Check the Orders tab for tracking.
    </div>

    <div class="card budget-panel">
      <div class="budget-label">Available Budget</div>
      <div class="slider-row">
        <input
          type="range"
          min="0"
          max="50000"
          step="500"
          v-model.number="budget"
          class="budget-slider"
        />
        <span class="budget-value">${{ budget.toLocaleString() }}</span>
      </div>
      <div class="budget-stats">
        <div class="budget-stat">
          <span class="budget-stat-label">Selected:</span>
          <span class="budget-stat-value">
            ${{ selectedCost.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}
          </span>
        </div>
        <div class="budget-stat">
          <span class="budget-stat-label">Remaining:</span>
          <span class="budget-stat-value" :class="{ 'remaining-negative': remainingBudget < 0 }">
            ${{ remainingBudget.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}
          </span>
        </div>
      </div>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else-if="recommendations.length === 0" class="empty-state">
      No restocking recommendations available.
    </div>
    <div v-else>
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommendations</h3>
        </div>
        <div class="table-container">
          <table class="data-table">
            <thead>
              <tr>
                <th class="col-check"></th>
                <th>Item</th>
                <th>SKU</th>
                <th>Warehouse</th>
                <th>Current Stock</th>
                <th>Forecast</th>
                <th>Restock Qty</th>
                <th>Unit Cost</th>
                <th>Total Cost</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendations"
                :key="item.item_sku"
                :class="{ 'row-disabled': isDisabled(item) }"
                @click="toggleItem(item)"
                class="recommendation-row"
              >
                <td class="col-check">
                  <input
                    type="checkbox"
                    :checked="selectedSkus.has(item.item_sku)"
                    :disabled="isDisabled(item)"
                    @click.stop="toggleItem(item)"
                  />
                </td>
                <td>{{ item.item_name }}</td>
                <td class="sku-cell">{{ item.item_sku }}</td>
                <td>{{ item.warehouse }}</td>
                <td>{{ item.current_stock != null ? item.current_stock.toLocaleString() : '-' }}</td>
                <td>{{ item.demand_forecast != null ? item.demand_forecast.toLocaleString() : '-' }}</td>
                <td>{{ item.restock_quantity.toLocaleString() }}</td>
                <td>{{ formatCurrency(item.unit_cost) }}</td>
                <td>
                  {{ formatCurrency(item.total_cost) }}
                  <span v-if="isDisabled(item)" class="over-budget-badge">Over budget</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <div v-if="selectedSkus.size > 0" class="order-summary-bar">
        <span class="summary-text">
          {{ selectedSkus.size }} item(s) selected &middot;
          Total: ${{ selectedCost.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }} &middot;
          Lead Time: 14 days
        </span>
        <button
          class="place-order-btn"
          :disabled="placing"
          @click="placeOrder"
        >
          {{ placing ? 'Placing...' : 'Place Order' }}
        </button>
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
    const budget = ref(10000)
    const recommendations = ref([])
    const selectedSkus = ref(new Set())
    const loading = ref(true)
    const error = ref(null)
    const placing = ref(false)
    const orderSuccess = ref(false)

    const selectedCost = computed(() =>
      recommendations.value
        .filter(r => selectedSkus.value.has(r.item_sku))
        .reduce((sum, r) => sum + r.total_cost, 0)
    )

    const remainingBudget = computed(() => budget.value - selectedCost.value)

    function isDisabled(item) {
      return !selectedSkus.value.has(item.item_sku) && item.total_cost > remainingBudget.value
    }

    function toggleItem(item) {
      if (isDisabled(item)) return
      const next = new Set(selectedSkus.value)
      if (next.has(item.item_sku)) next.delete(item.item_sku)
      else next.add(item.item_sku)
      selectedSkus.value = next
    }

    async function placeOrder() {
      placing.value = true
      try {
        const selected = recommendations.value.filter(r => selectedSkus.value.has(r.item_sku))
        for (const item of selected) {
          await api.createRestockingOrder({
            item_sku: item.item_sku,
            item_name: item.item_name,
            restock_quantity: item.restock_quantity,
            unit_cost: item.unit_cost,
            demand_forecast_id: item.demand_forecast_id,
          })
        }
        selectedSkus.value = new Set()
        orderSuccess.value = true
        setTimeout(() => { orderSuccess.value = false }, 4000)
      } finally {
        placing.value = false
      }
    }

    function formatCurrency(value) {
      return value.toLocaleString('en-US', { style: 'currency', currency: 'USD' })
    }

    onMounted(async () => {
      try {
        recommendations.value = await api.getRestockingRecommendations()
      } catch (err) {
        error.value = 'Failed to load recommendations'
      } finally {
        loading.value = false
      }
    })

    return {
      budget,
      recommendations,
      selectedSkus,
      loading,
      error,
      placing,
      orderSuccess,
      selectedCost,
      remainingBudget,
      isDisabled,
      toggleItem,
      placeOrder,
      formatCurrency
    }
  }
}
</script>

<style scoped>
.restocking {
  padding: 0;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 0.875rem 1.25rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-size: 0.938rem;
  font-weight: 500;
}

.budget-panel {
  margin-bottom: 1.25rem;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  margin-bottom: 0.75rem;
}

.slider-row {
  display: flex;
  align-items: center;
  gap: 1rem;
  margin-bottom: 0.75rem;
}

.budget-slider {
  flex: 1;
  height: 6px;
  accent-color: #2563eb;
  cursor: pointer;
}

.budget-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  min-width: 120px;
  text-align: right;
  letter-spacing: -0.025em;
}

.budget-stats {
  display: flex;
  gap: 2rem;
  padding-top: 0.75rem;
  border-top: 1px solid #e2e8f0;
}

.budget-stat {
  display: flex;
  gap: 0.5rem;
  align-items: center;
}

.budget-stat-label {
  font-size: 0.875rem;
  color: #64748b;
  font-weight: 500;
}

.budget-stat-value {
  font-size: 0.938rem;
  font-weight: 600;
  color: #0f172a;
}

.remaining-negative {
  color: #dc2626;
}

.empty-state {
  text-align: center;
  padding: 3rem;
  color: #64748b;
  font-size: 0.938rem;
}

.data-table {
  width: 100%;
  border-collapse: collapse;
}

.col-check {
  width: 40px;
  text-align: center;
}

.sku-cell {
  font-family: 'Courier New', monospace;
  font-size: 0.813rem;
  color: #64748b;
}

.recommendation-row {
  cursor: pointer;
}

.recommendation-row.row-disabled {
  opacity: 0.45;
  cursor: default;
}

.over-budget-badge {
  display: inline-block;
  margin-left: 0.5rem;
  padding: 0.125rem 0.5rem;
  background: #fecaca;
  color: #991b1b;
  border-radius: 4px;
  font-size: 0.688rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.025em;
}

.order-summary-bar {
  background: #0f172a;
  color: #f8fafc;
  border-radius: 10px;
  padding: 1rem 1.5rem;
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-top: 1.25rem;
}

.summary-text {
  font-size: 0.938rem;
  font-weight: 500;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.625rem 1.5rem;
  border-radius: 6px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}
</style>
