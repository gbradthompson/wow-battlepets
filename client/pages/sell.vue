<template>
  <v-layout column justify-center align-center>
    <v-flex xs12 sm8 md6>
      <div class="text-xs-center">
        <img src="/v.png" alt="Vuetify.js" class="mb-5" />
      </div>
      <v-card>
        <v-card-title class="headline">Sell your collection</v-card-title>
        <v-card-text>
          <h3>Main Character</h3>
          <v-select :items="regions" v-model="region" label="Region"></v-select>
          <v-autocomplete :items="realmListID" v-model="realm" label="Realm"></v-autocomplete>
          <v-text-field label="Character Name" v-model="character"></v-text-field>
          <h3>Sell Realms</h3>
          <v-autocomplete v-for="(item, index) in sellRealms" :items="realmListAHID" v-model="item.ahid" label="Realm">
            <v-icon slot="append-outer" @click="removeSellRealm(index)">cancel</v-icon>
          </v-autocomplete>
          <v-btn color="success" @click="addSellRealm" small fab><v-icon>add</v-icon></v-btn>
          <v-btn color="primary" @click="showIngest = !showIngest" small fab><v-icon>view_agenda</v-icon></v-btn>
          <div v-if="showIngest">
            <v-textarea
            outline
            label="New line separated list"
            v-model="ingestRealmsText"
            ></v-textarea>
            <v-btn color="primary" @click="ingestRealms"><v-icon>send</v-icon></v-btn>
          </div>
        </v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="primary" @click="requestData">Continue</v-btn>
        </v-card-actions>
      </v-card>
      <v-card class="mt-5">
        <v-card-text>
          {{ state }}
        </v-card-text>
      </v-card>
      <v-card v-if="listings.length > 0" class="mt-5">
        <v-card-title class="headline">Sell collection </v-card-title>
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
              <td><img :src="props.item.image" :class="'quality' + props.item.quality"></td>
              <td>{{ props.item.name }}</td>
              <td>{{ props.item.level }}</td>
              <td><display-gold :value="props.item.price"></display-gold></td>
              <td style="cursor: pointer" @click="props.item.sellIndex = (props.item.sellIndex + 1) % props.item.sellAt.length">{{ props.item.ahname }}</td>
              <td :class="'undercut_' + props.item.undercut">{{ props.item.undercut }}</td>
              <td><display-gold :value="props.item.competition"></display-gold></td>
              <td><v-checkbox v-model="props.item.selected" primary hide-details></v-checkbox></td>
              <td><v-checkbox v-model="props.item.match" primary hide-details></v-checkbox></td>
            </template>
          </v-data-table>
        </v-card-text>
      </v-card>
      <v-card v-if="listings.length > 0" class="mt-5">
        <v-card-title class="headline">Settings</v-card-title>
        <v-card-text>
          <div>
            <v-btn color="success" @click="matchAll">Match All</v-btn>
          </div>
          <v-radio-group label="Auction Length" row v-model="auctionLength">
            <v-radio :key="1" label="12H" :value="1"></v-radio>
            <v-radio :key="2" label="24H" :value="2"></v-radio>
            <v-radio :key="3" label="48H" :value="3"></v-radio>
          </v-radio-group>
          <v-radio-group label="Auto Match/Sell" row v-model="autoMatch">
            <v-radio :key="1" label="100%" :value="1"></v-radio>
            <v-radio :key="2" label="90%" :value="0.9"></v-radio>
            <v-radio :key="3" label="80%" :value="0.8"></v-radio>
            <v-radio :key="4" label="70%" :value="0.7"></v-radio>
            <v-radio :key="5" label="60%" :value="0.6"></v-radio>
            <v-radio :key="6" label="50%" :value="0.5"></v-radio>
            <v-radio :key="7" label="40%" :value="0.4"></v-radio>
          </v-radio-group>
          <v-text-field label="Discount (1-0)" v-model="discount"></v-text-field>
        </v-card-text>
      </v-card>
      <v-card v-if="listings.length > 0" class="mt-4"  v-for="ah in sellListings">
        <v-card-title class="headline">{{ah[0].ahname}}</v-card-title>
        <v-card-text>
          Total: <display-gold :value="ah.reduce((a, v) => { a += v.price; return a;}, 0)"></display-gold>
          <br>
          Num: {{ah.length}}
          <br>
          <br>
          <hr>
          <br>
          <code><span v-for="item in ah">{{`C_PetJournal.CagePetByID('BattlePet-0-${item.guid}')|PickupContainerItem(0, 1) ClickAuctionSellItemButton() ClearCursor() PostAuction(${item.price}, ${item.price}, ${auctionLength}) `}}</span></code>
        </v-card-text>
      </v-card>
    </v-flex>
  </v-layout>
</template>

<style>
.undercut_true {
  color: #00FF00
}
.undercut_false {
  color: #FF0000
}
.undercut_match {
  color: cyan
}
</style>

