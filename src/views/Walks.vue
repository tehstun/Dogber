<template>
  <div id="walks">
    <v-app id="inspire">
      <v-container grid-list-md text-xs-center>
        <v-layout row wrap>
          <v-flex xs12>
            <v-card>
              <v-layout row wrap>
                <v-flex xs6>
                  <v-card-title primary class="subheading text-sm-left">My Walks</v-card-title>
                </v-flex>
                <v-flex xs6 class="text-sm-right">
                  <v-btn
                    icon
                    large
                    :loading="refreshingMyWalks"
                    :disabled="refreshingMyWalks"
                    @click="refreshMyWalks"
                  >
                    <v-icon>refresh</v-icon>
                  </v-btn>
                </v-flex>
              </v-layout>
            </v-card>
          </v-flex>
          <v-flex xs12 style="margin-top: 25px">
            <SingleWalkResult v-for="(item, index) in walks" :key="index" class="walk-history" :id="item" />
          </v-flex>
        </v-layout>
      </v-container>
    </v-app>
  </div>
</template>

<script>
import SingleWalkResult from '@/components/SingleWalkResult.vue';
import firebaseWrapper from '../lib/firebaseWrapper.js';

export default {
  name: 'Walks',
  data: function() {
    return {
      dropdown_walkCompleted: ['Yes', 'No'],
      dropdown_date: ['Ascending', 'Descending'],
      walks: {},
      refreshingMyWalks: false
    };
  },

  created: async function() {
    // get the currenlty authenticated user.
    const profile = await firebaseWrapper.getProfile();

    // If the profile is new redirect to introduction
    if (profile.new) {
      this.$router.push({ name: 'introduction' });
    }

    this.walks = await firebaseWrapper.getAllWalkKeys();
  },

  methods: {
    refreshMyWalks: async function() {
      this.refreshingMyWalks = true;
      this.walks = {};
      this.walks = await firebaseWrapper.getAllWalkKeys();
      this.refreshingMyWalks = false;
    }
  },

  components: {
    SingleWalkResult
  }
};
</script>

<style scoped>
.walk-history {
  margin-bottom: 25px;
}
</style>
