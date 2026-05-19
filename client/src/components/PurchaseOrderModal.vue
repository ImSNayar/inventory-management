<template>
  <Teleport to="body">
    <Transition name="modal">
      <div v-if="isOpen" class="modal-overlay" @click="close">
        <div class="modal-container" @click.stop>
          <div class="modal-header">
            <h3 class="modal-title">{{ mode === 'create' ? 'Create Purchase Order' : 'Purchase Order Details' }}</h3>
            <button class="close-button" @click="close">
              <svg width="20" height="20" viewBox="0 0 20 20" fill="none">
                <path d="M15 5L5 15M5 5L15 15" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
              </svg>
            </button>
          </div>

          <!-- Create mode -->
          <form v-if="mode === 'create' && backlogItem" class="modal-body" @submit.prevent="submitForm">
            <div class="form-context">
              <div class="context-label">Backlog Item</div>
              <div class="context-value">{{ backlogItem.item_name }} (SKU: {{ backlogItem.item_sku }})</div>
            </div>

            <div class="form-grid">
              <div class="form-group">
                <label class="form-label" for="supplier_name">Supplier Name</label>
                <input
                  id="supplier_name"
                  v-model="form.supplier_name"
                  type="text"
                  class="form-input"
                  placeholder="Enter supplier name"
                  required
                />
              </div>

              <div class="form-group">
                <label class="form-label" for="quantity">Quantity</label>
                <input
                  id="quantity"
                  v-model.number="form.quantity"
                  type="number"
                  class="form-input"
                  min="1"
                  required
                />
              </div>

              <div class="form-group">
                <label class="form-label" for="unit_cost">Unit Cost ($)</label>
                <input
                  id="unit_cost"
                  v-model.number="form.unit_cost"
                  type="number"
                  class="form-input"
                  min="0"
                  step="0.01"
                  placeholder="0.00"
                  required
                />
              </div>

              <div class="form-group">
                <label class="form-label" for="expected_delivery_date">Expected Delivery Date</label>
                <input
                  id="expected_delivery_date"
                  v-model="form.expected_delivery_date"
                  type="date"
                  class="form-input"
                  required
                />
              </div>

              <div class="form-group form-group--full">
                <label class="form-label" for="notes">Notes <span class="optional">(optional)</span></label>
                <textarea
                  id="notes"
                  v-model="form.notes"
                  class="form-textarea"
                  rows="3"
                  placeholder="Add any relevant notes..."
                />
              </div>
            </div>

            <div class="total-preview" v-if="form.quantity && form.unit_cost">
              <span class="total-label">Estimated Total</span>
              <span class="total-value">{{ formatCurrency(form.quantity * form.unit_cost) }}</span>
            </div>

            <div class="modal-footer">
              <button type="button" class="btn-secondary" @click="close">Cancel</button>
              <button type="submit" class="btn-primary">Create Purchase Order</button>
            </div>
          </form>

          <!-- View mode -->
          <div v-else-if="mode === 'view'" class="modal-body">
            <div v-if="backlogItem && backlogItem.purchase_order">
              <div class="po-header">
                <div class="po-id">{{ backlogItem.purchase_order.id }}</div>
                <span class="status-badge" :class="backlogItem.purchase_order.status">
                  {{ backlogItem.purchase_order.status }}
                </span>
              </div>

              <div class="info-grid">
                <div class="info-item">
                  <div class="info-label">Supplier Name</div>
                  <div class="info-value">{{ backlogItem.purchase_order.supplier_name }}</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Quantity</div>
                  <div class="info-value">{{ backlogItem.purchase_order.quantity }} units</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Unit Cost</div>
                  <div class="info-value">{{ formatCurrency(backlogItem.purchase_order.unit_cost) }}</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Total Cost</div>
                  <div class="info-value highlight">
                    {{ formatCurrency(backlogItem.purchase_order.quantity * backlogItem.purchase_order.unit_cost) }}
                  </div>
                </div>

                <div class="info-item">
                  <div class="info-label">Expected Delivery</div>
                  <div class="info-value">{{ formatDate(backlogItem.purchase_order.expected_delivery_date) }}</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Created Date</div>
                  <div class="info-value">{{ formatDate(backlogItem.purchase_order.created_date) }}</div>
                </div>

                <div class="info-item form-group--full" v-if="backlogItem.purchase_order.notes">
                  <div class="info-label">Notes</div>
                  <div class="info-value notes">{{ backlogItem.purchase_order.notes }}</div>
                </div>
              </div>
            </div>

            <div v-else class="empty-state">
              No purchase order found for this backlog item.
            </div>

            <div class="modal-footer">
              <button class="btn-secondary" @click="close">Close</button>
            </div>
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<script>
import { ref, computed, watch } from 'vue'

