<script setup lang="ts">
import { computed, reactive, ref, watch } from "vue";

/* ========================= 类型定义 ========================= */

type OrderStatus = "pending" | "scheduled" | "signed";

type Building = {
  id: string;
  name: string;
  /** 轿厢内部尺寸，0 表示无电梯 */
  cabW: number;
  cabD: number;
  cabH: number;
  /** 轿厢门洞尺寸 */
  doorW: number;
  doorH: number;
};

type Rider = {
  id: string;
  name: string;
  /** 单人可搬重量上限 kg */
  maxLoad: number;
  /** 携带工具 */
  tools: string[];
};

type DraftAssignment = {
  buildingId: string;
  riderId: string;
  slot: string;
  tools: string[];
};

type HistoryEntry = {
  at: string;
  text: string;
};

type Order = {
  id: string;
  no: string;
  itemName: string;
  address: string;
  /** 外包装尺寸 cm / 重量 kg */
  pkgL: number;
  pkgW: number;
  pkgH: number;
  weight: number;
  floor: number;
  viaStairs: boolean;
  requireTools: string[];
  /** 预约日期 YYYY-MM-DD */
  date: string;
  customer: string;
  status: OrderStatus;
  /** 待排区里的派单草稿 */
  draft: DraftAssignment;
  /** 已确认（锁定）信息 */
  buildingId?: string;
  riderId?: string;
  slot?: string;
  lockedTools?: string[];
  lockedFloor?: number;
  lockedViaStairs?: boolean;
  /** 签收信息 */
  actualFloor?: number;
  entryWay?: string;
  signNote?: string;
  /** 未完成 */
  failReason?: string;
  failNote?: string;
  history: HistoryEntry[];
  createdAt: string;
};

type CheckResult = {
  ok: boolean;
  code: string;
  label: string;
  detail: string;
};

type Store = {
  orders: Order[];
  buildings: Building[];
  riders: Rider[];
  seq: number;
};

/* ========================= 基础数据 ========================= */

const STORAGE_KEY = "bulky-indoor-dispatch-v1";

const TOOL_OPTIONS = ["手推车", "捆扎带", "爬楼机", "吊装绳索", "护角毯", "双人搭班"] as const;

const SLOTS = [
  "08:00-10:00",
  "10:00-12:00",
  "12:00-14:00",
  "14:00-16:00",
  "16:00-18:00",
  "18:00-20:00"
] as const;

const ENTRY_WAYS = ["电梯入户", "楼梯搬运", "吊装入户", "平躺进梯后上楼", "拆除包装后入户"] as const;

const FAIL_REASONS = [
  "电梯实际放不下",
  "客户不在家",
  "楼道转弯受限",
  "重量超出搬运能力",
  "缺少工具",
  "时段冲突延误"
] as const;

const ALL_CLEARANCE = 2; // cm，比对电梯时预留余量

function todayStr() {
  const d = new Date();
  const m = `${d.getMonth() + 1}`.padStart(2, "0");
  const day = `${d.getDate()}`.padStart(2, "0");
  return `${d.getFullYear()}-${m}-${day}`;
}

function nowText() {
  return new Date().toLocaleString("zh-CN", { hour12: false });
}

function uid() {
  return crypto.randomUUID();
}

/* ========================= 初始演示数据 ========================= */

