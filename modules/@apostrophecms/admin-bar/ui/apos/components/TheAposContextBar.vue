<template>
  <div :class="classes">
    <template v-if="contextBarActive">
      <TheAposContextUndoRedo
        :v-if="editMode"
        :patches-since-loaded="patchesSinceLoaded"
        :undone="undone"
        @undo="undo"
        @redo="redo"
        :retrying="retrying"
        :editing="editing"
        :saving="saving"
        :saved="saved"
      />
      <TheAposContextTitle
        v-if="!hasCustomUi"
        :context="context"
        :draft-mode="draftMode"
        @switchDraftMode="switchDraftMode"
      />
      <TheAposContextModeAndSettings
        :context="context"
        :edit-mode="editMode"
        :has-custom-ui="hasCustomUi"
        :can-publish="canPublish"
        :ready-to-publish="readyToPublish"
        :custom-publish-label="customPublishLabel"
        @switchEditMode="switchEditMode"
        @discardDraft="onDiscardDraft"
        @publish="onPublish"
      />
    </template>
  </div>
</template>

<script>
import { klona } from 'klona';
import cuid from 'cuid';
import AposPublishMixin from 'Modules/@apostrophecms/ui/mixins/AposPublishMixin';
import AposAdvisoryLockMixin from 'Modules/@apostrophecms/ui/mixins/AposAdvisoryLockMixin';

