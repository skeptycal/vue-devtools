<template>
  <div class="settings grid">
    <GlobalPreferences />
  </div>
</template>

<script>
import GlobalPreferences from "./GlobalPreferences.vue";
import { mapState } from "vuex";

export const SETTINGS_VERSION = 1;
export const SETTINGS_VERSION_ID = "vue-devtools-settings-version";

export default {
  components: { GlobalPreferences },

  provide() {
    return {
      settingsVersion: SETTINGS_VERSION,
      currentSettingsVersion:
        parseInt(localStorage.getItem(SETTINGS_VERSION_ID)) || 0
    };
  },

  computed: mapState("events", ["enabled"])
};
</script>

<style lang="stylus" scoped>
.settings {
  overflow: auto !important;

  >>> .preferences {
    display: flex;
    flex-wrap: wrap;
    padding: 12px 4px;

    > * {
      flex-basis: 300px;
      margin: 16px 24px;
    }

    .vue-ui-form-field {
      > .wrapper > .content {
        min-height: 32px;
        justify-content: center;
      }
    }
  }
}
</style>