function seedStore(): Store {
  const buildings: Building[] = [
    { id: "b1", name: "阳光花园 3 栋", cabW: 110, cabD: 210, cabH: 230, doorW: 90, doorH: 210 },
    { id: "b2", name: "梧桐里 7 栋", cabW: 80, cabD: 140, cabH: 200, doorW: 75, doorH: 200 },
    { id: "b3", name: "江景苑 12 栋（老楼无电梯）", cabW: 0, cabD: 0, cabH: 0, doorW: 0, doorH: 0 }
  ];
  const riders: Rider[] = [
    { id: "r1", name: "张磊", maxLoad: 80, tools: ["手推车", "捆扎带", "护角毯"] },
    { id: "r2", name: "王强", maxLoad: 120, tools: ["手推车", "捆扎带", "爬楼机", "护角毯", "双人搭班"] },
    { id: "r3", name: "李军", maxLoad: 95, tools: ["手推车", "捆扎带", "吊装绳索"] }
  ];
  const t = todayStr();
  const log = (text: string): HistoryEntry[] => [{ at: nowText(), text }];
  const orders: Order[] = [
    {
      id: uid(),
      no: "BJ20260927-01",
      itemName: "对开门冰箱",
      address: "阳光花园 3 栋 1502",
      pkgL: 91, pkgW: 76, pkgH: 190, weight: 118,
      floor: 15, viaStairs: false,
      requireTools: ["手推车", "捆扎带"],
      date: t, customer: "陈先生",
      status: "pending",
      draft: { buildingId: "b1", riderId: "r1", slot: "10:00-12:00", tools: ["手推车", "捆扎带"] },
      history: log("订单登记，进入待排区")
    },
    {
      id: uid(),
      no: "BJ20260927-02",
      itemName: "滚筒洗衣机",
      address: "梧桐里 7 栋 601",
      pkgL: 65, pkgW: 60, pkgH: 85, weight: 72,
      floor: 6, viaStairs: false,
      requireTools: ["手推车"],
      date: t, customer: "刘女士",
      status: "pending",
      draft: { buildingId: "b2", riderId: "r2", slot: "14:00-16:00", tools: ["手推车"] },
      history: log("订单登记，进入待排区")
    },
    {
      id: uid(),
      no: "BJ20260927-03",
      itemName: "立式冰柜",
      address: "江景苑 12 栋 502",
      pkgL: 70, pkgW: 68, pkgH: 178, weight: 95,
      floor: 5, viaStairs: true,
      requireTools: ["爬楼机", "双人搭班"],
      date: t, customer: "赵师傅",
      status: "pending",
      draft: { buildingId: "b3", riderId: "r1", slot: "16:00-18:00", tools: ["爬楼机", "双人搭班"] },
      history: log("订单登记，进入待排区（老楼需走楼梯）")
    },
    {
      id: uid(),
      no: "BJ20260927-04",
      itemName: "65 寸电视",
      address: "阳光花园 3 栋 803",
      pkgL: 158, pkgW: 18, pkgH: 96, weight: 32,
      floor: 8, viaStairs: false,
      requireTools: ["护角毯"],
      date: t, customer: "孙女士",
      status: "scheduled",
      draft: { buildingId: "b1", riderId: "r2", slot: "08:00-10:00", tools: ["护角毯"] },
      buildingId: "b1", riderId: "r2", slot: "08:00-10:00",
      lockedTools: ["护角毯"], lockedFloor: 8, lockedViaStairs: false,
      history: log("核对通过：电梯/载重/工具/窗口四项满足，已确认并锁定 楼层·工具·时段")
    },
    {
      id: uid(),
      no: "BJ20260927-05",
      itemName: "嵌入式洗碗机",
      address: "梧桐里 7 栋 302",
      pkgL: 60, pkgW: 60, pkgH: 82, weight: 45,
      floor: 3, viaStairs: false,
      requireTools: ["手推车", "护角毯"],
      date: t, customer: "周先生",
      status: "signed",
      draft: { buildingId: "b2", riderId: "r1", slot: "10:00-12:00", tools: ["手推车", "护角毯"] },
      buildingId: "b2", riderId: "r1", slot: "10:00-12:00",
      lockedTools: ["手推车", "护角毯"], lockedFloor: 3, lockedViaStairs: false,
      actualFloor: 3, entryWay: "电梯入户", signNote: "客户现场确认无损",
      history: log("签收完成：实际搬运楼层 3 层，进门方式 电梯入户")
    }
  ];
  return { orders, buildings, riders, seq: 6 };
}

function loadStore(): Store {
  const raw = localStorage.getItem(STORAGE_KEY);
  if (!raw) return seedStore();
  try {
    const parsed = JSON.parse(raw) as Store;
    if (!parsed.orders || !parsed.buildings || !parsed.riders) return seedStore();
    return parsed;
  } catch {
    return seedStore();
  }
}

const store = reactive<Store>(loadStore());

watch(
  store,
  (value) => localStorage.setItem(STORAGE_KEY, JSON.stringify(value)),
  { deep: true }
);

/* ========================= 页面状态 ========================= */

const tab = ref<"dispatch" | "riders" | "base">("dispatch");
const workDate = ref(todayStr());

/* ---------------- 新增订单表单 ---------------- */

const blankForm = () => ({
  itemName: "",
  address: "",
  customer: "",
  pkgL: 200,
  pkgW: 80,
  pkgH: 80,
  weight: 60,
  floor: 1,
  viaStairs: false,
  requireTools: [] as string[],
  buildingId: store.buildings[0]?.id ?? ""
});

const form = reactive(blankForm());

function addOrder() {
  if (!form.itemName || !form.address) return;
  const buildingId = form.viaStairs ? form.buildingId : form.buildingId || store.buildings[0]?.id || "";
  store.orders.unshift({
    id: uid(),
    no: `BJ${workDate.value.replaceAll("-", "")}-${`${store.seq}`.padStart(2, "0")}`,
    itemName: form.itemName,
    address: form.address,
    customer: form.customer,
    pkgL: Number(form.pkgL),
    pkgW: Number(form.pkgW),
    pkgH: Number(form.pkgH),
    weight: Number(form.weight),
    floor: Number(form.floor),
    viaStairs: form.viaStairs,
    requireTools: [...form.requireTools],
    date: workDate.value,
    status: "pending",
    draft: {
      buildingId,
      riderId: store.riders[0]?.id ?? "",
      slot: SLOTS[0],
      tools: [...form.requireTools]
    },
    history: [{ at: nowText(), text: "订单登记，进入待排区" }],
    createdAt: new Date().toISOString()
  });
  store.seq += 1;
  Object.assign(form, blankForm());
}

