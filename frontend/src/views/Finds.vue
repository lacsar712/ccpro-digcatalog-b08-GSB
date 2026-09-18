<template>
  <div>
    <div class="toolbar">
      <div>
        <h2 class="page-title">出土文物</h2>
        <p class="page-sub">登记器物类型、材质、完整度与存放位置</p>
      </div>
      <div class="toolbar-actions">
        <button class="btn secondary" @click="openExport">导出 CSV</button>
        <button class="btn" @click="openCreate">新增文物</button>
      </div>
    </div>

    <div class="card">
      <div class="filters">
        <label>
          探方筛选
          <select v-model="filterUnitId" @change="load">
            <option value="">全部探方</option>
            <option v-for="u in units" :key="u.id" :value="String(u.id)">
              {{ u.site?.name || '' }} / {{ u.code }}
            </option>
          </select>
        </label>
        <label>
          器物类型
          <select v-model="filterType" @change="load">
            <option value="">全部类型</option>
            <option v-for="t in artifactTypes" :key="t" :value="t">{{ t }}</option>
          </select>
        </label>
        <label>
          出土日期从
          <input v-model="filterDateFrom" type="date" @change="load" />
        </label>
        <label>
          至
          <input v-model="filterDateTo" type="date" @change="load" />
        </label>
      </div>

      <table class="table">
        <thead>
          <tr>
            <th>登记号</th>
            <th>探方</th>
            <th>器物类型</th>
            <th>材质</th>
            <th>完整度</th>
            <th>出土日期</th>
            <th>存放位置</th>
            <th>操作</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="item in list" :key="item.id">
            <td>{{ item.registerNo }}</td>
            <td>{{ item.unit?.code || '-' }}</td>
            <td><span class="tag">{{ item.artifactType }}</span></td>
            <td>{{ item.materialName || item.material?.name || '-' }}</td>
            <td>{{ item.completeness || '-' }}</td>
            <td>{{ formatDate(item.findDate) }}</td>
            <td>{{ item.storageLoc || '-' }}</td>
            <td>
              <button class="btn secondary small" @click="openEdit(item)">编辑</button>
              <button class="btn danger small" @click="remove(item)">删除</button>
            </td>
          </tr>
        </tbody>
      </table>
      <p v-if="!list.length" class="page-sub">暂无数据</p>
      <p v-if="error" class="error">{{ error }}</p>
    </div>

    <div v-if="showModal" class="modal-mask" @click.self="showModal = false">
      <div class="modal">
        <h3>{{ form.id ? '编辑文物' : '新增文物' }}</h3>
        <div class="form-grid">
          <label>
            所属探方
            <select v-model.number="form.unitId">
              <option :value="0" disabled>请选择</option>
              <option v-for="u in units" :key="u.id" :value="u.id">
                {{ u.site?.name || '' }} / {{ u.code }}
              </option>
            </select>
          </label>
          <label>
            登记号
            <input v-model="form.registerNo" />
          </label>
          <label>
            器物类型
            <select v-model="form.artifactType">
              <option v-for="t in artifactTypes" :key="t" :value="t">{{ t }}</option>
            </select>
          </label>
          <label>
            材质
            <select v-model="form.materialId">
              <option :value="null">未指定</option>
              <option v-for="m in materials" :key="m.id" :value="m.id">{{ m.name }}</option>
            </select>
          </label>
          <label>
            完整度
            <select v-model="form.completeness">
              <option>完整</option>
              <option>残缺</option>
              <option>碎片</option>
            </select>
          </label>
          <label>
            出土日期
            <input v-model="form.findDate" type="date" />
          </label>
          <label class="full">
            存放位置
            <input v-model="form.storageLoc" />
          </label>
          <label class="full">
            描述
            <textarea v-model="form.description" />
          </label>
        </div>
        <p v-if="formError" class="error">{{ formError }}</p>
        <div class="modal-actions">
          <button class="btn secondary" @click="showModal = false">取消</button>
          <button class="btn" @click="save">保存</button>
        </div>
      </div>
    </div>

    <div v-if="showExport" class="modal-mask" @click.self="showExport = false">
      <div class="modal">
        <h3>导出文物 CSV</h3>
        <p class="page-sub">
          服务器将按当前筛选条件生成 CSV 文件（UTF-8 含 BOM，Excel 可直接打开），
          导出内容与列表数据一致。
        </p>
        <div class="export-summary">
          <div v-for="item in filterSummary" :key="item.label" class="summary-row">
            <span class="summary-label">{{ item.label }}</span>
            <span>{{ item.value }}</span>
          </div>
          <div class="summary-row">
            <span class="summary-label">预计导出</span>
            <span><strong>{{ exportCount }}</strong> 条</span>
          </div>
          <div class="summary-row">
            <span class="summary-label">导出列</span>
            <span class="cols">registerNo / artifactType / unitCode / materialName / completeness / findDate / storageLoc</span>
          </div>
        </div>
        <p v-if="exportCount === 0" class="export-warn">
          当前筛选条件下没有任何文物数据。确认后下载的文件将<strong>只包含表头、不含任何数据行</strong>；
          如非有意获取空模板，请取消并调整筛选条件。
        </p>
        <p v-if="exportError" class="error">{{ exportError }}</p>
        <div class="modal-actions">
          <button class="btn secondary" :disabled="exporting" @click="showExport = false">取消</button>
          <button class="btn" :disabled="exporting" @click="confirmExport">
            {{ exporting ? '导出中…' : (exportCount === 0 ? '仍导出仅表头 CSV' : '确认导出') }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { computed, onMounted, reactive, ref } from 'vue'
import api from '../api/http'

const artifactTypes = ['陶片', '青铜器', '骨器', '玉器', '石器', '铁器', '其他']
const list = ref([])
const units = ref([])
const materials = ref([])
const filterUnitId = ref('')
const filterType = ref('')
const filterDateFrom = ref('')
const filterDateTo = ref('')
const error = ref('')
const formError = ref('')
const showModal = ref(false)

const showExport = ref(false)
const exporting = ref(false)
const exportError = ref('')

const form = reactive({
  id: null,
  unitId: 0,
  materialId: null,
  registerNo: '',
  artifactType: '陶片',
  completeness: '完整',
  findDate: '',
  description: '',
  storageLoc: ''
})

function formatDate(v) {
  if (!v) return '-'
  return String(v).slice(0, 10)
}

// 列表与导出共用同一组筛选参数，保证“所见即所导”
function currentParams() {
  const params = {}
  if (filterUnitId.value) params.unitId = filterUnitId.value
  if (filterType.value) params.artifactType = filterType.value
  if (filterDateFrom.value) params.findDateFrom = filterDateFrom.value
  if (filterDateTo.value) params.findDateTo = filterDateTo.value
  return params
}

const exportCount = computed(() => list.value.length)

const filterSummary = computed(() => {
  const u = units.value.find((x) => String(x.id) === filterUnitId.value)
  let dateText = '不限'
  if (filterDateFrom.value && filterDateTo.value) {
    dateText = `${filterDateFrom.value} 至 ${filterDateTo.value}`
  } else if (filterDateFrom.value) {
    dateText = `${filterDateFrom.value} 起`
  } else if (filterDateTo.value) {
    dateText = `截至 ${filterDateTo.value}`
  }
  return [
    { label: '探方', value: u ? `${u.site?.name || ''} / ${u.code}` : '全部探方' },
    { label: '器物类型', value: filterType.value || '全部类型' },
    { label: '出土日期', value: dateText }
  ]
})

async function loadMeta() {
  const [u, m] = await Promise.all([api.get('/units'), api.get('/materials')])
  units.value = u.data
  materials.value = m.data
}

async function load() {
  error.value = ''
  try {
    const { data } = await api.get('/finds', { params: currentParams() })
    list.value = data
  } catch (e) {
    error.value = e.response?.data?.error || '加载失败'
  }
}

function openExport() {
  exportError.value = ''
  showExport.value = true
}

async function blobErrorMessage(e) {
  const data = e.response?.data
  if (data instanceof Blob) {
    try {
      const json = JSON.parse(await data.text())
      return json.error
    } catch {
      return null
    }
  }
  return e.response?.data?.error
}

async function confirmExport() {
  exporting.value = true
  exportError.value = ''
  try {
    // 文件由后端生成，前端只负责触发下载，不自行拼装 CSV
    const res = await api.get('/finds/export.csv', {
      params: currentParams(),
      responseType: 'blob'
    })
    const dispo = res.headers['content-disposition'] || ''
    const match = dispo.match(/filename="?([^";]+)"?/)
    const filename = match ? match[1] : 'finds-export.csv'
    const url = URL.createObjectURL(res.data)
    const a = document.createElement('a')
    a.href = url
    a.download = filename
    document.body.appendChild(a)
    a.click()
    a.remove()
    URL.revokeObjectURL(url)
    showExport.value = false
  } catch (e) {
    exportError.value = (await blobErrorMessage(e)) || '导出失败，请稍后重试'
  } finally {
    exporting.value = false
  }
}

