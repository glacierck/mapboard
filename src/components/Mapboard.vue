<template>
  <div id="mb-root"
       :class="rootClass"
       :style="mbRootStyle"
  >
      <topic-panel :class="this.shouldShowTopicPanel" />

      <map-panel :class="this.shouldShowMapPanel"
                 v-if="this.fullScreenTopicsOnly !== true"
      >
        <cyclomedia-widget v-if="this.shouldLoadCyclomediaWidget"
                           slot="cycloWidget"
                           v-show="cyclomediaActive"
                           screen-percent="2"
        />
        <pictometry-widget v-if="this.shouldLoadPictometryWidget"
                           slot="pictWidget"
                           v-show="pictometryActive"
                           :apiKey="this.ak"
                           :secretKey="this.sk"
        >
          <png-marker v-if="this.pictometryShowAddressMarker"
                      :latlng="[this.geocodeData.geometry.coordinates[1], this.geocodeData.geometry.coordinates[0]]"
                      :icon="'images/markers.png'"
                      :height="60"
                      :width="40"
                      :offsetX="0"
                      :offsetY="0"
          />
          <layer v-if="this.pictometryActive" />
          <png-marker v-if="this.cyclomediaActive && this.pictometryActive"
                      :latlng="cycloLatlng"
                      :icon="'images/camera2.png'"
                      :height="20"
                      :width="30"
                      :offsetX="-2"
                      :offsetY="-2"
          />
          <view-cone v-if="this.cyclomediaActive && this.pictometryActive"
                     :latlng="cycloLatlng"
                     :rotationAngle="cycloRotationAngle"
                     :hFov="cycloHFov"
          />
        </pictometry-widget>
      </map-panel>
  </div>
</template>

