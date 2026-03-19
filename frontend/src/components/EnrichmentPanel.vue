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
        融合
      </el-button>
    </el-form>

    <el-divider />

    <!-- Results List -->
    <div class="section-header">
      <el-icon><List /></el-icon>
      <span>识别结果列表</span>
      <el-button size="small" text @click="fetchEnrichmentResults" style="margin-left:auto">刷新</el-button>
    </div>

    <el-empty v-if="enrichmentResults.length === 0 && !loadingResults" description="暂无识别结果" :image-size="50" />

    <div v-for="result in enrichmentResults" :key="result.id" class="raster-item">
      <span class="raster-name" :title="result.name">{{ result.name }}</span>
      <div class="raster-controls">
        <el-select v-model="selectedColormaps[result.id]" size="small" placeholder="色带" style="width:110px">
          <el-option v-for="c in colormaps" :key="c.value" :label="c.label" :value="c.value" />
        </el-select>
        <el-button size="small" type="success" @click="loadToMap(result)">渲染到地图</el-button>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, computed } from 'vue'
import { TrendCharts, List } from '@element-plus/icons-vue'
import { ElMessage } from 'element-plus'
import type { RasterAsset, EnrichmentResultItem, EnrichmentGridPoint } from '../types'
import { rasterApi } from '../api/rasterApi'
import { enrichmentApi } from '../api/enrichmentApi'

const props = defineProps<{
  mapViewRef: {
    addRasterLayer: (url: string, name: string) => void
    showEnrichmentLayer: (grid: EnrichmentGridPoint[], colormap: string, name: string) => void
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

// Data layer rasters (for the form dropdown selector)
const rasters = ref<RasterAsset[]>([])
const loadingRasters = ref(false)

// Enrichment analysis results (for the results list)
const enrichmentResults = ref<EnrichmentResultItem[]>([])
const loadingResults = ref(false)

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

async function fetchEnrichmentResults() {
  loadingResults.value = true
  try {
    enrichmentResults.value = await enrichmentApi.list()
  } finally {
    loadingResults.value = false
  }
}

async function runAnalysis() {
  if (!form.value.name || form.value.selectedRasterIds.length === 0) return
  analyzing.value = true
  try {
    await enrichmentApi.analyze(
      form.value.name,
      form.value.selectedRasterIds,
      'overlay',
    )
    form.value.name = ''
    form.value.selectedRasterIds = []
    ElMessage.success('融合分析完成')
    await fetchEnrichmentResults()
  } catch {
    ElMessage.error('融合分析失败，请重试')
  } finally {
    analyzing.value = false
  }
}

async function loadToMap(result: EnrichmentResultItem) {
  if (!props.mapViewRef) return
  const colormap = selectedColormaps.value[result.id] || result.colormap || 'hot'
  try {
    const data = await enrichmentApi.getGrid(result.id)
    props.mapViewRef.showEnrichmentLayer(data.grid, colormap, result.name)
  } catch {
    ElMessage.error('加载融合图层失败')
  }
}

onMounted(() => {
  fetchRasters()
  fetchEnrichmentResults()
})
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
  display: flex;
  align-items: center;
  gap: 8px;
}

.raster-name {
  font-size: 12px;
  font-weight: 500;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  flex: 1;
  color: #ccc;
}

.raster-controls {
  display: flex;
  gap: 6px;
  align-items: center;
  flex-wrap: wrap;
}
</style>