/* ========================= 派单核对规则 ========================= */

function sorted3(a: number, b: number, c: number) {
  return [a, b, c].sort((x, y) => x - y);
}

function buildingOf(id?: string) {
  return store.buildings.find((b) => b.id === id);
}
function riderOf(id?: string) {
  return store.riders.find((r) => r.id === id);
}

/**
 * 四项核对：电梯尺寸 / 单人载重 / 工具 / 两小时安装窗口
 */
function evaluate(order: Order): CheckResult[] {
  const checks: CheckResult[] = [];
  const building = buildingOf(order.draft.buildingId);
  const rider = riderOf(order.draft.riderId);

  /* ① 电梯（走楼梯则核对楼梯可行性） */
  if (order.viaStairs) {
    if (order.floor > 6 && !order.draft.tools.includes("爬楼机")) {
      checks.push({
        ok: false,
        code: "elevator",
        label: "电梯 / 楼梯",
        detail: `该单确认走楼梯、位于 ${order.floor} 层（>6 层），必须携带爬楼机，否则不予派单`
      });
    } else {
      checks.push({
        ok: true,
        code: "elevator",
        label: "电梯 / 楼梯",
        detail: `确认走楼梯 ${order.floor} 层${order.floor > 6 ? "，已配爬楼机" : ""}，按楼梯方案入户`
      });
    }
  } else if (!building) {
    checks.push({ ok: false, code: "elevator", label: "电梯 / 楼梯", detail: "未选择楼栋，无法核对轿厢尺寸" });
  } else if (!building.cabW) {
    checks.push({
      ok: false,
      code: "elevator",
      label: "电梯 / 楼梯",
      detail: `「${building.name}」无电梯，但该单未勾选走楼梯，请改勾走楼梯或更换楼栋`
    });
  } else {
    const [pMin, pMid, pMax] = sorted3(order.pkgL, order.pkgW, order.pkgH);
    const [cMin, cMid, cMax] = sorted3(building.cabW, building.cabD, building.cabH);
    const cabFit =
      pMin + ALL_CLEARANCE <= cMin &&
      pMid + ALL_CLEARANCE <= cMid &&
      pMax + ALL_CLEARANCE <= cMax;
    // 门洞：包装最小的两个截面要过得了门
    const doorFit = pMin + ALL_CLEARANCE <= building.doorW && pMid + ALL_CLEARANCE <= building.doorH;
    if (cabFit && doorFit) {
      checks.push({
        ok: true,
        code: "elevator",
        label: "电梯 / 楼梯",
        detail: `包装 ${order.pkgL}×${order.pkgW}×${order.pkgH}cm 可旋转进入轿厢 ${building.cabW}×${building.cabD}×${building.cabH}cm，门洞 ${building.doorW}×${building.doorH}cm 可通过`
      });
    } else {
      const parts: string[] = [];
      if (!cabFit) {
        parts.push(
          `轿厢 ${building.cabW}×${building.cabD}×${building.cabH}cm 放不下包装 ${order.pkgL}×${order.pkgW}×${order.pkgH}cm（已预留 ${ALL_CLEARANCE}cm 余量，允许任意旋转）`
        );
      }
      if (!doorFit) {
        parts.push(`包装截面过不了轿厢门洞 ${building.doorW}×${building.doorH}cm`);
      }
      checks.push({ ok: false, code: "elevator", label: "电梯 / 楼梯", detail: parts.join("；") });
    }
  }

  /* ② 单人载重（双人搭班按 1.8 倍计） */
  if (!rider) {
    checks.push({ ok: false, code: "load", label: "单人载重", detail: "未选择骑手" });
  } else {
    const teamed = order.draft.tools.includes("双人搭班");
    const capacity = teamed ? Math.round(rider.maxLoad * 1.8) : rider.maxLoad;
    if (order.weight <= capacity) {
      checks.push({
        ok: true,
        code: "load",
        label: "单人载重",
        detail: `包装 ${order.weight}kg ≤ ${rider.name} 单人上限 ${rider.maxLoad}kg${teamed ? `（双人搭班按 ${capacity}kg 计）` : ""}`
      });
    } else {
      checks.push({
        ok: false,
        code: "load",
        label: "单人载重",
        detail: `包装 ${order.weight}kg 超过 ${rider.name} 单人载重 ${rider.maxLoad}kg${teamed ? `（双人搭班后 ${capacity}kg 仍不足）` : "，可加配双人搭班或改派他人"}`
      });
    }
  }

  /* ③ 工具配备 */
  if (!rider) {
    checks.push({ ok: false, code: "tools", label: "工具配备", detail: "未选择骑手，无法核对工具" });
  } else {
    const missing = order.draft.tools.filter((t) => !rider.tools.includes(t));
    if (missing.length === 0) {
      checks.push({
        ok: true,
        code: "tools",
        label: "工具配备",
        detail: order.draft.tools.length ? `所需 ${order.draft.tools.join("、")} 骑手均已携带` : "该单无需特殊工具"
      });
    } else {
      checks.push({
        ok: false,
        code: "tools",
        label: "工具配备",
        detail: `${rider.name} 缺少：${missing.join("、")}（现有：${rider.tools.join("、") || "无"}）`
      });
    }
  }

  /* ④ 两小时安装窗口：窗口时长 + 同骑手同日不撞单 */
  const range = /^(\d{2}):(\d{2})-(\d{2}):(\d{2})$/.exec(order.draft.slot);
  if (!range) {
    checks.push({ ok: false, code: "window", label: "两小时安装窗口", detail: "未选择时段" });
  } else {
    const start = Number(range[1]) * 60 + Number(range[2]);
    const end = Number(range[3]) * 60 + Number(range[4]);
    if (end - start !== 120) {
      checks.push({
        ok: false,
        code: "window",
        label: "两小时安装窗口",
        detail: `时段 ${order.draft.slot} 不是两小时窗口，安装+入户必须落在单个两小时窗口内`
      });
    } else if (!rider) {
      checks.push({ ok: false, code: "window", label: "两小时安装窗口", detail: "未选择骑手，无法核对窗口占用" });
    } else {
      const clash = store.orders.find(
        (o) =>
          o.id !== order.id &&
          o.status === "scheduled" &&
          o.date === order.date &&
          o.riderId === rider.id &&
          o.slot === order.draft.slot
      );
      if (clash) {
        checks.push({
          ok: false,
          code: "window",
          label: "两小时安装窗口",
          detail: `${rider.name} 在 ${order.date} ${order.draft.slot} 已占用（订单 ${clash.no}），请换窗口或骑手`
        });
      } else {
        checks.push({
          ok: true,
          code: "window",
          label: "两小时安装窗口",
          detail: `${order.draft.slot} 为两小时窗口，同骑手当日无撞单`
        });
      }
    }
  }

  return checks;
}

