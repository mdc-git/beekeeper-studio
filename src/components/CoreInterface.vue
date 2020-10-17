<template>
  <div id="interface" class="interface">
    <div class="interface-wrap row">
      <sidebar ref="sidebar">
        <core-sidebar
          @databaseSelected="databaseSelected"
          :connection="connection"
        ></core-sidebar>
        <statusbar>
          <ConnectionButton></ConnectionButton>
        </statusbar>
      </sidebar>
      <div ref="content" class="page-content flex-col" id="page-content">
        <core-tabs
          :connection="connection"
          @selectedRow="selectedRow"
        ></core-tabs>
      </div>
      <!-- Columneditor -->
      <div class="wrapper" ref="columneditor" v-if="rowCount > 0" style="height: 100%; overflow-y: scroll; padding: 10px; padding-top: 40px">
        <div v-for="(value, propertyName) in activerow" :key="propertyName">
          <label :for="propertyName"><strong>{{ propertyName }}</strong></label>
          <ResizeAuto>
            <template>
              <textarea
                class="textarea"
                :value="value"
                :rows="1"
                :id="'columneditor_' + propertyName"
                disabled
                readonly
              ></textarea>
            </template>
          </ResizeAuto>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import Sidebar from "./common/Sidebar";
import ResizeAuto from "./common/ResizeAuto";
import CoreSidebar from "./sidebar/CoreSidebar";
import CoreTabs from "./CoreTabs";
import Split from "split.js";
import Statusbar from "./common/StatusBar";
import ConnectionButton from "./sidebar/core/ConnectionButton";
export default {
  components: {
    CoreSidebar,
    CoreTabs,
    Sidebar,
    Statusbar,
    ConnectionButton,
    ResizeAuto,
  },
  props: ["connection"],
  data() {
    return {
      split: null,
      activerow: null,
    };
  },
  computed: {
    splitElements() {
      return [this.$refs.sidebar.$refs.sidebar, this.$refs.content];
    },
    rowCount() {
      return this.activerow && Object.keys(this.activerow).length
        ? Object.keys(this.activerow).length
        : 0;
    },
  },
  mounted() {
    this.$store.dispatch("updateHistory");
    this.$store.dispatch("updateFavorites");

    this.$nextTick(() => {
      this.split = Split(this.splitElements, {
        elementStyle: (dimension, size) => ({
          "flex-basis": `calc(${size}%)`,
        }),
        sizes: [10, 90],
        minSize: 200,
        expandToMin: true,
        gutterSize: 8,
      });
    });
  },
  beforeDestroy() {
    if (this.split) {
      console.log("destroying split");
      this.split.destroy();
    }
  },
  methods: {
    selectedRow(obj) {
      this.activerow = Object.freeze(obj);
      console.log(obj);
    },
    databaseSelected(database) {
      this.$emit("databaseSelected", database);
    },
  },
};
</script>
