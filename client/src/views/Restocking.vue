<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Recommendations</h2>
      <p>Budget-based restocking recommendations derived from demand forecasts</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget Slider -->
      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">Budget</h3>
        </div>
        <div class="budget-controls">
          <div class="budget-display">
            <span class="budget-label">Selected Budget:</span>
            <span class="budget-value">${{ budget.toLocaleString() }}</span>
          </div>
          <input
            type="range"
            v-model.number="budget"
            min="0"
            max="200000"
            step="1000"
            class="budget-slider"
            @input="onBudgetInput"
          />
          <div class="budget-range-labels">
            <span>$0</span>
            <span>$200,000</span>
          </div>
          <div class="budget-summary">
            <span class="used">Used: <strong>${{ budgetUsed.toLocaleString() }}</strong></span>
            <span class="remaining" :class="{ over: budgetUsed > budget }">
              Remaining: <strong>${{ (budget - budgetUsed).toLocaleString() }}</strong>
            </span>
          </div>
        </div>
      </div>

      <!-- Success Banner -->
      <div v-if="orderSuccess" class="success-banner">
        Order {{ lastOrderNumber }} placed successfully! Navigate to the Orders tab to view it.
      </div>

      <!-- Recommendations Table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items ({{ recommendedItems.length }} within budget)</h3>
          <button
            class="btn-place-order"
            :disabled="selectedSkus.size === 0 || placing"
            @click="placeOrder"
          >
            {{ placing ? 'Placing...' : 'Place Order' }}
          </button>
        </div>
        <div v-if="recommendedItems.length === 0" class="empty-state">
          No items with a demand gap found, or budget is too low for any item.
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th class="col-check"></th>
                <th>Item Name</th>
                <th>SKU</th>
                <th>Trend</th>
                <th class="col-num">Demand Gap</th>
                <th class="col-num">Unit Cost</th>
                <th class="col-num">Restock Qty</th>
                <th class="col-num">Total Cost</th>
                <th class="col-num">Lead Time</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendedItems" :key="item.item_sku">
                <td class="col-check">
                  <input
                    type="checkbox"
                    :checked="selectedSkus.has(item.item_sku)"
                    @change="toggleItem(item.item_sku)"
                  />
                </td>
                <td>{{ item.item_name }}</td>
                <td><code>{{ item.item_sku }}</code></td>
                <td>
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td class="col-num">{{ item.demand_gap }}</td>
                <td class="col-num">${{ item.unit_cost.toFixed(2) }}</td>
                <td class="col-num">{{ item.restock_qty }}</td>
                <td class="col-num">${{ item.total_cost.toLocaleString(undefined, { maximumFractionDigits: 0 }) }}</td>
                <td class="col-num">{{ item.lead_days }} days</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- All Items (outside budget) -->
      <div v-if="outsideBudgetItems.length > 0" class="card">
        <div class="card-header">
          <h3 class="card-title">Outside Budget ({{ outsideBudgetItems.length }} items)</h3>
        </div>
        <div class="table-container">
          <table>
            <thead>
              <tr>
                <th>Item Name</th>
                <th>SKU</th>
                <th>Trend</th>
                <th class="col-num">Demand Gap</th>
                <th class="col-num">Unit Cost</th>
                <th class="col-num">Restock Qty</th>
                <th class="col-num">Total Cost</th>
                <th class="col-num">Lead Time</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in outsideBudgetItems" :key="item.item_sku" class="outside-budget-row">
                <td>{{ item.item_name }}</td>
                <td><code>{{ item.item_sku }}</code></td>
                <td>
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td class="col-num">{{ item.demand_gap }}</td>
                <td class="col-num">${{ item.unit_cost.toFixed(2) }}</td>
                <td class="col-num">{{ item.restock_qty }}</td>
                <td class="col-num">${{ item.total_cost.toLocaleString(undefined, { maximumFractionDigits: 0 }) }}</td>
                <td class="col-num">{{ item.lead_days }} days</td>
              </tr>
            </tbody>
          </table>
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
    const demandForecasts = ref([])
    const inventoryItems = ref([])
    const budget = ref(50000)
    const selectedSkus = ref(new Set())
    const placing = ref(false)
    const orderSuccess = ref(false)
    const lastOrderNumber = ref('')

    const loadData = async () => {
      try {
        loading.value = true
        error.value = null
        const [forecasts, inventory] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory({})
        ])
        demandForecasts.value = forecasts
        inventoryItems.value = inventory
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
      } finally {
        loading.value = false
      }
    }

    // Join demand forecasts with inventory unit cost, compute restock fields
    const enrichedItems = computed(() => {
      return demandForecasts.value
        .map(forecast => {
          const inv = inventoryItems.value.find(i => i.sku === forecast.item_sku)
          const unit_cost = inv?.unit_cost ?? 0
          const restock_qty = Math.max(0, forecast.forecasted_demand - forecast.current_demand)
          const total_cost = restock_qty * unit_cost
          const lead_days = forecast.trend === 'increasing' ? 3 : forecast.trend === 'stable' ? 7 : 14
          const demand_gap = forecast.forecasted_demand - forecast.current_demand
          return { ...forecast, unit_cost, restock_qty, total_cost, lead_days, demand_gap }
        })
        .filter(item => item.demand_gap > 0)
        .sort((a, b) => b.demand_gap - a.demand_gap)
    })

    // Greedy selection within budget
    const recommendedItems = computed(() => {
      const result = []
      let cumulative = 0
      for (const item of enrichedItems.value) {
        if (cumulative + item.total_cost <= budget.value) {
          cumulative += item.total_cost
          result.push(item)
        }
      }
      return result
    })

    const outsideBudgetItems = computed(() => {
      const recommendedSkus = new Set(recommendedItems.value.map(i => i.item_sku))
      return enrichedItems.value.filter(i => !recommendedSkus.has(i.item_sku))
    })

    // Pre-check all recommended items in selection
    const syncSelection = () => {
      selectedSkus.value = new Set(recommendedItems.value.map(i => i.item_sku))
    }

    const budgetUsed = computed(() => {
      return recommendedItems.value
        .filter(i => selectedSkus.value.has(i.item_sku))
        .reduce((sum, i) => sum + i.total_cost, 0)
    })

    const toggleItem = (sku) => {
      const newSet = new Set(selectedSkus.value)
      if (newSet.has(sku)) {
        newSet.delete(sku)
      } else {
        newSet.add(sku)
      }
      selectedSkus.value = newSet
    }

    const selectedItems = computed(() => {
      return recommendedItems.value.filter(i => selectedSkus.value.has(i.item_sku))
    })

    const placeOrder = async () => {
      if (selectedItems.value.length === 0) return
      placing.value = true
      orderSuccess.value = false
      try {
        const today = new Date()
        const maxLeadDays = Math.max(...selectedItems.value.map(i => i.lead_days))
        const deliveryDate = new Date(today)
        deliveryDate.setDate(deliveryDate.getDate() + maxLeadDays)

        const orderData = {
          customer: 'Restocking Order',
          status: 'Submitted',
          order_date: today.toISOString().split('T')[0],
          expected_delivery: deliveryDate.toISOString().split('T')[0],
          items: selectedItems.value.map(i => ({
            sku: i.item_sku,
            name: i.item_name,
            quantity: i.restock_qty,
            unit_price: i.unit_cost
          })),
          total_value: selectedItems.value.reduce((sum, i) => sum + i.total_cost, 0)
        }

        const result = await api.createOrder(orderData)
        lastOrderNumber.value = result.order_number
        orderSuccess.value = true
        // Clear selection after successful order
        selectedSkus.value = new Set()
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
      } finally {
        placing.value = false
      }
    }

    // Re-sync selection when budget changes
    const onBudgetInput = () => {
      syncSelection()
    }

    onMounted(async () => {
      await loadData()
      syncSelection()
    })

    return {
      loading,
      error,
      budget,
      selectedSkus,
      placing,
      orderSuccess,
      lastOrderNumber,
      recommendedItems,
      outsideBudgetItems,
      budgetUsed,
      toggleItem,
      placeOrder,
      onBudgetInput
    }
  }
}
</script>