export default {
  name: 'TheAposContextBar',
  mixins: [ AposPublishMixin, AposAdvisoryLockMixin ],
  data() {
    const query = apos.http.parseQuery(location.search);
    // If the URL references a draft, go into draft mode but then clean up the URL
    const draftMode = query['apos-mode'] || 'published';
    if (draftMode === 'draft') {
      delete query['apos-mode'];
      history.replaceState(null, '', apos.http.addQueryToUrl(location.href, query));
    }
    return {
      patchesSinceLoaded: [],
      undone: [],
      patchesSinceSave: [],
      editMode: false,
      draftMode,
      original: null,
      saving: false,
      editing: false,
      editingTimeout: null,
      retrying: false,
      saved: false,
      savingTimeout: null,
      context: window.apos.adminBar.context ? {
        ...window.apos.adminBar.context
      } : {},
      contextStack: [],
      // If a published context doc itself is not editable this will contain a hint
      // that the draft version is editable, when appropriate. It should only be
      // consulted when the context doc is published and not editable
      draftIsEditable: false
    };
  },
  computed: {
    contextBarActive() {
      return window.apos.adminBar.contextBar && this.canEdit;
    },
    canEdit() {
      return this.context._edit || ((this.context.aposLocale && this.context.aposLocale.endsWith(':published')) && this.draftIsEditable);
    },
    classes() {
      if (!this.contextBarActive) {
        return {};
      } else {
        return {
          'apos-admin-bar__row': true,
          'apos-admin-bar__row--utils': true
        };
      }
    },
    needToAutosave() {
      return !!this.patchesSinceSave.length;
    },
    canPublish() {
      return this.getContextModule(this.context).canPublish;
    },
    readyToPublish() {
      return this.canPublish
        ? this.context.modified && (!this.needToAutosave) && (!this.editing)
        : (!this.context.submitted) || (this.context.updatedAt > this.context.submitted.at);
    },
    moduleOptions() {
      return window.apos.adminBar;
    },
    action() {
      return this.getContextModule(this.context).action;
    },
    hasCustomUi() {
      return this.contextStack.length > 0;
    },
    customPublishLabel() {
      return (this.hasCustomUi && apos.modules[this.context.type].publishLabel) || null;
    }
  },
  async mounted() {
    apos.bus.$on('revert-published-to-previous', this.onRevertPublishedToPrevious);
    apos.bus.$on('unpublish', this.onUnpublish);
    apos.bus.$on('set-context', this.onSetContext);
    apos.bus.$on('push-context', this.onPushContext);
    apos.bus.$on('pop-context', this.onPopContext);
    apos.bus.$on('context-editing', this.onContextEditing);
    apos.bus.$on('context-edited', this.onContextEdited);
    apos.bus.$on('content-changed', this.onContentChanged);

    window.addEventListener('beforeunload', this.onBeforeUnload);
    window.addEventListener('storage', this.onStorage);

    // sessionStorage because it is deliberately browser-tab specific
    let tabId = sessionStorage.getItem('aposTabId');
    if (!tabId) {
      tabId = cuid();
      sessionStorage.setItem('aposTabId', tabId);
    }
    window.apos.adminBar.tabId = tabId;

    if (this.editMode) {
      // Watch out for legacy situations where edit mode is active
      // but we are not in draft
      if (!await this.lock(`${this.action}/${this.context._id}`)) {
        this.lockNotAvailable();
        return;
      }
      if (this.draftMode !== 'draft') {
        // Also refreshes
        this.switchDraftMode('draft');
      } else {
        // The page always initially loads with fully rendered content,
        // so refetch the content with the area placeholders and data instead
        this.refresh();
      }
    }
    await this.updateDraftIsEditable();
  },
  methods: {
    // Implements the `set-context` Apostrophe event, which can change the mode
    // (`draft` or `published`), the locale (such as `en`), and the context
    // document (`doc`). Navigates to `doc._url` if it differs from the browser's
    // current URL in the new mode, whether it is the current context doc or not.
    //
    // Accepts `mode`, `locale` and `doc` properties in its options object. Whether
    // the mode and locale are changing or not, if the `_url` of `doc` in the
    // final mode and locale does not match the current URL, navigate to it.
    // `doc` becomes the new context doc if it is not already.
    //
    // You should not emit `set-context` with a doc that has no `_url`, nor
    // do you need to because the user's browsing context does not change
    // when creating such a doc.
    //
    // If `locale` or `mode` are not passed, those parameters remain unchanged.
    // If `doc` is not passed the current context doc is assumed.
    //
    // See also `onPushContext` and `onPopContext` for a way to set a temporary
    // context document, such as global or palette, while it is being edited
    // "on the page."
    //
    // TODO: locales are not fully implemented in the UI yet. They are considered
    // in this API to reduce bc breaks in forthcoming betas.
    async onSetContext({
      mode,
      locale,
      doc
    }) {
      await this.setContext({
        mode,
        locale,
        doc,
        navigate: true
      });
      apos.bus.$emit('context-changed', {
        mode,
        locale,
        doc
      });
    },
    async onPushContext({
      doc
    }) {
      if (!this.draftMode !== 'draft') {
        await this.switchDraftMode('draft');
      }
      if (!this.editMode) {
        await this.switchEditMode(true);
      }
      this.contextStack.push({
        doc: this.context,
        original: this.original,
        patchesSinceLoaded: this.patchesSinceLoaded,
        undone: this.undone
      });
      this.original = klona(doc);
      this.patchesSinceLoaded = [];
      this.undone = [];
      await this.setContext({
        doc,
        navigate: false
      });
      // So that on-page areas react like foreign areas while
      // palette or another nested context is up
      await this.refresh();
    },
    async onPopContext() {
      const layer = this.contextStack.pop();
      this.original = layer.original;
      this.patchesSinceLoaded = layer.patchesSinceLoaded;
      this.undone = layer.undone;
      await this.setContext({
        doc: layer.doc
      });
      // So that areas revert to being editable
      await this.refresh();
    },
    async onContextEditing() {
      // Accept a hint that someone is actively typing in a
      // rich text editor and a context-edited event is likely
      // coming; allows continuity of the "Saving..." indicator
      // so it doesn't flicker once a second as you type
      this.editing = true;
      if (this.editingTimeout) {
        clearTimeout(this.editingTimeout);
      }
      this.editingTimeout = setTimeout(() => {
        this.editing = false;
        // Wait slightly longer than the rich text editor does
        // before sending us a context-edited event
      }, 1100);
    },
    async onPublish(e) {
      if (!this.canPublish) {
        const submitted = await this.submitDraft(this.action, this.context._id);
        if (submitted) {
          this.context = {
            ...this.context,
            submitted
          };
        }
      } else {
        const published = await this.publish(this.action, this.context._id, !!this.context.lastPublishedAt);
        if (published) {
          this.context = {
            ...this.context,
            lastPublishedAt: Date.now(),
            modified: false
          };
        }
      }
    },
    onBeforeUnload(e) {
      if (this.patchesSinceSave.length || this.saving || this.editing) {
        e.preventDefault();
        // No actual control over the message is possible in modern browsers,
        // but Chrome requires we set a string here
        e.returnValue = '';
      }
    },
    async save() {
      // More patches could get pushed during the async call to
      // send the previous batch, so keep going until we clear
      // the queue
      while (this.patchesSinceSave.length) {
        const patchesSinceSave = this.patchesSinceSave;
        this.retrying = false;
        this.saving = true;
        this.patchesSinceSave = [];
        try {
          this.saved = false;
          const body = {
            _patches: patchesSinceSave
          };
          this.addLockToRequest(body);
          const doc = await apos.http.patch(`${this.action}/${this.context._id}`, {
            body
          });
          this.context = {
            ...this.context,
            modified: doc.modified,
            submitted: doc.submitted,
            updatedAt: doc.updatedAt
          };
          this.retrying = false;
        } catch (e) {
          if (this.isLockedError(e)) {
            await this.showLockedError(e);
            return this.lockNotAvailable();
          }
          this.patchesSinceSave = [ ...patchesSinceSave, ...this.patchesSinceSave ];
          // Wait 5 seconds between attempts if errors occur
          await new Promise((resolve, reject) => {
            setTimeout(() => resolve(), 5000);
          });
          this.retrying = true;
        }
      }
      this.saving = false;
      this.saved = true;
    },
    // Switch the mode to 'published' or 'draft'.
    //
    // May refresh or navigate to another URL if needed, depending on whether
    // _url differs between draft and published. May do nothing if the mode
    // matches the existing one
    switchDraftMode(mode) {
      apos.bus.$emit('set-context', {
        mode
      });
    },
    getContextModule(doc) {
      return doc.slug.match(/^\//) ? self.apos.page : self.apos.modules[doc.type];
    },
    // Implementation detail of onSetContext and onPushContext.
    async setContext({
      mode,
      locale,
      doc,
      navigate = false
    }) {
      mode = mode || this.draftMode;
      locale = locale || apos.locale;
      doc = doc || this.context;
      if ((mode === this.draftMode) && (locale === apos.locale)) {
        if ((this.context._id === doc._id) && (!this.urlDiffers(doc._url))) {
          return;
        } else if (navigate && this.urlDiffers(doc._url)) {
          await this.unlock();
          return window.location.assign(doc._url);
        } else {
          await this.unlock();
        }
      }
      try {
        // Returns the doc as represented in the new locale and mode
        const action = this.getContextModule(doc).action;
        const modeDoc = await apos.http.get(`${action}/${doc._id}`, {
          qs: {
            'apos-mode': mode,
            'apos-locale': locale
          }
        });
        if (navigate && (!modeDoc._url)) {
          await apos.alert({
            heading: 'Page Does Not Exist Yet',
            description: `The page that provides a listing for this type of piece is not yet available as ${mode} in the ${locale} locale.`
          });
          return;
        }
        window.sessionStorage.setItem('aposStateChange', Date.now());
        window.sessionStorage.setItem('aposStateChangeSeen', '{}');
        if (mode === 'published') {
          this.editMode = false;
        }
        // Patch the module options. This is necessary because we're simulating
        // something that normally would involve a new page load, but without
        // the UX negatives of that. TODO: VueX as a long term fix
        window.apos.adminBar.context = modeDoc;
        window.apos.adminBar.contextId = modeDoc._id;
        this.context = modeDoc;
        await this.updateDraftIsEditable();
        this.draftMode = mode;
        if (navigate) {
          if (!await this.refreshOrReload(modeDoc._url)) {
            if (this.editMode) {
              if (!await this.lock(`${this.action}/${this.context._id}`)) {
                this.lockNotAvailable();
              }
            }
          }
        } else {
          if (this.editMode) {
            if (!await this.lock(`${this.action}/${this.context._id}`)) {
              this.lockNotAvailable();
            }
          }
        }
      } catch (e) {
        if (e.status === 404) {
          // TODO don't get this far, check this in advance and disable it in the UI
          await apos.alert({
            heading: 'Does Not Exist Yet',
            description: `That document is not yet available as ${mode} in the ${locale} locale.`
          });
        } else {
          await apos.alert({
            heading: 'An Error Occurred',
            description: 'Unable to switch modes.'
          });
        }
      }
    },
    onContextEdited(patch) {
      this.patchesSinceLoaded.push(patch);
      this.patchesSinceSave.push(patch);
      this.undone = [];
      if (!this.saving) {
        this.save();
      }
    },
    async onContentChanged() {
      this.refresh();
    },
    async switchEditMode(editing) {
      this.editMode = editing;
      if (editing) {
        if (!await this.lock(`${this.action}/${this.context._id}`)) {
          this.lockNotAvailable();
          return;
        }
      }
      if (this.draftMode !== 'draft') {
        // Entering edit mode implies entering draft mode.
        // Also takes care of refresh
        this.switchDraftMode('draft');
      } else {
        await this.refresh();
      }
    },
    async refresh() {
      let url = window.location.href;
      const qs = {
        ...apos.http.parseQuery(window.location.search),
        'apos-refresh': '1',
        'apos-mode': this.draftMode,
        ...(this.editMode ? {
          'apos-edit': '1'
        } : {})
      };
      url = url.replace(/\?.*$/, '');
      url = apos.http.addQueryToUrl(url, qs);
      const content = await apos.http.get(url, {
        qs,
        headers: {
          'Cache-Control': 'no-cache'
        },
        busy: true
      });
      const refreshable = document.querySelector('[data-apos-refreshable]');
      if (refreshable) {
        refreshable.innerHTML = content;
        if (!this.original) {
          // the first time we enter edit mode on the page, we need to
          // establish a baseline for undo/redo. Use our
          // "@ notation" PATCH feature. Sort the areas by DOM depth
          // to ensure parents patch before children
          this.original = {};
          const els = Array.from(document.querySelectorAll('[data-apos-area-newly-editable]')).filter(el => el.getAttribute('data-doc-id') === this.context._id);
          els.sort((a, b) => {
            const da = depth(a);
            const db = depth(b);
            if (da < db) {
              return -1;
            } else if (db > da) {
              return 1;
            } else {
              return 0;
            }
          });
          for (const el of els) {
            const data = JSON.parse(el.getAttribute('data'));
            this.original[`@${data._id}`] = data;
          }
        }
      }
      apos.bus.$emit('refreshed');
    },
    async onDiscardDraft(e) {
      const result = await this.discardDraft(this.action, this.context._id, !!this.context.lastPublishedAt);
      this.context = {
        ...this.context,
        modified: false
      };
      if (result.doc) {
        if (this.contextStack.length) {
          // Pushed contexts might edit any property of the doc
          this.original = klona(result.doc);
        } else {
          // On-page we only edit areas
          this.original = Object.fromEntries(
            Object.entries(
              result.doc
            ).filter(
              ([ key, value ]) => value && value.metaType === 'area'
            ).map(
              ([ key, value ]) => [ key, klona(value) ]
            )
          );
        }
      }
      this.patchesSinceLoaded = [];
      this.undone = [];
      if (!this.contextStack.length) {
        if (result.doc) {
          this.refreshOrReload(result.doc._url);
        } else {
          // With the current page gone, we need to move to safe ground
          location.assign(`${window.apos.prefix}/`);
        }
      } else {
        apos.bus.$emit('context-history-changed', result && result.doc);
      }
    },
    async onRevertPublishedToPrevious(data) {
      try {
        const response = await apos.http.post(`${data.action}/${data._id}/revert-published-to-previous`, {
          body: {},
          busy: true
        });
        apos.notify('Restored previously published version.', {
          type: 'success',
          dismiss: true
        });
        // This handler covers the modals too, so make sure it's
        // for the context document before altering any admin bar state
        // because of it
        if (data._id.replace(/:.*$/, '') === (this.context._id.replace(/:.*$/, ''))) {
          this.context = {
            ...this.context,
            modified: true,
            // If lastPublishedAt isn't present use a reasonable fallback, as
            // there can be published documents that never went through the
            // published API (parked pages not published since, etc)
            lastPublishedAt: response && (response.lastPublishedAt || response.updatedAt)
          };
          // No refresh is needed here because we're still in draft mode
          // looking at the draft mode, and the thing that changed is the
          // published mode
        }
      } catch (e) {
        await apos.alert({
          heading: 'An Error Occurred',
          description: e.message || 'An error occurred while restoring the previously published version.'
        });
      }
    },
    async onUnpublish(data) {
      try {
        await apos.http.post(`${data.action}/${data._id}/unpublish`, {
          body: {},
          busy: true
        });
        apos.notify('No longer published.', {
          type: 'success',
          dismiss: true
        });
        // This handler covers the modals too, so make sure it's
        // for the context document before altering any admin bar state
        // because of it
        if (data._id.replace(/:.*$/, '') === (this.context._id.replace(/:.*$/, ''))) {
          this.context = {
            ...this.context,
            modified: true,
            lastPublishedAt: null
          };
          // No refresh is needed here because we're still in draft mode
          // looking at the draft mode, and the thing that changed is the
          // published mode
        }
      } catch (e) {
        await apos.alert({
          heading: 'An Error Occurred',
          description: e.message || 'An error occurred while unpublishing the document.'
        });
      }
    },
    async undo() {
      this.undone.push(this.patchesSinceLoaded.pop());
      await this.refreshAfterHistoryChange('The operation could not be undone.');
    },
    async redo() {
      this.patchesSinceLoaded.push(this.undone.pop());
      await this.refreshAfterHistoryChange('The operation could not be redone.');
    },
    async refreshAfterHistoryChange(errorMessage) {
      this.saving = true;
      try {
        const updated = await apos.http.patch(`${this.action}/${this.context._id}`, {
          body: {
            _patches: [
              this.original,
              ...this.patchesSinceLoaded
            ]
          },
          busy: true
        });
        if (!this.contextStack.length) {
          await this.refresh();
        } else {
          apos.bus.$emit('context-history-changed', updated);
        }
      } catch (e) {
        console.error(e);
        apos.notify(errorMessage, { type: 'error' });
      } finally {
        this.saving = false;
      }
    },
    // returns true if the browser is about to navigate away
    async refreshOrReload(url) {
      if (this.urlDiffers(url)) {
        // Slug changed, must navigate
        window.location.assign(url);
        return true;
      } else {
        // No URL change means we can refresh just the content area
        await this.refresh();
        return false;
      }
    },
    urlDiffers(url) {
      // URL might or might not include hostname part
      url = url.replace(/^https?:\/\/.*?\//, '/');
      if (url === (window.location.pathname + (window.location.search || ''))) {
        return false;
      } else {
        return true;
      }
    },
    lockNotAvailable() {
      if (this.contextStack.length) {
        // If we try to edit palette and someone else has it locked,
        // we should just revert to the page context. Ask the palette
        // (or similar tool) to close itself, including popping the context
        apos.bus.$emit('context-close', this.context);
      } else {
        // If the context is the page, we should stay, but in preview mode
        this.switchEditMode(false);
      }
    },
    async updateDraftIsEditable() {
      if (this.context.aposLocale && this.context.aposLocale.endsWith('published') && !this.context._edit) {
        // A contributor might be able to edit the draft
        const draftContext = await apos.http.get(`${this.action}/${this.context._id}`, {
          busy: true,
          qs: {
            'apos-mode': 'draft',
            'apos-locale': this.context.aposLocale.split(':')[0]
          }
        });
        this.draftIsEditable = draftContext && draftContext._edit;
      }
    }
  }
};

function depth(el) {
  let depth = 0;
  while (el) {
    el = el.parentNode;
    depth++;
  }
  return depth;
}
</script>
<style lang="scss" scoped>
.apos-admin-bar__row--utils {
  display: flex;
  align-items: center;
}
</style>