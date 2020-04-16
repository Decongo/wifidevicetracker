<template>
  <v-app>
    <v-navigation-drawer v-if="authenticated" app permanent>
      <template v-slot:prepend>
        <v-list>
          <v-list-item @click="$router.push('/')" style="cursor: pointer">
            <v-list-item-title>Analytics Dash</v-list-item-title>
          </v-list-item>

          <v-menu offset-y>
            <template v-slot:activator="{ on }">
              <v-list-item two-line v-on="on">
                <v-list-item-content>
                  <v-list-item-title class="title">{{dataStore.currentStore.readableName}}</v-list-item-title>
                  <v-list-item-subtitle>{{dataStore.currentStore.readableSubName}}</v-list-item-subtitle>
                </v-list-item-content>
                <v-list-item-action>
                  <v-icon>mdi-menu-down</v-icon>
                </v-list-item-action>
              </v-list-item>
            </template>
            <v-list>
              <v-list-item
                v-for="(item, index) in dataStore.ownedStores"
                :key="index"
                two-line
                @click="setStore(item)"
              >
                <v-list-item-title>{{ item.readableName }}</v-list-item-title>
                <v-list-item-subtitle>{{item.readableSubName}}</v-list-item-subtitle>
              </v-list-item>
            </v-list>
          </v-menu>
        </v-list>
      </template>

      <v-divider></v-divider>

      <v-list nav dense>
        <v-list-item link to="/daily">
          <v-list-item-icon>
            <v-icon>mdi-calendar-today</v-icon>
          </v-list-item-icon>
          <v-list-item-title>Daily Stats</v-list-item-title>
        </v-list-item>
        <v-list-item link to="/weekly">
          <v-list-item-icon>
            <v-icon>mdi-calendar-week</v-icon>
          </v-list-item-icon>
          <v-list-item-title>Weekly Stats</v-list-item-title>
        </v-list-item>
        <!-- <v-list-item link to="/monthly">
          <v-list-item-icon>
            <v-icon>mdi-calendar-month</v-icon>
          </v-list-item-icon>
          <v-list-item-title>Monthly Stats</v-list-item-title>
        </v-list-item>-->
        <v-list-item link to="/leaderboard">
          <v-list-item-icon>
            <v-icon>mdi-podium-gold</v-icon>
          </v-list-item-icon>
          <v-list-item-title>Leaderboard</v-list-item-title>
        </v-list-item>
        <v-list-item link to="/campaigns">
          <v-list-item-icon>
            <v-icon>mdi-calendar-plus</v-icon>
          </v-list-item-icon>
          <v-list-item-title>Campaigns</v-list-item-title>
        </v-list-item>
        <v-list-item link to="/competitors">
          <v-list-item-icon>
            <v-icon>mdi-podium-gold</v-icon>
          </v-list-item-icon>
          <v-list-item-title>Competitors</v-list-item-title>
        </v-list-item>
        <v-list-item link to="/range">
          <v-list-item-icon>
            <v-icon>mdi-signal-distance-variant</v-icon>
          </v-list-item-icon>
          <v-list-item-title>Range Config</v-list-item-title>
        </v-list-item>
      </v-list>
      <amplify-sign-out align="center"></amplify-sign-out>
    </v-navigation-drawer>

    <v-content>
      <router-view @authenticated="setAuthenticated(true)" />
    </v-content>
  </v-app>
</template>

<script>
import { dataStore } from "./misc/dataStore";
export default {
  name: "App",

  data() {
    return {
      authenticated: false,
      dataStore
    };
  },

  mounted() {
    if (!this.authenticated && this.$router.name != "login") {
      this.$router.push("/login").catch(err => {
        console.log(err);
      });
    }
  },
  methods: {
    setAuthenticated(status) {
      this.authenticated = status;
      this.dataStore.authenticated = status;
    },
    logout() {
      this.authenticated = false;
    },
    setStore(store) {
      dataStore.currentStore = store;
      dataStore.currentStoreName = store.storeName;
    }
  },

  components: {}
};
</script>
