<script setup>
import { ref, computed } from "vue";
import { iconPaths } from "../icons";

const props = defineProps({
  subscriptions: {
    type: Array,
    required: true,
  },
  categories: {
    type: Array,
    required: true,
  },
});

const emit = defineEmits(["add", "edit", "delete", "import"]);

function exportData() {
  const dataStr = JSON.stringify(props.subscriptions, null, 2);
  const dataUri =
    "data:application/json;charset=utf-8," + encodeURIComponent(dataStr);

  const exportFileDefaultName = "subscriptions.json";

  const linkElement = document.createElement("a");
  linkElement.setAttribute("href", dataUri);
  linkElement.setAttribute("download", exportFileDefaultName);
  linkElement.click();
}

function handleImportClick() {
  const input = document.createElement("input");
  input.type = "file";
  input.accept = ".json";
  input.onchange = (e) => {
    const file = e.target.files[0];
    if (!file) return;

    const reader = new FileReader();
    reader.onload = (e) => {
      try {
        const data = JSON.parse(e.target.result);
        if (Array.isArray(data)) {
          emit("import", data);
          alert("Import successful!");
        } else {
          alert("Invalid file format. Expected a JSON array of subscriptions.");
        }
      } catch (err) {
        alert("Error parsing JSON file: " + err.message);
      }
    };
    reader.readAsText(file);
  };
  input.click();
}

const searchQuery = ref("");
const statusFilter = ref("All Status");
const categoryFilter = ref("All Categories");
const sortKey = ref("name"); // name, price, expiry
const sortOrder = ref("asc");

const statusOptions = ["All Status", "Active", "Inactive", "Expiring Soon"];

const filteredSubs = computed(() => {
  let result = [...props.subscriptions];

  // Search
  if (searchQuery.value) {
    const q = searchQuery.value.toLowerCase();
    result = result.filter((s) => s.name.toLowerCase().includes(q));
  }

  // Status Filter
  if (statusFilter.value !== "All Status") {
    const today = new Date();
    result = result.filter((s) => {
      if (statusFilter.value === "Active") return s.status === "ACTIVE";
      if (statusFilter.value === "Inactive") return s.status !== "ACTIVE";
      if (statusFilter.value === "Expiring Soon") {
        const diff = new Date(s.expiry) - today;
        const days = Math.ceil(diff / (1000 * 60 * 60 * 24));
        return days > 0 && days <= 30;
      }
      return true;
    });
  }

  // Category Filter
  if (categoryFilter.value !== "All Categories") {
    result = result.filter((s) => s.category === categoryFilter.value);
  }

  // Sort
  result.sort((a, b) => {
    let valA = a[sortKey.value];
    let valB = b[sortKey.value];

    if (sortKey.value === "price") {
      // normalize to base currency approx if needed, but for now just raw number
      // Assuming mixed currencies might be tricky, usually convert to one base
      // Simple hack: if currency different, strict compare might be off
      valA = Number(valA);
      valB = Number(valB);
    } else if (sortKey.value === "expiry") {
      valA = new Date(valA).getTime();
      valB = new Date(valB).getTime();
    } else {
      valA = (valA || "").toString().toLowerCase();
      valB = (valB || "").toString().toLowerCase();
    }

    if (valA < valB) return sortOrder.value === "asc" ? -1 : 1;
    if (valA > valB) return sortOrder.value === "asc" ? 1 : -1;
    return 0;
  });

  return result;
});

function toggleSort(key) {
  if (sortKey.value === key) {
    sortOrder.value = sortOrder.value === "asc" ? "desc" : "asc";
  } else {
    sortKey.value = key;
    sortOrder.value = "asc";
  }
}

function formatPriceDisplay(sub) {
  return (
    new Intl.NumberFormat("vi-VN", {
      style: "currency",
      currency: sub.currency,
    }).format(sub.price) +
    "/" +
    (sub.cycle === "Gói tháng"
      ? "tháng"
      : sub.cycle === "Gói Quý"
        ? "quý"
        : sub.cycle === "Gói 6 tháng"
          ? "6 tháng"
          : "năm")
  );
}

