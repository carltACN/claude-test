<template>
  <Teleport to="body">
    <Transition name="modal">
      <div v-if="isOpen && backlogItem" class="modal-overlay" @click="close">
        <div class="modal-container" @click.stop>
          <div class="modal-header">
            <h3 class="modal-title">
              {{ mode === 'create' ? 'Create Purchase Order' : 'Purchase Order Submitted' }}
            </h3>
            <button class="close-button" @click="close">
              <svg width="20" height="20" viewBox="0 0 20 20" fill="none">
                <path d="M15 5L5 15M5 5L15 15" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
              </svg>
            </button>
          </div>

          <div class="modal-body">
            <!-- Item summary banner shown in both modes -->
            <div class="item-banner">
              <div class="item-banner-info">
                <div class="item-name">{{ backlogItem.item_name }}</div>
                <div class="item-sku">SKU: {{ backlogItem.item_sku }}</div>
              </div>
              <span class="priority-badge" :class="backlogItem.priority">
                {{ backlogItem.priority }} Priority
              </span>
            </div>

            <!-- CREATE MODE: form -->
            <form v-if="mode === 'create'" class="po-form" @submit.prevent="submitForm">
              <div class="form-section">
                <div class="section-title">Order Details</div>
                <div class="form-grid">
                  <div class="form-group full-width">
                    <label class="form-label" for="supplier-name">Supplier Name <span class="required">*</span></label>
                    <input
                      id="supplier-name"
                      v-model="form.supplier_name"
                      type="text"
                      class="form-input"
                      :class="{ 'input-error': errors.supplier_name }"
                      placeholder="Enter supplier name"
                      required
                    />
                    <span v-if="errors.supplier_name" class="error-message">{{ errors.supplier_name }}</span>
                  </div>

                  <div class="form-group">
                    <label class="form-label" for="quantity">Quantity <span class="required">*</span></label>
                    <input
                      id="quantity"
                      v-model.number="form.quantity"
                      type="number"
                      class="form-input"
                      :class="{ 'input-error': errors.quantity }"
                      min="1"
                      placeholder="Units to order"
                      required
                    />
                    <span v-if="errors.quantity" class="error-message">{{ errors.quantity }}</span>
                    <span class="field-hint">Shortage: {{ shortage }} units</span>
                  </div>

                  <div class="form-group">
                    <label class="form-label" for="unit-cost">Unit Cost (USD) <span class="required">*</span></label>
                    <input
                      id="unit-cost"
                      v-model.number="form.unit_cost"
                      type="number"
                      class="form-input"
                      :class="{ 'input-error': errors.unit_cost }"
                      min="0.01"
                      step="0.01"
                      placeholder="0.00"
                      required
                    />
                    <span v-if="errors.unit_cost" class="error-message">{{ errors.unit_cost }}</span>
                  </div>

                  <div class="form-group">
                    <label class="form-label" for="delivery-date">Expected Delivery Date <span class="required">*</span></label>
                    <input
                      id="delivery-date"
                      v-model="form.expected_delivery_date"
                      type="date"
                      class="form-input"
                      :class="{ 'input-error': errors.expected_delivery_date }"
                      :min="today"
                      required
                    />
                    <span v-if="errors.expected_delivery_date" class="error-message">{{ errors.expected_delivery_date }}</span>
                  </div>

                  <div v-if="form.quantity && form.unit_cost" class="form-group">
                    <label class="form-label">Estimated Total</label>
                    <div class="total-display">{{ formatCurrency(form.quantity * form.unit_cost) }}</div>
                  </div>

                  <div class="form-group full-width">
                    <label class="form-label" for="notes">Notes <span class="optional">(optional)</span></label>
                    <textarea
                      id="notes"
                      v-model="form.notes"
                      class="form-textarea"
                      rows="3"
                      placeholder="Additional notes or instructions..."
                    ></textarea>
                  </div>
                </div>
              </div>

              <div v-if="submitError" class="submit-error">{{ submitError }}</div>
            </form>

            <!-- VIEW MODE: read-only summary -->
            <div v-else class="view-summary">
              <div class="submitted-notice">
                <svg width="20" height="20" viewBox="0 0 20 20" fill="none">
                  <path d="M7 10L9 12L13 8M17 10C17 13.866 13.866 17 10 17C6.13401 17 3 13.866 3 10C3 6.13401 6.13401 3 10 3C13.866 3 17 6.13401 17 10Z" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
                </svg>
                <span>Purchase order has been submitted for this backlog item.</span>
              </div>

              <div class="summary-cards">
                <div class="summary-card danger">
                  <div class="summary-label">Shortage Amount</div>
                  <div class="summary-value">{{ shortage }} units</div>
                </div>
                <div class="summary-card warning">
                  <div class="summary-label">Days Delayed</div>
                  <div class="summary-value">{{ backlogItem.days_delayed }} days</div>
                </div>
              </div>

              <div class="info-grid">
                <div class="info-item">
                  <div class="info-label">Item Name</div>
                  <div class="info-value">{{ backlogItem.item_name }}</div>
                </div>
                <div class="info-item">
                  <div class="info-label">SKU</div>
                  <div class="info-value sku">{{ backlogItem.item_sku }}</div>
                </div>
                <div class="info-item">
                  <div class="info-label">Quantity Needed</div>
                  <div class="info-value">{{ backlogItem.quantity_needed }} units</div>
                </div>
                <div class="info-item">
                  <div class="info-label">Quantity Available</div>
                  <div class="info-value">{{ backlogItem.quantity_available }} units</div>
                </div>
                <div class="info-item">
                  <div class="info-label">Priority</div>
                  <div class="info-value">
                    <span class="priority-badge" :class="backlogItem.priority">
                      {{ backlogItem.priority }} Priority
                    </span>
                  </div>
                </div>
                <div class="info-item">
                  <div class="info-label">Days Delayed</div>
                  <div class="info-value">{{ backlogItem.days_delayed }} days</div>
                </div>
              </div>
            </div>
          </div>

          <div class="modal-footer">
            <button class="btn-secondary" @click="close" :disabled="submitting">Cancel</button>
            <button
              v-if="mode === 'create'"
              class="btn-primary"
              :disabled="submitting"
              @click="submitForm"
            >
              <span v-if="submitting">Submitting...</span>
              <span v-else>Submit Purchase Order</span>
            </button>
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<script setup>
import { ref, computed, watch } from 'vue'
import { api } from '../api'

