<template>
  <div ref="chart" :style="chartStyle" />
</template>

<script>
import { debounce } from 'quasar';
import echarts from 'echarts';

require('echarts/theme/macarons');

export default {
  name: 'maps',
  props: {
    config: {
      required: true,
      default: () => { },
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
  components: {
  },
  data() {
    return {
      chart: null,
      allParam: {},
      chartData: {},
      data: [],
      min: 0,
      max: 100,
      interval: this.config.interval,
      ti: 0,
      loading: false,
      tableData: [],
      color: ['#a6c84c', '#ffa022', '#46bee9'],
      planePath: 'path://M1705.06,1318.313v-89.254l-319.9-221.799l0.073-208.063c0.521-84.662-26.629-121.796-63.961-121.491c-37.332-0.305-64.482,36.829-63.961,121.491l0.073,208.063l-319.9,221.799v89.254l330.343-157.288l12.238,241.308l-134.449,92.931l0.531,42.034l175.125-42.917l175.125,42.917l0.531-42.034l-134.449-92.931l12.238-241.308L1705.06,1318.313z',
    };
  },
  computed: {
    orders() {
      return [
        ...this.config.cols.filter((t) => t.sort).map((t) => ({
          column: t.name,
          direction: t.sort,
        })),
        ...this.config.rows.filter((t) => t.sort).map((t) => ({
          column: `${t.agg}(${t.name.split('@')[0]})`,
          direction: t.sort,
        })),
      ];
    },
    loopHandler() {
      return this.config.loop && (this.config.interval - 0);
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
        this.getData();
      },
    },
    'config.series.maps.id': {
      handler(n, o) {
        if (n !== o) {
          this.registerMap();
        }
      },
      deep: true,
    },
    'config.loop': {
      handler(n, o) {
        if (n !== o) {
          this.confirmLoop();
        }
      },
    },
    'config.interval': {
      handler(n, o) {
        if (n !== o) {
          this.confirmLoop();
        }
      },
    },
  },
  methods: {
    getOrders() {
      const orders = [];
      this.config.orders.forEach((o) => {
        orders.push({
          column: o.name,
          direction: o.sort,
        });
      });
      return orders;
    },
    doLoadData() {
      if (this.config.viewId && this.config.cols && this.config.cols.length > 0) {
        this.loading = true;
        this.$axios.post(`/bi/view/getChartData/${this.config.viewId}`, {
          type: this.config.type,
          aggregators: this.config.rows.map((v) => ({
            column: v.name.split('@')[0],
            func: v.agg,
            alias: v.field.alias,
            axisIndex: v.leftRight ? '1' : '0',
          })),
          allParam: this.allParam,
          cache: false,
          expired: 0,
          filters: this.config.filters,
          groups: this.config.cols,
          nativeQuery: false,
          orders: this.getOrders(),
          pageNo: 1,
          pageSize: this.config.length,
        }).then(({ result }) => {
          this.data = result.dataList || [];
          this.config.series.maps.visualMap.min = result.min;
          this.config.series.maps.visualMap.max = result.max;
          this.renderChart();
        }).finally(() => {
          this.loading = false;
          this.confirmLoop();
        });
      }
    },
    loop() {
      clearTimeout(this.ti);
      this.interval -= 1;
      if (this.interval < 1) {
        this.getData();
      } else {
        this.ti = setTimeout(() => {
          this.loop();
        }, 999);
      }
    },
    confirmLoop() {
      if (this.loopHandler) {
        this.interval = this.config.interval - (-1);
        this.loop();
      } else {
        clearTimeout(this.ti);
      }
    },
    allParamChange(allParam) {
      this.allParam = allParam;
      this.getData();
    },
    handleResize() {
      if (this.chart) {
        this.chart.resize();
      }
    },
    registerMap() {
      if (this.config.series.maps.id) {
        this.$axios.get(`/bi/map/queryById?id=${this.config.series.maps.id}`).then(({ result }) => {
          echarts.registerMap('MP', result.json);
          this.makeOptions();
        }).finally(() => {
        });
      }
    },
    makeOptions() {
      const series = [
        {
          type: 'map',
          mapType: 'MP',
          zoom: this.config.series.maps.zoom / 10,
          label: {
            show: this.config.series.maps.label.show,
            color: this.config.series.maps.label.color,
          },
          itemStyle: {
            borderColor: this.config.series.maps.itemStyle.borderColor,
            borderWidth: this.config.series.maps.itemStyle.borderWidth,
            borderType: this.config.series.maps.itemStyle.borderType,
            opacity: this.config.series.maps.itemStyle.opacity / 100,
          },
          left: `${this.config.grid.left}%`,
          top: `${this.config.grid.top}%`,
          right: `${this.config.grid.right}%`,
          bottom: `${this.config.grid.bottom}%`,
          data: this.data,
          effect: {
            show: true,
            period: 6,
            trailLength: 0,
            symbol: this.planePath,
            symbolSize: 15,
          },
          zlevel: 2,
          symbol: ['none', 'arrow'],
          symbolSize: 10,
          lineStyle: {
            normal: {
              color: this.color[0],
              width: 1,
              opacity: 0.6,
              curveness: 0.2,
            },
          },
        },
      ];
      const option = {
        title: this.config.title,
        tooltip: this.config.tooltip,
        visualMap: this.config.series.maps.visualMap,
        series,
      };
      this.chart.clear();
      this.chart.setOption(option);
      setTimeout(() => {
        this.chart.resize();
      }, 100);
    },
    renderChart() {
      if (!this.chart) {
        this.chart = echarts.init(this.$refs.chart, this.config.theme);
        this.registerMap();
      }
      this.makeOptions();
    },
  },
  mounted() {
    this.doLoadData();
    this.renderChart();
  },
  created() {
    // 这里监听所有请求参数
    this.$root.$on('allParamChange', this.allParamChange);
    this.getData = debounce(this.doLoadData, 500);
  },
  beforeDestroy() {
    clearTimeout(this.ti);
    this.$root.$off('allParamChange', this.allParamChange);
  },
};
</script>

<style lang="stylus"></style>
