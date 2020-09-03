<!--
  AposMediaManager will be in charge of all media-related state logic
  this includes doing the selecting and deselecting of items, deciding the editor/selection view,
  emitting batch action events, etc. All sub views will recieve `media` as a prop
-->

<template>
  <AposModal
    :modal="modal" modal-title="Manage Media"
    class="apos-media-manager"
    @inactive="modal.active = false" @show-modal="modal.showModal = true"
    @esc="cancel" @no-modal="$emit('safe-close')"
  >
    <template #primaryControls>
      <AposButton
        type="default" label="Finished"
        @click="cancel"
      />
    </template>
    <template #leftRail>
      <AposModalRail>
        <AposTagList title="Filter by Tag" :tags="tagList" />
      </AposModalRail>
    </template>
    <template #main>
      <AposModalBody>
        <template #bodyHeader v-if="!!media.length">
          <AposMediaManagerToolbar
            :checked="checked" :media="media"
            @select-click="selectClick"
            @trash-click="trashClick"
            @search="search"
          />
        </template>
        <template #bodyMain>
          <AposMediaManagerDisplay
            ref="display"
            :media="media"
            :module-options="options"
            @edit="updateEditing"
            v-model="checked"
            @select="select"
            @select-series="selectSeries"
            @select-another="selectAnother"
            @uploaded="getMedia"
          />
        </template>
      </AposModalBody>
    </template>
    <template #rightRail>
      <AposModalRail type="right">
        <div
          class="apos-media-manager__sidebar"
          :class="{ 'apos-media-manager__sidebar--empty' : !checked.length }"
        >
          <AposMediaManagerEditor
            v-show="editing"
            :media="editing" :selected="selected"
            @back="updateEditing(null)" @save="saveMedia"
          />
          <AposMediaManagerSelections
            :items="selected"
            @clear="clearSelected" @edit="updateEditing"
            v-show="!editing"
          />
        </div>
      </AposModalRail>
    </template>
  </AposModal>
</template>

<script>
import AposModalParentMixin from 'Modules/@apostrophecms/modal/mixins/AposModalParentMixin';

export default {
  mixins: [ AposModalParentMixin ],
  props: {
    moduleName: {
      type: String,
      required: true
    }
  },
  emits: [ 'safe-close', 'trash', 'save', 'search' ],
  data() {
    return {
      media: [],
      tagList: [],
      modal: {
        active: false,
        type: 'overlay',
        showModal: false
      },
      editing: null,
      checked: [],
      lastSelected: null,
      emptyDisplay: {
        title: 'No Media Found',
        message: 'Uploaded media will appear here',
        emoji: '🖼'
      }
    };
  },
  computed: {
    options() {
      return window.apos.modules[this.moduleName];
    },
    selected() {
      return this.media.filter(item => this.checked.includes(item._id));
    }
  },
  watch: {
    checked (newVal) {
      if (newVal.length > 1) {
        this.editing = null;
      }
    }
  },
  async mounted() {
    // TODO: Get data here.
    this.modal.active = true;
    await this.getMedia();
  },
  methods: {
    async getMedia () {

      const qs = {};

      // Avoid undefined properties.
      for (const prop in qs) {
        if (qs[prop] === undefined) {
          delete qs[prop];
        };
      }
      const getResponse = (await apos.http.get(
        this.options.action, {
          busy: true,
          qs
        }
      ));

      this.media = getResponse.results;
    },
    clearSelected() {
      this.checked = [];
      this.editing = null;
    },
    updateEditing(id) {
      this.editing = this.media.find(item => item._id === id);
    },

    // select setters
    select(id) {
      if (this.checked.includes(id)) {
        this.checked = [];
      } else {
        this.checked = [ id ];
      }

      this.updateEditing(id);
      this.lastSelected = id;
    },

    selectAnother(id) {
      if (this.checked.includes(id)) {
        this.checked = this.checked.filter(checkedId => checkedId !== id);
      } else {
        this.checked.push(id);
      }

      this.lastSelected = id;
      this.editing = null;
    },

    selectSeries(id) {
      if (!this.lastSelected) {
        this.select(id);
        return;
      }

      let beginIndex = this.media.findIndex(item => item._id === this.lastSelected);
      let endIndex = this.media.findIndex(item => item._id === id);
      const direction = beginIndex > endIndex ? -1 : 1;

      if (direction < 0) {
        [ beginIndex, endIndex ] = [ endIndex, beginIndex ];
      } else {
        endIndex++;
      }

      const sliced = this.media.slice(beginIndex, endIndex);
      // always want to check, never toggle
      sliced.forEach(item => {
        if (!this.checked.includes(item._id)) {
          this.checked.push(item._id);
        }
      });

      this.lastSelected = sliced[sliced.length - 1]._id;
      this.editing = null;
    },

    // Toolbar handlers
    selectClick() {
      if (this.checked.length === this.media.length) {
        // unselect all
        this.clearSelected();
      } else {
        // select all
        this.checked = this.media.map(item => item._id);
        this.editing = null;
        this.lastSelected = this.media[this.media.length - 1]._id;
      }
    },

    // TODO stub
    trashClick() {
      this.$emit('trash', this.checked);
    },

    // TODO stub
    saveMedia() {
      this.$emit('save');
    },

    search(query) {
      // TODO stub
      this.$emit('search', query);
    }

  }
};
</script>

<style lang="scss" scoped>
.apos-media-manager /deep/ .apos-media-manager-toolbar {
  z-index: $z-index-manager-toolbar;
  position: relative;
}

.apos-media-manager__empty {
  display: flex;
  justify-content: center;
  align-items: center;
  width: 100%;
  height: 100%;
  margin-top: 130px;
}

.apos-media-manager__sidebar {
  position: relative;
  width: 100%;
}

.apos-media-manager__sidebar--empty {
  height: 100%;
}

</style>