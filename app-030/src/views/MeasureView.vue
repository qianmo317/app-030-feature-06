<script setup lang="ts">
import { computed, nextTick, reactive, ref, watch } from 'vue'
import { useRoute } from 'vue-router'
import { ensureMerged, flushProject, getProject, getRule, persistProject, store } from '../logic/store'
import { analyzeDraft, findDuplicateIds, makePersonId, type PersonDraft } from '../logic/analyze'
import { estimateInitialSize, type EstimateResult } from '../logic/estimate'
import { formatCm, parseLengthCm, parseWeightKg } from '../logic/precision'
import { downloadText, toCsvText } from '../logic/csv'
import { computeRuleSize, runMerge } from '../logic/merge'
import { specialFlagLabel } from '../logic/sizeRules'
import { detailRows } from '../logic/exporter'
import type { Gender, ManualOverride, Person, Project } from '../logic/types'

const route = useRoute()
const project = computed(() => getProject(route.params.id as string))
const rule = computed(() => getRule(project.value?.ruleVersion ?? store.rules[0].version))

const formRef = ref<HTMLFormElement | null>(null)
const heightRef = ref<HTMLInputElement | null>(null)

const form = reactive({
  name: '',
  gender: 'male' as Gender,
  orgUnit: '',
  batch: '',
  heightCm: '',
  weightKg: '',
  chestCm: '',
  waistCm: '',
  specialFlag: '',
  note: ''
})

const sticky = reactive({ orgUnit: '', gender: 'male' as Gender, batch: '' })
const notice = ref('')
const warnText = ref('')
const savedCount = ref(0)
const saving = ref(false)
const editingId = ref<string | null>(null)
const recentKeyword = ref('')

watch(
  project,
  (value) => {
    if (!value) return
    editingId.value = null
    sticky.batch = sticky.batch || value.batches[0] || ''
    form.batch = sticky.batch
  },
  { immediate: true }
)

const editingPerson = computed(() => project.value?.persons.find((person) => person.id === editingId.value) ?? null)
const editingIndex = computed(() => project.value?.persons.findIndex((person) => person.id === editingId.value) ?? -1)

const duplicateIds = computed(() => {
  if (!project.value) return []
  const heightCm = parseLengthCm(form.heightCm)
  const weightKg = parseWeightKg(form.weightKg)
  return findDuplicateIds(
    project.value.persons,
    {
      name: form.name,
      gender: form.gender,
      orgUnit: form.orgUnit,
      batch: form.batch,
      heightCm,
      weightKg,
      chestCm: null,
      waistCm: null,
      specialFlag: null,
      note: '',
      sourceRow: null,
      source: 'manual'
    },
    editingId.value ?? undefined
  )
})

const duplicateNames = computed(() =>
  duplicateIds.value
    .map((id) => project.value?.persons.find((person) => person.id === id)?.name ?? '')
    .filter((name) => name !== '')
)

const estimate = computed<EstimateResult | null>(() => {
  const heightCm = parseLengthCm(form.heightCm)
  const weightKg = parseWeightKg(form.weightKg)
  return estimateInitialSize(rule.value, form.gender, heightCm, weightKg)
})

const recent = computed<Person[]>(() => {
  const persons = project.value?.persons ?? []
  const keyword = recentKeyword.value.trim()
  if (keyword === '') return [...persons].slice(-6).reverse()
  // 搜索时覆盖全部记录（含导入行），让任意一条都能被找出来修改
  return persons
    .filter(
      (person) =>
        person.name.includes(keyword) ||
        person.orgUnit.includes(keyword) ||
        String(person.sourceRow ?? '') === keyword
    )
    .slice(-20)
    .reverse()
})

function sourceText(person: Person): string {
  return person.source === 'import' ? `导入·第 ${person.sourceRow ?? '—'} 行` : '手录'
}

/** 最近保存表里的号型展示：已归并的用归并结果，未归并的临时按规则算一个（仅展示，不写入） */
function displaySize(person: Person): string {
  if (person.status !== 'active') return '—'
  if (person.result?.sizeCode) return person.result.sizeCode
  return computeRuleSize(rule.value, person)?.sizeCode ?? '—'
}

function resetForm(): void {
  form.name = ''
  form.heightCm = ''
  form.weightKg = ''
  form.chestCm = ''
  form.waistCm = ''
  form.note = ''
  form.specialFlag = ''
  form.orgUnit = sticky.orgUnit
  form.gender = sticky.gender
  form.batch = sticky.batch
}

