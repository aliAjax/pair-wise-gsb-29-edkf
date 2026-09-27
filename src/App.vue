<script setup lang="ts">
import { computed, reactive, ref } from "vue";

type Status = "待排" | "已派单" | "已签收" | "未完成";

type Elevator = { depth: number; width: number; height: number };

type Building = {
  id: string;
  name: string;
  elevator: Elevator | null;
  carryLimit: number; // 单人载重上限 kg
};

type LockedPlan = {
  floor: number;
  slot: string;
  tools: string[];
};

type Order = {
  id: string;
  code: string;
  item: string;
  rider: string;
  buildingId: string;
  address: string;
  length: number; // 外包装长 cm
  width: number; // 外包装宽 cm
  height: number; // 外包装高 cm
  weight: number; // kg
  floor: number;
  needStairs: boolean; // 是否需要走楼梯
  slot: string; // 两小时安装窗口
  tools: string[];
  date: string; // 任务日期
  status: Status;
  blocked: string[]; // 卡在哪一项核对
  locked: LockedPlan | null; // 确认派单后锁住楼层、工具、时段
  actualFloor: number | null; // 签收时登记的实际搬运楼层
  entryMethod: string; // 签收时登记的进门方式
  unfinishedReason: string;
  createdAt: string;
};

type SignForm = {
  open: boolean;
  actualFloor: number;
  entryMethod: string;
  outcome: "已签收" | "未完成";
  reason: string;
  error: string;
};

const storageKey = "hxwlfront-15-bulky-scheduler";
const riders = ["骑手A", "骑手B", "骑手C"];
const riderFilters = ["全部骑手", ...riders];
const slots = ["08:00-10:00", "10:00-12:00", "13:00-15:00", "15:00-17:00", "17:00-19:00"];
const toolOptions = ["爬楼机", "液压搬运车", "绑带套装", "防护毛毯", "拆门工具"];
const entryMethods = ["电梯入户", "楼梯搬运", "吊装入户", "拆门入户"];
const statuses: Status[] = ["待排", "已派单", "已签收", "未完成"];
const metricLabels = ["今日订单", "待排区", "已派单", "已签收"];
const stack = ["Vue3", "Vite", "TypeScript", "Element Plus", "Leaflet"];

const buildings: Building[] = [
  { id: "b1", name: "锦绣家园3栋", elevator: { depth: 200, width: 150, height: 230 }, carryLimit: 80 },
  { id: "b2", name: "湖畔公寓B座", elevator: { depth: 140, width: 110, height: 190 }, carryLimit: 60 },
  { id: "b3", name: "老城厢12号", elevator: null, carryLimit: 40 },
];

function todayStr() {
  const d = new Date();
  const m = String(d.getMonth() + 1).padStart(2, "0");
  const day = String(d.getDate()).padStart(2, "0");
  return `${d.getFullYear()}-${m}-${day}`;
}

function buildingOf(order: Order) {
  return buildings.find((b) => b.id === order.buildingId);
}

function buildingName(id: string) {
  return buildings.find((b) => b.id === id)?.name ?? "未知楼栋";
}

function dimsText(o: { length: number; width: number; height: number }) {
  return `${o.length}×${o.width}×${o.height}cm`;
}

function elevatorText(b: Building) {
  if (!b.elevator) return "无电梯";
  return `轿厢 ${b.elevator.depth}×${b.elevator.width}×${b.elevator.height}cm`;
}

function fitsElevator(order: Order, elevator: Elevator) {
  const pkg = [order.length, order.width, order.height].sort((a, b) => a - b);
  const car = [elevator.depth, elevator.width, elevator.height].sort((a, b) => a - b);
  return pkg[0] <= car[0] && pkg[1] <= car[1] && pkg[2] <= car[2];
}

