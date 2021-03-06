<template>
  <div id="stats" class="stats">
    <div v-if="statsChart" class="stats__chart" ref="chart"></div>
    <ul class="stats__counters">
      <li v-for="(value, name) in data" :key="name" class="stat-counter" @click="editByName(name)">
        <div class="stat-counter__name" :style="{ color: colors[name] }">{{ name }}</div>
        <div class="stat-counter__value">{{ value }}</div>
      </li>
    </ul>
  </div>
</template>

<script>
import { mapState } from 'vuex'
import * as TV from 'lightweight-charts'
import socket from '../services/socket'
import Counter from '../utils/counter'
import { getColor } from '../utils/colors'
import MultiCounter from '../utils/multiCounter'
import { defaultChartOptions, defaultLineOptions, } from './chart/chartOptions'

import options from './chart/options.json'

const chartOptions = JSON.parse(JSON.stringify(defaultChartOptions))
chartOptions.priceScale.position = 'none'
chartOptions.timeScale.rightOffset = 0
chartOptions.timeScale.secondsVisible = false
chartOptions.priceScale.mode = 0

const lineOptions = JSON.parse(JSON.stringify(defaultLineOptions))
lineOptions.overlay = true
lineOptions.lineWidth = 2
lineOptions.priceLineVisible = false
lineOptions.scaleMargins = {
  top: 0.05,
  bottom: 0.05,
}

import { formatAmount } from '../utils/helpers'
/** @type {Counter[]} */
const counters = []
/** @type {TV.IChartApi} */
let chart
let colors = []
export default {
  data() {
    return {
      isCrosshairing: false,
      data: {},
    }
  },
  computed: {
    ...mapState('settings', [
      'statsPeriod',
      'statsChart',
      'statsCounters',
      'preferQuoteCurrencySize',
    ]),
    ...mapState('app', ['actives']),
    colors() {
      return this.statsCounters.reduce((obj, counter) => {
        obj[counter.name] = counter.color
        return obj
      }, {})
    },
  },
  created() {
    this.onStoreMutation = this.$store.subscribe((mutation, state) => {
      switch (mutation.type) {
        case 'settings/TOGGLE_STAT':
          if (mutation.payload.value) {
            this.createCounter(this.statsCounters[mutation.payload.index])
          } else {
            this.removeCounter(this.statsCounters[mutation.payload.index].name)
          }
          break
        case 'settings/SET_STAT_PERIOD':
        case 'settings/SET_STAT_OUTPUT':
        case 'settings/SET_STAT_SMOOTHING':
          this.refreshCounter(this.statsCounters[mutation.payload.index].name)
          break
        case 'settings/SET_STAT_NAME':
          this.renameCounter(mutation.payload.value)
          break
        case 'settings/SET_PAIR':
          this.prepareCounters()
          break
        case 'settings/SET_STAT_COLOR':
          this.recolorCounter(
            this.statsCounters[mutation.payload.index].name,
            mutation.payload.value
          )
          break
        case 'settings/TOGGLE_STATS_CHART':
          if (mutation.payload) {
            this.createChart()
          } else {
            this.removeChart()
          }
          break
      }
    })
  },
  mounted() {
    if (this.statsChart) {
      this.createChart()
    }
    this.prepareCounters()
    socket.$on('trades', this.onTrades)
  },
  beforeDestroy() {
    socket.$off('trades', this.onTrades)
    this.clearCounters()
    this.removeChart()
    this.onStoreMutation()
  },
  methods: {
    createChart() {
      clearTimeout(this._createChartTimeout)
      this._createChartTimeout = setTimeout(() => {
        chart = TV.createChart(this.$refs.chart, chartOptions)
        for (let i = 0; i < counters.length; i++) {
          this.createCounterSerie(counters[i]);
        }
      }, 100)
    },
    createCounterSerie(counter) {
      if (counter.serie) {
        return
      }
      counter.serie = chart.addLineSeries(
        Object.assign({}, lineOptions, {
          color: counter.options.color
        })
      )
    },
    removeChart() {
      if (!chart) {
        return
      }
      for (let i = 0; i < counters.length; i++) {
        if (counters[i].serie) {
          chart.removeSeries(counters[i].serie);
          delete counters[i].serie
        }
      }
      chart.remove()
    },
    prepareCounters() {
      console.log(`[stats.prepareCounters]`)
      this.clearCounters()
      for (let i = 0; i < this.statsCounters.length; i++) {
        this.createCounter(this.statsCounters[i])
      }
    },
    onTrades(trades) {
      

      for (let i = 0; i < counters.length; i++) {
        if (counters[i].stacks.length) {
          let value = counters[i].getValue()
          if (value.length) {
            this.$set(
              this.data,
              counters[i].name,
              value.map((a) => formatAmount(a)).join('/')
            )
          } else {
            if (counters[i].serie) {
              counters[i].serie.update({
                time: counters[i].timestamp / 1000,
                value,
              })
            }
            if (!isNaN(counters[i].options.precision)) {
              value = value.toFixed(counters[i].precision)
            }
            if (!this.isCrosshairing) {
              this.$set(this.data, counters[i].name, formatAmount(value))
            }
          }
        }
        counters[i].onTrades(trades, stats)
      }
    },
    clearCounters() {
      for (let i = counters.length - 1; i >= 0; i--) {
        if (!counters[i]) {
          console.log('clearCoutner: warning counter', i, 'doesnt exists')
          continue
        }
        this.removeCounter(i)
      }
    },
    removeCounter(index) {
      const counter = this.getCounter(index)
      if (!counter) {
        console.log(`[removeCounter] couldnt find counter ${index}`)
        return
      }
      index = counters.indexOf(counter)
      console.log(`\tunbind counter ${counter.name} index ${index}`)
      counter.unbind()
      if (counter.serie) {
        chart.removeSeries(counter.serie)
      }
      this.$delete(this.data, counter.name)
      counters.splice(index, 1)
    },
    refreshCounter(index) {
      const counter = this.getCounter(index)
      if (!counter) {
        console.log(`[refreshCounter] couldnt find counter ${index}`)
        return
      }
      const options = this.statsCounters.filter(
        (a) => a.name === counter.name
      )[0]
      if (!options) {
        console.log(
          `[refreshCounter] couldnt find options for counter ${counter.name}`
        )
        return
      }
      this.removeCounter(counter.name)
      this.createCounter(options)
    },
    recolorCounter(index, color) {
      const counter = this.getCounter(index)
      if (!counter) {
        console.log(`[refreshCounter] couldnt find counter ${index}`)
        return
      }
      if (!counter.serie) {
        return
      }
      console.log('[recolorCounter]', counter.name, color)
      counter.serie.applyOptions({
        color: color,
      })
    },
    createCounter(counterOptions) {
      if (
        counterOptions.enabled &&
        typeof this.data[counterOptions.name] === 'undefined'
      ) {
        console.log(`create counter ${counterOptions.name}`, counterOptions)
        const outputType = this.getOutputType(counterOptions.output)
        const outputFn = (stats, trades) => eval(counterOptions.output)
        let counter
        if (typeof outputType === 'number') {
          console.log(
            'instanciate counter with model',
            new Array(outputType).fill(0),
            counterOptions.output
          )
          counter = new MultiCounter(outputFn, {
            options: counterOptions,
            model: new Array(outputType).fill(0),
          })
        } else {
          console.log('instanciate single counter', counterOptions.output)
          counter = new Counter(outputFn, {
            options: counterOptions
          })
          if (chart) {
            this.createCounterSerie(counter);
          }
        }
        counter.name = counterOptions.name
        counters.push(counter)
        this.$set(this.data, counter.name, 0)
      } else {
        console.log(
          `counter ${counterOptions.name} was skipped`,
          counterOptions.enabled ? 'ENABLED' : 'DISABLED',
          typeof this.data[counterOptions.name] !== 'undefined'
            ? 'ALREADY IN DATA'
            : 'NOT IN DATA'
        )
      }
    },
    renameCounter(name) {
      const names = this.statsCounters.map((a) => a.name)
      console.log('rename counter', names, name)
      let counter
      for (let i = 0; i < counters.length; i++) {
        if (names.indexOf(counters[i].name) === -1) {
          counter = counters[i]
          break
        }
      }
      if (!counter) {
        console.log('no matching counter found')
        return
      }
      console.log('rename', counter)
      this.data[name] = this.data[counter.name]
      delete this.data[counter.name]
      counter.name = name
    },
    getCounter(name) {
      if (typeof name === 'number') {
        return counters[name]
      }
      let counter
      console.log('getCounterByName', name)
      for (let i = 0; i < counters.length; i++) {
        if (counters[i].name === name) {
          counter = counters[i]
          break
        }
      }
      return counter
    },
    editByName(name) {
      let index
      for (let i = 0; i < this.statsCounters.length; i++) {
        if (this.statsCounters[i].name === name) {
          index = i
          break
        }
      }
      if (typeof index !== 'number') {
        return
      }
      this.$store.dispatch('app/openModal', {
        name: 'stat',
        id: index,
      })
    },
    getOutputType(fn) {
      var trades = [
        {
          exchange: 'binance',
          timestamp: 1563897189117,
          price: 9975.67,
          size: 0.049051,
          side: 'sell',
        },
        {
          exchange: 'binance',
          timestamp: 1563897189120,
          price: 9976.86,
          size: 0.256507,
          side: 'buy',
        },
      ]
      var stats = socket.getStatsByTrades(trades)
      const output = eval(fn)
      if (Array.isArray(output)) {
        return output.length
      } else {
        return false
      }
    },
  },
}
</script>

