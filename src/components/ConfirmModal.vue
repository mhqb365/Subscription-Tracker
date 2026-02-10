<script setup>
import { onMounted, onUnmounted } from "vue";

const props = defineProps({
  show: Boolean,
  title: {
    type: String,
    default: "Confirm action",
  },
  message: {
    type: String,
    default: "Are you sure you want to proceed?",
  },
  confirmText: {
    type: String,
    default: "Confirm",
  },
  cancelText: {
    type: String,
    default: "Cancel",
  },
  isDanger: {
    type: Boolean,
    default: false,
  },
});

const emit = defineEmits(["confirm", "cancel"]);

function handleKeydown(e) {
  if (e.key === "Escape" && props.show) {
    emit("cancel");
  }
}

onMounted(() => window.addEventListener("keydown", handleKeydown));
onUnmounted(() => window.removeEventListener("keydown", handleKeydown));
</script>

<template>
  <Teleport to="body">
    <Transition name="fade">
      <div v-if="show" class="modal-overlay" @click.self="$emit('cancel')">
        <Transition name="scale">
          <div
            v-if="show"
            class="modal-content"
            role="dialog"
            aria-modal="true"
          >
            <div class="modal-header">
              <div class="icon-box" :class="{ danger: isDanger }">
                <svg
                  v-if="isDanger"
                  viewBox="0 0 24 24"
                  fill="none"
                  stroke="currentColor"
                  stroke-width="2"
                >
                  <path
                    d="M12 9v2m0 4h.01m-6.938 4h13.856c1.54 0 2.502-1.667 1.732-3L13.732 4c-.77-1.333-2.694-1.333-3.464 0L3.34 16c-.77 1.333.192 3 1.732 3z"
                  />
                </svg>
                <svg
                  v-else
                  viewBox="0 0 24 24"
                  fill="none"
                  stroke="currentColor"
                  stroke-width="2"
                >
                  <circle cx="12" cy="12" r="10" />
                  <line x1="12" y1="8" x2="12" y2="12" />
                  <line x1="12" y1="16" x2="12.01" y2="16" />
                </svg>
              </div>
              <h3>{{ title }}</h3>
            </div>

            <div class="modal-body">
              <p>{{ message }}</p>
            </div>

            <div class="modal-footer">
              <button class="btn btn-text" @click="$emit('cancel')">
                {{ cancelText }}
              </button>
              <button
                class="btn btn-pill"
                :class="isDanger ? 'btn-danger' : 'btn-primary'"
                @click="$emit('confirm')"
              >
                {{ confirmText }}
              </button>
            </div>
          </div>
        </Transition>
      </div>
    </Transition>
  </Teleport>
</template>

<style scoped>
.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.7);
  backdrop-filter: blur(8px);
  z-index: 2000;
  display: grid;
  place-items: center;
  padding: 20px;
}

.modal-content {
  background: #0f182c;
  border: 1px solid rgba(255, 255, 255, 0.08);
  width: 100%;
  max-width: 400px;
  border-radius: 24px;
  padding: 32px;
  box-shadow: 0 40px 80px rgba(0, 0, 0, 0.6);
  text-align: center;
}

[data-theme="light"] .modal-content {
  background: #ffffff;
  border-color: #e5e7eb;
  box-shadow: 0 20px 40px rgba(0, 0, 0, 0.1);
}

.modal-header {
  margin-bottom: 20px;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 16px;
}

.icon-box {
  width: 56px;
  height: 56px;
  border-radius: 16px;
  background: rgba(99, 102, 241, 0.1);
  color: #6366f1;
  display: grid;
  place-items: center;
}

.icon-box svg {
  width: 28px;
  height: 28px;
}

.icon-box.danger {
  background: rgba(239, 68, 68, 0.1);
  color: #ef4444;
}

h3 {
  margin: 0;
  font-size: 20px;
  font-weight: 700;
  color: #fff;
}

[data-theme="light"] h3 {
  color: #111827;
}

.modal-body p {
  margin: 0;
  color: var(--muted, #94a3b8);
  font-size: 15px;
  line-height: 1.6;
}

.modal-footer {
  margin-top: 32px;
  display: flex;
  gap: 12px;
}

.btn {
  flex: 1;
  padding: 12px;
  border-radius: 12px;
  font-weight: 600;
  font-size: 15px;
  cursor: pointer;
  transition: all 0.2s;
  border: none;
}

.btn-text {
  background: transparent;
  color: var(--muted, #94a3b8);
}

.btn-text:hover {
  background: rgba(255, 255, 255, 0.05);
  color: #fff;
}

[data-theme="light"] .btn-text:hover {
  background: #f3f4f6;
  color: #111827;
}

.btn-primary {
  background: linear-gradient(135deg, #6366f1, #8b5cf6);
  color: #fff;
}

.btn-primary:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 20px rgba(99, 102, 241, 0.3);
}

.btn-danger {
  background: #ef4444;
  color: #fff;
}

.btn-danger:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 20px rgba(239, 68, 68, 0.3);
}

/* Transitions */
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s ease;
}
.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

.scale-enter-active,
.scale-leave-active {
  transition: all 0.3s cubic-bezier(0.34, 1.56, 0.64, 1);
}
.scale-enter-from,
.scale-leave-to {
  opacity: 0;
  transform: scale(0.9) translateY(20px);
}
</style>
