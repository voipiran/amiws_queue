<template>
  <v-container fluid grid-list-lg v-if="queueSelect">
    <v-toolbar card color="white" prominent>
      <v-toolbar-title>
        <v-tooltip bottom>
          <v-btn fab small class="btn-close-panel" @click.stop="queueSelect = ''" slot="activator">
            <v-icon>exit_to_app</v-icon>
          </v-btn>
          <span>Close</span>
        </v-tooltip>
        {{ queueSelect }}
      </v-toolbar-title>
    </v-toolbar>
    <v-list three-line subheader class="members">
      <v-subheader>
        <v-icon>headset_mic</v-icon>Queue agents ({{ members.length }})
      </v-subheader>
      <template v-for="(member, index) in members">
        <v-list-tile avatar @click=""
          :key="index"
          draggable="true"
          @mouseover="memberHover = member.interface"
          @mouseout="memberHover = ''"
          @dragstart="memberDragStart(member)"
          @dragend="memberDragStop()"
          class="member-card">
          <v-list-tile-avatar>
            <v-icon v-if="member.paused"
              class="orange lighten-2 white--text">phone_paused</v-icon>
            <v-icon v-else-if="member.incall"
              class="green darken-2 white--text">phone_in_talk</v-icon>
            <v-icon v-else-if="member.ringing"
              class="cyan lighten-2 white--text">ring_volume</v-icon>
            <v-icon v-else
              class="grey lighten-1 white--text">contact_phone</v-icon>
          </v-list-tile-avatar>
          <v-list-tile-content>
            <v-list-tile-title>{{ member.name }}</v-list-tile-title>
            <v-list-tile-sub-title class="caption">
              <span>Penalty: {{ member.penalty }}</span>
              <v-tooltip left>
                <span slot="activator">
                  <v-icon class="data-icon">check_box</v-icon>
                  {{ member.callsTaken}} calls
                </span>
                <span>Calls taken</span>
              </v-tooltip>
              <v-tooltip left>
                <span slot="activator">
                  <v-icon class="data-icon">event</v-icon>
                  <span class="last-call-taken">{{ member.lastCall | formatFromUnixtime }}</span>
                </span>
                <span>Last call taken</span>
              </v-tooltip>
            </v-list-tile-sub-title>

            <v-list-tile-sub-title class="caption">
              <v-tooltip left>
                <span slot="activator">
                  <v-icon class="data-icon">alarm</v-icon>
                  {{ member.lastHoldtime | formatTime }}
                </span>
                <span>Last Hold time</span>
              </v-tooltip>
              <v-tooltip left>
                <span slot="activator">
                  <v-icon class="data-icon">alarm_on</v-icon>
                  {{ member.lastTalktime | formatTime }}
                </span>
                <span>Last Talk time</span>
              </v-tooltip>
            </v-list-tile-sub-title>

            <v-list-tile-sub-title v-if="member.incall" class="caption">
              <v-tooltip left>
                <span slot="activator" class="green--text darken-2--text">
                  <v-icon
                    class="data-icon">phone_in_talk</v-icon>
                  <strong>{{ member.incallTime | formatTime }}</strong>
                </span>
                <span>Time on call</span>
              </v-tooltip>
              <v-tooltip left>
                <span slot="activator" class="green--text darken-2--text">
                  <v-icon
                    class="data-icon">account_circle</v-icon>
                  <strong>{{ member.callerName }}
                  &lt;{{ member.callerNum }}&gt;</strong>
                </span>
                <span>Caller name and number</span>
              </v-tooltip>
            </v-list-tile-sub-title>

          </v-list-tile-content>
          <v-list-tile-action v-show="memberHover === member.interface">
            <v-tooltip top>
              <v-btn slot="activator" class="btn-agent-remove"
                icon ripple top @click="removeAgent(member)">
                <v-icon color="red lighten-1">delete_forever</v-icon>
              </v-btn>
              <span>Remove agent from the queue</span>
            </v-tooltip>
            <v-tooltip top>
              <v-btn slot="activator" icon ripple class="btn-agent-toggle" @click="pauseAgentToggle(member)">
                <v-icon color="grey lighten-1">
                  {{ member.paused ? 'play_circle_outline' : 'pause_circle_outline' }}
                </v-icon>
              </v-btn>
              <span>{{ member.paused ? 'Activate' : 'Pause'}} agent in queue</span>
            </v-tooltip>
          </v-list-tile-action>
        </v-list-tile>
        <v-divider></v-divider>
      </template>
    </v-list>

    <v-list two-line class="callers">
      <v-subheader>
        <v-icon>account_circle</v-icon>
        Queue callers ({{ callers.length }})
      </v-subheader>
      <v-list-tile avatar v-for="(caller, i) in callers" :key="i"
        @click="" class="caller-card">
        <v-list-tile-avatar>
          <v-icon :class="[caller.incall ? 'green darken-2':'grey lighten-1', 'white--text']">
            {{ caller.incall ? 'phone_in_talk' : 'phone_forwarded' }}
          </v-icon>
        </v-list-tile-avatar>
        <v-list-tile-content>
          <v-list-tile-title>{{ caller.clidName }} &lt;{{ caller.clidNum }}&gt;</v-list-tile-title>
          <v-list-tile-sub-title class="caption">
            <v-tooltip v-if="caller.incall" left>
              <span slot="activator" class="green--text darken-2--text">
                <v-icon class="data-icon">phone_in_talk</v-icon>
                <strong>{{ caller.answerTime | formatTime }}</strong>
              </span>
              <span>On call time</span>
            </v-tooltip>
            <v-tooltip left>
              <span slot="activator">
                <v-icon class="data-icon">hourglass_full</v-icon>
                {{ caller.wait | formatTime }}
              </span>
              <span>Wait in queue time</span>
            </v-tooltip>
          </v-list-tile-sub-title>
          <v-list-tile-sub-title class="caption">
            <span>
              <v-icon class="data-icon">watch_later</v-icon>
              Position in queue: {{ caller.position }}
            </span>
          </v-list-tile-sub-title>
        </v-list-tile-content>
      </v-list-tile>
    </v-list>
    <v-snackbar v-model="notify" :timeout="3000" color="info" top right>
      <v-icon color="white">info_outline</v-icon>
      {{ notifyText }}
    </v-snackbar>

    <v-dialog v-model="confirmDlg" max-width="300">
      <v-card>
        <v-card-text class="modal-body">{{ dlgBody }}</v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn @click.native="confirmDlg = false">Cancel</v-btn>
          <v-btn class="btn-confirm-remove" @click.native.stop="doRemove()" dark color="green darken-1">Yes</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

  </v-container>
