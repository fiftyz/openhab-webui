<template>
  <f7-page @page:afterin="onPageAfterIn" @page:beforeout="onPageBeforeOut" class="plan-editor">
    <f7-navbar :title="(!ready) ? '' : (createMode) ? 'Create plan page' : page.config.label" back-link="Back" no-hairline>
      <f7-nav-right>
        <f7-link @click="save()" v-if="$theme.md" icon-md="material:save" icon-only />
        <f7-link @click="save()" v-if="!$theme.md">
          Save<span v-if="$device.desktop">&nbsp;(Ctrl-S)</span>
        </f7-link>
      </f7-nav-right>
    </f7-navbar>
    <f7-toolbar tabbar position="top">
      <f7-link @click="switchTab('design', fromYaml)" :tab-link-active="currentTab === 'design'" class="tab-link">
        Design
      </f7-link>
      <f7-link @click="switchTab('code', toYaml)" :tab-link-active="currentTab === 'code'" class="tab-link">
        Code
      </f7-link>
    </f7-toolbar>
    <f7-toolbar bottom class="toolbar-details">
      <div style="margin-left: auto">
        <f7-toggle :checked="previewMode" @toggle:change="(value) => togglePreviewMode(value)" /> Run mode<span v-if="$device.desktop">&nbsp;(Ctrl-R)</span>
      </div>
    </f7-toolbar>

    <f7-tabs class="plan-editor-tabs">
      <f7-tab id="design" class="plan-editor-design-tab" @tab:show="() => this.currentTab = 'design'" :tab-active="currentTab === 'design'">
        <f7-block v-if="!ready" class="text-align-center">
          <f7-preloader />
          <div>Loading...</div>
        </f7-block>
        <f7-block class="block-narrow" v-if="ready && !previewMode">
          <page-settings :page="page" :createMode="createMode" />
        </f7-block>

        <f7-block class="block-narrow" style="padding-bottom: 8rem" v-if="ready && !previewMode">
          <f7-col>
            <f7-block-title>Background Configuration</f7-block-title>
            <config-sheet
              :parameterGroups="pageWidgetDefinition.props.parameterGroups || []"
              :parameters="pageWidgetDefinition.props.parameters || []"
              :configuration="page.config"
              @updated="dirty = true" />

            <f7-block-title class="padding-bottom">
              Markers
            </f7-block-title>
            <f7-menu v-if="clipboardType === 'oh-plan-marker'" class="padding-bottom">
              <f7-menu-item style="margin-left: auto" icon-f7="square_on_square" dropdown>
                <f7-menu-dropdown right>
                  <f7-menu-dropdown-item @click="pasteWidget(page, null)" href="#" text="Paste" />
                </f7-menu-dropdown>
              </f7-menu-item>
            </f7-menu>

            <f7-list media-list class="markers-list">
              <f7-list-item media-item v-for="(marker, idx) in page.slots.default" :key="idx"
                            :title="marker.config.name" :subtitle="marker.config.item || marker.config.location"
                            link="#" @click.native="(ev) => configureMarker(ev, marker, context)">
                <oh-icon v-if="marker.config.icon" slot="media" :icon="marker.config.icon" height="32" width="32" />
                <f7-menu slot="content-start" class="configure-layout-menu">
                  <f7-menu-item icon-f7="list_bullet" dropdown>
                    <f7-menu-dropdown>
                      <f7-menu-dropdown-item @click="configureWidget(marker, { component: page })" href="#" text="Configure marker" />
                      <f7-menu-dropdown-item @click="editWidgetCode(marker, { component: page })" href="#" text="Edit YAML" />
                      <f7-menu-dropdown-item divider />
                      <f7-menu-dropdown-item @click="cutWidget(marker, { component: page })" href="#" text="Cut" />
                      <f7-menu-dropdown-item @click="copyWidget(marker, { component: page })" href="#" text="Copy" />
                      <f7-menu-dropdown-item divider />
                      <f7-menu-dropdown-item @click="moveWidgetUp(marker, { component: page })" href="#" text="Move Up" />
                      <f7-menu-dropdown-item @click="moveWidgetDown(marker, { component: page })" href="#" text="Move Down" />
                      <f7-menu-dropdown-item divider />
                      <f7-menu-dropdown-item @click="removeWidget(marker, { component: page })" href="#" text="Remove marker" />
                    </f7-menu-dropdown>
                  </f7-menu-item>
                </f7-menu>
              </f7-list-item>
              <f7-list-button color="blue" title="Add marker" @click="addWidget(page, 'oh-plan-marker')" />
            </f7-list>
            <f7-block-footer class="param-description">
              You can also <f7-link style="z-index: inherit" href="#" @click="previewMode = true">
                switch to Run mode
              </f7-link> to add markers and position them on the plan.
            </f7-block-footer>
          </f7-col>
        </f7-block>

        <oh-plan-page class="plan-page" v-else-if="ready && previewMode" :context="context" :key="pageKey" />
      </f7-tab>

      <f7-tab id="code" @tab:show="() => { this.currentTab = 'code' }" :tab-active="currentTab === 'code'">
        <editor v-if="currentTab === 'code'" :style="{ opacity: previewMode ? '0' : '' }" class="page-code-editor" mode="application/vnd.openhab.uicomponent+yaml;type=plan" :value="pageYaml" @input="onEditorInput" />
        <!-- <pre class="yaml-message padding-horizontal" :class="[yamlError === 'OK' ? 'text-color-green' : 'text-color-red']">{{yamlError}}</pre> -->

        <oh-plan-page class="plan-page" v-if="ready && previewMode" :context="context" :key="pageKey + '2'" />
      </f7-tab>
    </f7-tabs>
  </f7-page>
