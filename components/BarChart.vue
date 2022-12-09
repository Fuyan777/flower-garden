<template>
  <Bar
    :chart-options="chartOptions"
    :chart-data="chartData"
    :chart-id="chartId"
    :dataset-id-key="datasetIdKey"
    :plugins="plugins"
    :css-classes="cssClasses"
    :styles="styles"
    :width="width"
    :height="height"
  />
</template>

<script>
import { Bar } from 'vue-chartjs'
import { Chart as ChartJS, Title, Tooltip, Legend, BarElement, CategoryScale, LinearScale } from 'chart.js'

ChartJS.register(Title, Tooltip, Legend, BarElement, CategoryScale, LinearScale)

export default {
  name: 'BarChart',
  components: { Bar },
  props: {
    chartId: {
      type: String,
      default: 'bar-chart'
    },
    datasetIdKey: {
      type: String,
      default: 'label'
    },
    width: {
      type: Number,
      default: 600
    },
    height: {
      type: Number,
      default: 300
    },
    cssClasses: {
      default: '',
      type: String
    },
    styles: {
      type: Object,
      default: () => {}
    },
    plugins: {
      type: Array,
      default: () => []
    },
    datasets: {
      type: Array,
      default: {
        label: 'Data One',
        backgroundColor: '#f87979',
        data: [30, 20, 12]
      }
    },
  },
  watch: {
    'chartData.datasets': {
      handler() {
        this.makeChart();
      },
      deep: true
    }
  },
  data() {
    return {
      chartData: {
        labels: [ 'player-1', 'player-2', 'player-3', 'player-4', 'player-5' ],
        datasets: this.datasets
      },
      chartOptions: {
        responsive: true,
        maintainAspectRatio: true,
        animation: false
      }
    }
  },
  mounted: function() {
  },
  methods: {
    makeChart: function() {
      this.renderChart(this.chartData, this.chartOptions);
    }
  },
}
</script>