<template>
  <div class="enrichment-panel">
    <!-- Create New Analysis -->
    <div class="section-header">
      <el-icon><TrendCharts /></el-icon>
      <span>煤矿富集指数识别</span>
    </div>

    <el-form label-position="top" size="small">
      <el-form-item label="分析名称">
        <el-input v-model="form.name" placeholder="例：山西北部富集分析" />
      </el-form-item>

      <el-form-item label="选择数据图层（多选）">
        <el-select
          v-model="form.selectedRasterIds"
          multiple
          collapse-tags
          placeholder="选择TIF数据层"
          style="width:100%"
          :loading="loadingRasters"
        >
          <el-option
            v-for="r in readyRasters"
            :key="r.id"
            :label="r.filename"
            :value="r.id"
          />
        </el-select>
        <div v-if="readyRasters.length === 0" class="hint">暂无TIF数据，请先在数据图层模块上传TIF文件</div>
      </el-form-item>

      <el-button
        type="primary"
        :disabled="!form.name || form.selectedRasterIds.length === 0"
        :loading="analyzing"
        @click="runAnalysis"
        style="width:100%"
      >
        开始富集指数识别
      </el-button>
    </el-form>

    <el-divider />

    <!-- Results List -->
    <div class="section-header">
      <el-icon><List /></el-icon>
      <span>识别结果列表</span>
      <el-button size="small" text @click="fetchRasters" style="margin-left:auto">刷新</el-button>
    </div>

    <el-empty v-if="rasters.length === 0 && !loadingRasters" description="暂无TIF数据" :image-size="50" />

    <div v-for="raster in rasters" :key="raster.id" class="raster-item">
      <div class="raster-info">
        <span class="raster-name" :title="raster.filename">{{ raster.filename }}</span>
        <el-tag :type="statusType(raster.status)" size="small">{{ statusLabel(raster.status) }}</el-tag>
      </div>
      <div class="raster-meta-row" v-if="raster.crs || raster.band_count">
        <span v-if="raster.crs">{{ raster.crs }}</span>
        <span v-if="raster.band_count">{{ raster.band_count }}波段</span>
      </div>

      <div class="raster-controls">
        <el-select v-model="selectedColormaps[raster.id]" size="small" placeholder="色带" style="width:110px">
          <el-option v-for="c in colormaps" :key="c.value" :label="c.label" :value="c.value" />
        </el-select>
        <el-button size="small" type="success" @click="loadToMap(raster)">渲染到地图</el-button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, computed } from 'vue'
import { TrendCharts, List } from '@element-plus/icons-vue'
import { ElMessage } from 'element-plus'
import type { RasterAsset } from '../types'
import { rasterApi } from '../api/rasterApi'
import { enrichmentApi } from '../api/enrichmentApi'

const props = defineProps<{
  mapViewRef: {
    addRasterLayer: (url: string, name: string) => void
  } | null
}>()

const form = ref({
  name: '',
  selectedRasterIds: [] as string[],
})

const colormaps = [
  { value: 'viridis', label: 'Viridis' },
  { value: 'terrain', label: 'Terrain' },
  { value: 'rainbow', label: 'Rainbow' },
  { value: 'plasma', label: 'Plasma' },
  { value: 'inferno', label: 'Inferno' },
  { value: 'hot', label: 'Hot' },
  { value: 'jet', label: 'Jet' },
  { value: 'RdYlGn', label: 'RdYlGn' },
]

const rasters = ref<RasterAsset[]>([])
const loadingRasters = ref(false)
const analyzing = ref(false)
const selectedColormaps = ref<Record<string, string>>({})

const readyRasters = computed(() => rasters.value.filter((r) => r.status === 'ready'))

async function fetchRasters() {
  loadingRasters.value = true
  try {
    rasters.value = await rasterApi.list()
  } finally {
    loadingRasters.value = false
  }
}

async function runAnalysis() {
  if (!form.value.name || form.value.selectedRasterIds.length === 0) return
  analyzing.value = true
  try {
    await enrichmentApi.analyze(
      form.value.name,
      form.value.selectedRasterIds,
    )
    form.value.name = ''
    form.value.selectedRasterIds = []
    ElMessage.success('富集指数识别完成')
    await fetchRasters()
  } catch {
    ElMessage.error('识别分析失败，请重试')
  } finally {
    analyzing.value = false
  }
}

function loadToMap(raster: RasterAsset) {
  if (!props.mapViewRef) return
  const filePath = raster.cog_path || raster.original_path
  if (!filePath) return
  const colormap = selectedColormaps.value[raster.id] || 'viridis'
  const titilerBase = import.meta.env.VITE_TITILER_URL || 'http://localhost:8080'
  const titilerPath = filePath.replace(/^\/app\/data\//, '/data/')
  const url = `${titilerBase}/cog/tiles/{z}/{x}/{y}.png?url=${titilerPath}&colormap_name=${colormap}`
  props.mapViewRef.addRasterLayer(url, raster.filename)
}

function statusType(status: string) {
  return { ready: 'success', processing: 'warning', pending: 'info', failed: 'danger' }[status] || 'info'
}

function statusLabel(status: string) {
  return { ready: '就绪', processing: '处理中', pending: '待处理', failed: '失败' }[status] || status
}

onMounted(fetchRasters)
</script>

<style scoped>
.enrichment-panel {
  padding: 12px;
  font-size: 13px;
}

.section-header {
  display: flex;
  align-items: center;
  gap: 6px;
  font-weight: bold;
  font-size: 13px;
  color: #e0e0e0;
  margin-bottom: 10px;
}

.hint {
  font-size: 11px;
  color: #888;
  margin-top: 4px;
}

.raster-item {
  background: #1e2d4a;
  border-radius: 6px;
  padding: 10px;
  margin-bottom: 8px;
  border: 1px solid #2c3e5a;
}

.raster-info {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 6px;
}

.raster-name {
  font-size: 12px;
  font-weight: 500;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  max-width: 160px;
  color: #ccc;
}

.raster-meta-row {
  font-size: 11px;
  color: #888;
  margin-bottom: 6px;
  display: flex;
  gap: 8px;
}

.raster-controls {
  display: flex;
  gap: 6px;
  align-items: center;
  flex-wrap: wrap;
}
</style>