// 派单核对：电梯尺寸、单人载重、两小时安装窗口放在一起核对，返回全部卡住的原因
function runChecks(order: Order, all: Order[]): string[] {
  const reasons: string[] = [];
  const building = buildingOf(order);
  if (!order.needStairs) {
    if (!building?.elevator) {
      reasons.push(`电梯尺寸：${building?.name ?? "该楼栋"}没有电梯，包装 ${dimsText(order)} 需改走楼梯或吊装`);
    } else if (!fitsElevator(order, building.elevator)) {
      reasons.push(`电梯尺寸：包装 ${dimsText(order)} 放不进${building.name}（${elevatorText(building)}）`);
    }
  }
  if (building && order.weight > building.carryLimit) {
    reasons.push(`单人载重：${order.weight}kg 超过 ${building.name} 上限 ${building.carryLimit}kg`);
  }
  const clash = all.some(
    (other) =>
      other.id !== order.id &&
      other.rider === order.rider &&
      other.date === order.date &&
      other.status === "已派单" &&
      other.locked?.slot === order.slot
  );
  if (clash) {
    reasons.push(`安装窗口：${order.rider} 当天 ${order.slot} 已被其他订单占用`);
  }
  return reasons;
}

function buildSeeds(): Order[] {
  const date = todayStr();
  const now = new Date().toISOString();
  const seeds: Order[] = [
    {
      id: "seed-1", code: "JD-101", item: "对开门冰箱", rider: "骑手A", buildingId: "b2",
      address: "湖畔路88号 1202室", length: 90, width: 80, height: 200, weight: 95, floor: 12,
      needStairs: false, slot: "10:00-12:00", tools: ["液压搬运车", "防护毛毯"], date,
      status: "待排", blocked: [], locked: null, actualFloor: null, entryMethod: "",
      unfinishedReason: "", createdAt: now,
    },
    {
      id: "seed-2", code: "JD-102", item: "滚筒洗衣机", rider: "骑手B", buildingId: "b1",
      address: "锦绣路12号 503室", length: 60, width: 65, height: 85, weight: 70, floor: 5,
      needStairs: false, slot: "10:00-12:00", tools: ["液压搬运车", "防护毛毯"], date,
      status: "待排", blocked: [], locked: null, actualFloor: null, entryMethod: "",
      unfinishedReason: "", createdAt: now,
    },
    {
      id: "seed-3", code: "JD-103", item: "实木书柜", rider: "骑手C", buildingId: "b3",
      address: "老城厢12号 602室", length: 120, width: 40, height: 180, weight: 35, floor: 6,
      needStairs: true, slot: "13:00-15:00", tools: ["爬楼机", "绑带套装"], date,
      status: "待排", blocked: [], locked: null, actualFloor: null, entryMethod: "",
      unfinishedReason: "", createdAt: now,
    },
    {
      id: "seed-4", code: "JD-104", item: "嵌入式洗碗机", rider: "骑手A", buildingId: "b1",
      address: "锦绣路12号 901室", length: 65, width: 60, height: 85, weight: 45, floor: 9,
      needStairs: false, slot: "15:00-17:00", tools: ["液压搬运车"], date,
      status: "已派单", blocked: [], locked: { floor: 9, slot: "15:00-17:00", tools: ["液压搬运车"] },
      actualFloor: null, entryMethod: "", unfinishedReason: "", createdAt: now,
    },
    {
      id: "seed-5", code: "JD-105", item: "激光电视", rider: "骑手B", buildingId: "b2",
      address: "湖畔路88号 304室", length: 150, width: 35, height: 95, weight: 38, floor: 3,
      needStairs: false, slot: "08:00-10:00", tools: ["防护毛毯"], date,
      status: "未完成", blocked: [], locked: null, actualFloor: null, entryMethod: "",
      unfinishedReason: "客户临时外出，电话约定改到次日上午", createdAt: now,
    },
  ];
  for (const seed of seeds) {
    if (seed.status === "待排") seed.blocked = runChecks(seed, seeds);
  }
  return seeds;
}

function loadOrders(): Order[] {
  const raw = localStorage.getItem(storageKey);
  if (!raw) return buildSeeds();
  try {
    return JSON.parse(raw) as Order[];
  } catch {
    return buildSeeds();
  }
}

function createBlankForm() {
  return {
    item: "",
    rider: riders[0],
    buildingId: buildings[0].id,
    address: "",
    length: 0,
    width: 0,
    height: 0,
    weight: 0,
    floor: 1,
    needStairs: "否",
    slot: slots[1],
    tools: [] as string[],
    date: todayStr(),
  };
}

