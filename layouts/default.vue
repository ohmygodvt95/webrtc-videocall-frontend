<template>
  <v-app>
    <v-app-bar
      :clipped-left="clipped"
      fixed
      app
      flat
    >
      <v-btn
        icon
        to="/"
        @click.stop="clipped = !clipped"
      >
        <v-icon>mdi-message</v-icon>
      </v-btn>
      <v-toolbar-title v-text="title" />
      <v-spacer />
      {{ name }}
    </v-app-bar>
    <v-content>
      <div class="main">
        <nuxt />
      </div>
    </v-content>
    <v-overlay :value="overlay">
      <p>
        <v-progress-circular
          indeterminate
          color="primary"
        />
        Lost connection to server, trying to reconnect ...
      </p>
    </v-overlay>
  </v-app>
</template>

<style>
  .main {
    height: calc(100vh - 64px);
    width: 100vw;
    overflow-y: hidden;
  }
</style>

<script>
import { mapState } from 'vuex'

export default {
  data () {
    return {
      clipped: false,
      title: 'Video call 1-1'
    }
  },
  computed: mapState({
    name: state => state.setting.name,
    overlay: state => state.setting.overlay
  }),
  mounted () {
    this.$socket.on('reconnecting', () => {
      this.$store.commit('setting/setOverlay', {
        overlay: true
      })
    })

    this.$socket.on('reconnect', () => {
      this.$store.commit('setting/setOverlay', {
        overlay: false
      })
    })

    this.$socket.on('reconnect', () => {
      this.$store.commit('setting/setOverlay', {
        overlay: false
      })
    })

    this.$socket.on('reject', (data) => {
      this.$router.push('/error')
    })
  }
}
</script>
