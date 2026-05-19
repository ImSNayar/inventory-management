<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Budget-based restock recommendations from demand forecasts</p>
    </div>

    <div v-if="successOrder" class="success-banner">
      Order {{ successOrder.order_number }} placed — delivery expected in 14 days.
      <router-link to="/orders">View in Orders tab</router-link>
      <button @click="successOrder = null">Dismiss</button>
    </div>

    <div v-if="submitError" class="error">{{ submitError }}</div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Budget</h3>
        </div>
        <div class="budget-section">
          <div class="budget-label">Available Budget</div>
          <input
            type="range"
            min="0"
            max="500000"
            step="1000"
            v-model.number="budget"
            class="budget-slider"
          />
          <div class="budget-display">${{ budget.toLocaleString() }}</div>
          <div class="budget-stats">
            <div class="budget-stat">
              <span class="budget-stat-label">Items Recommended</span>
              <span class="budget-stat-value">{{ includedItems.length }}</span>
            </div>
            <div class="budget-stat">
              <span class="budget-stat-label">Total Cost</span>
              <span class="budget-stat-value">${{ totalCost.toLocaleString() }}</span>
            </div>
            <div class="budget-stat">
              <span class="budget-stat-label">Remaining Budget</span>
              <span :class="['budget-stat-value', remainingBudget < 0 ? 'over' : 'ok']">
                ${{ remainingBudget.toLocaleString() }}
              </span>
            </div>
          </div>
        </div>
      </div>

      <!-- Recommendations Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommendations</h3>
          <button
            class="place-order-btn"
            :disabled="!canPlaceOrder"
            @click="placeOrder"
          >
            {{ submitting ? 'Placing Order...' : 'Place Order' }}
          </button>
        </div>

        <div v-if="restockItems.length === 0" class="loading">
          No items need restocking based on current demand data.
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>Trend</th>
                <th>SKU</th>
                <th>Item Name</th>
                <th>Current</th>
                <th>Forecasted</th>
                <th>Qty</th>
                <th>Unit Cost</th>
                <th>Row Total</th>
                <th>Include</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in restockItems"
                :key="item.item_sku"
                :class="{ 'row-included': item.included }"
              >
                <td>
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td>{{ item.item_sku }}</td>
                <td>{{ item.item_name }}</td>
                <td>{{ item.current_demand }}</td>
                <td>{{ item.forecasted_demand }}</td>
                <td>
                  <input
                    type="number"
                    :min="1"
                    v-model.number="item.qty"
                    @change="onQtyChange(item)"
                    class="qty-input"
                  />
                </td>
                <td>${{ item.unit_cost.toFixed(2) }}</td>
                <td>${{ (item.qty * item.unit_cost).toLocaleString() }}</td>
                <td>
                  <input
                    type="checkbox"
                    v-model="item.included"
                    @change="onIncludeChange(item)"
                  />
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const budget = ref(100000)
    const demandForecasts = ref([])
    const inventoryMap = ref({})
    const restockItems = ref([])
    const submitting = ref(false)
    const successOrder = ref(null)
    const submitError = ref(null)
    const loading = ref(false)
    const error = ref(null)

    const includedItems = computed(() => restockItems.value.filter(i => i.included))

    const totalCost = computed(() =>
      includedItems.value.reduce((sum, i) => sum + i.qty * i.unit_cost, 0)
    )

    const remainingBudget = computed(() => budget.value - totalCost.value)

    const canPlaceOrder = computed(
      () => includedItems.value.length > 0 && budget.value > 0 && !submitting.value
    )

    // Greedy budget allocation: iterate items in priority order, include while budget allows.
    // Does NOT reset user-edited quantities — only toggles the included flag.
    const runGreedy = () => {
      let runningTotal = 0
      for (const item of restockItems.value) {
        const cost = item.qty * item.unit_cost
        if (runningTotal + cost <= budget.value) {
          item.included = true
          runningTotal += cost
        } else {
          item.included = false
        }
      }
    }

    watch(budget, runGreedy)

    const buildRestockItems = () => {
      const trendPriority = { increasing: 0, stable: 1 }

      const filtered = demandForecasts.value
        .filter(f => f.forecasted_demand > f.current_demand)
        .filter(f => inventoryMap.value[f.item_sku] !== undefined)

      const sorted = [...filtered].sort((a, b) => {
        const pa = trendPriority[a.trend] !== undefined ? trendPriority[a.trend] : 2
        const pb = trendPriority[b.trend] !== undefined ? trendPriority[b.trend] : 2
        return pa - pb
      })

      restockItems.value = sorted.map(f => ({
        ...f,
        unit_cost: inventoryMap.value[f.item_sku],
        qty: Math.max(1, f.forecasted_demand - f.current_demand),
        included: false
      }))

      runGreedy()
    }

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const [forecasts, inventory] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory()
        ])

        demandForecasts.value = forecasts

        // Build SKU-to-unit-cost map from inventory
        const map = {}
        for (const item of inventory) {
          map[item.sku] = item.unit_cost
        }
        inventoryMap.value = map

        buildRestockItems()
      } catch (err) {
        error.value = 'Failed to load data. Please try again.'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    // When user manually changes qty, recomputed totals update automatically
    // via computed properties — no need to re-run greedy.
    const onQtyChange = (_item) => {
      // Totals are computed properties and update reactively.
    }

    // When user manually toggles checkbox, totals update automatically.
    const onIncludeChange = (_item) => {
      // Totals are computed properties and update reactively.
    }

    const placeOrder = async () => {
      submitting.value = true
      submitError.value = null
      try {
        const order = await api.createOrder({
          customer: 'Restocking Order',
          items: includedItems.value.map(i => ({
            sku: i.item_sku,
            name: i.item_name,
            quantity: i.qty,
            unit_price: i.unit_cost
          }))
        })
        successOrder.value = order
        // Reset state after success
        budget.value = 100000
        restockItems.value = restockItems.value.map(i => ({ ...i, included: false }))
        runGreedy()
      } catch (err) {
        submitError.value = 'Failed to place order. Please try again.'
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadData)

    return {
      budget,
      restockItems,
      submitting,
      successOrder,
      submitError,
      loading,
      error,
      includedItems,
      totalCost,
      remainingBudget,
      canPlaceOrder,
      onQtyChange,
      onIncludeChange,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-section {
  padding: 0.5rem 0;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #64748b;
  margin-bottom: 0.25rem;
}

.budget-slider {
  width: 100%;
  margin: 1rem 0 0.5rem;
  accent-color: #2563eb;
}

.budget-display {
  font-size: 2rem;
  font-weight: 700;
  color: #0f172a;
}

.budget-stats {
  display: flex;
  gap: 1.5rem;
  margin-top: 1rem;
  padding-top: 1rem;
  border-top: 1px solid #e2e8f0;
}

.budget-stat {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.budget-stat-label {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  color: #64748b;
}

.budget-stat-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
}

.budget-stat-value.over {
  color: #dc2626;
}

.budget-stat-value.ok {
  color: #059669;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 0.875rem 1rem;
  border-radius: 8px;
  margin-bottom: 1rem;
  display: flex;
  align-items: center;
  gap: 1rem;
}

.success-banner a {
  color: #059669;
  font-weight: 600;
}

.success-banner button {
  margin-left: auto;
  background: none;
  border: none;
  cursor: pointer;
  color: #065f46;
  font-size: 1rem;
}

.row-included {
  background: #eff6ff !important;
}

.qty-input {
  width: 70px;
  padding: 0.25rem 0.5rem;
  border: 1px solid #e2e8f0;
  border-radius: 4px;
  font-size: 0.875rem;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.5rem 1.25rem;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #cbd5e1;
  cursor: not-allowed;
}
</style>
