<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <!-- Success Toast -->
    <div v-if="showToast" class="toast toast-success">
      {{ t('restocking.orderSuccess') }}
    </div>

    <!-- Budget Section -->
    <div class="card">
      <div class="card-header">
        <h3 class="card-title">{{ t('restocking.budgetSlider') }}</h3>
      </div>
      <div class="budget-section">
        <div class="budget-slider-row">
          <span class="budget-label">{{ currencySymbol }}1,000</span>
          <input
            type="range"
            class="budget-slider"
            v-model.number="budget"
            :min="1000"
            :max="100000"
            :step="1000"
          />
          <span class="budget-label">{{ currencySymbol }}100,000</span>
        </div>
        <div class="budget-current">
          {{ t('restocking.budgetSlider') }}: <strong>{{ currencySymbol }}{{ budget.toLocaleString() }}</strong>
        </div>
        <div class="budget-bar-section">
          <div class="budget-bar-labels">
            <span class="budget-used-label">
              {{ t('restocking.budgetUsed') }}: <strong>{{ currencySymbol }}{{ budgetUsed.toLocaleString() }}</strong>
            </span>
            <span class="budget-remaining-label">
              {{ t('restocking.budgetRemaining') }}: <strong>{{ currencySymbol }}{{ budgetRemaining.toLocaleString() }}</strong>
            </span>
          </div>
          <div class="budget-progress-bar">
            <div
              class="budget-progress-fill"
              :style="{ width: budgetUsedPercent + '%' }"
            ></div>
          </div>
        </div>
      </div>
    </div>

    <!-- Lead Times Reference -->
    <div class="card lead-times-card">
      <div class="card-header">
        <h3 class="card-title">{{ t('restocking.leadTimes') }}</h3>
      </div>
      <div v-if="leadTimesLoading" class="loading-sm">{{ t('common.loading') }}</div>
      <div v-else class="lead-times-grid">
        <div
          v-for="(days, category) in leadTimes"
          :key="category"
          class="lead-time-item"
        >
          <span class="lead-time-category">{{ category }}</span>
          <span class="lead-time-days">{{ days }} {{ t('restocking.days') }}</span>
        </div>
      </div>
    </div>

    <!-- Recommendations Table -->
    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">
            {{ t('restocking.recommendations') }}
            ({{ t('restocking.itemsSelected', { count: selectedCount }) }})
          </h3>
        </div>
        <div v-if="recommendations.length === 0" class="empty-state">
          {{ t('restocking.noRecommendations') }}
        </div>
        <div v-else class="table-container">
          <table class="recommendations-table">
            <thead>
              <tr>
                <th class="col-sku">{{ t('demand.table.sku') }}</th>
                <th class="col-name">{{ t('demand.table.itemName') }}</th>
                <th class="col-category">{{ t('restocking.category') }}</th>
                <th class="col-trend">{{ t('restocking.trend') }}</th>
                <th class="col-forecast">{{ t('demand.table.forecastedDemand') }}</th>
                <th class="col-cost">{{ t('restocking.unitCost') }}</th>
                <th class="col-qty">{{ t('restocking.quantity') }}</th>
                <th class="col-total">{{ t('restocking.totalCost') }}</th>
                <th class="col-lead">{{ t('restocking.leadTime') }}</th>
                <th class="col-status">{{ t('restocking.status') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendations"
                :key="item.item_sku"
                :class="{ 'row-dimmed': !item.selected }"
              >
                <td class="col-sku"><strong>{{ item.item_sku }}</strong></td>
                <td class="col-name">{{ translateProductName(item.item_name) }}</td>
                <td class="col-category">{{ item.category }}</td>
                <td class="col-trend">
                  <span :class="['trend-indicator', item.trend]">
                    <span class="trend-dot"></span>
                    {{ item.trend }}
                  </span>
                </td>
                <td class="col-forecast">{{ item.forecasted_demand.toLocaleString() }}</td>
                <td class="col-cost">{{ currencySymbol }}{{ item.unit_cost.toLocaleString() }}</td>
                <td class="col-qty">{{ item.recommended_qty.toLocaleString() }}</td>
                <td class="col-total"><strong>{{ currencySymbol }}{{ item.total_cost.toLocaleString() }}</strong></td>
                <td class="col-lead">{{ item.lead_time_days }} {{ t('restocking.days') }}</td>
                <td class="col-status">
                  <span v-if="item.selected" class="badge success">{{ t('restocking.selected') }}</span>
                  <span v-else class="badge" style="background:#f1f5f9;color:#64748b;">{{ t('restocking.notSelected') }}</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Place Order Button -->
      <div class="place-order-row">
        <button
          class="btn-place-order"
          :disabled="selectedCount === 0 || submitting"
          @click="placeOrder"
        >
          {{ submitting ? t('common.loading') : t('restocking.placeOrder') }}
        </button>
      </div>
    </div>

    <!-- Submitted Orders Section -->
    <div class="card">
      <div class="card-header">
        <h3 class="card-title">{{ t('restocking.submittedOrders') }}</h3>
      </div>
      <div v-if="submittedOrdersLoading" class="loading-sm">{{ t('common.loading') }}</div>
      <div v-else-if="submittedOrders.length === 0" class="empty-state">
        {{ t('restocking.noSubmittedOrders') }}
      </div>
      <div v-else class="table-container">
        <table class="submitted-orders-table">
          <thead>
            <tr>
              <th class="col-order-number">{{ t('restocking.orderNumber') }}</th>
              <th class="col-order-date">{{ t('restocking.orderDate') }}</th>
              <th class="col-items-count">{{ t('orders.table.items') }}</th>
              <th class="col-delivery">{{ t('restocking.expectedDelivery') }}</th>
              <th class="col-lead-time">{{ t('restocking.leadTime') }}</th>
              <th class="col-total-value">{{ t('restocking.totalValue') }}</th>
              <th class="col-order-status">{{ t('restocking.status') }}</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="order in submittedOrders" :key="order.id">
              <td class="col-order-number"><strong>{{ order.order_number }}</strong></td>
              <td class="col-order-date">{{ formatDate(order.order_date) }}</td>
              <td class="col-items-count">{{ order.items.length }}</td>
              <td class="col-delivery">{{ formatDate(order.expected_delivery) }}</td>
              <td class="col-lead-time">{{ order.lead_time_days }} {{ t('restocking.days') }}</td>
              <td class="col-total-value"><strong>{{ currencySymbol }}{{ order.total_value.toLocaleString() }}</strong></td>
              <td class="col-order-status">
                <span :class="['badge', getOrderStatusClass(order.status)]">{{ order.status }}</span>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted, watch } from 'vue'
import { api } from '../api'
import { useFilters } from '../composables/useFilters'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency, translateProductName, currentLocale } = useI18n()
    const { selectedLocation } = useFilters()

    const currencySymbol = computed(() => {
      return currentCurrency.value === 'JPY' ? '¥' : '$'
    })

    // Budget state
    const budget = ref(50000)

    // Recommendations state
    const recommendations = ref([])
    const budgetUsed = ref(0)
    const budgetRemaining = ref(50000)
    const loading = ref(false)
    const error = ref(null)

    // Lead times state
    const leadTimes = ref({})
    const leadTimesLoading = ref(false)

    // Submitted orders state
    const submittedOrders = ref([])
    const submittedOrdersLoading = ref(false)

    // Submission state
    const submitting = ref(false)
    const showToast = ref(false)

    // Debounce timer
    let debounceTimer = null

    const selectedCount = computed(() => {
      return recommendations.value.filter(item => item.selected).length
    })

    const budgetUsedPercent = computed(() => {
      if (budget.value === 0) return 0
      const pct = (budgetUsed.value / budget.value) * 100
      return Math.min(100, Math.max(0, pct))
    })

    const loadRecommendations = async () => {
      loading.value = true
      error.value = null
      try {
        const result = await api.getRestockingRecommendations(budget.value)
        recommendations.value = result.recommendations || []
        budgetUsed.value = result.budget_used || 0
        budgetRemaining.value = result.budget_remaining || 0
      } catch (err) {
        error.value = 'Failed to load recommendations: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const loadLeadTimes = async () => {
      leadTimesLoading.value = true
      try {
        leadTimes.value = await api.getLeadTimes()
      } catch (err) {
        console.error('Failed to load lead times:', err)
      } finally {
        leadTimesLoading.value = false
      }
    }

    const loadSubmittedOrders = async () => {
      submittedOrdersLoading.value = true
      try {
        submittedOrders.value = await api.getSubmittedRestockingOrders()
      } catch (err) {
        console.error('Failed to load submitted orders:', err)
      } finally {
        submittedOrdersLoading.value = false
      }
    }

    // Debounced budget watch
    watch(budget, () => {
      if (debounceTimer) clearTimeout(debounceTimer)
      debounceTimer = setTimeout(() => {
        loadRecommendations()
      }, 400)
    })

    const placeOrder = async () => {
      if (selectedCount.value === 0 || submitting.value) return

      submitting.value = true
      try {
        const selectedItems = recommendations.value
          .filter(item => item.selected)
          .map(item => ({
            sku: item.item_sku,
            name: item.item_name,
            quantity: item.recommended_qty,
            unit_price: item.unit_cost
          }))

        const orderData = {
          items: selectedItems,
          total_budget: budget.value,
          warehouse: selectedLocation.value !== 'all' ? selectedLocation.value : 'Default'
        }

        await api.submitRestockingOrder(orderData)

        showToast.value = true
        setTimeout(() => {
          showToast.value = false
        }, 3000)

        await loadSubmittedOrders()
      } catch (err) {
        console.error('Failed to submit restocking order:', err)
        error.value = 'Failed to submit order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    const formatDate = (dateString) => {
      const date = new Date(dateString)
      if (isNaN(date.getTime())) return dateString
      const locale = currentLocale.value === 'ja' ? 'ja-JP' : 'en-US'
      return date.toLocaleDateString(locale, {
        year: 'numeric',
        month: 'short',
        day: 'numeric'
      })
    }

    const getOrderStatusClass = (status) => {
      const statusMap = {
        'Delivered': 'success',
        'Shipped': 'info',
        'Processing': 'warning',
        'Backordered': 'danger'
      }
      return statusMap[status] || 'info'
    }

    onMounted(() => {
      loadRecommendations()
      loadLeadTimes()
      loadSubmittedOrders()
    })

    return {
      t,
      currencySymbol,
      budget,
      recommendations,
      budgetUsed,
      budgetRemaining,
      budgetUsedPercent,
      loading,
      error,
      leadTimes,
      leadTimesLoading,
      submittedOrders,
      submittedOrdersLoading,
      submitting,
      showToast,
      selectedCount,
      placeOrder,
      formatDate,
      getOrderStatusClass,
      translateProductName
    }
  }
}
</script>