const props = defineProps({
  isOpen: {
    type: Boolean,
    default: false
  },
  backlogItem: {
    type: Object,
    default: null
  },
  mode: {
    type: String,
    default: 'create',
    validator: (val) => ['create', 'view'].includes(val)
  }
})

const emit = defineEmits(['close', 'po-created'])

const submitting = ref(false)
const submitError = ref(null)
const errors = ref({})

const form = ref({
  supplier_name: '',
  quantity: 0,
  unit_cost: null,
  expected_delivery_date: '',
  notes: ''
})

// Pre-fill quantity with shortage when modal opens or backlogItem changes
watch(
  () => [props.isOpen, props.backlogItem],
  ([open, item]) => {
    if (open && item) {
      const qty = Math.max(0, (item.quantity_needed ?? 0) - (item.quantity_available ?? 0))
      form.value = {
        supplier_name: '',
        quantity: qty,
        unit_cost: null,
        expected_delivery_date: '',
        notes: ''
      }
      errors.value = {}
      submitError.value = null
    }
  },
  { immediate: true }
)

const shortage = computed(() => {
  if (!props.backlogItem) return 0
  return Math.max(0, props.backlogItem.quantity_needed - props.backlogItem.quantity_available)
})

const today = computed(() => {
  return new Date().toISOString().split('T')[0]
})

const formatCurrency = (value) => {
  return value.toLocaleString('en-US', { style: 'currency', currency: 'USD' })
}

const validate = () => {
  const errs = {}
  if (!form.value.supplier_name.trim()) {
    errs.supplier_name = 'Supplier name is required.'
  }
  if (!form.value.quantity || form.value.quantity < 1) {
    errs.quantity = 'Quantity must be at least 1.'
  }
  if (!form.value.unit_cost || form.value.unit_cost <= 0) {
    errs.unit_cost = 'Unit cost must be greater than 0.'
  }
  if (!form.value.expected_delivery_date) {
    errs.expected_delivery_date = 'Expected delivery date is required.'
  }
  errors.value = errs
  return Object.keys(errs).length === 0
}

const submitForm = async () => {
  if (!validate()) return

  submitting.value = true
  submitError.value = null

  try {
    const payload = {
      backlog_item_id: props.backlogItem.id,
      supplier_name: form.value.supplier_name.trim(),
      quantity: form.value.quantity,
      unit_cost: form.value.unit_cost,
      expected_delivery_date: form.value.expected_delivery_date,
      notes: form.value.notes.trim() || null
    }

    const data = await api.createPurchaseOrder(payload)
    emit('po-created', data)
    close()
  } catch (err) {
    submitError.value = 'Failed to submit purchase order. Please try again.'
    console.error('PO creation error:', err)
  } finally {
    submitting.value = false
  }
}

const close = () => {
  emit('close')
}
</script>

<style scoped>
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2000;
  padding: 1rem;
}

.modal-container {
  background: white;
  border-radius: 12px;
  box-shadow: 0 20px 50px rgba(0, 0, 0, 0.15);
  max-width: 640px;
  width: 100%;
  max-height: 90vh;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1.5rem;
  border-bottom: 1px solid #e2e8f0;
}

