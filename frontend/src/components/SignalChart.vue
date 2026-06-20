<script setup lang="ts">
import { computed, watch, ref, nextTick } from 'vue';
import VChart from 'vue-echarts';
import { use } from 'echarts/core';
import { CanvasRenderer } from 'echarts/renderers';
import { LineChart } from 'echarts/charts';
import {
  GridComponent,
  TooltipComponent,
  LegendComponent,
  DataZoomComponent,
  ToolboxComponent
} from 'echarts/components';
import { useCanBusStore } from '../store/canbus';
import type { ECharts } from 'echarts';

use([CanvasRenderer, LineChart, GridComponent, TooltipComponent, LegendComponent, DataZoomComponent, ToolboxComponent]);

const store = useCanBusStore();
const chartRef = ref<InstanceType<typeof VChart> | null>(null);
const defaultDataZoomStart = ref(70);
const defaultDataZoomEnd = ref(100);
const dataZoomStart = ref(70);
const dataZoomEnd = ref(100);

const chartOption = computed(() => {
  const signalEntries = Array.from(store.signals.entries());

  const colors = ['#06b6d4', '#22c55e', '#ef4444', '#eab308', '#a855f7'];
  const series = signalEntries.map(([name, sig], idx) => ({
    name,
    type: 'line' as const,
    smooth: true,
    symbol: 'none',
    lineStyle: { width: 2 },
    itemStyle: { color: colors[idx % colors.length] },
    data: sig.data.map(d => [d.time, d.value])
  }));

  return {
    backgroundColor: '#111827',
    tooltip: {
      trigger: 'axis',
      backgroundColor: '#1f2937',
      borderColor: '#374151',
      textStyle: { color: '#e5e7eb', fontSize: 12 },
      formatter: (params: any) => {
        if (!Array.isArray(params) || params.length === 0) return '';
        const time = new Date(params[0].value[0]).toLocaleTimeString('zh-CN', { hour12: false });
        let html = `<div style="font-size:11px;color:#9ca3af">${time}</div>`;
        for (const p of params) {
          html += `<div style="display:flex;align-items:center;gap:6px">
            <span style="display:inline-block;width:8px;height:8px;border-radius:50%;background:${p.color}"></span>
            <span>${p.seriesName}: <b>${Number(p.value[1]).toFixed(1)}</b></span>
          </div>`;
        }
        return html;
      }
    },
    legend: {
      top: 8,
      textStyle: { color: '#9ca3af', fontSize: 11 },
      itemWidth: 12,
      itemHeight: 2
    },
    grid: {
      left: 60,
      right: 20,
      top: 45,
      bottom: 35
    },
    xAxis: {
      type: 'value',
      axisLabel: {
        color: '#6b7280',
        fontSize: 10,
        formatter: (val: number) => {
          const d = new Date(val);
          return d.toLocaleTimeString('zh-CN', { hour12: false });
        }
      },
      axisLine: { lineStyle: { color: '#374151' } },
      splitLine: { lineStyle: { color: '#1f2937' } }
    },
    yAxis: {
      type: 'value',
      axisLabel: { color: '#6b7280', fontSize: 10 },
      axisLine: { lineStyle: { color: '#374151' } },
      splitLine: { lineStyle: { color: '#1f2937' } }
    },
    dataZoom: [
      {
        type: 'inside',
        xAxisIndex: [0],
        start: dataZoomStart.value,
        end: dataZoomEnd.value,
        zoomOnMouseWheel: true,
        moveOnMouseMove: true,
        moveOnMouseWheel: false
      },
      {
        type: 'slider',
        xAxisIndex: [0],
        start: dataZoomStart.value,
        end: dataZoomEnd.value,
        height: 20,
        bottom: 8,
        borderColor: '#374151',
        backgroundColor: '#1f2937',
        fillerColor: 'rgba(6, 182, 212, 0.2)',
        handleStyle: {
          color: '#06b6d4',
          borderColor: '#06b6d4'
        },
        textStyle: { color: '#6b7280' },
        brushSelect: false
      }
    ],
    toolbox: {
      right: 60,
      top: 8,
      iconStyle: {
        borderColor: '#6b7280',
        color: '#6b7280'
      },
      feature: {
        dataZoom: {
          yAxisIndex: 'none',
          icon: {
            zoom: 'path://M17 11v2h-4v4h-2v-4H7v-2h4V7h2v4h4zM12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm0 18c-4.41 0-8-3.59-8-8s3.59-8 8-8 8 3.59 8 8-3.59 8-8 8z',
            back: 'path://M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm1 15h-2v-2h2v2zm0-4h-2V7h2v6z'
          }
        }
      }
    },
    series
  };
});