<style lang="scss">
@import '../assets/sass/variables';
.stats {
  position: relative;
  background-color: rgba(white, 0.05);
  &__chart {
    width: 100%;
    height: 180px;
    + .stats__counters {
      position: absolute;
      margin: .25em;
      .stat-counter {
        flex-direction: row;
        padding: 0.25em;
        &__value {
          text-align: left;
          flex-grow: 0;
        }
        &__name {
          font-size: 12px;
          margin-left: 1em;
          order: 2;
          opacity: 0;
          transform: translateX(8px);
          transition: transform .2s $easeOutExpo, opacity .2s $easeOutExpo;
        }
        &:hover {
          .stat-counter__name {
            opacity: 1;
            transform: none;
          }
        }
      }
    }
  }
  &__counters {
    padding: 0;
    margin: 0;
    list-style: none;
    top: 0;
    z-index: 11;
  }
  .stat-counter {
    display: flex;
    align-items: center;
    padding: 0.75em;
    cursor: pointer;
    + .stat-counter {
      padding-top: 0;
    }
    &__name {
      color: rgba(white, 0.5);
      letter-spacing: 0.4px;
      transition: opacity 0.2s $easeOutExpo;
    }
    &__value {
      text-align: right;
      white-space: nowrap;
      font-family: 'Roboto Condensed';
      z-index: 1;
      flex-grow: 1;
    }
    &:hover {
      .custom-stat__name {
        color: white;
      }
    }
  }
}
</style>