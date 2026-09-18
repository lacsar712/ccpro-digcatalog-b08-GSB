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

    <div v-if="showExport" class="modal-mask" @click.self="closeExport">
      <div class="modal">
        <h3>导出出土文物 CSV</h3>
        <div class="export-summary">
          <dl>
            <dt>探方筛选</dt>
            <dd>{{ exportSummary.unit }}</dd>
            <dt>器物类型</dt>
            <dd>{{ exportSummary.type }}</dd>
            <dt>预估条数</dt>
            <dd><strong>{{ list.length }}</strong> 条</dd>
          </dl>
        </div>
        <p class="page-sub export-note">
          导出列：registerNo、artifactType、unitCode、materialName、completeness、findDate、storageLoc。
          文件采用 UTF-8（含 BOM）编码，可用 Excel 直接打开；数据由服务器按当前筛选条件生成。
        </p>
        <p v-if="list.length === 0" class="export-empty">
          当前筛选下暂无文物记录。仍可下载仅包含表头的空 CSV 文件；如不需要请取消并调整筛选条件。
        </p>
        <p v-if="exportError" class="error">{{ exportError }}</p>
        <div class="modal-actions">
          <button class="btn secondary" :disabled="exporting" @click="closeExport">取消</button>
          <button class="btn" :disabled="exporting" @click="doExport">
            {{ exporting ? '导出中…' : '下载 CSV' }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { onMounted, reactive, ref } from 'vue'
import api from '../api/http'

const artifactTypes = ['陶片', '青铜器', '骨器', '玉器', '石器', '铁器', '其他']
const list = ref([])
const units = ref([])
const materials = ref([])
const filterUnitId = ref('')
const filterType = ref('')
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

function currentParams() {
  const params = {}
  if (filterUnitId.value) params.unitId = filterUnitId.value
  if (filterType.value) params.artifactType = filterType.value
  return params
}

const exportSummary = ref({ unit: '全部探方', type: '全部类型' })

function openExport() {
  const unit = units.value.find((u) => String(u.id) === filterUnitId.value)
  exportSummary.value = {
    unit: unit ? `${unit.site?.name || ''} / ${unit.code}` : '全部探方',
    type: filterType.value || '全部类型'
  }
  exportError.value = ''
  showExport.value = true
}

function closeExport() {
  if (exporting.value) return
  showExport.value = false
}

async function doExport() {
  exportError.value = ''
  exporting.value = true
  try {
    const { data } = await api.get('/finds/export.csv', {
      params: currentParams(),
      responseType: 'blob'
    })
    const url = URL.createObjectURL(data)
    const a = document.createElement('a')
    a.href = url
    a.download = 'finds-export.csv'
    document.body.appendChild(a)
    a.click()
    a.remove()
    URL.revokeObjectURL(url)
    showExport.value = false
  } catch (e) {
    // 错误响应也是 blob，需读回文本里的 error 字段
    const msg = await readBlobError(e)
    exportError.value = msg || '导出失败，请稍后重试'
  } finally {
    exporting.value = false
  }
}

async function readBlobError(e) {
  const blob = e.response?.data
  if (!(blob instanceof Blob) || !blob.size) return e.response?.data?.error || ''
  try {
    const parsed = JSON.parse(await blob.text())
    return parsed.error || ''
  } catch {
    return ''
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
.toolbar-actions {
  display: flex;
  gap: 0.6rem;
}

.filters {
  display: flex;
  gap: 1rem;
  margin-bottom: 1rem;
  flex-wrap: wrap;
}

.filters label {
  min-width: 200px;
}

.export-summary dl {
  display: grid;
  grid-template-columns: 90px 1fr;
  gap: 0.4rem 1rem;
  margin: 0 0 0.75rem;
  padding: 0.85rem 1rem;
  background: #f7f1e7;
  border: 1px solid var(--border);
  border-radius: 10px;
  font-size: 0.92rem;
}

.export-summary dt {
  color: var(--muted);
}

.export-summary dd {
  margin: 0;
}

.export-note {
  line-height: 1.6;
}

.export-empty {
  margin-top: 0.75rem;
  padding: 0.6rem 0.85rem;
  border-radius: 8px;
  background: #f6e3de;
  color: var(--danger);
  font-size: 0.88rem;
  line-height: 1.6;
}
</style>
