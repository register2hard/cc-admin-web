<template>
  <div ref="chart" :style="chartStyle"></div>
</template>

<script>
import echarts from 'echarts';
import SimplexNoise from 'components/third/bi/simplex';
import 'echarts-gl';

export default {
  name: 'customline3d',
  props: {
    config: {
      required: true,
    },
    chartStyle: {
      require: false,
      type: Object,
      default: () => ({
        height: '100%',
        position: 'absolute',
        width: '100%',
        top: '0px',
        bottom: '0px',
        left: '0px',
      }),
    },
  },
  data() {
    return {
      chart: null,
    };
  },
  created() {

  },
  mounted() {
    this.renderChart();
  },
  methods: {
    handleResize() {
      if (this.chart) {
        this.chart.resize();
      }
    },
    renderChart() {
      if (!this.chart) {
        this.chart = echarts.init(this.$refs.chart, this.config.theme);
      }
      const option = this.makeOptions(this.config, this.chartData);
      this.chart.clear();
      this.chart.setOption(option);
      setTimeout(() => {
        this.chart.resize();
      }, 100);
    },
    makeOptions() {
      const noise = new SimplexNoise(Math.random);
      function generateData() {
        let valMin = Infinity;
        let valMax = -Infinity;
        const data = [];
        for (let i = 0; i <= 50; i += 1) {
          for (let j = 0; j <= 50; j += 1) {
            const value = noise.noise2D(i / 20, j / 20);
            valMax = Math.max(valMax, value);
            valMin = Math.min(valMin, value);
            data.push([i, j, value * 2 + 4]);
          }
        }
        return data;
      }
      const data = generateData(2, -5, 5);

      return {
        visualMap: {
          show: false,
          min: 2,
          max: 6,
          inRange: {
            color: ['#313695', '#4575b4', '#74add1', '#abd9e9', '#e0f3f8', '#ffffbf', '#fee090', '#fdae61', '#f46d43', '#d73027', '#a50026'],
          },
        },
        xAxis3D: {
          type: 'value',
        },
        yAxis3D: {
          type: 'value',
        },
        zAxis3D: {
          type: 'value',
          max: 10,
          min: 0,
        },
        grid3D: {
          axisLine: {
            lineStyle: { color: '#fff' },
          },
          axisPointer: {
            lineStyle: { color: '#fff' },
          },
          viewControl: {
            autoRotate: true,
          },
          light: {
            main: {
              shadow: true,
              quality: 'ultra',
              intensity: 1.5,
            },
          },
        },
        series: [{
          type: 'bar3D',
          data,
          shading: 'lambert',
          label: {
            formatter(param) {
              return param.value[2].toFixed(1);
            },
          },
        }],
      };
    },
  },
  watch: {
    config: {
      deep: true,
      handler() {
        if (this.config.needResize) {
          this.config.needResize = false;
          this.handleResize();
        }
      },
    },
  },
};
</script>

<style scoped lang="stylus"></style>
