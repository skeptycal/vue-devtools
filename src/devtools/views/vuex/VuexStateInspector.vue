<template>
  <scroll-pane>
    <action-header slot="header">
      <div class="search">
        <VueIcon icon="search" />
        <input v-model.trim="filter" placeholder="Filter inspected state" />
      </div>
      <a
        v-tooltip="'Export Vuex State'"
        class="button export"
        @click="copyStateToClipboard"
      >
        <VueIcon icon="content_copy" />
        <span>Export</span>
        <transition name="slide-up">
          <span v-show="showStateCopiedMessage" class="message">
            (Copied to clipboard!)
          </span>
        </transition>
      </a>
      <a
        v-tooltip="'Import Vuex State'"
        class="button import"
        @click="toggleImportStatePopup"
      >
        <VueIcon icon="content_paste" />
        <span>Import</span>
      </a>
      <transition name="slide-down">
        <div v-if="showImportStatePopup" class="import-state">
          <textarea
            placeholder="Paste state object here to import it..."
            @input="importState"
            @keydown.esc.stop="closeImportStatePopup"
          />
          <span v-show="showBadJSONMessage" class="message invalid-json">
            INVALID JSON!
          </span>
        </div>
      </transition>
    </action-header>
    <div
      slot="scroll"
      class="vuex-state-inspector"
      :class="{
        pointer: isOnlyMutationPayload
      }"
      @click="isOnlyMutationPayload && loadState()"
    >
      <state-inspector
        :state="filteredState"
        :dim-after="isOnlyMutationPayload ? 1 : -1"
      />
    </div>
    <div
      v-if="$shared.snapshotLoading"
      slot="footer"
      class="state-info loading-vuex-state"
    >
      <div class="label">
        Loading state...
      </div>

      <VueLoadingIndicator />
    </div>
    <div
      v-else-if="isOnlyMutationPayload"
      slot="footer"
      class="state-info recording-vuex-state"
    >
      <div class="label">
        <VueIcon class="medium" icon="cached" />
        <span>Recording state on-demand...</span>
        <span v-if="lastReceivedState" class="note"
          >displaying last received state</span
        >
      </div>

      <div>
        <VueButton
          data-id="load-vuex-state"
          icon-left="arrow_forward"
          class="accent flat"
          @click="loadState()"
        >
          Load state
        </VueButton>
      </div>
    </div>
  </scroll-pane>
</template>

<script>
import ScrollPane from "components/ScrollPane.vue";
import ActionHeader from "components/ActionHeader.vue";
import StateInspector from "components/StateInspector.vue";

import { searchDeepInObject, sortByKey, stringify, parse } from "src/util";
import debounce from "lodash.debounce";
import groupBy from "lodash.groupby";
import { mapState, mapGetters, mapActions } from "vuex";

