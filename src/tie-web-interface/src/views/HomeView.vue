<template>
  <div class="home-page">
    <div class="matrix-section theme-dark">
      <div class="matrix-section-contents">
        <EnterpriseMatrix
          :techniques="matrixTechniques"
          :highlightedTechniques="highlightedTechniques"
        />
      </div>
    </div>
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
import EnterpriseMatrix from "@/components/Elements/EnterpriseMatrix.vue";
// Store
import { useInferenceEngineStore } from "@/stores/InferenceEngineStore";

interface Technique {
  id: string;
  name: string;
  tactics: string[];
}

export default defineComponent({
  name: "App",
  data: () => ({
    tools: [
      {
        name: "Predict Techniques",
        component: "PredictTechniquesTool"
      }
    ],
    activeTool: 0,
    matrixTechniques: [] as Technique[],
    highlightedTechniques: new Map<string, { likelihood: number }>()
  }),
  computed: {

  },
  methods: {
    async loadTechniques() {
      const store = useInferenceEngineStore();
      try {
        const enrichmentFile = await store.getEnrichmentFile;
        // Convert enrichment file techniques object to array
        const techniques: Technique[] = [];
        for (const [id, technique] of Object.entries(enrichmentFile.techniques)) {
          techniques.push({
            id: (technique as any).id,
            name: (technique as any).name,
            tactics: (technique as any).tactics || []
          });
        }
        this.matrixTechniques = techniques;
      } catch (error) {
        console.error("Failed to load techniques:", error);
      }
    }
  },
  async mounted() {
    await this.loadTechniques();
  },
  components: { RouterLink, PredictTechniquesTool, EnterpriseMatrix }
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

.matrix-section {
  display: flex;
  justify-content: center;
  width: 100%;
  background: var(--engenuity-navy, #0a1628);
  padding: scale.size("m") 0;
}

.matrix-section-contents {
  width: 100%;
  max-width: 100%;
  padding: 0em scale.size("h");
  overflow: hidden;
}
</style>