function blockers(order: Order): CheckResult[] {
  return evaluate(order).filter((c) => !c.ok);
}
function canConfirm(order: Order) {
  return order.status === "pending" && blockers(order).length === 0;
}

/* ========================= 派单动作 ========================= */

function pushLog(order: Order, text: string) {
  order.history.unshift({ at: nowText(), text });
}

function confirmOrder(order: Order) {
  if (!canConfirm(order)) return;
  order.status = "scheduled";
  order.buildingId = order.draft.buildingId;
  order.riderId = order.draft.riderId;
  order.slot = order.draft.slot;
  order.lockedTools = [...order.draft.tools];
  order.lockedFloor = order.floor;
  order.lockedViaStairs = order.viaStairs;
  order.failReason = undefined;
  order.failNote = undefined;
  pushLog(order, `核对通过，已确认派单并锁定：楼层 ${order.floor} 层 / 工具 ${order.draft.tools.join("、") || "无"} / 时段 ${order.draft.slot}（改约须先释放原时段）`);
}

/** 改约：先释放原时段，订单回到待排区重新核对 */
function releaseOrder(order: Order) {
  const oldSlot = order.slot;
  order.status = "pending";
  order.failReason = undefined;
  order.failNote = undefined;
  order.buildingId = undefined;
  order.riderId = undefined;
  order.slot = undefined;
  order.lockedTools = undefined;
  order.lockedFloor = undefined;
  order.lockedViaStairs = undefined;
  order.draft.slot = SLOTS[(SLOTS.indexOf(oldSlot as (typeof SLOTS)[number]) + 1) % SLOTS.length] ?? SLOTS[0];
  pushLog(order, `改约：已释放原时段 ${oldSlot}，楼层/工具/时段解锁，退回待排区重新核对`);
}

function removeOrder(order: Order) {
  store.orders = store.orders.filter((o) => o.id !== order.id);
}

/* ---------------- 签收 / 未完成 弹窗 ---------------- */

const signTarget = ref<Order | null>(null);
const signForm = reactive({ actualFloor: 1, entryWay: ENTRY_WAYS[0], signNote: "" });
const failTarget = ref<Order | null>(null);
const failForm = reactive({ reason: FAIL_REASONS[0], note: "" });