<script>
  import philaVueMapping from '@cityofphiladelphia/phila-vue-mapping';

  import TopicPanel from './TopicPanel.vue';
  import MapPanel from './MapPanel.vue';

  const CyclomediaWidget = philaVueMapping.CyclomediaWidget;
  const PictometryWidget = philaVueMapping.PictometryWidget;
  const Layer = philaVueMapping.PictometryLayer;
  const ViewCone = philaVueMapping.PictometryViewCone;
  const PngMarker = philaVueMapping.PictometryPngMarker;

  export default {
    components: {
      TopicPanel,
      MapPanel,
      CyclomediaWidget,
      PictometryWidget,
      Layer,
      PngMarker,
      ViewCone
    },
    data() {
      const data = {
        // this will only affect the app size if the app is set to "plugin" mode
        mbRootStyle: {
          'height': '100px'
        }
      };
      return data;
    },
    created() {
      console.log('this.$config:', this.$config);
      if (this.$config.panels) {
        if (!this.$config.panels.includes('map')) {
          this.$store.commit('setTopicsOnly', true);
        } else if (!this.$config.panels.includes('topics')) {
          this.$store.commit('setMapOnly', true);
        }
      }
      window.addEventListener('click', this.closeAddressCandidateList);
      window.addEventListener('resize', this.handleWindowResize);
      this.handleWindowResize();
    },
    computed: {
      rootClass() {
        if (this.$config.plugin) {
          return 'grid-x';
        } else {
          return 'cell medium-auto grid-x';
        }
      },
      isMobileOrTablet() {
        return this.$store.state.isMobileOrTablet;
      },
      shouldLoadCyclomediaWidget() {
        return this.$config.cyclomedia.enabled && !this.isMobileOrTablet;
      },
      shouldLoadPictometryWidget() {
        return this.$config.pictometry.enabled && !this.isMobileOrTablet;
      },
      fullScreenMapOnly() {
        return this.$store.state.fullScreen.mapOnly;
      },
      fullScreenMapEnabled() {
        return this.$store.state.fullScreenMapEnabled;
      },
      fullScreenTopicsOnly() {
        return this.$store.state.fullScreen.topicsOnly;
      },
      fullScreenTopicsEnabled() {
        return this.$store.state.fullScreenTopicsEnabled;
      },
      shouldShowTopicPanel() {
        let value;
        if (!this.fullScreenMapEnabled && !this.fullScreenMapOnly) {
          value = 'topic-panel-true';
        } else {
          value = 'topic-panel-false';
        }
        return value;
      },
      shouldShowMapPanel() {
        let value;
        if (!this.fullScreenTopicsEnabled && !this.fullScreenTopicsOnly) {
          value = 'map-panel-true';
        } else {
          value = 'map-panel-false';
        }
        return value;
      },
      cyclomediaActive() {
        return this.$store.state.cyclomedia.active
      },
      cycloLatlng() {
        if (this.$store.state.cyclomedia.orientation.xyz !== null) {
          const xyz = this.$store.state.cyclomedia.orientation.xyz;
          return [xyz[1], xyz[0]];
        } else {
          const center = this.$config.map.center;
          return center;
        }
      },
      cycloRotationAngle() {
        return this.$store.state.cyclomedia.orientation.yaw * (180/3.14159265359);
      },
      cycloHFov() {
        return this.$store.state.cyclomedia.orientation.hFov;
      },
      pictometryActive() {
        return this.$store.state.pictometry.active
      },
      pictometryZoom() {
        return this.$store.state.pictometry.zoom
      },
      pictometryShowAddressMarker() {
        if (!this.pictometryActive || !this.geocodeData) {
          return false;
        } else if (this.pictometryZoom < 20 && this.cyclomediaActive) {
          return false;
        } else {
          return true;
        }
      },
      geocodeData() {
        return this.$store.state.geocode.data
      },
      ak() {
        const host = window.location.hostname;
        if (host === 'atlas.phila.gov') {
          return this.$config.pictometry.apiKey;
        }
        if (host === 'atlas-dev.phila.gov') {
          return this.$config.pictometryDev.apiKey;
        }
        if (host === 'cityatlas.phila.gov') {
          return this.$config.pictometryCity.apiKey;
        }
        if (host === 'cityatlas-dev.phila.gov') {
          return this.$config.pictometryCityDev.apiKey;
        }
        if (host === '10.8.101.67') {
          return this.$config.pictometryLocal.apiKey;
        }
      },
      sk() {
        const host = window.location.hostname;
        if (host === 'atlas.phila.gov') {
          return this.$config.pictometry.secretKey;
        }
        if (host === 'atlas-dev.phila.gov') {
          return this.$config.pictometryDev.secretKey;
        }
        if (host === 'cityatlas.phila.gov') {
          return this.$config.pictometryCity.secretKey;
        }
        if (host === 'cityatlas-dev.phila.gov') {
          return this.$config.pictometryCityDev.secretKey;
        }
        if (host === '10.8.101.67') {
          return this.$config.pictometryLocal.secretKey;
        }
      }
    },
    watch: {
      pictometryShowAddressMarker(nextValue) {
        console.log('watch pictometryShowAddressMarker', nextValue);
      }
    },
    methods: {
      closeAddressCandidateList() {
        this.$store.commit('setShouldShowAddressCandidateList', false);
      },
      handleWindowResize() {
        // this only actually affects the size if it is set to "plugin mode"
        if ($(window).width() >= 750) {
          this.mbRootStyle.height = '600px'
        } else {
          this.mbRootStyle.height = 'auto';
        }
      }
    },
  };
</script>

<style>
  /*don't highlight any form elements*/
  input:focus,
  select:focus,
  textarea:focus,
  button:focus {
    outline: none;
  }

  .mb-panel-topics-with-widget {
    height: 50%;
  }

  /* standards applies padding to buttons, which causes some weirdness with
  buttons on the map panel. override here. */
  button {
    padding: inherit;
  }

  .topic-panel-false {
    /* display: none; */
  }

  @media screen and (min-width: 46.875em) {
    .topic-panel-false {
      display: none;
    }

    .map-panel-false {
      display: none;
    }
  }

</style>
