<template>
  <v-layout column justify-center align-center>
    <v-flex xs12 sm8 md6>
      <div class="text-xs-center">
        <img src="/v.png" alt="Vuetify.js" class="mb-5" />
      </div>
      <v-card class="mt-5">
        <v-card-title class="headline">Server Region</v-card-title>
        <v-card-text>
          <v-select :items="regions" v-model="region" label="Region"></v-select>
        </v-card-text>
      </v-card>
      <v-card v-if="listings.length > 0" class="mt-5">
        <v-card-title class="headline">Server Health</v-card-title>
        <v-card-text>
          <v-data-table
            :headers="listingsHeadings"
            :items="listings"
            class="elevation-1"
            :rows-per-page-items="[ 50, 100, 500 ]"
            must-sort
            :pagination.sync="pagination"
          >
            <template slot="items" slot-scope="props">
              <td>{{ props.item.name }}</td>
              <td>{{ msToTimeString(props.item.request) }}</td>
              <td>{{ msToTimeString(props.item.process) }}</td>
              <td>{{ msToTimeString(props.item.datastore) }}</td>
              <td>{{ props.item.found }}</td>
              <td>{{ props.item.lost }}</td>
              <td>{{ msToTimeString(Date.now() - props.item.time) }}</td>
              <td>{{ props.item.error }}</td>
            </template>
          </v-data-table>
        </v-card-text>
      </v-card>
    </v-flex>
  </v-layout>
</template>

<style>
</style>

<script>
  export default {
    computed: {
      regions () {
        return Object.keys(this.$store.state.realmIndex)
        .map(r => this.$store.state.realmIndex[r].regionTag)
        .reduce((a, v) => { if (!a.includes(v)) a.push(v); return a }, [])
      },
      auctionHouseNameLookup () {
        let ahLookup = {}
        Object.keys(this.$store.state.realmIndex).forEach(index => {
          let realm = this.$store.state.realmIndex[index]
          if (typeof ahLookup[realm.ahid] === 'undefined') ahLookup[realm.ahid] = []
          ahLookup[realm.ahid].push(realm.name)
        })
        Object.keys(ahLookup).forEach(index => {
          ahLookup[index] = ahLookup[index].join(', ')
        })
        return ahLookup
      },
      auctionHouseRegionLookup () {
        let ahLookup = {}
        Object.keys(this.$store.state.realmIndex).forEach(index => {
          let realm = this.$store.state.realmIndex[index]
          ahLookup[realm.ahid] = realm.regionTag
        })
        return ahLookup
      },
      listings () {
        let list = []
        this.listingsRaw
        .filter(item => {
          return this.auctionHouseRegionLookup[item.ahid] === this.region
        })
        .forEach(item => {
          let name = this.auctionHouseNameLookup[item.ahid]
          list.push(Object.assign(item, {name}))
        })

        return list
      }
    },
    data () {
      return {
        region: 'US',
        listingsRaw: [],
        pagination: {descending: false, page: 0, rowsPerPage: -1, sortBy: 'sellRate'},
        listingsHeadings: [
          {text: 'Name', value: 'name', width: 250},
          {text: 'Request', value: 'request'},
          {text: 'Process', value: 'process'},
          {text: 'Store', value: 'datastore'},
          {text: 'Found', value: 'found'},
          {text: 'Lost', value: 'lost'},
          {text: 'Time', value: 'time'},
          {text: 'Error', value: 'error'}
        ]
      }
    },
    async asyncData ({ params, $axios, store }) {
      let { data } = await $axios.get(`http://${process.server ? store.state.harvestServer.replace(store.state.publicIp, 'localhost') : store.state.harvestServer}/timing`)
      return {listingsRaw: data}
    },
    methods: {
      msToTimeString (ms) {
        let str = ''
        let hours = Math.floor(ms / (1000*60*60))
        ms = ms % (1000*60*60)
        let minutes = Math.floor(ms / (1000*60))
        ms = ms % (1000*60)
        let seconds = Math.floor(ms / (1000))
        ms = ms % (1000)
        if (hours) return `${hours}h ${minutes}m ${seconds}s`
        if (minutes) return `${minutes}m ${seconds}s`
        if (seconds) return `${seconds}s`
        return `${ms}ms`
      }
    }
  }
</script>