function openSign(order: Order) {
  signTarget.value = order;
  signForm.actualFloor = order.lockedFloor ?? order.floor;
  signForm.entryWay = order.lockedViaStairs ? "楼梯搬运" : "电梯入户";
  signForm.signNote = "";
}

function submitSign() {
  const order = signTarget.value;
  if (!order) return;
  order.status = "signed";
  order.actualFloor = Number(signForm.actualFloor);
  order.entryWay = signForm.entryWay;
  order.signNote = signForm.signNote;
  order.failReason = undefined;
  order.failNote = undefined;
  pushLog(order, `签收完成：实际搬运楼层 ${signForm.actualFloor} 层，进门方式「${signForm.entryWay}」${signForm.signNote ? `，备注：${signForm.signNote}` : ""}`);
  signTarget.value = null;
}

function openFail(order: Order) {
  failTarget.value = order;
  failForm.reason = FAIL_REASONS[0];
  failForm.note = "";
}

function submitFail() {
  const order = failTarget.value;
  if (!order) return;
  order.status = "pending";
  // 原派单选择同步回草稿，待排区再次核对的就是现场这套方案
  order.draft.buildingId = order.buildingId ?? order.draft.buildingId;
  order.draft.riderId = order.riderId ?? order.draft.riderId;
  order.draft.slot = order.slot ?? order.draft.slot;
  order.draft.tools = order.lockedTools ? [...order.lockedTools] : order.draft.tools;
  order.buildingId = undefined;
  order.riderId = undefined;
  order.slot = undefined;
  order.lockedTools = undefined;
  order.lockedFloor = undefined;
  order.lockedViaStairs = undefined;
  order.failReason = failForm.reason;
  order.failNote = failForm.note;
  pushLog(order, `未完成退回待排：原因「${failForm.reason}」${failForm.note ? `，说明：${failForm.note}` : ""}`);
  failTarget.value = null;
}

/* ========================= 列表与统计 ========================= */

const dayOrders = computed(() => store.orders.filter((o) => o.date === workDate.value));
const pendingOrders = computed(() => dayOrders.value.filter((o) => o.status === "pending"));
const scheduledOrders = computed(() => dayOrders.value.filter((o) => o.status === "scheduled"));
const signedOrders = computed(() => dayOrders.value.filter((o) => o.status === "signed"));

const metrics = computed(() => [
  { label: "当日订单", value: dayOrders.value.length },
  { label: "待排（含卡点）", value: pendingOrders.value.length },
  { label: "已锁定排班", value: scheduledOrders.value.length },
  { label: "已签收", value: signedOrders.value.length }
]);

/* 骑手视图 */
const riderFilter = ref<string>("all");
const riderTasks = computed(() => {
  return store.riders
    .filter((r) => riderFilter.value === "all" || r.id === riderFilter.value)
    .map((r) => {
      const tasks = scheduledOrders.value
        .filter((o) => o.riderId === r.id)
        .sort((a, b) => (a.slot! < b.slot! ? -1 : 1));
      const done = signedOrders.value.filter((o) => o.riderId === r.id);
      const unfinished = pendingOrders.value.filter(
        (o) => o.failReason && (o.riderId ?? o.draft.riderId) === r.id
      );
      return { rider: r, tasks, done, unfinished };
    });
});

/* ---------------- 楼栋 / 骑手 维护 ---------------- */

function addBuilding() {
  store.buildings.push({ id: uid(), name: "新楼栋", cabW: 100, cabD: 200, cabH: 220, doorW: 90, doorH: 200 });
}
function removeBuilding(b: Building) {
  store.buildings = store.buildings.filter((x) => x.id !== b.id);
}
function addRider() {
  store.riders.push({ id: uid(), name: "新骑手", maxLoad: 80, tools: ["手推车"] });
}
function removeRider(r: Rider) {
  store.riders = store.riders.filter((x) => x.id !== r.id);
}
function toggleTool(list: string[], tool: string) {
  const i = list.indexOf(tool);
  if (i >= 0) list.splice(i, 1);
  else list.push(tool);
}
</script>