function openCreate() {
  Object.assign(form, {
    id: null,
    unitId: units.value[0]?.id || 0,
    materialId: materials.value[0]?.id ?? null,
    registerNo: '',
    artifactType: '陶片',
    completeness: '完整',
    findDate: '',
    description: '',
    storageLoc: ''
  })
  formError.value = ''
  showModal.value = true
}

function openEdit(item) {
  Object.assign(form, {
    id: item.id,
    unitId: item.unitId,
    materialId: item.materialId,
    registerNo: item.registerNo,
    artifactType: item.artifactType,
    completeness: item.completeness || '完整',
    findDate: formatDate(item.findDate) === '-' ? '' : formatDate(item.findDate),
    description: item.description || '',
    storageLoc: item.storageLoc || ''
  })
  formError.value = ''
  showModal.value = true
}

async function save() {
  formError.value = ''
  try {
    const payload = {
      unitId: form.unitId,
      materialId: form.materialId || null,
      registerNo: form.registerNo,
      artifactType: form.artifactType,
      completeness: form.completeness,
      findDate: form.findDate || null,
      description: form.description,
      storageLoc: form.storageLoc
    }
    if (form.id) {
      await api.put(`/finds/${form.id}`, payload)
    } else {
      await api.post('/finds', payload)
    }
    showModal.value = false
    await load()
  } catch (e) {
    formError.value = e.response?.data?.error || '保存失败'
  }
}