.modal-title {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.close-button {
  background: none;
  border: none;
  color: #64748b;
  cursor: pointer;
  padding: 0.5rem;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 6px;
  transition: all 0.15s ease;
}

.close-button:hover {
  background: #f1f5f9;
  color: #0f172a;
}

.modal-body {
  flex: 1;
  overflow-y: auto;
  padding: 1.5rem 2rem;
}

/* Item banner */
.item-banner {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
  padding: 1rem 1.25rem;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  margin-bottom: 1.5rem;
}

.item-banner-info {
  min-width: 0;
}

.item-name {
  font-size: 1rem;
  font-weight: 600;
  color: #0f172a;
  margin-bottom: 0.25rem;
}

.item-sku {
  font-size: 0.813rem;
  color: #64748b;
  font-family: 'Monaco', 'Courier New', monospace;
}

/* Priority badge */
.priority-badge {
  padding: 0.375rem 0.75rem;
  border-radius: 6px;
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.025em;
  flex-shrink: 0;
}

.priority-badge.high {
  background: #fecaca;
  color: #991b1b;
}

.priority-badge.medium {
  background: #fed7aa;
  color: #92400e;
}

.priority-badge.low {
  background: #dbeafe;
  color: #1e40af;
}

/* Form */
.po-form {
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
}

.form-section {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.section-title {
  font-size: 0.875rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.form-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 1rem;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
}

.form-group.full-width {
  grid-column: 1 / -1;
}

.form-label {
  font-size: 0.875rem;
  font-weight: 500;
  color: #334155;
}

.required {
  color: #dc2626;
}

.optional {
  font-weight: 400;
  color: #94a3b8;
}

.form-input {
  padding: 0.625rem 0.875rem;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.875rem;
  color: #0f172a;
  background: white;
  font-family: inherit;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;
  outline: none;
}

.form-input:focus {
  border-color: #2563eb;
  box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.1);
}

.form-input.input-error {
  border-color: #ef4444;
}

.form-input.input-error:focus {
  box-shadow: 0 0 0 3px rgba(239, 68, 68, 0.1);
}

.form-textarea {
  padding: 0.625rem 0.875rem;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.875rem;
  color: #0f172a;
  background: white;
  font-family: inherit;
  resize: vertical;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;
  outline: none;
  min-height: 80px;
}

.form-textarea:focus {
  border-color: #2563eb;
  box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.1);
}

.error-message {
  font-size: 0.813rem;
  color: #dc2626;
}

.field-hint {
  font-size: 0.813rem;
  color: #64748b;
}

.total-display {
  padding: 0.625rem 0.875rem;
  background: #f0fdf4;
  border: 1px solid #bbf7d0;
  border-radius: 8px;
  font-size: 1rem;
  font-weight: 700;
  color: #15803d;
}

.submit-error {
  padding: 0.75rem 1rem;
  background: #fef2f2;
  border: 1px solid #fecaca;
  border-radius: 8px;
  font-size: 0.875rem;
  color: #dc2626;
}

/* View mode */
.view-summary {
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
}

.submitted-notice {
  display: flex;
  align-items: center;
  gap: 0.625rem;
  padding: 0.875rem 1rem;
  background: #f0fdf4;
  border: 1px solid #bbf7d0;
  border-radius: 8px;
  font-size: 0.875rem;
  font-weight: 500;
  color: #15803d;
}

.summary-cards {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 1rem;
}

.summary-card {
  padding: 1.25rem;
  border-radius: 10px;
  border: 2px solid;
}

.summary-card.danger {
  border-color: #fecaca;
  background: #fef2f2;
}

.summary-card.warning {
  border-color: #fed7aa;
  background: #fffbeb;
}

.summary-label {
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
  margin-bottom: 0.5rem;
}

.summary-value {
  font-size: 1.875rem;
  font-weight: 700;
  color: #0f172a;
}

.summary-card.danger .summary-value {
  color: #dc2626;
}

.summary-card.warning .summary-value {
  color: #f59e0b;
}

.info-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
  gap: 1.25rem;
}

.info-item {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.info-label {
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.info-value {
  font-size: 0.938rem;
  color: #0f172a;
  font-weight: 500;
}

.info-value.sku {
  font-family: 'Monaco', 'Courier New', monospace;
  color: #2563eb;
}

/* Footer */
.modal-footer {
  padding: 1.5rem;
  border-top: 1px solid #e2e8f0;
  display: flex;
  justify-content: flex-end;
  gap: 0.75rem;
}

.btn-secondary {
  padding: 0.625rem 1.25rem;
  background: #f1f5f9;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-weight: 500;
  font-size: 0.875rem;
  color: #334155;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.btn-secondary:hover:not(:disabled) {
  background: #e2e8f0;
  border-color: #cbd5e1;
}

.btn-secondary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn-primary {
  padding: 0.625rem 1.25rem;
  background: #2563eb;
  border: 1px solid #2563eb;
  border-radius: 8px;
  font-weight: 600;
  font-size: 0.875rem;
  color: white;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
  border-color: #1d4ed8;
}

.btn-primary:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

/* Modal transition animations */
.modal-enter-active,
.modal-leave-active {
  transition: opacity 0.2s ease;
}

.modal-enter-from,
.modal-leave-to {
  opacity: 0;
}

.modal-enter-active .modal-container,
.modal-leave-active .modal-container {
  transition: transform 0.2s ease;
}

.modal-enter-from .modal-container,
.modal-leave-to .modal-container {
  transform: scale(0.95);
}
</style>
