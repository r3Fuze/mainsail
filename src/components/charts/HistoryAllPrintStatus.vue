<template>
    <e-chart
        ref="historyAllPrintStatus"
        v-observe-visibility="visibilityChanged"
        :option="chartOptions"
        :init-options="{ renderer: 'svg' }"
        style="height: 250px; width: 100%"></e-chart>
</template>

<script lang="ts">
import Component from 'vue-class-component'
import { Mixins, Watch } from 'vue-property-decorator'
import BaseMixin from '@/components/mixins/base'
import type { ECharts } from 'echarts/core'

@Component({
    components: {},
})
export default class HistoryAllPrintStatus extends Mixins(BaseMixin) {
    declare $refs: {
        historyAllPrintStatus: any
    }

    private chartOptions: any = {
        darkMode: true,
        animation: false,
        grid: {
            top: 10,
            right: 0,
            bottom: 0,
            left: 10,
        },
        tooltip: {
            trigger: 'item',
            borderWidth: 0,
        },
        series: [
            {
                type: 'pie',
                data: [],
                avoidLabelOverlap: false,
                radius: ['35%', '60%'],
                emphasis: {
                    itemStyle: {
                        shadowBlur: 10,
                        shadowOffsetX: 0,
                        shadowColor: 'rgba(0, 0, 0, 0.5)',
                    },
                },
            },
        ],
    }

    get selectedJobs() {
        return this.$store.state.gui.view.history.selectedJobs ?? []
    }

    get allPrintStatusArray() {
        return this.$store.getters['server/history/getAllPrintStatusArray']
    }

    get selectedPrintStatusArray() {
        return this.$store.getters['server/history/getSelectedPrintStatusArray']
    }

    get printStatusArray() {
        return this.selectedJobs.length ? this.selectedPrintStatusArray : this.allPrintStatusArray
    }

    get chart(): ECharts | null {
        const historyAllPrintStatus = this.$refs.historyAllPrintStatus
        return historyAllPrintStatus?.inst ?? null
    }

    mounted() {
        this.chartOptions.series[0].data = this.printStatusArray
        this.chart?.setOption(this.chartOptions)

        window.addEventListener('resize', this.eventListenerResize)
    }

    destroyed() {
        window.removeEventListener('resize', this.eventListenerResize)
    }

    beforeDestroy() {
        if (typeof window === 'undefined') return
        if (this.chart) this.chart.dispose()
    }

    @Watch('printStatusArray')
    printStatusArrayChanged(newVal: any) {
        this.chart?.setOption({
            series: {
                data: newVal,
            },
        })
    }

    visibilityChanged(isVisible: boolean) {
        if (isVisible) this.chart?.resize()
    }

    eventListenerResize() {
        this.chart?.resize()
    }
}
</script>