function onEnter(event: KeyboardEvent): void {
  const element = event.target as HTMLElement
  const isTextarea = element.tagName === 'TEXTAREA'
  if (isTextarea && event.shiftKey) return
  event.preventDefault()
  const container = formRef.value
  if (!container) return
  const fields = Array.from(container.querySelectorAll<HTMLElement>('[data-field]'))
  const index = fields.indexOf(element)
  if (index >= 0 && index < fields.length - 1) {
    fields[index + 1].focus()
    return
  }
  void save()
}

function startEdit(person: Person): void {
  editingId.value = person.id
  form.name = person.name
  form.gender = person.gender
  form.orgUnit = person.orgUnit
  form.batch = person.batch
  form.heightCm = formatCm(person.heightCm)
  form.weightKg = person.weightKg === null ? '' : String(person.weightKg)
  form.chestCm = formatCm(person.chestCm)
  form.waistCm = formatCm(person.waistCm)
  form.specialFlag = person.specialFlag ?? ''
  form.note = person.note
  notice.value = ''
  warnText.value = ''
  void nextTick(() => {
    formRef.value?.scrollIntoView({ behavior: 'smooth', block: 'start' })
    heightRef.value?.focus()
  })
}

function cancelEdit(): void {
  editingId.value = null
  resetForm()
  notice.value = ''
  warnText.value = ''
}

async function save(): Promise<void> {
  const current = project.value
  if (!current || saving.value) return
  notice.value = ''
  warnText.value = ''
  if (form.name.trim() === '') {
    warnText.value = '请先填写姓名，姓名用于量体明细回贴核对'
    return
  }
  saving.value = true
  const editing = editingPerson.value
  if (editing) saveEdit(current, editing)
  else saveNew(current)
  saving.value = false
  await nextTick()
  heightRef.value?.focus()
}

function saveNew(current: Project): void {
  const draft: PersonDraft = {
    name: form.name,
    gender: form.gender,
    orgUnit: form.orgUnit,
    batch: form.batch || current.batches[0] || '未分批',
    heightCm: parseLengthCm(form.heightCm),
    weightKg: parseWeightKg(form.weightKg),
    chestCm: parseLengthCm(form.chestCm),
    waistCm: parseLengthCm(form.waistCm),
    specialFlag: form.specialFlag || null,
    note: form.note,
    sourceRow: current.persons.length + 1,
    source: 'manual'
  }
  const outcome = analyzeDraft(draft, rule.value)
  const duplicated = findDuplicateIds(current.persons, draft)
  const person: Person = {
    id: makePersonId(),
    name: draft.name.trim(),
    gender: draft.gender ?? 'male',
    orgUnit: draft.orgUnit.trim(),
    batch: draft.batch,
    heightCm: draft.heightCm ?? 0,
    weightKg: draft.weightKg,
    chestCm: draft.chestCm ?? 0,
    waistCm: draft.waistCm ?? 0,
    specialFlag: draft.specialFlag,
    note: draft.note,
    status: outcome.status,
    statusReason: outcome.statusReason,
    anomaly: outcome.anomaly,
    needsConfirm: outcome.needsConfirm || duplicated.length > 0,
    possibleDuplicateOf: duplicated.length > 0 ? `既有行「${duplicated[0]}」` : null,
    sourceRow: draft.sourceRow,
    source: 'manual',
    result: null,
    createdAt: Date.now()
  }
  current.persons.push(person)
  sticky.orgUnit = person.orgUnit
  sticky.gender = person.gender
  sticky.batch = person.batch
  savedCount.value += 1
  resetForm()
  persistProject(current, true)
  notice.value = `第 ${current.persons.length} 条（${person.name}）已保存到本机 IndexedDB，断网也不丢`
  if (outcome.status === 'invalid') {
    warnText.value = `已拦截：${outcome.statusReason}；该行记为无效行，不计入有效人数，可在归并页复核`
  } else if (outcome.messages.length > 0) {
    warnText.value = outcome.messages.join('；')
  } else if (duplicated.length > 0) {
    warnText.value = `可能与「${duplicated[0]}」重复（同名 + 同班级 + 同身高体重），已保留并标记，未自动删除`
  }
}

