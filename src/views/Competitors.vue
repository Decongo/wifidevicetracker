<template>
  <v-container>
    <v-row class="mb-4">
      <v-col>
        <h1 class="display-2 font-weight-bold mb-3">
          Compare Store Stats
        </h1>
        <p class="subheading font-weight-regular">
          Choose a store to compare with
        </p>
        <v-spacer></v-spacer>
        <v-select
          v-model="selectedStore"
          :items="stores"
          item-text="displayName"
          item-value="storeName"
          return-object
          label="Select a store"
          solo
          @change="getStoreData()"
        ></v-select>

        <div v-if="!selectedQuery.chart">
          <v-card class="pa-5">
            <v-card-title>{{selectedQuery.chartTitle}}</v-card-title>
            <v-card-subtitle>Week ending on: {{date}}</v-card-subtitle>
            <h1 class="pl-5">{{crossover}}</h1>
            <v-card-text>
              {{summary}}
            </v-card-text>
            <div v-if="percentage">
              <v-card-title>{{percentage}} of total visitors.</v-card-title>
            </div>
          </v-card>
        </div>
      </v-col>
      <v-col>
        <v-tabs
          v-model="tab"
          centered
          fixed-tabs
          @change="getStoreData()"
          >
          <v-tab>Daily</v-tab>
          <v-tab>Weekly</v-tab>
          <v-tab>All time</v-tab>
        </v-tabs>
        <v-date-picker 
          v-if="tab!==2" 
          v-model="date" 
          :max="today" 
          @input="getStoreData()" 
          landscape
        ></v-date-picker>
      </v-col>
    </v-row>
    <v-card v-if="mode==='daily'" class="pa-5">
      <GChart
        :settings="{packages: ['bar']}"    
        :data="dataTable"
        :options="chartOptions"
        :createChart="(el, google) => new google.charts.Bar(el)"
        @ready="onChartReady"
        width="90%"
      />
    </v-card>
  </v-container>
</template>

<script>
import {dataStore} from '../misc/dataStore'

export default {
  name: 'Competitors',

  data() {
    return {
      dataStore,
      dataTable: [],
      chartsLib: null,
      selectedStore: {
        storeName: "all", 
        readableName: "All Stores", 
        readableSubName: "",
        displayName: "All Stores"
      },
      tab: 0,
      date: "",
      today: "",
      crossover: 0,
      totalDistinct: 0,
    }
  },

  computed: {
    chartOptions () {
      if (!this.chartsLib) return null
      return this.chartsLib.charts.Bar.convertOptions({
        chart: {
          title: this.selectedQuery.chartTitle,
          subtitle: this.date
        },
        height: 300
      })
    },
    stores() {
      let stores = this.dataStore.ownedStores.filter(store => {
        return store.storeName != this.dataStore.currentStoreName
      });
      stores.push({
        storeName: "all", 
        readableName: "All Stores", 
        readableSubName: "",
        displayName: "All Stores"
      })

      return stores;
    },
    mode() {
      const modes = ["daily", "weekly", "to-date"];
      return modes[this.tab];
    },
    selectedQuery() {
      let query = {};
      if (this.selectedStore.storeName === 'all') {
        query.name = `Crossover with all other stores`;
      } else {
        query.name = `Crossover with ${this.selectedStore.displayName}`;
      }
      query.query = `with_${this.selectedStore.storeName}`;
      query.chartTitle = `Crossover with ${this.selectedStore.displayName}`;
      query.chart = false;
      return query;
    },
    percentage() {
      if (this.totalDistinct) 
        return ((this.crossover / this.totalDistinct) * 100).toPrecision(2) + "%";
      else return null;
    },

    summary() {
      let summary = "";
      if (this.crossover != null) {
        summary += `${this.crossover} ${this.dataStore.currentStore.displayName} customers also visited `;
      }
      if (this.selectedStore.storeName === "all") {
        summary += "other stores";
      } else {
        summary += `${this.selectedStore.displayName}`;
      }
      summary += ".";
      return summary;
    }
  },

  created() {
    this.setCurrentDate()
  },

  methods: {
    onChartReady (chart, google) {
      this.chartsLib = google
    },
    setCurrentDate () {
      const today = new Date()
      const year = today.getFullYear();
      let month = today.getMonth()+1;
      if (month < 10) month = `0${month}`;
      let day = today.getDate();
      if (day < 10) day = `0${day}`;

      let date = `${year}-${month}-${day}`;

      this.date = date
      this.today = date
    },
    async getStoreData() {
      if (this.mode === 'daily') {
        this.populateDataTable();
      }

      const basePrefix = `athena-query-logs/${this.dataStore.currentStoreName}`;

      const crossoverPrefix = `${basePrefix}/crossover/with_${this.selectedStore.storeName}/${this.mode}/${this.date}/`;
      this.fetchData(crossoverPrefix, "crossover");

      let distinctPrefix = "";
      if (this.mode === "to_date") {
        distinctPrefix = `${basePrefix}/crossover/total_distinct_visitors/${this.date}/`
      } else if (this.mode === "weekly") {
        distinctPrefix = `${basePrefix}/weekly_total_unique/${this.date}/`
      }
      if (distinctPrefix) this.fetchData(distinctPrefix, "totalDistinct")
    },
    fetchData(prefix, resultLocation) {
      const base = "https://s3.us-east-2.amazonaws.com/jolt.capstone/";
      
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
              // if (this.selectedQuery.query === "daily_avg_duration") {
              //   results.data[1][0] = results.data[1][0].toFixed(1)
              // }
              // this.dataTable = results.data
              this[resultLocation] = results.data[1][0]
            }
          })
          .catch((error) => {
              this.dataTable = [];
          })
        })
        .catch((error) => {
          this.dataTable = [];
        })
    },

    populateDataTable() {
      this.dataTable = [["string", "Element"]];
      for (let i = 2; i < 9; ++i) {
        let prefix = `athena-query-logs/heritage_15/crossover/with_heritage_16/daily/2020-04-0${i}/`

        const base = "https://s3.us-east-2.amazonaws.com/jolt.capstone/";
      
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
              this.dataTable.push([`2020-04-0${i}`, results.data[1][0]]);
            }
          })
          .catch((error) => {
              // this.dataTable = [];
          })
        })
        .catch((error) => {
          // this.dataTable = [];
        })
      }
    }
  }
}
</script>