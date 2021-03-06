<template>
  <v-container grid-list-md text-xs-center>
    <v-layout row wrap>
      <v-flex xs12>
        <v-card>
          <v-card-title primary class="subheading text-sm-left">Walk Request</v-card-title>
        </v-card>
      </v-flex>

      <v-flex md6 xs12>
        <v-card>
          <v-card-title>
            <div class="text-xs-center" style="margin: auto">
              <v-avatar size="50">
                <img :src="owner.photo" alt="avatar" />
              </v-avatar>
              <div style="margin-top: 5px; margin-bottom: -25px;">
                Owner
                <span v-if="isOwner">(You)</span>
                <span v-else>(them)</span>
              </div>
            </div>
          </v-card-title>
          <v-card-text>
            <div class="core-text-inner">
              <div>Name: {{ owner.name }}</div>
              <div>Email: {{ owner.email }}</div>
              <div>Age: {{ owner.age }}</div>
            </div>
          </v-card-text>
        </v-card>
      </v-flex>
      <v-flex md6 xs12>
        <v-card>
          <v-card-title>
            <div class="text-xs-center" style="margin: auto">
              <v-avatar size="50">
                <img :src="walker.photo" alt="avatar" />
              </v-avatar>
              <div style="margin-top: 5px; margin-bottom: -25px;">
                Walker
                <span v-if="!isOwner">(You)</span>
                <span v-else>(them)</span>
              </div>
            </div>
          </v-card-title>
          <v-card-text>
            <div class="core-text-inner">
              <div>Name: {{ walker.name }}</div>
              <div>Email: {{ walker.email }}</div>
              <div>Age: {{ walker.age }}</div>
            </div>
          </v-card-text>
        </v-card>
      </v-flex>

      <v-flex xs12>
        <v-card grid-list-md text-xs-center>
          <v-layout row wrap>
            <v-flex xs4 style="margin: auto">
              <div>
                Status:
                <span :style="{ color: getStatusColor() }">{{ getStatus() }}</span>
              </div>
            </v-flex>
            <v-flex xs4>
              <ActionWithNotes
                :disabled="isOwner"
                v-if="walk.status != null && walk.status === walkStats.PENDING"
                title="Accepting"
                button-text="Accept"
                :submit="acceptWalk"
              />
              <ActionWithNotes
                v-if="walk.status != null && walk.status === walkStats.ACTIVE"
                title="Completing"
                button-text="Complete"
                :disabled="!isOwner"
                :submit="completeWalk"
              >
                <v-flex>
                  <v-card-text
                    >How well do you rate {{ this.walker.name || this.walker.email }} (walker)?</v-card-text
                  >
                  <v-rating
                    v-model="completeRating"
                    background-color="grey darken-1"
                    hover
                    half-increments
                  ></v-rating>
                </v-flex>
              </ActionWithNotes>
            </v-flex>
            <v-flex xs4>
              <ActionWithNotes
                :disabled="isOwner"
                v-if="walk.status != null && !isOwner && walk.status === walkStats.PENDING"
                title="Rejecting"
                button-text="Reject"
                :submit="rejectWalk"
              />
              <ActionWithNotes
                v-if="
                  (walk.status != null && isOwner && walk.status === walkStats.PENDING) ||
                    walk.status === walkStats.ACTIVE
                "
                title="Cancelling"
                button-text="Cancel"
                :submit="cancelWalk"
              />
            </v-flex>
          </v-layout>
        </v-card>
      </v-flex>
      <v-flex xs12>
        <v-card>
          <GmapMap
            ref="mapRef"
            :center="this.walkLocation"
            :zoom="14"
            :options="googleMapsSettings"
            class="maps-style"
          >
            <GmapMarker ref="mapMarkerRef" :position="this.walk.location" />
          </GmapMap>
        </v-card>
      </v-flex>
      <v-flex md6 xs12>
        <v-card elevation="0" style="background: none">
          <v-card-title>Notes</v-card-title>
          <v-card-text dense>
            <ul class="text-xs-left">
              <li color="gray lighten-3" :key="index" v-for="(item, index) in walk.notes">{{ item }}</li>
            </ul>
          </v-card-text>
        </v-card>
      </v-flex>
      <v-flex md6 xs12>
        <v-card elevation="0" style="background: none">
          <v-card-title>History</v-card-title>
          <v-card-text>
            <v-timeline dense>
              <v-timeline-item
                color="primary lighten-3"
                small
                v-for="(item, index) in walk.history"
                :key="index"
              >
                <v-layout>
                  <v-flex class="text-sm-left">{{ item }}</v-flex>
                </v-layout>
              </v-timeline-item>
            </v-timeline>
          </v-card-text>
        </v-card>
      </v-flex>
      <DogsGrid :profile="this.owner" :dogs="this.dogs" :owner-id="this.walk.owner" />
    </v-layout>
  </v-container>