/** 修改写回：在原行上就地更新，不新增；号型按当前规则版本重新判定，人工覆写保留并对比提示 */
function saveEdit(current: Project, target: Person): void {
  const before = {
    gender: target.gender,
    heightCm: target.heightCm,
    chestCm: target.chestCm,
    waistCm: target.waistCm,
    specialFlag: target.specialFlag,
    override: target.result?.manualOverride ? { ...target.result.manualOverride } : null
  }
  const draft: PersonDraft = {
    name: form.name,
    gender: form.gender,
    orgUnit: form.orgUnit,
    batch: form.batch, // 修改写回：批次以表单为准，允许改回「未分批」
    heightCm: parseLengthCm(form.heightCm),
    weightKg: parseWeightKg(form.weightKg),
    chestCm: parseLengthCm(form.chestCm),
    waistCm: parseLengthCm(form.waistCm),
    specialFlag: form.specialFlag || null,
    note: form.note,
    sourceRow: target.sourceRow,
    source: target.source
  }
  const outcome = analyzeDraft(draft, rule.value)
  const duplicated = findDuplicateIds(current.persons, draft, target.id)

  target.name = draft.name.trim()
  target.gender = draft.gender ?? 'male'
  target.orgUnit = draft.orgUnit.trim()
  target.batch = draft.batch
  target.heightCm = draft.heightCm ?? 0
  target.weightKg = draft.weightKg
  target.chestCm = draft.chestCm ?? 0
  target.waistCm = draft.waistCm ?? 0
  target.specialFlag = draft.specialFlag
  target.note = draft.note
  // 「重复行已排除」是归并页的人工决定，修改数据不擅自恢复；其余状态按校验结果重算
  const keptDuplicate = target.status === 'duplicate'
  if (!keptDuplicate) {
    target.status = outcome.status
    target.statusReason = outcome.statusReason
  }
  target.anomaly = outcome.anomaly
  target.needsConfirm = outcome.needsConfirm || duplicated.length > 0
  target.possibleDuplicateOf = duplicated.length > 0 ? `既有行「${duplicated[0]}」` : null

  // 重新归并整个项目（幂等）：保留人工覆写、刷新规则号型，归并页与汇总页的人数套数随之更新
  ensureMerged(current)
  persistProject(current, true)

  const rowNo = current.persons.indexOf(target) + 1
  editingId.value = null
  resetForm()
  notice.value = `第 ${rowNo} 条（${target.name}）已写回修改，未新增行；号型已按规则 ${current.ruleVersion} 重新判定`

  const notes: string[] = []
  if (keptDuplicate) {
    notes.push('本条仍保持「重复行已排除」状态，不计入有效人数；如需恢复计入，请到归并页操作')
  } else if (outcome.status === 'invalid') {
    notes.push(`已拦截：${outcome.statusReason}；该行记为无效行，不计入有效人数，可在归并页复核`)
  } else {
    notes.push(...outcome.messages)
  }
  if (duplicated.length > 0) {
    const other = current.persons.find((person) => person.id === duplicated[0])
    notes.push(`可能与「${other?.name ?? duplicated[0]}」重复（同名 + 同班级 + 同身高体重），已保留并标记，未自动删除`)
  }
  notes.push(...overrideNotes(target, before.override))
  notes.push(...specialFlagNotes(target, before))
  warnText.value = notes.join('；')
}

/** 人工覆写保留提示：覆写不动，但要说清它与重新判定结果的差异 */
function overrideNotes(person: Person, beforeOverride: ManualOverride | null): string[] {
  const current = person.result?.manualOverride ?? null
  if (beforeOverride && !current) {
    return [
      `原人工覆写号型 ${beforeOverride.sizeCode}（${beforeOverride.by}：${beforeOverride.reason}）因本条已非有效行，随归并结果一并清除；恢复有效后如需覆写请重新操作`
    ]
  }
  if (!current) return []
  const fresh = person.result?.ruleSizeCode ?? ''
  if (fresh && fresh !== current.sizeCode) {
    return [
      `本条号型为人工覆写 ${current.sizeCode}（${current.by}：${current.reason}），已保留；按修改后的数据重新判定为 ${fresh}，两者不一致，如以实测为准请到归并页撤销覆写`
    ]
  }
  if (fresh && fresh === current.sizeCode) {
    return [`人工覆写 ${current.sizeCode} 与按修改后数据重新判定的结果一致，覆写继续保留`]
  }
  return [`本条号型为人工覆写 ${current.sizeCode}，已保留；但修改后的数据按当前规则无法归并，请在归并页复核`]
}