export default {
  components: {
    ScrollPane,
    ActionHeader,
    StateInspector
  },

  provide() {
    return {
      InspectorInjection: this.injection
    };
  },

  data() {
    return {
      showStateCopiedMessage: false,
      showBadJSONMessage: false,
      showImportStatePopup: false,
      filter: "",
      injection: {
        editable: false
      }
    };
  },

  computed: {
    ...mapState("vuex", ["activeIndex", "inspectedIndex", "lastReceivedState"]),

    ...mapGetters("vuex", [
      "inspectedState",
      "filteredHistory",
      "inspectedEntry"
    ]),

    filteredState() {
      const inspectedState =
        this.isOnlyMutationPayload && this.inspectedState.mutation
          ? {
              mutation: this.inspectedState.mutation,
              ...this.lastReceivedState
            }
          : this.inspectedState;

      const getProcessedState = (state, type) => {
        if (!Array.isArray(state)) {
          return Object.keys(state).map(key => ({
            key,
            editable: !this.isOnlyMutationPayload && type === "state",
            value: state[key]
          }));
        } else {
          return state;
        }
      };

      const result = [].concat(
        ...Object.keys(inspectedState).map(type => {
          const state = inspectedState[type];
          let processedState;

          if (type === "mutation" && this.inspectedEntry) {
            const { options } = this.inspectedEntry;
            if (options.registerModule || options.unregisterModule) {
              processedState = getProcessedState(state.payload, type);
              type = options.registerModule
                ? "register module"
                : "unregister module";
            }
          }

          if (!processedState) {
            processedState = getProcessedState(state, type);
          }

          return processedState.map(item => ({
            type,
            ...item
          }));
        })
      );

      return groupBy(
        sortByKey(
          result.filter(el =>
            searchDeepInObject(
              {
                [el.key]: el.value
              },
              this.filter
            )
          )
        ),
        "type"
      );
    },

    isOnlyMutationPayload() {
      return (
        (Object.keys(this.inspectedState).length === 1 &&
          !!this.inspectedState.mutation) ||
        Object.keys(this.inspectedState).length < 1
      );
    },

    isActive() {
      return this.activeIndex === this.inspectedIndex;
    }
  },

  watch: {
    showImportStatePopup(val) {
      if (val) {
        this.$nextTick(() => {
          this.$el.querySelector("textarea").focus();
        });
      }
    },

    isActive: {
      handler(value) {
        this.injection.editable = value;
      },
      immediate: true
    }
  },

  mounted() {
    bridge.on("vuex:mutation", this.onMutation);
    if (this.isOnlyMutationPayload && this.$shared.vuexAutoload) {
      this.loadState();
    }

    bridge.on("vuex:init", this.onVuexInit);
  },

  destroyed() {
    bridge.off("vuex:mutation", this.onMutation);
    bridge.off("vuex:init", this.onVuexInit);
  },

  methods: {
    ...mapActions("vuex", ["inspect"]),

    copyStateToClipboard() {
      copyToClipboard(this.inspectedState.state);
      this.showStateCopiedMessage = true;
      window.setTimeout(() => {
        this.showStateCopiedMessage = false;
      }, 2000);
    },

    toggleImportStatePopup() {
      if (this.showImportStatePopup) {
        this.closeImportStatePopup();
      } else {
        this.showImportStatePopup = true;
      }
    },

    closeImportStatePopup() {
      this.showImportStatePopup = false;
    },

    importState: debounce(function(e) {
      const importedStr = e.target.value;
      if (importedStr.length === 0) {
        this.showBadJSONMessage = false;
      } else {
        try {
          // Try to parse here so we can provide invalid feedback
          parse(importedStr, true);
          bridge.send("vuex:import-state", importedStr);
          this.showBadJSONMessage = false;
        } catch (e) {
          this.showBadJSONMessage = true;
        }
      }
    }, 250),

    loadState() {
      const history = this.filteredHistory;
      this.inspect(history[history.length - 1]);
    },

    onMutation: debounce(function() {
      if (this.$shared.vuexAutoload) {
        this.loadState();
      }
    }, 300),

    onVuexInit() {
      if (this.$shared.vuexAutoload) {
        this.loadState();
      }
    }
  }
};

function copyToClipboard(state) {
  const dummyTextArea = document.createElement("textarea");
  dummyTextArea.textContent = stringify(state);
  document.body.appendChild(dummyTextArea);
  dummyTextArea.select();
  document.execCommand("copy");
  document.body.removeChild(dummyTextArea);
}
</script>

<style lang="stylus" scoped>
.state-info {
  display: flex;
  align-items: center;
  padding: 2px 2px 2px 14px;
  min-height: 36px;
  font-size: 14px;

  .label {
    flex: 1;
    display: flex;
    align-items: center;
    color: $blueishGrey;

    .vue-ui-icon {
      margin-right: 8px;

      >>> svg {
        fill: @color;
      }
    }
  }

  .note {
    opacity: 0.7;
    margin-left: 4px;
  }
}

.loading-vuex-state {
  padding-right: 14px;
}

.pointer {
  cursor: pointer;
}

.message {
  margin-left: 5px;
  transition: all 0.3s ease;
  color: $blue;
}

.invalid-json {
  right: 20px;
  left: initial;
  top: 1px;
  font-size: 12px;
  color: $red;
  background-color: $background-color;

  .vue-ui-dark-mode & {
    background-color: $dark-background-color;
  }
}

.import-state {
  transition: all 0.2s ease;
  width: 300px;
  position: absolute;
  z-index: 1;
  left: 220px;
  right: 10px;
  top: 45px;
  box-shadow: 4px 4px 6px 0 $border-color;
  border: 1px solid $border-color;
  padding: 3px;
  background-color: $background-color;

  .vue-ui-dark-mode & {
    background-color: $dark-background-color;
    box-shadow: 4px 4px 6px 0 $dark-border-color;
    border: 1px solid $dark-border-color;
  }

  &:after {
    content: 'Press ESC to close';
    position: absolute;
    bottom: 0;
    padding: 5px;
    color: inherit;
    opacity: 0.5;
  }

  textarea {
    width: 100%;
    height: 100px;
    display: block;
    outline: none;
    border: none;
    resize: vertical;

    .vue-ui-dark-mode & {
      color: #DDD;
      background-color: $dark-background-color;
    }
  }
}
</style>
