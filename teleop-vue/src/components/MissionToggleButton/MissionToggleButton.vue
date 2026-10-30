<template>
  <button
    type="button"
    class="mission-toggle-button"
    v-bind:class="{ enabled: missionEnabled }"
    v-on:click="onClick"
  >
    <span class="mission-indicator" />
    <div>toggle_mission</div>
  </button>
</template>

<script>
export default {
  name: "mission-toggle-button",
  computed: {
    // The robot is the source of truth: it records the last requested mode
    // and keeps reporting it through "mission_mode_status" messages, so the
    // indicator survives page reloads
    missionEnabled() {
      return this.$store.state.localClient.openteraTeleop.missionMode;
    },
  },
  methods: {
    onClick() {
      if (this.$store.state.localClient.openteraTeleop.client) {
        this.$store.state.localClient.openteraTeleop.client.sendToAll(
          JSON.stringify({
            type: "toggle_mission_mode",
            state: !this.missionEnabled,
          })
        );
      }
    },
  },
};
</script>

<style lang="scss" scoped>
@import "./MissionToggleButton.scss";
</style>