<style scoped>
/* Toast */
.toast {
  position: fixed;
  top: 1.5rem;
  right: 1.5rem;
  z-index: 1000;
  padding: 0.875rem 1.5rem;
  border-radius: 8px;
  font-weight: 600;
  font-size: 0.938rem;
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.12);
  animation: slide-in 0.2s ease;
}

.toast-success {
  background: #059669;
  color: #ffffff;
}

@keyframes slide-in {
  from {
    opacity: 0;
    transform: translateY(-0.5rem);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

/* Budget section */
.budget-section {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.budget-slider-row {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.budget-label {
  font-size: 0.813rem;
  color: #64748b;
  font-weight: 500;
  white-space: nowrap;
}

.budget-slider {
  flex: 1;
  -webkit-appearance: none;
  appearance: none;
  height: 6px;
  border-radius: 3px;
  background: #e2e8f0;
  outline: none;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #3b82f6;
  cursor: pointer;
  box-shadow: 0 1px 4px rgba(59, 130, 246, 0.4);
  transition: box-shadow 0.15s ease;
}

.budget-slider::-webkit-slider-thumb:hover {
  box-shadow: 0 1px 8px rgba(59, 130, 246, 0.6);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #3b82f6;
  cursor: pointer;
  border: none;
  box-shadow: 0 1px 4px rgba(59, 130, 246, 0.4);
}

.budget-current {
  font-size: 1rem;
  color: #334155;
}

.budget-bar-section {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.budget-bar-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.875rem;
  color: #64748b;
}

.budget-progress-bar {
  height: 10px;
  background: #e2e8f0;
  border-radius: 5px;
  overflow: hidden;
}

.budget-progress-fill {
  height: 100%;
  background: #3b82f6;
  border-radius: 5px;
  transition: width 0.4s ease;
}

/* Lead times card */
.lead-times-card .card-header {
  margin-bottom: 0.75rem;
}

.lead-times-grid {
  display: flex;
  flex-wrap: wrap;
  gap: 0.75rem;
}

.lead-time-item {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  padding: 0.375rem 0.875rem;
  font-size: 0.875rem;
}

.lead-time-category {
  color: #334155;
  font-weight: 500;
}

.lead-time-days {
  color: #3b82f6;
  font-weight: 700;
}

/* Recommendations table */
.recommendations-table {
  table-layout: fixed;
  width: 100%;
}

.col-sku { width: 90px; }
.col-name { width: 180px; }
.col-category { width: 130px; }
.col-trend { width: 110px; }
.col-forecast { width: 130px; }
.col-cost { width: 100px; }
.col-qty { width: 90px; }
.col-total { width: 120px; }
.col-lead { width: 90px; }
.col-status { width: 110px; }

/* Trend indicator */
.trend-indicator {
  display: inline-flex;
  align-items: center;
  gap: 0.375rem;
  font-size: 0.813rem;
  font-weight: 500;
  text-transform: capitalize;
}

.trend-indicator .trend-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  flex-shrink: 0;
}

.trend-indicator.increasing {
  color: #059669;
}
.trend-indicator.increasing .trend-dot {
  background: #059669;
}

.trend-indicator.stable {
  color: #64748b;
}
.trend-indicator.stable .trend-dot {
  background: #94a3b8;
}

.trend-indicator.decreasing {
  color: #dc2626;
}
.trend-indicator.decreasing .trend-dot {
  background: #dc2626;
}

/* Dimmed rows for over-budget items */
.row-dimmed td {
  opacity: 0.5;
}

/* Place order row */
.place-order-row {
  display: flex;
  justify-content: flex-end;
  margin-bottom: 1.25rem;
}

.btn-place-order {
  padding: 0.75rem 2rem;
  background: #2563eb;
  color: #ffffff;
  border: none;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease, opacity 0.2s ease;
}

.btn-place-order:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-place-order:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

/* Submitted orders table */
.submitted-orders-table {
  table-layout: fixed;
  width: 100%;
}

.col-order-number { width: 150px; }
.col-order-date { width: 130px; }
.col-items-count { width: 90px; }
.col-delivery { width: 130px; }
.col-lead-time { width: 100px; }
.col-total-value { width: 130px; }
.col-order-status { width: 120px; }

/* Empty state */
.empty-state {
  padding: 2rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

/* Small loading */
.loading-sm {
  padding: 0.75rem 0;
  color: #64748b;
  font-size: 0.875rem;
}
</style>