export default {
  name: 'PurchaseOrderModal',
  props: {
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
      validator: (v) => ['create', 'view'].includes(v)
    }
  },
  emits: ['close', 'po-created'],
  setup(props, { emit }) {
    const defaultQuantity = computed(() => {
      if (!props.backlogItem) return 1
      const needed = props.backlogItem.quantity_needed ?? 0
      const available = props.backlogItem.quantity_available ?? 0
      return Math.max(1, needed - available)
    })

    const form = ref({
      supplier_name: '',
      quantity: 1,
      unit_cost: null,
      expected_delivery_date: '',
      notes: ''
    })

    // Reset form when modal opens or backlogItem changes
    watch(
      () => [props.isOpen, props.backlogItem],
      ([open]) => {
        if (open && props.mode === 'create') {
          form.value = {
            supplier_name: '',
            quantity: defaultQuantity.value,
            unit_cost: null,
            expected_delivery_date: '',
            notes: ''
          }
        }
      },
      { immediate: true }
    )

    const close = () => {
      emit('close')
    }

    const submitForm = () => {
      if (!props.backlogItem) return

      const po = {
        id: `PO-${Date.now()}`,
        backlog_item_id: props.backlogItem.id,
        supplier_name: form.value.supplier_name,
        quantity: form.value.quantity,
        unit_cost: form.value.unit_cost,
        expected_delivery_date: form.value.expected_delivery_date,
        status: 'pending',
        created_date: new Date().toISOString().slice(0, 10),
        notes: form.value.notes
      }

      emit('po-created', po)
    }

    const formatCurrency = (value) => {
      if (value == null || isNaN(value)) return '$0.00'
      return value.toLocaleString('en-US', { style: 'currency', currency: 'USD' })
    }

    const formatDate = (dateString) => {
      if (!dateString) return 'N/A'
      const date = new Date(dateString)
      if (isNaN(date.getTime())) return 'N/A'
      return date.toLocaleDateString('en-US', {
        year: 'numeric',
        month: 'long',
        day: 'numeric'
      })
    }

    return {
      form,
      close,
      submitForm,
      formatCurrency,
      formatDate
    }
  }
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
  flex-shrink: 0;
}

.modal-title {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
  margin: 0;
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
  padding: 1.5rem;
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
}

/* Create mode */
.form-context {
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  padding: 0.875rem 1rem;
}

.context-label {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
  margin-bottom: 0.25rem;
}

.context-value {
  font-size: 0.938rem;
  font-weight: 500;
  color: #0f172a;
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

.form-group--full {
  grid-column: 1 / -1;
}

.form-label {
  font-size: 0.813rem;
  font-weight: 600;
  color: #374151;
}

.optional {
  font-weight: 400;
  color: #94a3b8;
}

.form-input,
.form-textarea {
  padding: 0.5rem 0.75rem;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  font-size: 0.875rem;
  color: #0f172a;
  background: white;
  font-family: inherit;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;
}

.form-input:focus,
.form-textarea:focus {
  outline: none;
  border-color: #2563eb;
  box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.1);
}

.form-textarea {
  resize: vertical;
  min-height: 80px;
}

.total-preview {
  display: flex;
  align-items: center;
  justify-content: space-between;
  background: #eff6ff;
  border: 1px solid #bfdbfe;
  border-radius: 8px;
  padding: 0.875rem 1rem;
}

.total-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #1e40af;
}

.total-value {
  font-size: 1.125rem;
  font-weight: 700;
  color: #1e40af;
}

/* View mode */
.po-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding-bottom: 1rem;
  border-bottom: 1px solid #e2e8f0;
  margin-bottom: 1.25rem;
}

.po-id {
  font-size: 0.938rem;
  font-weight: 600;
  font-family: 'Monaco', 'Courier New', monospace;
  color: #2563eb;
}

.status-badge {
  padding: 0.25rem 0.75rem;
  border-radius: 20px;
  font-size: 0.813rem;
  font-weight: 600;
  text-transform: capitalize;
}

.status-badge.pending {
  background: #fef9c3;
  color: #854d0e;
}

.status-badge.approved {
  background: #dcfce7;
  color: #166534;
}

.status-badge.shipped {
  background: #dbeafe;
  color: #1e40af;
}

.status-badge.delivered {
  background: #f0fdf4;
  color: #15803d;
}

.status-badge.cancelled {
  background: #fee2e2;
  color: #991b1b;
}

.info-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 1.25rem;
}

.info-item {
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
}

.info-label {
  font-size: 0.75rem;
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

.info-value.highlight {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
}

.info-value.notes {
  color: #475569;
  font-weight: 400;
  line-height: 1.5;
}

.empty-state {
  text-align: center;
  padding: 3rem 1rem;
  color: #64748b;
  font-size: 0.938rem;
}

/* Footer */
.modal-footer {
  padding: 1.25rem 1.5rem;
  border-top: 1px solid #e2e8f0;
  display: flex;
  justify-content: flex-end;
  gap: 0.75rem;
  flex-shrink: 0;
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

.btn-secondary:hover {
  background: #e2e8f0;
  border-color: #cbd5e1;
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

.btn-primary:hover {
  background: #1d4ed8;
  border-color: #1d4ed8;
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