</template>

<script>
import * as _ from 'lodash';

import * as firebaseConstants from '../constants/firebaseConstants.js';
import ActionWithNotes from '@/components/ActionWithNotes.vue';
import firebaseWrapper from '../lib/firebaseWrapper.js';
import DogsGrid from '@/components/DogsGrid.vue';

export default {
  data: function() {
    return {
      // the walker of the dogs.
      walker: {},
      // the owner of the dogs.
      owner: {},
      // the walk object.
      walk: {},
      // the dogs on the walk.
      dogs: {},
      // the walk location, this will be updated once we have actually got the walk location.
      walkLocation: { lat: 1, lng: 1 },
      // the local walk id, used / passed through the url path, this includes the use of gathering
      // walk related data and the people on the walks.
      localWalkId: '',
      // if the user is the current owner, this will be used for displaying the related accepting,
      // rejceting or completing the walk. Including the displaying of related information.
      isOwner: false,
      // Cleans up the google maps controls to better allow for selecting a location for the user.
      googleMapsSettings: {
        zoomControl: true,
        mapTypeControl: false,
        scaleControl: true,
        streetViewControl: false,
        rotateControl: true,
        fullscreenControl: false,
        disableDefaultUi: true
      },
      walkStats: firebaseConstants.WALK_STATUS,
      completeRating: 0
    };
  },

  /**
   * When the single walk is first created, we will just have a single id related to the walk, with
   * this walk id will result in gathering all the related information about a given walk and two
   * ids, these ids will be used to gather the profiles for the given walker and owner. Both
   * required to display the related information.
   *
   * Depending on what kind of person the current viewer is (walker, owner), will depend on how we
   * display and adjust the page.
   */
  created: async function() {
    await this.initalizePage();
  },

  watch: {
    // call again the method if the route changes
    $route: 'initalizePage'
  },

  methods: {
    initalizePage: async function() {
      // the local walk id, this is the id that is used to reference the id, this will be used to
      // gather the walk and the people related to the walk (e.g walker and owner).
      this.localWalkId = this.$route.params.id || '';

      // gather the related walker and owners profile including the walker.
      this.walk = await firebaseWrapper.getWalkByKey(this.localWalkId);
      this.owner = await firebaseWrapper.getProfile(this.walk.owner);
      this.walker = await firebaseWrapper.getProfile(this.walk.walker);

      // determine if the current authenticated user is the walker or the owner so we can do simple
      // little displays indercating who is who.
      this.isOwner = firebaseWrapper.getUid() === this.walk.owner;
      this.walkLocation = this.walk.location;

      // lets revserse the history logs so that the related history object is in order when displaying
      // it for the given user. This will correctly format it for a timeline.
      this.walk.history = _.values(this.walk.history).reverse();
      this.walk.notes = _.values(this.walk.notes).reverse();

      // lets build up the dogs object for settings.
      const dogs = {};

      // iterate and perform the requests for gathering the dogs, for of to allow the syncing of the
      // request process. Then set the dogs onto the object.
      for (const dog of this.walk.dogs) {
        dogs[dog] = await firebaseWrapper.getSingleDog(this.walk.owner, dog);
      }

      // update the current set walks.
      this.dogs = dogs;

      // setup all related listeners, including the walk (owner and walker).
      await this.setupWalkListener();
    },

    // if changes occure on the current walk, we want to update the whole object when the changes
    // happen. Keeping all related information live. This will also setup listeners for the owner
    // and the walker, just incase the profiles change at anypoint.
    setupWalkListener: async function() {
      const walkerReference = firebaseWrapper.getProfileReference(this.walk.walker);
      const ownerReference = firebaseWrapper.getProfileReference(this.walk.owner);
      const walkReference = firebaseWrapper.getWalkReferenceByKey(this.localWalkId);

      walkReference.on('value', (snapshot) => {
        this.walk = snapshot.val();
        this.walk.history = _.values(this.walk.history).reverse();
        this.walk.notes = _.values(this.walk.notes).reverse();
      });

      ownerReference.on('value', (snapshot) => (this.owner = snapshot.val()));
      walkerReference.on('value', (snapshot) => (this.walker = snapshot.val()));
    },
    // gets the display related text for a given walk message, this is what the user is going to see
    // as the text for a given status. This has to be clean and easy to read.
    getStatus: function() {
      switch (this.walk.status) {
        case firebaseConstants.WALK_STATUS.ACTIVE:
          return 'Active';
        case firebaseConstants.WALK_STATUS.PENDING:
          return 'Pending';
        case firebaseConstants.WALK_STATUS.CANCELLED:
          return 'Cancelled';
        case firebaseConstants.WALK_STATUS.COMPLETE:
          return 'Complete';
        case firebaseConstants.WALK_STATUS.REJECTED:
          return 'Rejected';
        default:
          return 'N/A';
      }
    },
    // gets the display related text color for a given walk message, this is what the user is going to see
    // as the text for a given status. This has to be clean and easy to read.
    getStatusColor: function() {
      switch (this.walk.status) {
        case firebaseConstants.WALK_STATUS.ACTIVE:
        case firebaseConstants.WALK_STATUS.COMPLETE:
          return 'green';
        case firebaseConstants.WALK_STATUS.PENDING:
        case firebaseConstants.WALK_STATUS.CANCELLED:
          return 'orange';
        case firebaseConstants.WALK_STATUS.REJECTED:
          return 'red';
        default:
          return 'black';
      }
    },
    // sets and marks the walk as cancelled, no action can take place now since this has gone
    // through, the other party will be notified about these actions.
    cancelWalk: async function(notes) {
      return this.walkAction(firebaseWrapper.cancelWalkRequest.bind(firebaseWrapper), notes);
    },

    // sets and marks the walk as accepted.
    acceptWalk: async function(notes) {
      return this.walkAction(firebaseWrapper.acceptWalkRequest.bind(firebaseWrapper), notes);
    },

    // sets and marks the walk as rejected.
    rejectWalk: async function(notes) {
      return this.walkAction(firebaseWrapper.rejectWalkRequest.bind(firebaseWrapper), notes);
    },

    // sets and marks the walk as complete.
    completeWalk: async function(notes) {
      return this.walkAction(
        firebaseWrapper.completeWalkRequest.bind(firebaseWrapper),
        notes,
        this.completeRating
      );
    },

    // standard walk request action format, all methods have the same standard param input so this
    // can be used instead of repeating lots of codes.
    walkAction: async function(asyncMethod, notes, ...params) {
      const localUserId = this.isOwner ? this.owner.id : this.walker.id;
      await asyncMethod(localUserId, this.localWalkId, notes === '' ? null : notes, ...params);
      return true;
    }
  },

  components: {
    DogsGrid,
    ActionWithNotes
  }
};
</script>

<style scoped>
.point {
  background: none;
}
.maps-style {
  max-width: 100%;
  min-width: 100%;
  min-height: 250px;
}
</style>
