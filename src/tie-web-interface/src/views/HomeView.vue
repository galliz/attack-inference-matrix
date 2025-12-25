<template>
  <div class="home-page">
    <div class="matrix-section theme-dark">
      <div class="matrix-section-contents">
        <EnterpriseMatrix
          :techniques="matrixTechniques"
          :highlightedTechniques="highlightedTechniques"
          :selectedTechniques="selectedTechniques"
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
        <component
          class="active-tool-component"
          :is="tools[activeTool].component"
          @predictions-updated="onPredictionsUpdated"
          @observed-updated="onObservedUpdated"
        />
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
    highlightedTechniques: {} as Record<string, { likelihood: number }>,
    selectedTechniques: {} as Record<string, boolean>
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
    },
    onPredictionsUpdated(predictions: Map<string, { likelihood: number, rank: number }> | null) {
      if (predictions) {
        // Only highlight the top 25 predictions by rank
        const TOP_N = 25;
        const highlighted: Record<string, { likelihood: number }> = {};

        // Convert to array, sort by rank, and take top N
        const sortedPredictions = [...predictions.entries()]
          .sort((a, b) => a[1].rank - b[1].rank)
          .slice(0, TOP_N);

        // Track parent technique scores (aggregate from subtechniques)
        const parentScores: Record<string, number> = {};

        for (const [id, technique] of sortedPredictions) {
          // Recalculate likelihood relative to the top predictions only
          const topScore = sortedPredictions[0][1].likelihood;
          const relativeScore = topScore > 0 ? (technique.likelihood / topScore) * 100 : 0;

          // Check if this is a subtechnique (contains a dot, e.g., T1204.002)
          if (id.includes('.')) {
            const parentId = id.split('.')[0]; // T1204.002 -> T1204

            // Store subtechnique score
            highlighted[id] = { likelihood: relativeScore };

            // Aggregate to parent: use the maximum subtechnique score
            const existingParentScore = parentScores[parentId] || 0;
            if (relativeScore > existingParentScore) {
              parentScores[parentId] = relativeScore;
            }
          } else {
            // Regular technique (not a subtechnique)
            highlighted[id] = { likelihood: relativeScore };
          }
        }

        // Add parent techniques with their aggregated scores
        for (const parentId of Object.keys(parentScores)) {
          const score = parentScores[parentId];
          // Only set if not already set (direct prediction takes priority)
          if (!highlighted[parentId]) {
            highlighted[parentId] = { likelihood: score };
          } else {
            // If parent was directly predicted, use the higher of the two scores
            if (score > highlighted[parentId].likelihood) {
              highlighted[parentId] = { likelihood: score };
            }
          }
        }

        this.highlightedTechniques = highlighted;
      } else {
        // Clear highlights when no predictions
        this.highlightedTechniques = {};
      }
    },
    onObservedUpdated(observed: string[]) {
      // Convert observed techniques array to a lookup object
      const selected: Record<string, boolean> = {};
      for (const id of observed) {
        selected[id] = true;
        // Also mark parent technique if this is a subtechnique
        if (id.includes('.')) {
          const parentId = id.split('.')[0];
          selected[parentId] = true;
        }
      }
      this.selectedTechniques = selected;
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