const orders = ref<Order[]>(loadOrders());
const form = reactive(createBlankForm());
const riderFilter = ref(riderFilters[0]);
const checkRider = ref(riders[0]);
const checkDate = ref(todayStr());
const signForms = reactive<Record<string, SignForm>>({});

const formBuilding = computed(() => buildings.find((b) => b.id === form.buildingId) ?? buildings[0]);

const todayOrders = computed(() => orders.value.filter((o) => o.date === todayStr()));

const metrics = computed(() => [
  todayOrders.value.length,
  todayOrders.value.filter((o) => o.status === "待排").length,
  todayOrders.value.filter((o) => o.status === "已派单").length,
  todayOrders.value.filter((o) => o.status === "已签收").length,
]);

const visibleOrders = computed(() => {
  if (riderFilter.value.startsWith("全部")) return orders.value;
  return orders.value.filter((o) => o.rider === riderFilter.value);
});

const pendingOrders = computed(() => visibleOrders.value.filter((o) => o.status === "待排"));
const dispatchedOrders = computed(() => visibleOrders.value.filter((o) => o.status === "已派单"));
const finishedOrders = computed(() =>
  visibleOrders.value.filter((o) => o.status === "已签收" || o.status === "未完成")
);

const chartRows = computed(() =>
  statuses.map((status) => ({
    status,
    value: orders.value.filter((o) => o.status === status).length,
  }))
);

const maxChart = computed(() => Math.max(1, ...chartRows.value.map((row) => row.value)));

const checkList = computed(() =>
  orders.value
    .filter((o) => o.rider === checkRider.value && o.date === checkDate.value)
    .sort((a, b) => a.slot.localeCompare(b.slot))
);

const checkSummary = computed(() => {
  const list = checkList.value;
  const count = (s: Status) => list.filter((o) => o.status === s).length;
  return `共 ${list.length} 单 · 待排 ${count("待排")} · 已派单 ${count("已派单")} · 已签收 ${count("已签收")} · 未完成 ${count("未完成")}`;
});

function persist() {
  localStorage.setItem(storageKey, JSON.stringify(orders.value));
}

function statusClass(status: Status) {
  return { 待排: "pending", 已派单: "dispatched", 已签收: "done", 未完成: "failed" }[status];
}

function submit() {
  const order: Order = {
    id: crypto.randomUUID(),
    code: `JD-${Date.now().toString(36).toUpperCase().slice(-6)}`,
    item: form.item.trim(),
    rider: form.rider,
    buildingId: form.buildingId,
    address: form.address.trim(),
    length: Number(form.length),
    width: Number(form.width),
    height: Number(form.height),
    weight: Number(form.weight),
    floor: Number(form.floor),
    needStairs: form.needStairs === "是",
    slot: form.slot,
    tools: [...form.tools],
    date: form.date || todayStr(),
    status: "待排",
    blocked: [],
    locked: null,
    actualFloor: null,
    entryMethod: "",
    unfinishedReason: "",
    createdAt: new Date().toISOString(),
  };
  // 登记时先核对一遍，不满足的直接留在待排区并写明卡在哪一项
  order.blocked = runChecks(order, orders.value);
  orders.value = [order, ...orders.value];
  Object.assign(form, createBlankForm());
  persist();
}

function dispatch(order: Order) {
  const reasons = runChecks(order, orders.value);
  order.blocked = reasons;
  if (reasons.length === 0) {
    order.status = "已派单";
    // 确认后楼层、工具和时段锁住
    order.locked = { floor: order.floor, slot: order.slot, tools: [...order.tools] };
  }
  persist();
}

function release(order: Order) {
  const slot = order.locked?.slot ?? order.slot;
  const ok = window.confirm(`改约需先释放原时段：${order.rider} 当天 ${slot} 将被释放，订单回到待排区。确认释放？`);
  if (!ok) return;
  order.status = "待排";
  order.locked = null;
  order.blocked = [];
  persist();
}

function signForm(order: Order): SignForm {
  if (!signForms[order.id]) {
    signForms[order.id] = {
      open: false,
      actualFloor: order.locked?.floor ?? order.floor,
      entryMethod: entryMethods[0],
      outcome: "已签收",
      reason: "",
      error: "",
    };
  }
  return signForms[order.id];
}