function getNextBillingDate(sub) {
  if (sub.status !== "ACTIVE" || !sub.autoRenew) return new Date(sub.expiry);

  const today = new Date();
  const start = new Date(sub.startDate);
  let next = new Date(start);

  if (next > today) return next;

  while (next <= today) {
    if (sub.cycle === "Gói tháng") {
      next.setMonth(next.getMonth() + 1);
    } else if (sub.cycle === "Gói Quý") {
      next.setMonth(next.getMonth() + 3);
    } else if (sub.cycle === "Gói 6 tháng") {
      next.setMonth(next.getMonth() + 6);
    } else {
      next.setFullYear(next.getFullYear() + 1);
    }
    if (next.getFullYear() > today.getFullYear() + 10) break;
  }
  return next;
}

function getDaysLeft(sub) {
  const today = new Date();
  const nextBilling = getNextBillingDate(sub);
  const diff = nextBilling - today;
  const days = Math.ceil(diff / (1000 * 60 * 60 * 24));
  return days;
}

function formatDate(dateStr) {
  if (!dateStr) return "";
  return new Date(dateStr).toLocaleDateString("vi-VN", {
    month: "long",
    day: "numeric",
    year: "numeric",
  });
}
</script>

<template>
  <div class="subscriptions-page">
    <header class="page-header">
      <div class="header-content">
        <h1>Subscription Management</h1>
        <p class="subtitle">Manage all your subscriptions in one place</p>
      </div>
      <div class="header-actions">
        <button class="pill-btn secondary" @click="handleImportClick">
          <svg viewBox="0 0 24 24"><path :d="iconPaths.upload" /></svg>
          Import
        </button>
        <button class="pill-btn secondary" @click="exportData">
          <svg viewBox="0 0 24 24"><path :d="iconPaths.download" /></svg>
          Export
        </button>
        <button class="pill-btn primary" @click="$emit('add')">
          <svg viewBox="0 0 24 24"><path :d="iconPaths.plus" /></svg>
          Add Subscription
        </button>
      </div>
    </header>

    <div class="toolbar">
      <div class="search-box">
        <svg viewBox="0 0 24 24"><path :d="iconPaths.search" /></svg>
        <input
          v-model="searchQuery"
          type="text"
          placeholder="Search subscriptions..."
        />
        <button
          v-if="searchQuery"
          class="clear-btn"
          @click="searchQuery = ''"
          type="button"
          aria-label="Clear search"
        >
          <svg viewBox="0 0 24 24"><path :d="iconPaths.close" /></svg>
        </button>
      </div>

      <div class="filter-separator">
        <svg viewBox="0 0 24 24"><path :d="iconPaths.filter" /></svg>
      </div>

      <div class="filters-row">
        <div class="dropdown">
          <select v-model="statusFilter">
            <option v-for="opt in statusOptions" :key="opt" :value="opt">
              {{ opt }}
            </option>
          </select>
          <div class="select-arrow">
            <svg viewBox="0 0 24 24"><path d="M6 9l6 6 6-6" /></svg>
          </div>
        </div>

        <div class="dropdown">
          <select v-model="categoryFilter">
            <option value="All Categories">All Categories</option>
            <option v-for="cat in categories" :key="cat.id" :value="cat.id">
              {{ cat.label }}
            </option>
          </select>
          <div class="select-arrow">
            <svg viewBox="0 0 24 24"><path d="M6 9l6 6 6-6" /></svg>
          </div>
        </div>

        <div class="sort-group">
          <button
            class="sort-btn"
            :class="{ active: sortKey === 'name' }"
            @click="toggleSort('name')"
          >
            Name
            <span v-if="sortKey === 'name'">{{
              sortOrder === "asc" ? "↑" : "↓"
            }}</span>
          </button>
          <button
            class="sort-btn"
            :class="{ active: sortKey === 'price' }"
            @click="toggleSort('price')"
          >
            Price
          </button>
          <button
            class="sort-btn"
            :class="{ active: sortKey === 'expiry' }"
            @click="toggleSort('expiry')"
          >
            Expiration
          </button>
        </div>
      </div>
    </div>

    <div class="subs-count">
      Showing {{ filteredSubs.length }} of
      {{ subscriptions.length }} subscriptions
    </div>

    <div class="subs-grid">
      <article v-for="sub in filteredSubs" :key="sub.id" class="sub-card-large">
        <div class="card-top">
          <div class="card-brand">
            <div
              class="brand-icon"
              :style="{
                background: `linear-gradient(135deg, ${sub.accent[0]}, ${sub.accent[1]})`,
              }"
            >
              <svg viewBox="0 0 24 24"><path :d="iconPaths[sub.icon]" /></svg>
            </div>
            <div class="brand-info">
              <h3>{{ sub.name }}</h3>
              <div class="brand-cat">
                <svg
                  viewBox="0 0 24 24"
                  style="
                    width: 12px;
                    height: 12px;
                    margin-right: 4px;
                    fill: none;
                    stroke: currentColor;
                  "
                >
                  <path :d="iconPaths.cloud" />
                </svg>
                {{ sub.category }}
              </div>
            </div>
          </div>
          <div class="card-actions">
            <button class="icon-btn-sm" @click="$emit('edit', sub)">
              <svg viewBox="0 0 24 24"><path :d="iconPaths.edit" /></svg>
            </button>
            <button class="icon-btn-sm danger" @click="$emit('delete', sub.id)">
              <svg viewBox="0 0 24 24"><path :d="iconPaths.trash" /></svg>
            </button>
          </div>
        </div>

        <div class="card-mid">
          <div class="big-price">{{ formatPriceDisplay(sub) }}</div>
          <div class="days-badge" :class="{ warning: getDaysLeft(sub) <= 7 }">
            {{ getDaysLeft(sub) }}D LEFT
          </div>
        </div>

        <div class="card-dates">
          <div class="date-col">
            <span class="label">START</span>
            <span class="val">{{ formatDate(sub.startDate) }}</span>
          </div>
          <div class="arrow">→</div>
          <div class="date-col">
            <span class="label">{{
              sub.autoRenew ? "NEXT BILL" : "EXPIRES"
            }}</span>
            <span class="val">{{ formatDate(getNextBillingDate(sub)) }}</span>
          </div>
        </div>

        <div class="card-footer">
          <div class="auto-renew" :class="{ active: sub.autoRenew }">
            <svg viewBox="0 0 24 24" v-if="sub.autoRenew">
              <path :d="iconPaths.clock" />
            </svg>
            <span>{{
              sub.autoRenew ? "Auto-renewal enabled" : "Manual renewal"
            }}</span>
          </div>
        </div>
      </article>
    </div>
  </div>