function onChartReady(chart: ECharts) {
  chart.on('dataZoom', () => {
    const option = chart.getOption();
    if (option && option.dataZoom && option.dataZoom.length > 0) {
      dataZoomStart.value = option.dataZoom[0].start as number;
      dataZoomEnd.value = option.dataZoom[0].end as number;
    }
  });
}

function resetZoom() {
  dataZoomStart.value = defaultDataZoomStart.value;
  dataZoomEnd.value = defaultDataZoomEnd.value;
  nextTick(() => {
    if (chartRef.value && chartRef.value.getEchartsInstance) {
      const chart = chartRef.value.getEchartsInstance();
      chart.dispatchAction({
        type: 'dataZoom',
        start: defaultDataZoomStart.value,
        end: defaultDataZoomEnd.value
      });
    }
  });
}

function zoomIn() {
  const range = dataZoomEnd.value - dataZoomStart.value;
  const newRange = Math.max(range * 0.8, 5);
  const center = (dataZoomStart.value + dataZoomEnd.value) / 2;
  dataZoomStart.value = Math.max(0, center - newRange / 2);
  dataZoomEnd.value = Math.min(100, center + newRange / 2);
  nextTick(() => {
    if (chartRef.value && chartRef.value.getEchartsInstance) {
      const chart = chartRef.value.getEchartsInstance();
      chart.dispatchAction({
        type: 'dataZoom',
        start: dataZoomStart.value,
        end: dataZoomEnd.value
      });
    }
  });
}

function zoomOut() {
  const range = dataZoomEnd.value - dataZoomStart.value;
  const newRange = Math.min(range * 1.25, 100);
  const center = (dataZoomStart.value + dataZoomEnd.value) / 2;
  dataZoomStart.value = Math.max(0, center - newRange / 2);
  dataZoomEnd.value = Math.min(100, center + newRange / 2);
  nextTick(() => {
    if (chartRef.value && chartRef.value.getEchartsInstance) {
      const chart = chartRef.value.getEchartsInstance();
      chart.dispatchAction({
        type: 'dataZoom',
        start: dataZoomStart.value,
        end: dataZoomEnd.value
      });
    }
  });
}
</script>

<template>
  <div class="flex flex-col h-full bg-gray-900 rounded-lg overflow-hidden relative">
    <div class="px-4 py-2 bg-gray-800 border-b border-gray-700 flex items-center justify-between">
      <h3 class="text-sm font-semibold text-gray-300">信号趋势图</h3>
      <div class="flex items-center gap-2">
        <span class="text-xs text-gray-500 mr-2">
          {{ store.signals.size }} 个信号活跃
        </span>
        <div class="flex items-center gap-1">
          <button
            @click="zoomOut"
            class="px-2 py-1 text-xs bg-gray-700 hover:bg-gray-600 text-gray-300 rounded transition-colors"
            title="缩小时间窗口"
          >
            −
          </button>
          <button
            @click="zoomIn"
            class="px-2 py-1 text-xs bg-gray-700 hover:bg-gray-600 text-gray-300 rounded transition-colors"
            title="放大时间窗口"
          >
            +
          </button>
          <button
            @click="resetZoom"
            class="px-2 py-1 text-xs bg-cyan-600 hover:bg-cyan-500 text-white rounded transition-colors"
            title="复位时间窗口"
          >
            复位
          </button>
        </div>
      </div>
    </div>
    <div class="flex-1 p-2">
      <VChart
        ref="chartRef"
        :option="chartOption"
        autoresize
        class="w-full h-full"
        style="min-height: 200px;"
        @ready="onChartReady"
      />
    </div>
    <div v-if="store.signals.size === 0" class="absolute inset-0 flex items-center justify-center pointer-events-none">
      <p class="text-gray-600 text-sm">等待信号数据...</p>
    </div>
  </div>
</template>