/** 特殊体型标记复核提示：改了性别或数值后，说明先前标记是否仍然合适 */
function specialFlagNotes(
  person: Person,
  before: { gender: Gender; heightCm: number; chestCm: number; waistCm: number; specialFlag: string | null }
): string[] {
  const notes: string[] = []
  const flag = person.specialFlag
  if (before.specialFlag && !flag) {
    notes.push(`已取消特殊体型标记「${specialFlagLabel(rule.value, before.specialFlag)}」，本条按常规档参与归并`)
  }
  if (!flag) return notes
  const genderChanged = before.gender !== person.gender
  const valuesChanged =
    before.heightCm !== person.heightCm || before.chestCm !== person.chestCm || before.waistCm !== person.waistCm
  if (!genderChanged && !valuesChanged) return notes
  const label = specialFlagLabel(rule.value, flag)
  if (genderChanged) {
    notes.push(
      `性别已由「${genderText(before.gender)}」改为「${genderText(person.gender)}」，特殊体型标记「${label}」仍保留，请确认对该性别仍然合适`
    )
  }
  const flagDef = rule.value.specialFlags.find((item) => item.code === flag)
  const semantic = `${flag} ${flagDef?.label ?? ''}`.toLowerCase()
  const heightInRange =
    person.heightCm >= rule.value.heightRangeCm.minCm && person.heightCm <= rule.value.heightRangeCm.maxCm
  const chestInRange =
    person.chestCm >= rule.value.chestRangeCm.minCm && person.chestCm <= rule.value.chestRangeCm.maxCm
  if (/tall|超高/.test(semantic) && heightInRange) {
    notes.push(
      `身高 ${formatCm(person.heightCm)}cm 已落在规则可判定范围（${rule.value.heightRangeCm.minCm}~${rule.value.heightRangeCm.maxCm}cm）内，「${label}」标记可能不再合适，请确认或在「特殊体型标记」改回「无」`
    )
  } else if (/plus|加肥/.test(semantic) && chestInRange) {
    notes.push(
      `胸围 ${formatCm(person.chestCm)}cm 已落在规则可判定范围（${rule.value.chestRangeCm.minCm}~${rule.value.chestRangeCm.maxCm}cm）内，「${label}」标记可能不再合适，请确认或在「特殊体型标记」改回「无」`
    )
  } else if (valuesChanged) {
    notes.push(
      `特殊体型标记「${label}」仍保留；量体数值已修改，请确认该标记对修改后的数据仍然合适，不需要可在「特殊体型标记」改回「无」`
    )
  }
  return notes
}

async function removePerson(person: Person): Promise<void> {
  const current = project.value
  if (!current) return
  if (editingId.value === person.id) cancelEdit()
  current.persons = current.persons.filter((item) => item.id !== person.id)
  persistProject(current, true)
  notice.value = `已删除「${person.name}」`
}

async function exportFallbackCsv(): Promise<void> {
  const current = project.value
  if (!current) return
  runMerge(current, rule.value)
  await flushProject(current)
  const rows = detailRows({ project: current, rule: rule.value })
  downloadText(
    toCsvText(rows),
    `${current.name.replace(/[\\/:*?"<>|\s]/g, '_')}-量体明细-离线兜底.csv`
  )
  notice.value = '已导出本地 CSV（兜底），可直接交给办公室汇总'
}

function focusHeight(): void {
  heightRef.value?.focus()
}

function genderText(gender: Gender): string {
  return gender === 'male' ? '男' : '女'
}
</script>