</template>

<style lang="stylus">
.page-code-editor.vue-codemirror
  display block
  top calc(var(--f7-navbar-height) + var(--f7-tabbar-height))
  height calc(100% - 3*var(--f7-navbar-height))
  width 100%
.yaml-message
  display block
  position absolute
  top 80%
  white-space pre-wrap
.plan-editor
  .oh-plan-page-lmap
    top calc(var(--f7-navbar-height) + var(--f7-toolbar-height)) !important
    height calc(100% - var(--f7-navbar-height) - 2 * var(--f7-toolbar-height)) !important
.markers-list
  .item-link
    overflow inherit
    z-index inherit !important
</style>

<script>
import PageDesigner from '../pagedesigner-mixin'

import YAML from 'yaml'

import OhPlanPage from '@/components/widgets/plan/oh-plan-page.vue'
import OhPlanMarker from '@/components/widgets/plan/oh-plan-marker.vue'

const ConfigurableWidgets = {
  OhPlanMarker
}

import PageSettings from '@/components/pagedesigner/page-settings.vue'

import ConfigSheet from '@/components/config/config-sheet.vue'

export default {
  mixins: [PageDesigner],
  components: {
    'editor': () => import(/* webpackChunkName: "script-editor" */ '@/components/config/controls/script-editor.vue'),
    OhPlanPage,
    PageSettings,
    ConfigSheet
  },
  props: ['createMode', 'uid'],
  data () {
    return {
      pageWidgetDefinition: OhPlanPage.widget(),
      forceEditMode: true,
      page: {
        uid: 'page_' + this.$f7.utils.id(),
        component: 'oh-plan-page',
        config: {},
        slots: { default: [] }
      }
    }
  },
  methods: {
    markerDefaultIcon (marker) {
      const widgetDefinition = Object.values(ConfigurableWidgets).find((c) => c.widget && typeof c.widget === 'function' && c.widget().name === marker.component)
      if (widgetDefinition) {
        return widgetDefinition.widget().icon
      }
      return null
    },
    addWidget (component, widgetType, parentContext, slot) {
      if (!slot) slot = 'default'
      if (!component.slots) component.slots = {}
      if (!component.slots[slot]) component.slots[slot] = []
      if (widgetType) {
        component.slots[slot].push({
          component: widgetType,
          config: {
            name: 'New Marker'
          },
          slots: { default: [] }
        })
        this.forceUpdate()
      }
    },
    getWidgetDefinition (componentType) {
      const component = Object.values(ConfigurableWidgets).find((w) => w.widget && typeof w.widget === 'function' && w.widget().name === componentType)
      if (!component) return null
      return component.widget()
    },
    configureMarker (ev, marker, context) {
      let el = ev.target
      ev.cancelBubble = true
      while (!el.classList.contains('media-item')) {
        if (el && el.classList.contains('menu')) return
        el = el.parentElement
      }
      this.context.editmode.configureWidget(marker, context)
    },
    toYaml () {
      this.pageYaml = YAML.stringify({
        config: this.page.config,
        markers: this.page.slots.default
      })
    },
    fromYaml () {
      try {
        const updatedPage = YAML.parse(this.pageYaml)
        this.$set(this.page, 'config', updatedPage.config)
        this.$set(this.page.slots, 'default', updatedPage.markers)
        this.forceUpdate()
        return true
      } catch (e) {
        this.$f7.dialog.alert(e).open()
        return false
      }
    }
  }
}
</script>
