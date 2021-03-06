<template>
  <v-container grid-list-md text-xs-center>
    <v-layout row wrap>
      <v-flex xs12 sm6 md6 class="box-spacing">
        <GenericPanel top-text="Welcome to Dogber" bottom-text="Pending walk requests">{{
          getPendingWalkCount()
        }}</GenericPanel>
      </v-flex>

      <v-flex xs12 sm6 md6 class="box-spacing">
        <GenericPanel top-text="Activities" bottom-text="Confirmed/Future Walks">{{
          getConfirmedWalks()
        }}</GenericPanel>
      </v-flex>

      <v-flex xs12 sm6 md3 class="box-spacing">
        <GenericPanel bottom-text="Miles Walked" top-text-color="blue" :top-text="milesWalked">
          <v-icon>directions_walk</v-icon>
        </GenericPanel>
      </v-flex>

      <v-flex xs12 sm6 md3 class="box-spacing">
        <RatingPanel bottom-text="My 5 Star Rating" top-text-color="red" :rating="currentRating">
          <v-icon>star</v-icon>
        </RatingPanel>
      </v-flex>

      <v-flex xs12 sm6 md3 class="box-spacing">
        <GenericPanel bottom-text="Completed Walks" top-text-color="green" :top-text="completedWalks">
          <v-icon>check</v-icon>
        </GenericPanel>
      </v-flex>

      <v-flex xs12 sm6 md3 class="box-spacing">
        <GenericPanel bottom-text="Available" top-text-color="orange" :top-text="availableIncome">
          <v-icon>credit_card</v-icon>
        </GenericPanel>
      </v-flex>

      <v-flex xs12 class="calendar-wrapper">
        <Calendar />
      </v-flex>
    </v-layout>
  </v-container>
</template>

<script>
import _ from 'lodash';

import firebaseWrapper from '../lib/firebaseWrapper.js';
import * as firebsaeConstants from '../constants/firebaseConstants.js';
import GenericPanel from '@/components/GenericPanel.vue';
import RatingPanel from '@/components/RatingPanel.vue';
import Calendar from '@/components/Calendar.vue';

export default {
  name: 'Home',

  data: function() {
    return {
      milesWalked: 0,
      currentRating: 0,
      completedWalks: 0,
      availableIncome: `£0`,
      // these are all the current walks that the person is enrolled in, we will use this
      // information to display related data for the user, including pending, miles, active etc.
      walks: {}
    };
  },

  created: async function() {
    // get the currenlty authenticated user.
    const user = firebaseWrapper.getCurrentUser();
    const profile = await firebaseWrapper.getProfile();
    this.walks = await firebaseWrapper.getAllWalks();

    // if the profile is new redirect to introduction page
    if (profile.new) {
      this.$router.push({ name: 'introduction' });
    }

    if (!_.isNil(profile) && !_.isNil(user)) {
      // set all the required data fields on the home page for displaying.
      this.currentRating = profile.walk.rating / profile.walk.completed;
      this.completedWalks = profile.walk.completed;
      this.availableIncome = `£${profile.walk.balance}`;
      this.milesWalked = profile.walk.miles;

      // when a user has 0 rating and 0 complted walks then there value is going to be NaN, if this
      // is true then we are just just to set the value back to 0
      if (_.isNaN(this.currentRating)) this.currentRating = 0;
    }
  },

  methods: {
    // returns the count of the total amount of walks that are currently pending for the given user,
    // this are walks in which the walker (or self if walker) has not confirmed, rejected or
    // completed a given walk.
    getPendingWalkCount: function() {
      let totalPendingWalks = 0;

      _.forEach(this.walks, (walk) => {
        if (walk.status === firebsaeConstants.WALK_STATUS.PENDING) totalPendingWalks += 1;
      });

      return totalPendingWalks;
    },
    // returns the count of the total amount of walks that are currently confirmed for the given
    // user, this are walks in which the walker (or self if walker) has confirmed completed a given
    // walk.
    getConfirmedWalks: function() {
      let totalConfirmedWalks = 0;

      _.forEach(this.walks, (walk) => {
        if (walk.status === firebsaeConstants.WALK_STATUS.ACTIVE) totalConfirmedWalks += 1;
      });

      return totalConfirmedWalks;
    }
  },

  components: {
    GenericPanel,
    Calendar,
    RatingPanel
  }
};
</script>

<style scoped>
.avatar {
  vertical-align: middle;
  width: 50px;
  height: 50px;
  border-radius: 50%;
}

.calendar-wrapper {
  margin-top: 25px;
}

.box-spacing {
  margin-bottom: 5px;
}
</style>
