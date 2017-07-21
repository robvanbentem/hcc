<template>
  <div id="app">
    <div class="alert alert-info">
      <strong>HCC</strong>: Home Control Center
    </div>


    <div class="menu">
      <div class="interval" v-for="i in intervals">
        <a href="javascript:;" @click="setInt(i)">
            <span v-if="i >= 60">
            {{ i / 60 }} min
            </span>
          <span v-else>
              {{ i }} sec
            </span>
        </a>
      </div>
      <div>/</div>
      <div class="start" v-for="i in starts">
        <a href="javascript:;" @click="setStart(i)">
            <span v-if="i.min >= 60">
            {{ i.min / 60 }} hr
            </span>
          <span v-else>
            {{ i.min }} min
            </span>
        </a>
      </div>

    </div>
    <div class="main">
      <div class="graph" v-for="s in sources">
        <canvas :id="s.device"></canvas>
      </div>
    </div>


    <router-view></router-view>
  </div>
</template>

<script>
  import Chart from 'chart.js'
  import axios from 'axios'
  import moment from 'moment'


  function number_format (number, decimals, decPoint, thousandsSep) {
    decimals = decimals || 0;
    number = parseFloat(number);

    if (!decPoint || !thousandsSep) {
      decPoint = '.';
      thousandsSep = ',';
    }

    var roundedNumber = Math.round(Math.abs(number) * ('1e' + decimals)) + '';
    var numbersString = decimals ? roundedNumber.slice(0, decimals * -1) : roundedNumber;
    var decimalsString = decimals ? roundedNumber.slice(decimals * -1) : '';
    var formattedNumber = "";

    while (numbersString.length > 3) {
      formattedNumber += thousandsSep + numbersString.slice(-3)
      numbersString = numbersString.slice(0, -3);
    }

    return (number < 0 ? '-' : '') + numbersString + formattedNumber + (decimalsString ? (decPoint + decimalsString) : '');
  }

  export default {
    name: 'app',
    data() {
      return {
        looper: null,
        chart: null,

        interval: 360,
        start: 300,

        intervals: [
          30,
          60,
          150,
          300,
          900,
        ],
        starts: [
          {min: 15, int: 30},
          {min: 60, int: 60},
          {min: 180, int: 150},
          {min: 360, int: 300},
          {min: 720, int: 600},
          {min: 1440, int: 900},
          {min: 10080, int: 7200}
        ],
        sources: [
          {
            type: 'DS18B20',
            device: '07cd17',
            color: '#5898ff',
            chart: null
          },
          {
            type: 'DS18B20',
            device: '07d75f',
            color: '#ff355a',
            chart: null
          }
        ]
      }
    },
    mounted() {
      this.render()
    },
    methods: {
      setInt(i) {
        this.interval = i
        this.render()
      },
      setStart(i) {
        this.interval = i.int
        this.start = i.min

        this.render()
      },
      render() {

        clearInterval(this.looper)
        this.looper = setInterval(() => {
          this.render()
        }, 5000)


        var time = (Math.round(+new Date() / 1000)) - (2 * 60 * 60) - (this.start * 60)
        for (let idx in this.sources) {
          let s = this.sources[idx]

          axios.get(`http://192.168.1.6:4422/data?type=${s.type}&device=${s.device}&interval=${this.interval}&start=${time}`).then((resp) => {
            let ctx = document.getElementById(s.device).getContext('2d');

            let sets = []

            let labels = resp.data.map((obj) => moment.unix(obj.Timestamp).format("DD-MM HH:mm"))

            sets.push({
              label: 'MAX',
              backgroundColor: '#eee',
              borderWidth: 1,
              borderDash: [2, 2],
              borderColor: '#999',
              data: resp.data.map((obj) => obj.Count > 0 ? number_format(obj.Max, 2, '.', '') : null),
              steppedLine: false,
              fill: false,
              pointRadius: 0
            })

            sets.push({
              label: 'AVG',
              backgroundColor: s.color,
              borderColor: s.color,
              data: resp.data.map((obj) => obj.Count > 0 ? number_format(obj.Avg, 2, '.', '') : null),
              steppedLine: false,
              fill: false,
              pointRadius: 2
            })

            sets.push({
              label: 'MIN',
              backgroundColor: '#eee',
              borderWidth: 1,
              borderDash: [2, 3],
              borderColor: '#999',
              data: resp.data.map((obj) => obj.Count > 0 ? number_format(obj.Min, 2, '.', '') : null),
              steppedLine: false,
              fill: '-2',
              pointRadius: 0
            })

            let config = {
              type: 'line',
              data: {
                labels: labels,
                datasets: sets
              },
              options: {
                animation: false,
                responsive: true,
                maintainAspectRatio: false,
                legend: {
                  labels: {
                    usePointStyle: false
                  }
                },
                title: {
                  display: true,
                  text: 'GoGarden - DS18B20 - ' + s.device
                },
                tooltips: {
                  mode: 'nearest',
                  intersect: false,
                },
                hover: {
                  mode: null,
                  intersect: true
                },
                scales: {
                  xAxes: [{
                    display: true,
                    scaleLabel: {
                      display: true,
                      labelString: 'Time'
                    }
                  }],
                  yAxes: [{
                    display: true,
                    scaleLabel: {
                      display: true,
                      labelString: 'Temperature'
                    }
                  }]
                }
              }
            };

            if (s.chart !== null) {
              s.chart.destroy()
            }

            s.chart = new Chart(ctx, config);

          })
        }
      }
    }
  }
</script>

<style>
  #app {
    font-family: 'Avenir', Helvetica, Arial, sans-serif;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    text-align: center;
    color: #2c3e50;
    margin-top: 60px;
  }

  .main {
    display: flex;
    flex-wrap: nowrap;
    justify-content: space-around;
    align-items: flex-start;
  }

  .graph {
    flex-grow: 0;
    flex-basis: 40%;
    height: 500px;
  }

  .menu {
    display: flex;
    flex-wrap: wrap;
    margin-top: 72px;
    margin-left: 24px;
    border: 1px solid #bcdff1;
    background-color: #fafafa;
    padding: 8px;
  }

  .menu > div {
    padding: 4px 12px 4px 12px;
  }

  .menu > div.interval, div.start {
    padding: 4px;
  }
</style>
