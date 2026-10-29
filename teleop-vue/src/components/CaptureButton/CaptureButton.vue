<template>
  <div class="capture-container">
    <action-button
      label="CapEventImg ▾"
      class="CapEventImg-button"
      @clicked="toggleShowMenu"
    />
    <div
      class="capture-menu"
      v-if="showMenu"
      v-click-away="onClickAway"
      @click="onClickAway"
    >
      <div
        class="capture-menu-item"
        v-for="category in categories"
        :key="category"
        @click="onSelectCategory(category)"
      >
        {{ category }}
      </div>
    </div>
    <div class="capture-note-panel" v-if="selectedCategory">
      <div class="capture-note-header">
        <span class="capture-note-title">{{ selectedCategory }}</span>
        <button
          type="button"
          class="capture-note-close"
          title="Skip note (Esc)"
          @click="onSkip"
        >
          &#x2715;
        </button>
      </div>
      <textarea
        ref="note"
        v-model="note"
        class="capture-note-textarea"
        :maxlength="noteMaxLength"
        placeholder="Optional note (Enter: send, Shift+Enter: new line, Esc: skip)"
        @keydown.stop="onNoteKeyDown"
        @keyup.stop
      />
      <div class="capture-note-buttons">
        <action-button
          label="Send"
          class="capture-note-send"
          @clicked="onSend"
        />
      </div>
    </div>
  </div>
</template>

<script>
import ActionButton from "@/components/ActionButton/ActionButton.vue";

const CAPTURE_CATEGORIES = [
  "Mortality",
  "Downer",
  "Leak",
  "Spill",
  "Misplaced_Item",
  "Miscellaneous",
];

export default {
  name: "capture-button",
  components: {
    ActionButton,
  },
  data() {
    return {
      categories: CAPTURE_CATEGORIES,
      noteMaxLength: 500,
      showMenu: false,
      canShowMenu: true,
      selectedCategory: null,
      note: "",
    };
  },
  methods: {
    toggleShowMenu() {
      if (this.selectedCategory) {
        // A capture is already waiting for its note
        return;
      }
      if (!this.showMenu && this.canShowMenu) {
        this.showMenu = true;
        this.canShowMenu = false;
      } else if (this.showMenu) {
        this.showMenu = false;
        this.canShowMenu = true;
      }
    },
    onClickAway() {
      this.showMenu = false;
      setTimeout(() => {
        this.canShowMenu = true;
      }, 200);
    },
    onSelectCategory(category) {
      this.selectedCategory = category;
      this.note = "";
      this.$nextTick(() => {
        if (this.$refs.note) {
          this.$refs.note.focus();
        }
      });
    },
    onNoteKeyDown(event) {
      // Key events are stopped from propagating so that typing a note (e.g.
      // arrow keys) doesn't drive the robot through window-level listeners
      if (event.key === "Escape") {
        this.onSkip();
      } else if (event.key === "Enter" && !event.shiftKey) {
        event.preventDefault();
        this.onSend();
      }
    },
    onSkip() {
      this.note = "";
      this.onSend();
    },
    onSend() {
      if (!this.selectedCategory) {
        return;
      }
      if (this.$store.state.localClient.openteraTeleop.client) {
        this.$store.state.localClient.openteraTeleop.client.sendToAll(
          JSON.stringify({
            type: "CapEventImg",
            state: true,
            category: this.selectedCategory,
            note: this.note.trim(),
          })
        );
      }
      this.selectedCategory = null;
      this.note = "";
    },
  },
};
</script>

<style lang="scss" scoped>
@import "./CaptureButton.scss";
</style>