</template>

<style scoped>
.subscriptions-page {
  animation: fadeIn 0.3s ease;
}
@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(5px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.page-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 32px;
  gap: 20px;
}
.header-actions {
  display: flex;
  gap: 12px;
  align-items: center;
}
.header-content h1 {
  font-size: 24px;
  margin-bottom: 4px;
}
.subtitle {
  color: var(--muted);
  margin: 0;
  font-size: 14px;
}

/* Toolbar */
.toolbar {
  display: flex;
  flex-wrap: wrap;
  gap: 16px;
  align-items: center;
  margin-bottom: 24px;
}

.search-box {
  flex: 1;
  min-width: 300px;
  height: 48px;
  display: flex;
  align-items: center;
  gap: 12px;
  background: #0f182c;
  padding: 0 16px;
  border-radius: 12px;
  border: 1px solid rgba(255, 255, 255, 0.06);
  transition: border-color 0.2s;
}
.search-box:focus-within {
  border-color: #6366f1;
}
.search-box svg {
  width: 20px;
  height: 20px;
  stroke: #64748b;
  fill: none;
  stroke-width: 2;
}
.search-box input {
  background: transparent;
  border: none;
  color: white;
  width: 100%;
  height: 100%;
  outline: none;
  font-size: 14px;
}

.clear-btn {
  background: none;
  border: none;
  color: #64748b;
  cursor: pointer;
  display: flex;
  align-items: center;
  padding: 4px;
  margin-left: 4px;
  border-radius: 50%;
  transition: all 0.2s;
  flex-shrink: 0;
}
.clear-btn:hover {
  background: rgba(255, 255, 255, 0.1);
  color: #fff;
}
.clear-btn svg {
  width: 16px;
  height: 16px;
  stroke: currentColor;
  stroke-width: 2.5;
  fill: none;
}