async function remove(item) {
  if (!confirm(`确认删除文物「${item.registerNo}」？`)) return
  try {
    await api.delete(`/finds/${item.id}`)
    await load()
  } catch (e) {
    alert(e.response?.data?.error || '删除失败')
  }
}

onMounted(async () => {
  await loadMeta()
  await load()
})
</script>

<style scoped>
.filters {
  display: flex;
  gap: 1rem;
  margin-bottom: 1rem;
  flex-wrap: wrap;
}

.filters label {
  min-width: 180px;
}

.toolbar-actions {
  display: flex;
  gap: 0.6rem;
  align-items: center;
}

.export-summary {
  margin: 1rem 0;
  border: 1px solid var(--border);
  border-radius: 10px;
  padding: 0.75rem 1rem;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  background: #fbf7ef;
}

.summary-row {
  display: flex;
  gap: 0.75rem;
  font-size: 0.92rem;
}

.summary-label {
  color: var(--muted);
  min-width: 4.5rem;
  flex-shrink: 0;
}

.cols {
  font-family: ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
  font-size: 0.82rem;
  word-break: break-all;
}

.export-warn {
  margin: 0 0 0.25rem;
  padding: 0.6rem 0.8rem;
  border-radius: 8px;
  background: #fdf0e6;
  border: 1px solid #e8c9a8;
  color: var(--accent-hover);
  font-size: 0.9rem;
}
</style>