<template>
  <main class="app">
    <div class="shell">
      <header class="topbar">
        <div>
          <p class="eyebrow">大件家电 · 末端入户调度</p>
          <h1>大件入户排班台</h1>
          <p class="subtitle">
            登记包装尺寸重量与楼层，派单时把电梯尺寸、单人载重、工具和两小时安装窗口放在一起核对；
            不满足的订单留在待排区并标明卡点，确认后锁定楼层·工具·时段。
          </p>
        </div>
        <div class="head-side">
          <label class="date-pick">
            排班日期
            <input type="date" v-model="workDate" />
          </label>
          <div class="stack">
            <span class="tag">Vue3</span><span class="tag">本地持久化</span><span class="tag">四项核对</span>
          </div>
        </div>
      </header>

      <nav class="tabs">
        <button :class="{ active: tab === 'dispatch' }" @click="tab = 'dispatch'">排班工作台</button>
        <button :class="{ active: tab === 'riders' }" @click="tab = 'riders'">骑手当日任务</button>
        <button :class="{ active: tab === 'base' }" @click="tab = 'base'">楼栋 / 骑手档案</button>
      </nav>

      <section class="metrics">
        <article v-for="m in metrics" :key="m.label" class="metric">
          <span>{{ m.label }}</span>
          <strong>{{ m.value }}</strong>
        </article>
      </section>

      <!-- ================= 排班工作台 ================= -->
      <section v-if="tab === 'dispatch'" class="board">
        <!-- 登记 -->
        <form class="panel form-panel" @submit.prevent="addOrder">
          <h2>大件订单登记</h2>
          <div class="form-grid">
            <label>货品名称
              <input v-model="form.itemName" placeholder="如：对开门冰箱" required />
            </label>
            <label>客户
              <input v-model="form.customer" placeholder="客户称呼 / 联系电话" />
            </label>
            <label class="span2">送达地址
              <input v-model="form.address" placeholder="小区楼栋门牌号" required />
            </label>
            <label>外包装长 (cm)
              <input type="number" min="1" v-model.number="form.pkgL" />
            </label>
            <label>外包装宽 (cm)
              <input type="number" min="1" v-model.number="form.pkgW" />
            </label>
            <label>外包装高 (cm)
              <input type="number" min="1" v-model.number="form.pkgH" />
            </label>
            <label>重量 (kg)
              <input type="number" min="1" v-model.number="form.weight" />
            </label>
            <label>楼层
              <input type="number" min="1" v-model.number="form.floor" />
            </label>
            <label>所属楼栋
              <select v-model="form.buildingId">
                <option v-for="b in store.buildings" :key="b.id" :value="b.id">{{ b.name }}</option>
              </select>
            </label>
            <label class="check-line">
              <input type="checkbox" v-model="form.viaStairs" />
              <span>该单需要走楼梯（无电梯 / 电梯放不下时改楼梯）</span>
            </label>
            <div class="span2 tool-pick">
              <p>入户所需工具</p>
              <label v-for="t in TOOL_OPTIONS" :key="t" class="chip">
                <input type="checkbox" :checked="form.requireTools.includes(t)" @change="toggleTool(form.requireTools, t)" />
                <span>{{ t }}</span>
              </label>
            </div>
            <button class="span2" type="submit">登记并进入待排区</button>
          </div>
        </form>

        <!-- 待排区 -->
        <section class="panel zone zone-pending">
          <div class="zone-head">
            <h2>待排区 <em>{{ pendingOrders.length }}</em></h2>
            <p>核对不通过的订单一律留在这里，红色条目即卡点</p>
          </div>
          <div v-if="pendingOrders.length === 0" class="empty">待排区已清空</div>
          <article v-for="order in pendingOrders" :key="order.id" class="card" :class="{ blocked: blockers(order).length > 0, failed: !!order.failReason }">
            <div class="card-head">
              <div>
                <p class="card-title">{{ order.itemName }} <span class="order-no">{{ order.no }}</span></p>
                <p class="card-sub">{{ order.address }} · {{ order.customer || "客户未填" }} · {{ order.floor }} 层
                  <span v-if="order.viaStairs" class="badge warn">走楼梯</span>
                  <span v-else class="badge">电梯</span>
                </p>
              </div>
              <button class="danger tiny" @click="removeOrder(order)">删除</button>
            </div>
            <p class="pkg">包装 {{ order.pkgL }}×{{ order.pkgW }}×{{ order.pkgH }}cm · {{ order.weight }}kg · 需用：{{ order.requireTools.join("、") || "无" }}</p>

            <!-- 派单核对：电梯 / 载重 / 工具 / 窗口 放在一起 -->
            <div class="draft-row">
              <label>楼栋电梯
                <select v-model="order.draft.buildingId">
                  <option v-for="b in store.buildings" :key="b.id" :value="b.id">{{ b.name }}</option>
                </select>
              </label>
              <label>骑手
                <select v-model="order.draft.riderId">
                  <option v-for="r in store.riders" :key="r.id" :value="r.id">{{ r.name }}（{{ r.maxLoad }}kg）</option>
                </select>
              </label>
              <label>两小时窗口
                <select v-model="order.draft.slot">
                  <option v-for="s in SLOTS" :key="s" :value="s">{{ s }}</option>
                </select>
              </label>
            </div>
            <div class="tool-pick small">
              <p>本单配给工具</p>
              <label v-for="t in TOOL_OPTIONS" :key="t" class="chip">
                <input type="checkbox" :checked="order.draft.tools.includes(t)" @change="toggleTool(order.draft.tools, t)" />
                <span>{{ t }}</span>
              </label>
            </div>

            <ul class="checks">
              <li v-for="c in evaluate(order)" :key="c.code" :class="c.ok ? 'pass' : 'fail'">
                <span class="dot" />
                <div>
                  <strong>{{ c.label }}</strong>
                  <p>{{ c.detail }}</p>
                </div>
              </li>
            </ul>

            <div v-if="order.failReason" class="fail-reason">
              上次未完成原因：<strong>{{ order.failReason }}</strong>
              <span v-if="order.failNote">（{{ order.failNote }}）</span>
            </div>

            <div class="actions">
              <button :disabled="!canConfirm(order)" @click="confirmOrder(order)">
                {{ blockers(order).length ? `卡住 ${blockers(order).length} 项，不能派单` : "四项核对通过 · 确认派单并锁定" }}
              </button>
            </div>
          </article>
        </section>

        <!-- 已锁定 -->
        <section class="panel zone zone-locked">
          <div class="zone-head">
            <h2>已锁定排班 <em>{{ scheduledOrders.length }}</em></h2>
            <p>楼层 / 工具 / 时段已锁，改约须先释放原时段</p>
          </div>
          <div v-if="scheduledOrders.length === 0" class="empty">当天暂无已确认排班</div>
          <article v-for="order in scheduledOrders" :key="order.id" class="card locked">
            <div class="card-head">
              <div>
                <p class="card-title">{{ order.itemName }} <span class="order-no">{{ order.no }}</span></p>
                <p class="card-sub">{{ order.address }} · {{ order.customer || "客户未填" }}</p>
              </div>
              <span class="badge lock">🔒 已锁定</span>
            </div>
            <div class="lock-grid">
              <span>骑手：<b>{{ riderOf(order.riderId)?.name }}</b></span>
              <span>楼层：<b>{{ order.lockedFloor }} 层（{{ order.lockedViaStairs ? "走楼梯" : "乘电梯" }}）</b></span>
              <span>窗口：<b>{{ order.slot }}</b></span>
              <span>工具：<b>{{ order.lockedTools?.join("、") || "无" }}</b></span>
            </div>
            <div class="actions">
              <button @click="openSign(order)">签收登记</button>
              <button class="warn-btn" @click="openFail(order)">未完成 / 登记原因</button>
              <button class="secondary" @click="releaseOrder(order)">改约：先释放原时段</button>
            </div>
          </article>

          <div v-if="signedOrders.length" class="signed-block">
            <h3>当日已签收 {{ signedOrders.length }}</h3>
            <article v-for="order in signedOrders" :key="order.id" class="card signed">
              <p class="card-title">{{ order.itemName }} <span class="order-no">{{ order.no }}</span></p>
              <p class="card-sub">
                {{ riderOf(order.riderId)?.name }} · 实际搬运 <b>{{ order.actualFloor }}</b> 层 ·
                进门方式 <b>{{ order.entryWay }}</b> · {{ order.slot }}
              </p>
              <p v-if="order.signNote" class="pkg">备注：{{ order.signNote }}</p>
            </article>
          </div>
        </section>
      </section>

      <!-- ================= 骑手当日任务 ================= -->
      <section v-else-if="tab === 'riders'" class="panel rider-view">
        <div class="toolbar">
          <h2>{{ workDate }} 骑手任务核对</h2>
          <select v-model="riderFilter">
            <option value="all">全部骑手</option>
            <option v-for="r in store.riders" :key="r.id" :value="r.id">{{ r.name }}</option>
          </select>
        </div>
        <div v-for="row in riderTasks" :key="row.rider.id" class="rider-block">
          <header>
            <h3>{{ row.rider.name }}
              <span class="muted">单人载重 {{ row.rider.maxLoad }}kg · 携带 {{ row.rider.tools.join("、") || "无工具" }}</span>
            </h3>
            <span class="badge">待送 {{ row.tasks.length }} · 已签收 {{ row.done.length }} · 未完成 {{ row.unfinished.length }}</span>
          </header>

          <p v-if="!row.tasks.length && !row.done.length && !row.unfinished.length" class="empty">当天无任务</p>

          <ul class="timeline">
            <li v-for="t in row.tasks" :key="t.id">
              <span class="time">{{ t.slot }}</span>
              <div>
                <strong>{{ t.itemName }}</strong>
                <p>{{ t.address }} · {{ t.lockedFloor }} 层（{{ t.lockedViaStairs ? "走楼梯" : "电梯" }}）· {{ t.lockedTools?.join("、") || "无工具" }}</p>
              </div>
              <span class="badge">待送</span>
            </li>
            <li v-for="t in row.done" :key="t.id" class="done">
              <span class="time">{{ t.slot }}</span>
              <div>
                <strong>{{ t.itemName }}</strong>
                <p>{{ t.address }} · 实际 {{ t.actualFloor }} 层 · {{ t.entryWay }}</p>
              </div>
              <span class="badge ok">已签收</span>
            </li>
          </ul>

          <div v-if="row.unfinished.length" class="unfinished">
            <h4>未完成原因（重开页面仍保留）</h4>
            <article v-for="t in row.unfinished" :key="t.id">
              <strong>{{ t.itemName }}</strong>
              <span class="order-no">{{ t.no }}</span>
              <p>原因：{{ t.failReason }}<template v-if="t.failNote"> — {{ t.failNote }}</template></p>
            </article>
          </div>
        </div>
      </section>

      <!-- ================= 档案维护 ================= -->
      <section v-else class="panel base-view">
        <h2>楼栋 / 电梯档案</h2>
        <table class="grid-table">
          <thead>
            <tr><th>楼栋</th><th>轿厢宽×深×高 (cm)</th><th>门洞宽×高 (cm)</th><th></th></tr>
          </thead>
          <tbody>
            <tr v-for="b in store.buildings" :key="b.id">
              <td><input v-model="b.name" /></td>
              <td class="num3">
                <input type="number" v-model.number="b.cabW" /><span>×</span>
                <input type="number" v-model.number="b.cabD" /><span>×</span>
                <input type="number" v-model.number="b.cabH" />
                <small v-if="!b.cabW">填 0 表示无电梯（订单须勾选走楼梯）</small>
              </td>
              <td class="num2">
                <input type="number" v-model.number="b.doorW" /><span>×</span>
                <input type="number" v-model.number="b.doorH" />
              </td>
              <td><button class="danger tiny" @click="removeBuilding(b)">删除</button></td>
            </tr>
          </tbody>
        </table>
        <button class="secondary" @click="addBuilding">+ 新增楼栋</button>

        <h2 style="margin-top:26px">骑手档案</h2>
        <table class="grid-table">
          <thead>
            <tr><th>姓名</th><th>单人载重 (kg)</th><th>携带工具</th><th></th></tr>
          </thead>
          <tbody>
            <tr v-for="r in store.riders" :key="r.id">
              <td><input v-model="r.name" /></td>
              <td><input type="number" v-model.number="r.maxLoad" /></td>
              <td>
                <label v-for="t in TOOL_OPTIONS" :key="t" class="chip">
                  <input type="checkbox" :checked="r.tools.includes(t)" @change="toggleTool(r.tools, t)" />
                  <span>{{ t }}</span>
                </label>
              </td>
              <td><button class="danger tiny" @click="removeRider(r)">删除</button></td>
            </tr>
          </tbody>
        </table>
        <button class="secondary" @click="addRider">+ 新增骑手</button>
      </section>
    </div>

    <!-- 签收弹窗 -->
    <div v-if="signTarget" class="modal-mask" @click.self="signTarget = null">
      <div class="modal">
        <h2>签收登记 · {{ signTarget.itemName }}</h2>
        <p class="muted">{{ signTarget.address }} · 预约 {{ signTarget.slot }}</p>
        <label>实际搬运楼层
          <input type="number" min="1" v-model.number="signForm.actualFloor" />
        </label>
        <label>进门方式
          <select v-model="signForm.entryWay">
            <option v-for="w in ENTRY_WAYS" :key="w" :value="w">{{ w }}</option>
          </select>
        </label>
        <label>签收备注
          <textarea v-model="signForm.signNote" placeholder="现场情况、客户确认等" />
        </label>
        <div class="actions">
          <button @click="submitSign">确认签收</button>
          <button class="secondary" @click="signTarget = null">取消</button>
        </div>
      </div>
    </div>

    <!-- 未完成弹窗 -->
    <div v-if="failTarget" class="modal-mask" @click.self="failTarget = null">
      <div class="modal">
        <h2>登记未完成原因 · {{ failTarget.itemName }}</h2>
        <p class="muted">订单将退回待排区，骑手视图中保留未完成记录。</p>
        <label>原因
          <select v-model="failForm.reason">
            <option v-for="r in FAIL_REASONS" :key="r" :value="r">{{ r }}</option>
          </select>
        </label>
        <label>现场说明
          <textarea v-model="failForm.note" placeholder="如：客户要求下午再来 / 轿厢实测 78cm" />
        </label>
        <div class="actions">
          <button class="warn-btn" @click="submitFail">提交并退回待排</button>
          <button class="secondary" @click="failTarget = null">取消</button>
        </div>
      </div>
    </div>
  </main>
</template>