.filter-separator {
  color: #64748b;
  display: grid;
  place-items: center;
  padding: 0 4px;
}
.filter-separator svg {
  width: 20px;
  height: 20px;
  fill: currentColor;
}

.filters-row {
  display: flex;
  gap: 12px;
  align-items: center;
  flex-wrap: wrap;
}

.dropdown {
  position: relative;
  min-width: 140px;
}
.dropdown select {
  width: 100%;
  height: 48px;
  background: #0f182c;
  color: #dfe7ff;
  border: 1px solid rgba(255, 255, 255, 0.06);
  padding: 0 36px 0 16px;
  border-radius: 12px;
  cursor: pointer;
  font-size: 14px;
  appearance: none;
  transition: border-color 0.2s;
}
.dropdown select:focus {
  border-color: #6366f1;
  outline: none;
}
.select-arrow {
  position: absolute;
  right: 14px;
  top: 50%;
  transform: translateY(-50%);
  color: #64748b;
  pointer-events: none;
  display: grid;
  place-items: center;
}
.select-arrow svg {
  width: 16px;
  height: 16px;
  fill: none;
  stroke: currentColor;
  stroke-width: 2;
}

.sort-group {
  display: flex;
  background: #0f182c;
  border-radius: 12px;
  border: 1px solid rgba(255, 255, 255, 0.06);
  padding: 4px;
  height: 48px;
  align-items: center;
}
.sort-btn {
  background: transparent;
  border: none;
  color: var(--muted);
  padding: 0 16px;
  height: 38px;
  font-size: 13px;
  cursor: pointer;
  border-radius: 8px;
  font-weight: 600;
  display: flex;
  align-items: center;
  gap: 6px;
  transition: all 0.2s;
}
.sort-btn:hover {
  color: #fff;
}
.sort-btn.active {
  background: rgba(99, 102, 241, 0.15);
  color: #818cf8;
  border: 1px solid rgba(99, 102, 241, 0.3);
}

.subs-count {
  margin: 24px 0 16px;
  color: var(--muted);
  font-size: 13px;
}

.subs-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(360px, 1fr));
  gap: 24px;
}

/* Large Card */
.sub-card-large {
  background: #0f182c;
  border: 1px solid rgba(255, 255, 255, 0.04);
  border-radius: 20px;
  padding: 24px;
  display: flex;
  flex-direction: column;
  gap: 20px;
  transition:
    transform 0.2s,
    box-shadow 0.2s;
  position: relative;
  overflow: hidden;
}
.sub-card-large:hover {
  transform: translateY(-4px);
  box-shadow: 0 14px 40px rgba(0, 0, 0, 0.3);
  border-color: rgba(123, 91, 255, 0.4);
}

.card-top {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
}
.card-brand {
  display: flex;
  gap: 14px;
  align-items: center;
}
.brand-icon {
  width: 52px;
  height: 52px;
  border-radius: 16px;
  display: grid;
  place-items: center;
  color: white;
  box-shadow: 0 8px 20px rgba(0, 0, 0, 0.2);
}
.brand-icon svg {
  width: 28px;
  height: 28px;
  fill: none;
  stroke: currentColor;
  stroke-width: 1.8;
}
.brand-info h3 {
  margin: 0 0 4px;
  font-size: 18px;
}
.brand-cat {
  font-size: 13px;
  color: var(--muted);
  display: flex;
  align-items: center;
  text-transform: capitalize;
}

.card-actions {
  display: flex;
  gap: 8px;
  opacity: 0;
  transition: 0.2s;
}
.sub-card-large:hover .card-actions {
  opacity: 1;
}
.icon-btn-sm {
  width: 32px;
  height: 32px;
  border-radius: 8px;
  display: grid;
  place-items: center;
  background: rgba(255, 255, 255, 0.05);
  color: var(--muted);
  border: none;
  cursor: pointer;
}
.icon-btn-sm:hover {
  background: rgba(255, 255, 255, 0.1);
  color: #fff;
}
.icon-btn-sm.danger:hover {
  color: #ff6b6b;
}
.icon-btn-sm svg {
  width: 16px;
  height: 16px;
  fill: none;
  stroke: currentColor;
  stroke-width: 2;
}