function toggleSign(order: Order) {
  const sf = signForm(order);
  sf.open = !sf.open;
  sf.error = "";
}

function confirmSign(order: Order) {
  const sf = signForm(order);
  sf.error = "";
  if (sf.outcome === "未完成" && !sf.reason.trim()) {
    sf.error = "请填写未完成原因";
    return;
  }
  if (sf.outcome === "已签收" && (sf.actualFloor === null || Number.isNaN(Number(sf.actualFloor)))) {
    sf.error = "请登记实际搬运楼层";
    return;
  }
  order.status = sf.outcome;
  order.actualFloor = Number(sf.actualFloor);
  order.entryMethod = sf.entryMethod;
  order.unfinishedReason = sf.outcome === "未完成" ? sf.reason.trim() : "";
  sf.open = false;
  persist();
}

function remove(id: string) {
  orders.value = orders.value.filter((o) => o.id !== id);
  persist();
}
</script>

<template>
  <main class="app">
    <div class="shell">
      <header class="topbar">
        <div>
          <p class="eyebrow">物流行业前端最小闭环</p>
          <h1>大件入户排班台</h1>
          <p class="subtitle">
            登记外包装尺寸、重量、楼层和走楼梯需求；派单时把电梯尺寸、单人载重、两小时安装窗口放在一起核对，
            不满足的订单留在待排区并写明卡在哪一项。
          </p>
        </div>
        <div class="stack">
          <span v-for="item in stack" :key="item" class="tag">{{ item }}</span>
        </div>
      </header>

      <section class="metrics">
        <article v-for="(label, index) in metricLabels" :key="label" class="metric">
          <span>{{ label }}</span>
          <strong>{{ metrics[index] }}</strong>
        </article>
      </section>

      <section class="workspace">
        <form class="panel" @submit.prevent="submit">
          <h2>新订单登记</h2>
          <div class="form-grid">
            <label>
              品名
              <input v-model="form.item" required placeholder="如：对开门冰箱" />
            </label>
            <label>
              骑手
              <select v-model="form.rider" required>
                <option v-for="rider in riders" :key="rider">{{ rider }}</option>
              </select>
            </label>
            <label>
              楼栋
              <select v-model="form.buildingId" required>
                <option v-for="b in buildings" :key="b.id" :value="b.id">{{ b.name }}</option>
              </select>
              <p class="form-hint">{{ elevatorText(formBuilding) }} · 单人载重上限 {{ formBuilding.carryLimit }}kg</p>
            </label>
            <label>
              地址
              <input v-model="form.address" required placeholder="路名 + 门牌室号" />
            </label>
            <div class="inline-fields">
              <label>外包装长 cm<input v-model.number="form.length" type="number" min="1" required /></label>
              <label>宽 cm<input v-model.number="form.width" type="number" min="1" required /></label>
              <label>高 cm<input v-model.number="form.height" type="number" min="1" required /></label>
            </div>
            <div class="inline-fields">
              <label>重量 kg<input v-model.number="form.weight" type="number" min="1" required /></label>
              <label>楼层<input v-model.number="form.floor" type="number" min="1" required /></label>
              <label>
                是否走楼梯
                <select v-model="form.needStairs">
                  <option>否</option>
                  <option>是</option>
                </select>
              </label>
            </div>
            <label>
              两小时安装窗口
              <select v-model="form.slot">
                <option v-for="slot in slots" :key="slot">{{ slot }}</option>
              </select>
            </label>
            <div>
              <span class="muted">搬运工具</span>
              <div class="tools-row">
                <label v-for="tool in toolOptions" :key="tool">
                  <input v-model="form.tools" type="checkbox" :value="tool" />{{ tool }}
                </label>
              </div>
            </div>
            <label>
              任务日期
              <input v-model="form.date" type="date" required />
            </label>
            <button type="submit">登记并核对</button>
          </div>
        </form>

        <section class="list-panel">
          <div class="toolbar">
            <h2>排班台</h2>
            <select v-model="riderFilter" class="filter-select">
              <option v-for="item in riderFilters" :key="item">{{ item }}</option>
            </select>
          </div>

          <h3 class="group-title">待排区 <span class="count">{{ pendingOrders.length }}</span></h3>
          <div class="record-grid">
            <div v-if="pendingOrders.length === 0" class="empty">待排区已清空</div>
            <article v-for="order in pendingOrders" :key="order.id" class="record">
              <div class="record-head">
                <p class="record-title">{{ order.item }} · {{ order.code }}</p>
                <span class="status" :class="statusClass(order.status)">{{ order.status }}</span>
              </div>
              <div class="details">
                <span>骑手: {{ order.rider }}</span>
                <span>楼栋: {{ buildingName(order.buildingId) }}</span>
                <span>包装: {{ dimsText(order) }}</span>
                <span>重量: {{ order.weight }}kg</span>
                <span>楼层: {{ order.floor }} 层</span>
                <span>走楼梯: {{ order.needStairs ? "是（免电梯核对）" : "否" }}</span>
                <span>地址: {{ order.address }}</span>
                <span>日期: {{ order.date }}</span>
              </div>
              <div v-if="order.blocked.length" class="blocked">
                卡在以下核对项：
                <ul>
                  <li v-for="reason in order.blocked" :key="reason">{{ reason }}</li>
                </ul>
              </div>
              <p v-else class="ok-line">三项核对通过，可直接派单</p>
              <div class="pending-editor">
                <div class="inline-fields">
                  <label>楼层<input v-model.number="order.floor" type="number" min="1" @change="persist" /></label>
                  <label>
                    安装窗口
                    <select v-model="order.slot" @change="persist">
                      <option v-for="slot in slots" :key="slot">{{ slot }}</option>
                    </select>
                  </label>
                  <label>
                    走楼梯
                    <select v-model="order.needStairs" @change="persist">
                      <option :value="false">否（电梯）</option>
                      <option :value="true">是（走楼梯）</option>
                    </select>
                  </label>
                </div>
                <div>
                  <span class="muted">搬运工具</span>
                  <div class="tools-row">
                    <label v-for="tool in toolOptions" :key="tool">
                      <input v-model="order.tools" type="checkbox" :value="tool" @change="persist" />{{ tool }}
                    </label>
                  </div>
                </div>
                <div class="actions">
                  <button type="button" @click="dispatch(order)">核对并派单</button>
                  <button class="danger" type="button" @click="remove(order.id)">删除</button>
                </div>
              </div>
            </article>
          </div>

          <h3 class="group-title">已派单 · 已锁定 <span class="count">{{ dispatchedOrders.length }}</span></h3>
          <div class="record-grid">
            <div v-if="dispatchedOrders.length === 0" class="empty">暂无已派单订单</div>
            <article v-for="order in dispatchedOrders" :key="order.id" class="record">
              <div class="record-head">
                <p class="record-title">{{ order.item }} · {{ order.code }}</p>
                <span class="status" :class="statusClass(order.status)">{{ order.status }}</span>
              </div>
              <div class="details">
                <span>骑手: {{ order.rider }}</span>
                <span>楼栋: {{ buildingName(order.buildingId) }}</span>
                <span>包装: {{ dimsText(order) }}</span>
                <span>重量: {{ order.weight }}kg</span>
                <span>地址: {{ order.address }}</span>
                <span>日期: {{ order.date }}</span>
              </div>
              <div class="locked-line">
                <span class="lock-tag">🔒 楼层 {{ order.locked?.floor }} 层</span>
                <span class="lock-tag">🔒 时段 {{ order.locked?.slot }}</span>
                <span class="lock-tag">🔒 工具 {{ order.locked?.tools.join("、") || "无" }}</span>
              </div>
              <div class="actions">
                <button type="button" @click="toggleSign(order)">签收登记</button>
                <button class="secondary" type="button" @click="release(order)">释放时段改约</button>
              </div>
              <div v-if="signForm(order).open" class="sign-panel">
                <div class="radio-row">
                  <label><input v-model="signForm(order).outcome" type="radio" value="已签收" />已签收</label>
                  <label><input v-model="signForm(order).outcome" type="radio" value="未完成" />未完成</label>
                </div>
                <div class="inline-fields">
                  <label>实际搬运楼层<input v-model.number="signForm(order).actualFloor" type="number" min="0" /></label>
                  <label>
                    进门方式
                    <select v-model="signForm(order).entryMethod">
                      <option v-for="m in entryMethods" :key="m">{{ m }}</option>
                    </select>
                  </label>
                </div>
                <label v-if="signForm(order).outcome === '未完成'">
                  未完成原因
                  <textarea v-model="signForm(order).reason" placeholder="如：客户临时外出，约定改期" />
                </label>
                <p v-if="signForm(order).error" class="error-text">{{ signForm(order).error }}</p>
                <div class="actions">
                  <button type="button" @click="confirmSign(order)">确认登记</button>
                  <button class="secondary" type="button" @click="toggleSign(order)">取消</button>
                </div>
              </div>
            </article>
          </div>

          <h3 class="group-title">已签收 / 未完成 <span class="count">{{ finishedOrders.length }}</span></h3>
          <div class="record-grid">
            <div v-if="finishedOrders.length === 0" class="empty">暂无完成记录</div>
            <article v-for="order in finishedOrders" :key="order.id" class="record">
              <div class="record-head">
                <p class="record-title">{{ order.item }} · {{ order.code }}</p>
                <span class="status" :class="statusClass(order.status)">{{ order.status }}</span>
              </div>
              <div class="details">
                <span>骑手: {{ order.rider }}</span>
                <span>楼栋: {{ buildingName(order.buildingId) }}</span>
                <span>日期: {{ order.date }}</span>
                <span>时段: {{ order.locked?.slot ?? order.slot }}</span>
                <span v-if="order.actualFloor !== null">实际搬运楼层: {{ order.actualFloor }} 层</span>
                <span v-if="order.entryMethod">进门方式: {{ order.entryMethod }}</span>
              </div>
              <p v-if="order.unfinishedReason" class="blocked">未完成原因：{{ order.unfinishedReason }}</p>
              <div class="actions">
                <button class="danger" type="button" @click="remove(order.id)">删除</button>
              </div>
            </article>
          </div>

          <div class="mini-chart">
            <div v-for="row in chartRows" :key="row.status" class="bar">
              <span>{{ row.status }}</span>
              <div class="bar-track"><div class="bar-fill" :style="{ width: `${(row.value / maxChart) * 100}%` }" /></div>
              <strong>{{ row.value }}</strong>
            </div>
          </div>
        </section>
      </section>

      <section class="panel rider-panel">
        <div class="toolbar">
          <h2>骑手当天任务核对</h2>
          <div class="check-controls">
            <select v-model="checkRider">
              <option v-for="rider in riders" :key="rider">{{ rider }}</option>
            </select>
            <input v-model="checkDate" type="date" />
          </div>
        </div>
        <p class="muted">{{ checkRider }} · {{ checkDate }} · {{ checkSummary }}</p>
        <table class="check-table">
          <thead>
            <tr>
              <th>单号</th>
              <th>品名</th>
              <th>楼栋 / 地址</th>
              <th>时段</th>
              <th>状态</th>
              <th>核对与原因</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="order in checkList" :key="order.id">
              <td>{{ order.code }}</td>
              <td>{{ order.item }}</td>
              <td>{{ buildingName(order.buildingId) }} · {{ order.address }}</td>
              <td>{{ order.locked?.slot ?? order.slot }}</td>
              <td><span class="status" :class="statusClass(order.status)">{{ order.status }}</span></td>
              <td>
                <span v-if="order.status === '未完成'" class="reason">未完成原因：{{ order.unfinishedReason }}</span>
                <span v-else-if="order.status === '已签收'">{{ order.entryMethod }} · 实际搬到 {{ order.actualFloor }} 层</span>
                <span v-else-if="order.status === '待排' && order.blocked.length" class="reason">卡在：{{ order.blocked.join("；") }}</span>
                <span v-else-if="order.status === '待排'">核对通过，待派单</span>
                <span v-else>已锁定 {{ order.locked?.floor }} 层 / {{ order.locked?.slot }}</span>
              </td>
            </tr>
            <tr v-if="checkList.length === 0">
              <td colspan="6" class="empty">该骑手当天暂无任务</td>
            </tr>
          </tbody>
        </table>
      </section>
    </div>
  </main>
</template>
