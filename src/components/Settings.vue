<script setup>
import { ref, computed } from "vue";
import {
  isAuthenticated,
  login,
  logout,
  saveToDrive,
  loadFromDrive,
  isSyncing,
  lastSyncTime,
  initError,
} from "../services/googleDrive";

const props = defineProps({
  subscriptions: {
    type: Array,
    required: true,
  },
});

const emit = defineEmits(["import"]);

const syncStatus = computed(() => {
  if (isSyncing.value) return "Syncing...";
  if (lastSyncTime.value) {
    // Format: "10/02/2026, 10:34:45"
    return `Last synced: ${lastSyncTime.value.toLocaleString("vi-VN", {
      day: "2-digit",
      month: "2-digit",
      year: "numeric",
      hour: "2-digit",
      minute: "2-digit",
      second: "2-digit",
    })}`;
  }
  return "Not synced yet";
});

async function handleBackup() {
  if (
    confirm(
      "This will overwrite the data on your Google Drive with current data. Continue?",
    )
  ) {
    try {
      await saveToDrive(props.subscriptions);
      alert("Backup successful!");
    } catch (e) {
      alert("Failed to backup: " + e.message);
    }
  }
}

async function handleRestore() {
  if (
    confirm(
      "This will overwrite your local data with data from Google Drive. Continue?",
    )
  ) {
    try {
      const data = await loadFromDrive();
      if (data) {
        emit("import", data); // Reuse handleImport from App.vue
        alert("Restore successful!");
      } else {
        alert("No configuration file found on Drive.");
      }
    } catch (e) {
      alert("Failed to restore data: " + e.message);
    }
  }
}
</script>

<template>
  <div class="settings-container">
    <header class="header">
      <h1>Settings</h1>
    </header>

    <div class="setting-section">
      <h2>Google Drive Sync</h2>
      <p class="description">
        Sync your subscriptions across devices using your personal Google Drive
        (App Data folder).
      </p>

      <div v-if="initError" class="error-alert">
        <div class="error-title">Initialization Error</div>
        <div class="error-message">{{ initError }}</div>
        <div class="error-hint">
          Please check your API Key restrictions in Google Cloud Console.
        </div>
      </div>

      <div class="sync-card">
        <div v-if="!isAuthenticated" class="auth-action">
          <button
            @click="login"
            :disabled="!!initError"
            class="btn btn-primary"
          >
            <svg viewBox="0 0 24 24" width="20" height="20" fill="currentColor">
              <path
                d="M12.545,10.239v3.821h5.445c-0.712,2.315-2.647,3.972-5.445,3.972c-3.332,0-6.033-2.701-6.033-6.032s2.701-6.032,6.033-6.032c1.498,0,2.866,0.549,3.921,1.453l2.814-2.814C17.503,2.988,15.139,2,12.545,2C7.021,2,2.543,6.477,2.543,12s4.478,10,10.002,10c8.396,0,10.249-7.85,9.426-11.748L12.545,10.239z"
              />
            </svg>
            Connect Google Drive
          </button>
        </div>

        <div v-else class="auth-action">
          <div class="status-row">
            <span class="status-badge connected">Connected</span>
            <span class="last-sync">{{ syncStatus }}</span>
          </div>

          <div class="actions-row">
            <button
              @click="handleBackup"
              :disabled="isSyncing || !!initError"
              class="btn btn-secondary"
            >
              Upload to Cloud
            </button>
            <button
              @click="handleRestore"
              :disabled="isSyncing || !!initError"
              class="btn btn-secondary"
            >
              Download from Cloud
            </button>
          </div>

          <button @click="logout" class="btn btn-danger text-sm">
            Disconnect
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.settings-container {
  max-width: 800px;
  margin: 0 auto;
}

.header {
  margin-bottom: 2rem;
}

h1 {
  font-size: 2rem;
  font-weight: 700;
  margin: 0;
  background: linear-gradient(135deg, #fff 0%, #a5b4fc 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}

.setting-section {
  background: rgba(255, 255, 255, 0.03);
  border: 1px solid rgba(255, 255, 255, 0.05);
  border-radius: 20px;
  padding: 24px;
}

h2 {
  font-size: 1.25rem;
  margin-bottom: 0.5rem;
  color: #fff;
}

.description {
  color: var(--muted, #94a3b8);
  font-size: 0.95rem;
  margin-bottom: 1.5rem;
}

.sync-card {
  background: rgba(0, 0, 0, 0.2);
  border-radius: 12px;
  padding: 20px;
}

.status-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16px;
}

.status-badge {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 4px 12px;
  border-radius: 100px;
  font-size: 13px;
  font-weight: 500;
}

.status-badge.connected {
  background: rgba(16, 185, 129, 0.1);
  color: #34d399;
  border: 1px solid rgba(16, 185, 129, 0.2);
}

.last-sync {
  font-size: 13px;
  color: var(--muted, #94a3b8);
}

.actions-row {
  display: flex;
  gap: 12px;
  margin-bottom: 20px;
}

.btn {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 10px 20px;
  border-radius: 12px;
  font-weight: 500;
  font-size: 14px;
  cursor: pointer;
  transition: all 0.2s;
  border: none;
}

.btn-primary {
  background: #fff;
  color: #000;
}
.btn-primary:hover {
  background: #f3f4f6;
}

.btn-secondary {
  background: rgba(255, 255, 255, 0.1);
  color: #fff;
}
.btn-secondary:hover:not(:disabled) {
  background: rgba(255, 255, 255, 0.15);
}
.btn-secondary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn-danger {
  background: transparent;
  color: #ef4444;
  padding: 8px 16px;
}
.btn-danger:hover {
  background: rgba(239, 68, 68, 0.1);
}

[data-theme="light"] h1 {
  background: linear-gradient(135deg, #1e293b 0%, #4338ca 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
}
[data-theme="light"] .setting-section {
  background: #ffffff;
  border-color: #e2e8f0;
  box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
}
[data-theme="light"] h2 {
  color: #1e293b;
}
[data-theme="light"] .description {
  color: #64748b;
}
[data-theme="light"] .sync-card {
  background: #f8fafc;
  border: 1px solid #e2e8f0;
}
[data-theme="light"] .btn-primary {
  background: #4f46e5;
  color: #fff;
}
[data-theme="light"] .btn-primary:hover {
  background: #4338ca;
}
[data-theme="light"] .btn-secondary {
  background: #e2e8f0;
  color: #334155;
}
[data-theme="light"] .btn-secondary:hover {
  background: #cbd5e1;
}
.btn-secondary:hover {
  background: #cbd5e1;
}

.error-alert {
  background: rgba(239, 68, 68, 0.1);
  border: 1px solid rgba(239, 68, 68, 0.2);
  border-radius: 12px;
  padding: 16px;
  margin-bottom: 20px;
  color: #ef4444;
}

.error-title {
  font-weight: 700;
  margin-bottom: 4px;
}

.error-message {
  font-size: 0.9em;
  margin-bottom: 4px;
}

.error-hint {
  font-size: 0.85em;
  opacity: 0.8;
}
</style>