.card-mid {
  display: flex;
  justify-content: space-between;
  align-items: center;
}
.big-price {
  font-size: 24px;
  font-weight: 800;
  font-feature-settings: "tnum";
  letter-spacing: -0.5px;
}
.days-badge {
  border: 1px solid rgba(58, 222, 139, 0.4);
  color: #3ade8b;
  padding: 6px 12px;
  border-radius: 20px;
  font-size: 11px;
  font-weight: 700;
  letter-spacing: 0.5px;
}
.days-badge.warning {
  border-color: rgba(255, 159, 67, 0.4);
  color: #ff9f43;
}

.card-dates {
  background: #0a0f1d;
  padding: 12px 16px;
  border-radius: 12px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  border: 1px solid rgba(255, 255, 255, 0.03);
}
.date-col {
  display: flex;
  flex-direction: column;
  gap: 4px;
}
.date-col .label {
  font-size: 10px;
  color: #64748b;
  font-weight: 700;
  letter-spacing: 0.5px;
}
.date-col .val {
  font-size: 13px;
  font-weight: 600;
  color: #dfe7ff;
}
.arrow {
  color: #475569;
  font-size: 14px;
}

.card-footer {
  margin-top: auto;
}
.auto-renew {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 12px;
  color: #6366f1;
  background: rgba(99, 102, 241, 0.1);
  padding: 8px 12px;
  border-radius: 20px;
  width: fit-content;
}
.auto-renew svg {
  width: 14px;
  height: 14px;
  fill: none;
  stroke: currentColor;
  stroke-width: 2;
  animation: spin 4s linear infinite;
}
@keyframes spin {
  100% {
    transform: rotate(360deg);
  }
}

[data-theme="light"] .search-box {
  background: white;
  border-color: #e5e7eb;
}
[data-theme="light"] .search-box input {
  color: #1f2937;
}
[data-theme="light"] .clear-btn:hover {
  background: #f3f4f6;
  color: #111827;
}
[data-theme="light"] .dropdown select {
  background: white;
  border-color: #e5e7eb;
  color: #374151;
}
[data-theme="light"] .sort-group {
  background: white;
  border-color: #e5e7eb;
}
[data-theme="light"] .sort-btn:hover {
  color: #111827;
}
[data-theme="light"] .sort-btn.active {
  background: #eff6ff;
  color: #1d4ed8;
  border-color: #3b82f6;
}
[data-theme="light"] .sub-card-large {
  background: white;
  border-color: rgba(0, 0, 0, 0.05);
  box-shadow:
    0 4px 6px -1px rgba(0, 0, 0, 0.05),
    0 2px 4px -1px rgba(0, 0, 0, 0.03);
}
[data-theme="light"] .card-dates,
[data-theme="light"] .card-top .brand-icon {
  /* brand icon uses gradient but maybe needs shadow adjustment */
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
}
[data-theme="light"] .card-dates {
  background: #f9fafb;
  border-color: #f3f4f6;
}
[data-theme="light"] .date-col .val {
  color: #111827;
}
[data-theme="light"] .icon-btn-sm {
  background: #f3f4f6;
  color: #6b7280;
}
[data-theme="light"] .icon-btn-sm:hover {
  background: #e5e7eb;
  color: #111827;
}

@media (max-width: 600px) {
  .page-header {
    flex-direction: column;
    align-items: flex-start;
    gap: 16px;
  }
  .header-content h1 {
    font-size: 20px;
  }
  .pill-btn {
    width: 100%;
    justify-content: center;
  }
  .header-actions {
    width: 100%;
    display: flex;
    flex-direction: column;
    gap: 8px;
  }
  .toolbar {
    flex-direction: column;
    align-items: stretch;
  }
  .search-box {
    min-width: 0;
  }
  .filter-separator {
    display: none;
  }
  .filters-row {
    flex-wrap: wrap;
  }
  .dropdown,
  .dropdown select {
    flex: 1;
    min-width: 0;
  }
  .sort-group {
    width: 100%;
    overflow-x: auto;
  }
  .sort-btn {
    flex: 1;
    justify-content: center;
    white-space: nowrap;
  }
  .subs-grid {
    grid-template-columns: 1fr;
  }
  .card-actions {
    opacity: 1;
  }
}
</style>