<script>
  var socket = null

  export default {
    mounted () {
      socket = new WebSocket(`ws://${this.$store.state.liveServer}`)
      socket.json = (obj) => { socket.send(JSON.stringify(obj)) }
      socket.addEventListener('message', (event) => {
        let obj = JSON.parse(event.data)
        if (obj.m === 'state') {
          this.state = obj.d
        } else if (obj.m === 'response') {
          this.listingsRaw = obj.d
        }
      });
    },
    beforeDestroy () {
      if (socket && socket.close) socket.close()
    },
    computed: {
      realmIndex () { return this.$store.state.realmIndex },
      petIndex () { return this.$store.state.petIndex },
      regions () {
        return Object.keys(this.$store.state.realmIndex)
        .map(r => this.$store.state.realmIndex[r].regionTag)
        .reduce((a, v) => { if (!a.includes(v)) a.push(v); return a }, [])
      },
      realmListID () {
        return Object.keys(this.$store.state.realmIndex)
        .map(r => { return {text: this.$store.state.realmIndex[r].name, value: this.$store.state.realmIndex[r].id, region: this.$store.state.realmIndex[r].regionTag} })
        .filter(r => { return r.region === this.region })
        .sort((a, b) => { return a.text > b.text ? 1:-1 })
      },
      realmListAHID () {
        return Object.keys(this.$store.state.realmIndex)
        .map(r => { return {text: this.$store.state.realmIndex[r].name, value: this.$store.state.realmIndex[r].ahid, region: this.$store.state.realmIndex[r].regionTag} })
        .filter(r => { return r.region === this.region })
        .sort((a, b) => { return a.text > b.text ? 1:-1 })
      },
      auctionHouseNameLookup () {
        let ahLookup = {}
        Object.keys(this.$store.state.realmIndex).forEach(index => {
          let realm = this.$store.state.realmIndex[index]
          if (typeof ahLookup[realm.ahid] === 'undefined') ahLookup[realm.ahid] = []
          ahLookup[realm.ahid].push(realm.name)
        })
        Object.keys(ahLookup).forEach(index => {
          ahLookup[index].sort()
          ahLookup[index] = ahLookup[index].join(', ')
        })
        return ahLookup
      },
      sellRealmsString () { return this.sellRealms.map(r => r.ahid).filter(r => r !== '').join('-')},
      listings () {
        let list = []
        this.listingsRaw
        .filter(item => { return typeof this.petIndex[item.psid] !== 'undefined'})
        .forEach(item => {
          let petInfo = this.petIndex[item.psid]
          item.sellAt.forEach(sa => {
            sa.ahname = this.auctionHouseNameLookup[sa.ahid]
          })

          let defaultPrice = Math.floor((item.sellAt[item.sellIndex].price * parseFloat(this.discount))/10000)*10000
          let matchPrice = Math.floor((Math.floor(item.sellAt[item.sellIndex].competition * 0.999))/10000)*10000
          if (defaultPrice > matchPrice && matchPrice > 1) defaultPrice = matchPrice + 10

          let price = item.match ? matchPrice + 5 : defaultPrice
          let undercut = item.match ? 'match' : item.sellAt[item.sellIndex].competition > price
          list.push(Object.assign(item, {name: petInfo.name, image: petInfo.image}, item.sellAt[item.sellIndex], {price, undercut}))
        })
        return list
      },
      sellListings () {
        let sl = {}
        this.listings
        .filter(item => item.selected)
        .forEach(item => {
          if (typeof sl[item.ahid] === 'undefined') sl[item.ahid] = []
          sl[item.ahid].push(item)
        })
        return Object.keys(sl).map(index => sl[index]).sort((a,b) => {return b.length - a.length})
      }
    },
    data () {
      return {
        ingestRealmsText: '',
        showIngest: false,
        state: '',
        discount: 1,
        autoMatch: 1,
        auctionLength: 3,
        region: 'US',
        realm: 106,
        character: 'Napri',
        sellRealms: [],
        listingsRaw: [],
        pagination: {descending: false, page: 0, rowsPerPage: -1, sortBy: 'name'},
        listingsHeadings: [
          {text: 'Image', value: 'image'},
          {text: 'Name', value: 'name'},
          {text: 'Level', value: 'level'},
          {text: 'Price', value: 'price'},
          {text: 'Realm', value: 'ahname'},
          {text: 'Can Undercut', value: 'undercut'},
          {text: 'Competition', value: 'competition'},
          {text: 'Sell', value: 'selected'},
          {text: 'Match', value: 'match'},
        ]
      }
    },
    watch: {
      autoMatch (val) {
        this.listingsRaw.forEach(item => {
          if (item.sellAt[item.sellIndex].competition <= item.sellAt[item.sellIndex].price) {
            if (item.sellAt[item.sellIndex].competition > item.sellAt[item.sellIndex].price * val){
              item.match = true
              item.selected = true
            } else {
              item.match = false
              item.selected = false
            }
            if (item.sellAt[item.sellIndex].competition === 0) item.selected = true
          }
        })
      }
    },
    methods: {
      async requestData (event) {
        this.listingsRaw = []
        socket.json({m: 'sell', rid: this.realm, name: this.character, sellat: this.sellRealmsString})
      },
      matchAll (event) {
        this.listingsRaw.forEach(item => {
          item.match = true
          item.selected = true
          if (item.sellAt[item.sellIndex].competition === 0) item.match = false
        })
      },
      addSellRealm (event) {
        this.sellRealms.push({ahid: ''})
      },
      removeSellRealm (index) {
        this.sellRealms.splice(index, 1)
      },
      ingestRealms () {
        let realms = this.ingestRealmsText.replace(/\n/gi, '-').split('-')
        realms.forEach(realm => {
          for (var index in this.realmListAHID) {
            if (this.realmListAHID[index].text === realm && this.realmListAHID[index].region === this.region) {
              this.sellRealms.push({ahid: this.realmListAHID[index].value})
            }
          }
        })
        this.ingestRealmsText = ''
      }
    }
  }
</script>