<style scoped>
.restocking {
  padding-bottom: 2rem;
}

.budget-card {
  margin-bottom: 1.25rem;
}

.budget-controls {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  max-width: 600px;
}

.budget-display {
  display: flex;
  align-items: baseline;
  gap: 0.75rem;
}

.budget-label {
  color: #64748b;
  font-size: 0.938rem;
  font-weight: 500;
}

.budget-value {
  font-size: 1.75rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.budget-slider {
  width: 100%;
  height: 6px;
  accent-color: #2563eb;
  cursor: pointer;
}

.budget-range-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.813rem;
  color: #94a3b8;
}

.budget-summary {
  display: flex;
  gap: 2rem;
  font-size: 0.875rem;
  color: #334155;
}

.budget-summary .remaining.over {
  color: #dc2626;
}

.btn-place-order {
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

.btn-place-order:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-place-order:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #a7f3d0;
  color: #065f46;
  padding: 0.875rem 1.25rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-size: 0.938rem;
  font-weight: 500;
}

.col-check {
  width: 40px;
}

.col-num {
  text-align: right;
}

.outside-budget-row td {
  color: #94a3b8;
}

.empty-state {
  padding: 2rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

code {
  font-family: monospace;
  font-size: 0.813rem;
  background: #f1f5f9;
  padding: 0.125rem 0.375rem;
  border-radius: 4px;
  color: #475569;
}
</style>
