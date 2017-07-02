<template>
  <div id="app">
    <div class="alert alert-info">
      <strong>HCC</strong>: Home Control Center
    </div>


    <div class="main">
      <canvas id="myChart"></canvas>
    </div>


    <router-view></router-view>
  </div>
</template>

<script>
  import Chart from 'chart.js'
  import axios from 'axios'
  import moment from 'moment'

  export default {
    name: 'app',
    mounted() {

      this.render()
      setInterval(() => {
        this.render()
      }, 30000)
    },
    methods: {
      render() {
        Promise.all([
          axios.get('http://192.168.1.6:4422/data?type=DS18B20&device=07d464&interval=300&start=2017-01-01'),
          axios.get('http://192.168.1.6:4422/data?type=DS18B20&device=26f902&interval=300&start=2017-01-01'),

        ]).then((resps) => {
          let ctx = document.getElementById("myChart").getContext('2d');

          let labels = {}
          let lbls = []
          let nums = {}

          for (const r in resps) {


            let resp = resps[r]

            labels[resp.data[0].device] = []
            nums[resp.data[0].device] = []

            for (const d in resp.data) {
              let e = resp.data[d];
              let m = moment(e.date, "YYYY-MM-DD HH:mm:ss").format("HH:mm")

              if (r == 0) {
                lbls.push(m)
              }

              labels[e.device].push(m)
              nums[e.device].push(e.avg)
            }

          }

          let sets = []

          for (let d in labels) {
            sets.push({
              label: d,
              backgroundColor: d == '07d464' ? '#0000ff' : '#00ff00',
              borderColor: d == '07d464' ? '#0000ff' : '#00ff00',
              data: nums[d],
              fill: false,
            })
          }

          var config = {
            type: 'line',
            data: {
              labels: lbls,
              datasets: sets
            },
            options: {
              animation: false,
              responsive: true,
              title: {
                display: true,
                text: 'Chart.js Line Chart'
              },
              tooltips: {
                mode: null,
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

          new Chart(ctx, config);
        }).catch((err) => {
          console.log(err)
        })
      }
    }
  }
</script>

<style lang="sass">
  #app
    font-family: 'Avenir', Helvetica, Arial, sans-serif
    -webkit-font-smoothing: antialiased
    -moz-osx-font-smoothing: grayscale
    text-align: center
    color: #2c3e50
    margin-top: 60px

    .main
      width: 100%
      max-height: 400px
      padding: 20px

</style>
