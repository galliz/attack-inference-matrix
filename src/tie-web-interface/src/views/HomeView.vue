<template>
  <div class="home-page">
    <div class="tool-set theme-dark">
      <div class="tool-set-contents" v-if="1 < tools.length">
        <div class="tool-tabs">
          <template v-for="(tool, i) of tools" :key="tool.component">
            <h6 :class="['tool-tab', { 'theme-light': activeTool === i }]" @click="activeTool = i">
              {{ tool.name }}
            </h6>
          </template>
        </div>
      </div>
    </div>
    <div class="active-tool">
      <div class="active-tool-contents">
        <component class="active-tool-component" :is="tools[activeTool].component" />
      </div>
    </div>
  </div>
</template>

<script lang="ts">
// Dependencies
import { defineComponent } from "vue";
// Components
import { RouterLink } from "vue-router";
import PredictTechniquesTool from "@/components/Elements/PredictTechniquesTool.vue";

export default defineComponent({
  name: "App",
  data: () => ({
    tools: [
      {
        name: "Predict Techniques",
        component: "PredictTechniquesTool"
      }
    ],
    activeTool: 0
  }),
  computed: {

  },
  methods: {

  },
  components: { RouterLink, PredictTechniquesTool }
});
</script>

<style lang="scss" scoped>
@use "@/assets/styles/engenuity_scaling_system.scss" as scale;

.home-page {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.tool-set,
.active-tool {
  display: flex;
  justify-content: center;
  width: 100%;
}

.tool-set-contents,
.active-tool-contents {
  display: flex;
  width: 100%;
  max-width: scale.$max-width;
}

.tool-set-contents {
  min-width: 0;
  padding: 0em scale.size("h");
}

.active-tool-contents {
  min-width: 0;
  padding: 0em scale.size("h");
}

.tool-tabs,
.active-tool-component {
  display: flex;
}

.active-tool-component {
  width: 100%;
  flex-direction: column;
}

.tool-tab {
  padding: scale.size("xs") scale.size("m");
  user-select: none;
  cursor: pointer;
}
</style>