<template>
  <section v-if="!project" class="empty">项目不存在，请回到项目列表重新选择。</section>
  <section v-else>
    <div class="page-head">
      <div>
        <h1>{{ project.name }} · 量体录入</h1>
        <div class="sub">
          规则版本 {{ project.ruleVersion }} ｜ 已录入 {{ project.persons.length }} 条 ｜
          本机离线保存，回办公室可一次性导出
        </div>
      </div>
      <div class="spacer"></div>
      <div class="toolbar">
        <RouterLink class="btn btn-sm" :to="`/import/${project.id}`">批量导入</RouterLink>
        <button class="btn btn-sm" type="button" @click="exportFallbackCsv">导出 CSV（兜底）</button>
      </div>
    </div>

    <div class="card" :class="editingPerson ? 'card-accent-warn' : ''">
      <div class="card-head">
        <h2>{{ editingPerson ? `正在修改第 ${editingIndex + 1} 条` : '连续录入（回车即存下一条）' }}</h2>
        <div class="spacer"></div>
        <span class="hint">{{ editingPerson ? '保存后写回本条，不新增行' : '光标自动回到身高；同班级数据只需改动身高体重胸腰围' }}</span>
      </div>
      <div class="card-body">
        <p v-if="editingPerson" class="notice notice-warn" style="margin-bottom: 12px">
          正在修改「{{ editingPerson.name }}」（{{ sourceText(editingPerson) }}）：姓名、性别、班级、数值与备注已回填；
          保存后写回本条，号型按规则 {{ project.ruleVersion }} 重新判定，人工覆写会保留并对比提示。
          <button class="btn btn-sm" type="button" style="margin-left: 8px" @click="cancelEdit">取消修改</button>
        </p>
        <form ref="formRef" @submit.prevent="save" @keydown.enter="onEnter">
          <div class="measure-grid">
            <label class="field span-2">
              <span class="field-label">姓名 <b class="req">*</b></span>
              <input v-model="form.name" class="input" data-field type="text" placeholder="张三" autocomplete="off" />
            </label>

            <div class="field">
              <span class="field-label">性别</span>
              <div class="gender-switch">
                <button
                  class="btn"
                  :class="{ active: form.gender === 'male' }"
                  type="button"
                  @click="form.gender = 'male'"
                >
                  男
                </button>
                <button
                  class="btn"
                  :class="{ active: form.gender === 'female' }"
                  type="button"
                  @click="form.gender = 'female'"
                >
                  女
                </button>
              </div>
            </div>

            <label class="field">
              <span class="field-label">班级 / 车间</span>
              <input v-model="form.orgUnit" class="input" data-field type="text" placeholder="高一(3)班" />
            </label>

            <label class="field">
              <span class="field-label">身高 (cm) <b class="req">*</b></span>
              <input
                ref="heightRef"
                v-model="form.heightCm"
                class="input measure-input"
                data-field
                type="text"
                inputmode="decimal"
                placeholder="170"
                autocomplete="off"
              />
            </label>

            <label class="field">
              <span class="field-label">体重 (kg)</span>
              <input
                v-model="form.weightKg"
                class="input measure-input"
                data-field
                type="text"
                inputmode="decimal"
                placeholder="65"
                autocomplete="off"
              />
            </label>

            <label class="field">
              <span class="field-label">胸围 (cm) <b class="req">*</b></span>
              <input
                v-model="form.chestCm"
                class="input measure-input"
                data-field
                type="text"
                inputmode="decimal"
                placeholder="88"
                autocomplete="off"
              />
            </label>

            <label class="field">
              <span class="field-label">腰围 (cm) <b class="req">*</b></span>
              <input
                v-model="form.waistCm"
                class="input measure-input"
                data-field
                type="text"
                inputmode="decimal"
                placeholder="72"
                autocomplete="off"
              />
            </label>

            <label class="field">
              <span class="field-label">批次</span>
              <select v-model="form.batch" class="select" data-field>
                <option value="">未分批</option>
                <option v-for="batch in project.batches" :key="batch" :value="batch">{{ batch }}</option>
                <option v-if="form.batch && !project.batches.includes(form.batch)" :value="form.batch">
                  {{ form.batch }}（导入批次）
                </option>
              </select>
            </label>

            <label class="field">
              <span class="field-label">特殊体型标记</span>
              <select v-model="form.specialFlag" class="select" data-field>
                <option value="">无（常规档）</option>
                <option v-for="flag in rule.specialFlags" :key="flag.code" :value="flag.code">
                  {{ flag.label }}
                </option>
                <option
                  v-if="form.specialFlag && !rule.specialFlags.some((flag) => flag.code === form.specialFlag)"
                  :value="form.specialFlag"
                >
                  {{ form.specialFlag }}（规则外标记）
                </option>
              </select>
            </label>

            <label class="field span-2">
              <span class="field-label">备注（Shift + 回车换行）</span>
              <textarea v-model="form.note" class="textarea" data-field placeholder="如：左肩略低、需留量 2cm" />
            </label>
          </div>

          <div class="toolbar" style="margin-top: 12px">
            <button class="btn btn-primary btn-accent" type="submit">
              {{ editingPerson ? '保存修改（写回本条）' : '保存并录入下一条（回车）' }}
            </button>
            <button v-if="editingPerson" class="btn" type="button" @click="cancelEdit">取消修改</button>
            <button class="btn" type="button" @click="focusHeight">回到身高</button>
            <span class="hint">身高 / 胸围 / 腰围按 0.5cm 精度存储与判定</span>
          </div>
        </form>

        <p v-if="notice" class="notice notice-ok" style="margin-top: 12px">{{ notice }}</p>
        <p v-if="warnText" class="notice notice-warn" style="margin-top: 12px">{{ warnText }}</p>
        <p v-if="duplicateNames.length" class="notice notice-warn" style="margin-top: 12px">
          检测到可能重复：同名 + 同班级 + 同身高体重 的既有记录「{{ duplicateNames.join('、') }}」。重复行只提示、
          不自动删除，保存后可在归并页标记排除。
        </p>

        <div v-if="estimate" class="estimate-box" style="margin-top: 12px">
          <div>
            建议初始号型 <span class="estimate-code">{{ estimate.sizeCode }}</span>
            （型别 {{ estimate.fit }}，估算胸围 {{ estimate.estimatedChestCm }}cm / 腰围 {{ estimate.estimatedWaistCm }}cm）
          </div>
          <div class="hint">{{ estimate.basis }}</div>
        </div>
      </div>
    </div>

    <div class="grid-2">
      <div class="card">
        <div class="card-head">
          <h3>{{ recentKeyword.trim() ? '搜索结果（全部记录）' : '最近保存（本机离线数据）' }}</h3>
          <div class="spacer"></div>
          <input
            v-model="recentKeyword"
            class="input btn-sm"
            style="width: 150px"
            type="search"
            placeholder="搜姓名/班级/行号"
          />
          <span class="badge badge-ok">本次会话新增 {{ savedCount }} 条</span>
        </div>
        <div v-if="recent.length === 0" class="empty">
          {{ recentKeyword.trim() ? '没有匹配的记录' : '还没有录入数据' }}
        </div>
        <div v-else class="table-wrap">
          <table class="data-table">
            <thead>
              <tr>
                <th>姓名</th>
                <th>性别</th>
                <th class="num">身高</th>
                <th class="num">胸围</th>
                <th class="num">腰围</th>
                <th>号型</th>
                <th>来源</th>
                <th>状态</th>
                <th></th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="person in recent"
                :key="person.id"
                :class="[
                  person.status === 'active' ? '' : 'row-invalid',
                  person.id === editingId ? 'row-highlight' : ''
                ]"
              >
                <td>{{ person.name }}</td>
                <td>{{ genderText(person.gender) }}</td>
                <td class="num">{{ formatCm(person.heightCm) }}</td>
                <td class="num">{{ formatCm(person.chestCm) }}</td>
                <td class="num">{{ formatCm(person.waistCm) }}</td>
                <td>
                  <b>{{ displaySize(person) }}</b>
                  <span v-if="person.result?.manualOverride" class="badge badge-warn" style="margin-left: 4px">覆写</span>
                  <span v-if="person.specialFlag" class="badge badge-warn" style="margin-left: 4px">
                    {{ specialFlagLabel(rule, person.specialFlag) }}
                  </span>
                </td>
                <td>
                  <span class="badge" :class="person.source === 'import' ? 'badge-info' : ''">
                    {{ sourceText(person) }}
                  </span>
                </td>
                <td>
                  <span class="badge" :class="person.status === 'active' ? 'badge-ok' : 'badge-danger'">
                    {{ person.status === 'active' ? '有效' : person.status === 'invalid' ? '无效' : '重复' }}
                  </span>
                </td>
                <td>
                  <div class="toolbar">
                    <button class="btn btn-sm" type="button" @click="startEdit(person)">改</button>
                    <button class="btn btn-sm btn-danger" type="button" @click="removePerson(person)">删除</button>
                  </div>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
        <div class="card-body tight">
          <span class="hint">
            点「改」把该行填回上方表单，保存后写回原行不新增；导入行可按「导入·第 N 行」回查原文件。
          </span>
        </div>
      </div>

      <div class="card">
        <div class="card-head"><h3>现场录入要点</h3></div>
        <div class="card-body tight">
          <p>· 回车即保存并自动聚焦身高，适合一人接一人连续录入。</p>
          <p>· 班级 / 性别 / 批次会沿用上一条，同班录入只需改数值。</p>
          <p>· 录错了不用删：「最近保存」每行点「改」回填表单，保存写回原行，号型自动重新判定。</p>
          <p>· 身高 80cm、胸围小于身高一半、胸腰差为负等异常会即时提示，并标记为待确认或无效行。</p>
          <p>· 特殊体型（加肥加大 / 特体定制 / 超高定制）请选择标记，将单列进定制清单，不混入常规档。</p>
          <p>· 无网也能录入：数据写入本机 IndexedDB，随时可用「导出 CSV（兜底）」带出。</p>
        </div>
      </div>
    </div>
  </section>
</template>