</template>

<script>
import { mapGetters, mapActions } from 'vuex'
export default {
  name: 'QueueData',
  data () {
    return {
      memberHover: '',
      notify: false,
      notifyText: '',
      confirmDlg: false,
      dlgBody: '',
      memberToRemove: ''
    }
  },
  computed: {
    ...mapGetters({
      queues: 'getAllQueues',
      members: 'getSelectedMembers',
      callers: 'getSelectedCallers',
      getSelectedQueue: 'getSelectedQueue'
    }),
    queueSelect: {
      get: function () {
        return this.getSelectedQueue
      },
      set: function (val) {
        this.selectedQueue(val)
      }
    }
  },
  methods: {
    ...mapActions(['pauseAgentInSelectedQueue', 'removeAgentFromQueue', 'selectedQueue', 'memberDragStart', 'memberDragStop']),
    pauseAgentToggle: function (member) {
      this.pauseAgentInSelectedQueue({ member: member, pause: !member.paused })
      this.notifyText = `${member.paused ? 'Activate' : 'Pause'} agent ${member.name}`
      this.notify = true
    },
    removeAgent: function (member) {
      this.dlgBody = `Remove agent ${member.name}?`
      this.memberToRemove = member.interface
      this.confirmDlg = true
    },
    doRemove: function () {
      this.removeAgentFromQueue({ iface: this.memberToRemove, qname: this.getSelectedQueue })
      this.notifyText = `Removing agent ${this.memberToRemove} from queue ${this.getSelectedQueue}`
      this.notify = true
      this.confirmDlg = false
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
.data-icon {font-size: 16px}
</style>
