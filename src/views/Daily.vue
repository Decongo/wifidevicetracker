<template>
  <v-container>
    <v-row class="mb-4">
      <v-col>
        <h1 class="display-2 font-weight-bold mb-3">
          Daily Visitor Stats
        </h1>
        <p class="subheading font-weight-regular">
          Choose a query to get started
        </p>
        <v-spacer></v-spacer>
        <v-select
          v-model="selectedQuery"
          :items="queries"
          item-text="name"
          item-value="query"
          return-object
          label="Select a query"
          solo
          @change="getChartData()"
        ></v-select>
        <v-btn @click="exportToCsv()" bottom>Export</v-btn>
      </v-col>
      <v-col>
        <v-date-picker v-model="date" @input="getChartData()" landscape></v-date-picker>
      </v-col>
    </v-row>
    <v-card v-if="dataTable.length > 0" class="pa-5">
      <GChart
        :settings="{packages: ['bar']}"    
        :data="dataTable"
        :options="chartOptions"
        :createChart="(el, google) => new google.charts.Bar(el)"
        @ready="onChartReady"
        width="90%"
      />
    </v-card>
    <div v-else>
      No results to show
    </div>
  </v-container>
</template>

<script>
import exportService from '@/misc/exportService.js'
export default {
  name: 'UniqueDailyCmp',
  components: {
    
  },
  data: () => ({
    dataTable: [],
    chartsLib: null,
    date: "",
    storeName: "store_name_1",
    query: "unique_per_hour",
    selectedQuery: "",
    queries: [
      {
        name: "Unique per hour",
        query: "unique_per_hour",
        chartTitle: "Number of Unique Visitors Per Hour"
      },
      {
        name: "Daily total",
        query: "daily_total_unique",
        chartTitle: "Total Visitors Per Day"
      },
    ]
  }),
  computed: {
    chartOptions () {
      if (!this.chartsLib) return null
      return this.chartsLib.charts.Bar.convertOptions({
        chart: {
          title: this.selectedQuery.chartTitle,
          subtitle: this.date
        },
        hAxis: { format: 'decimal' },
        colors: ['#1b9e77', '#d95f02', '#7570b3']
      })
    }
  },
  created() {
    this.setCurrentDate()
    this.getChartData()
  },
  methods: {
    onChartReady (chart, google) {
      this.chartsLib = google
    },
    setCurrentDate () {
      var today = new Date()
      var date = today.getFullYear()+'-0'+(today.getMonth()+1)+'-'+today.getDate()
      this.date = date
    },
    getChartData() {
      let base = "https://s3.us-east-2.amazonaws.com/jolt.capstone/"
      //let pretest = "athena-query-logs/store_name_1/unique_per_hour/2020-01-07"
      let prefix = "athena-query-logs/" + this.storeName + "/" + this.selectedQuery.query + "/" + this.date + "/"
      this.axios.get(base+"?prefix="+prefix)
        .then((response) => {
          var parser = new DOMParser();
          let data = parser.parseFromString(response.data, "text/xml");
          let key = data.getElementsByTagName("Key")[0].innerHTML
          return key
        })
        .then((key) => {
          this.$papa.parse(base+key, {
          download: true,
          dynamicTyping: true,
          complete: (results) => {
            results.data.pop()
            this.dataTable = results.data
          }
      })
        })
        .catch((error) => {
          this.dataTable = []
      })
    },
    exportToCsv() {
      exportService.exportToCSV(this.storeName + "_" + this.query + "_" + this.date, this.dataTable)
    }
  }
};
</script>