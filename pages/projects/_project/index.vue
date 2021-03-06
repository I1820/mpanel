<template>
  <v-layout column justify-center align-center>
    <v-flex>
      <h2 class="orange--text display-2">Things @ {{projectID}}</h2>
    </v-flex>
    <v-flex>
      Runner status: {{ project.inspects[0].State.Status }}
    </v-flex>
    <v-flex class="xs12 sm8 md6 pt-4"> <!-- thing listing section -->
      <i-thing v-for="(thing, i) in things" :key="i" :thing="thing" :project="projectID"></i-thing>
    </v-flex>
    <v-flex> <!-- create thing dialog -->
      <v-dialog v-model="dialog" persistent max-width="500px">
        <v-btn slot="activator" flat icon light><v-icon>add</v-icon></v-btn>
        <v-card>
          <v-card-title>
            <span class="headline">Thing</span>
          </v-card-title>
          <v-card-text>
            <v-container grid-list-md>
              <v-layout column>
                <v-flex xs12 sm6 md4>
                  <v-text-field v-model="name" label="*Name" required></v-text-field>
                </v-flex>
              </v-layout>
            </v-container>
            <small>*indicates required field</small>
          </v-card-text>
          <v-card-actions>
            <v-spacer></v-spacer>
            <v-btn color="blue darken-1" flat @click.native="dialog = false">Close</v-btn>
            <v-btn color="blue darken-1" flat @click.native="create">Create</v-btn>
          </v-card-actions>
        </v-card>
      </v-dialog>
    </v-flex>
    <v-flex align-center> <!-- realtime socket section  -->
      <v-btn color="blue darken-1" :disabled="isConnected" flat @click.native="connect">Connect</v-btn>
      <v-btn color="blue darken-1" :disabled="!isConnected" flat @click.native="disconnect">Disconnect</v-btn>
      <v-data-table
        :headers="headers"
        :items="states"
        :loading="isConnected"
        class="elevation-1"
        hide-actions
        >
        <v-progress-linear slot="progress" color="blue" indeterminate></v-progress-linear>
        <template slot="items" slot-scope="props">
          <td>{{ props.item.asset }}</td>
          <td class="text-xs-right">{{ props.item.thing_id }}</td>
          <td class="text-xs-right">{{ props.item.at | moment("DD-MM-YYYY hh:mm:ss") }}</td>
          <td class="text-xs-right">{{ props.item.raw }}</td>
        </template>
      </v-data-table>
    </v-flex>
  </v-layout>
</template>


<script>
import IThing from '~/components/i-thing.vue'
import io from 'socket.io-client'

export default {
  components: {
    'i-thing': IThing
  },

  methods: {
    disconnect () {
      this.socket.disconnect()
    },

    connect () {
      this.socket.open()
    },

    async refresh () {
      try {
        this.things = await this.$axios.$get(`/pm/api/projects/${this.projectID}/things`)
        console.log(this.things)
      } catch (e) {
        console.log(e)
      }
    },

    async create () {
      try {
        await this.$axios.$post(`/pm/api/projects/${this.projectID}/things`, {
          'name': this.name,
          'model': 'default'
        })
        this.$toast.global.iSuccess({message: 'Successfully builded'})
      } catch (e) {
        this.$toast.global.iError({message: `${e.response.data.code}: ${e.response.data.error.split('\n')[0]}`})
      }
      await this.refresh()
      this.dialog = false
    }
  },

  computed: {
    isConnected: function () {
      if (this.socket != null) { // before component is mounted
        return this.socket.connected
      }
      return false
    }
  },

  beforeDestroy () {
    this.socket.disconnect()
  },

  mounted () {
    // create socket.io instance when component is mounted
    this.socket = io('/I1820', { autoConnect: false })

    // register socket.io callbacks
    this.socket.on('connect', () => {
      this.$toast.global.iSuccess({message: 'We are coming with data to you'})
    })
    this.socket.on('connect_error', (error) => {
      this.$toast.global.iError({message: `connection failure: ${error}`})
      this.socket.disconnect()
    })
    this.socket.on(`projects/${this.projectID}`, (message) => {
      this.states.push(message)
    })
  },

  data: () => ({
    socket: null,
    states: [],
    headers: [
      {
        text: 'Asset',
        value: 'asset'
      },
      {
        text: 'ThingID',
        value: 'thing_id'
      }, {
        text: 'At',
        value: 'at'
      }, {
        text: 'Value',
        value: 'raw'
      }
    ],

    dialog: false, // create thing dialog visibility

    name: '' // name field in create thing
  }),

  async asyncData ({app, params}) {
    let project = {}
    let projectID = params.project
    let things = []
    try {
      things = await app.$axios.$get(`/pm/api/projects/${projectID}/things`)
      console.log(things)
    } catch (e) {
      console.log(e)
    }
    try {
      project = await app.$axios.$get(`/pm/api/projects/${projectID}`)
    } catch (e) {
      project = {
        inspects: [
          {
            State: {Status: 'internal error'}
          },
          {
            State: {Status: 'internal error'}
          }
        ]
      }
    }
    return {things, projectID, project} // returns object contains required data
  }
}
</script>
