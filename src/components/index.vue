<template>
  <section>
    <div class="container is-fluid">
      <div class="hero title">Explore</div>
      <div class="columns">
        <div class="column is-one-quarter" style="align-content: center; width: 250px">
          <b-field label="Dataset">
            <b-select
              placeholder="Select a dataset"
              v-model="select.dataset"
              @input="getData"
              expanded
            >
              <option value="covid19-country">Covid19 by country</option>
            </b-select>
          </b-field>

          <b-field label="Country" v-if="select.dataset === 'covid19-country'">
            <b-select
              placeholder="Select a country"
              v-model="select.country"
              @input="getData"
              expanded
            >
              <option
                v-for="country in countries"
                :key="country.code"
                :value="country.code"
              >{{ country.name }}</option>
            </b-select>
          </b-field>

          <!--div> TODO: syncing charts for country comparison https://apexcharts.com/vue-chart-demos/line-charts/syncing-charts/
            <span class="compared">Compare To (Optional)</span>
          </div>

          <b-field>
            <b-select
              placeholder="Select a country"
              v-model="select.countryCompare"
              @input="getCountryTrend"
              expanded
            >
              <option
                v-for="country in countries"
                :key="country.code"
                :value="country.code"
              >{{ country.name }}</option>
            </b-select>
          </b-field-->

          <b-field label="Select a date range" v-if="select.dataset === 'covid19-country'">
            <b-datepicker
              placeholder="Click to select..."
              v-model="select.dates"
              range
              @input="getData"
            ></b-datepicker>
          </b-field>

          <b-field label="Chart Type">
            <b-select
              placeholder="Select a chart type"
              v-model="select.chart"
              @input="generateChart"
              expanded
            >
              <option value="line">Line chart</option>
              <option value="bar">Bar chart</option>
            </b-select>
          </b-field>
        </div>
        <div class="column">
          <apexchart
            height="400"
            width="1000"
            :type="select.chart"
            :options="options"
            :series="series"
          />
        </div>
      </div>
    </div>
  </section>
</template>

<script>
export default {
  data() {
    return {
      countries: [],
      select: {
        dataset: null,
        country: null,
        dates: [],
        chart: null
      },
      trend: [],
      options: {
        chart: {
          id: "vue-chart"
        },
        xaxis: {
          type: "datetime",
          labels: {
            format: "dd MMM"
          }
        }
      },
      series: []
    };
  },

  created() {
    this.loadAffectedCountry();
  },

  methods: {
    loadAffectedCountry() {
      const url = "https://api.coronatracker.com/v2/analytics/country";
      this.$http.get(url).then(res => {
        const mappedInfo = res.data
          .map(m => {
            return {
              code: m.countryCode,
              name: m.countryName
            };
          })
          .filter(m => m.code != "OT");

        this.countries = [{ code: "global", name: "Global" }, ...mappedInfo];
      });
    },

    convertDate(date) {
      var tzoffset = date.getTimezoneOffset() * 60000;
      var localISOTime = new Date(date - tzoffset).toISOString().slice(0, 10);
      return localISOTime;
    },

    getCountryTrend() {
      const url =
        "https://api.coronatracker.com/v3/analytics/trend/country?" +
        "countryCode=" +
        this.select.country +
        "&startDate=" +
        this.convertDate(this.select.dates[0]) +
        "&endDate=" +
        this.convertDate(this.select.dates[1]);
      this.$http
        .get(url)
        .then(res => {
          this.trend = res.data;
        })
        .catch(error => {
          console.log(error);
        })
        .finally(() => {
          if (this.select.chart) this.generateChart();
        });
    },

    getData() {
      if (this.select.dataset && this.select.country && this.select.dates) {
        this.getCountryTrend();
      }
    },

    generateChart() {
      let countryTrendConfirmed = [];
      let countryTrendRecovered = [];
      let countryTrendDeath = [];
      let confirmedLastMax = 0;
      let recoveredLastMax = 0;
      let deadLastMax = 0;
      let dateRange = [];

      this.trend.forEach(day => {
        confirmedLastMax = Math.max(day["total_confirmed"], confirmedLastMax);
        recoveredLastMax = Math.max(day["total_recovered"], recoveredLastMax);
        deadLastMax = Math.max(day["total_deaths"], deadLastMax);

        countryTrendConfirmed.push(confirmedLastMax);
        countryTrendRecovered.push(recoveredLastMax);
        countryTrendDeath.push(deadLastMax);

        dateRange.push(day["last_updated"].slice(0, 10));
      });

      this.series = [
        {
          name: "confirmed",
          data: countryTrendConfirmed
        },
        {
          name: "recovered",
          data: countryTrendRecovered
        },
        {
          name: "death",
          data: countryTrendDeath
        }
      ];

      this.options = {
        chart: {
          stacked: this.select.chart === 'bar' ? true : false 
        },
        xaxis: {
          categories: dateRange
        }
      };
    }
  }
};
</script>

<style scoped>
.compared {
  font-size: 13px;
  color: dimgrey;
}
</style>