<template>
  <div class="home-page">
    <div class="matrix-section theme-dark">
      <div class="matrix-section-contents">
        <EnterpriseMatrix
          :techniques="matrixTechniques"
          :highlightedTechniques="highlightedTechniques"
          :selectedTechniques="selectedTechniques"
          :parentsWithSelectedChildren="parentsWithSelectedChildren"
          :parentsWithPredictedChildren="parentsWithPredictedChildren"
          :searchQuery="searchQuery"
          @technique-toggle="onTechniqueToggle"
          @search-change="onSearchChange"
          @clear-selection="onClearSelection"
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
    store: useInferenceEngineStore(),
    matrixTechniques: [] as Technique[],
    highlightedTechniques: {} as Record<string, { likelihood: number }>,
    selectedTechniques: {} as Record<string, boolean>,
    parentsWithSelectedChildren: {} as Record<string, boolean>,
    parentsWithPredictedChildren: {} as Record<string, { likelihood: number }>,
    observedTechniques: new Set<string>(),
    trainedTechniques: new Set<string>(),
    searchQuery: "",
    isLoading: false,
    predictionTime: ""
  }),
  computed: {
    selectedCount(): number {
      return Object.keys(this.selectedTechniques).length;
    }
  },
  methods: {
    async loadTechniques() {
      try {
        const enrichmentFile = await this.store.getEnrichmentFile;
        const trained = await this.store.getTrainedTechniques;
        this.trainedTechniques = trained;

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

    onSearchChange(query: string) {
      this.searchQuery = query;
    },

    async onTechniqueToggle(technique: { id: string }) {
      const id = technique.id;

      // Toggle selection
      if (this.observedTechniques.has(id)) {
        this.observedTechniques.delete(id);
      } else {
        this.observedTechniques.add(id);
      }

      // Clear search when a technique is selected
      this.searchQuery = "";

      // Update selectedTechniques object for reactivity
      this.updateSelectedTechniques();

      // Update predictions
      await this.updatePredictions();
    },

    async onClearSelection() {
      this.observedTechniques.clear();
      this.selectedTechniques = {};
      this.parentsWithSelectedChildren = {};
      this.parentsWithPredictedChildren = {};
      this.highlightedTechniques = {};
      this.predictionTime = "";
    },

    updateSelectedTechniques() {
      const selected: Record<string, boolean> = {};
      const parentsWithChildren: Record<string, boolean> = {};

      for (const id of this.observedTechniques) {
        selected[id] = true;
        // Track parent technique separately for cosmetic dropdown highlighting
        if (id.includes('.')) {
          const parentId = id.split('.')[0];
          parentsWithChildren[parentId] = true;
        }
      }

      this.selectedTechniques = selected;
      this.parentsWithSelectedChildren = parentsWithChildren;
    },

    async updatePredictions() {
      if (this.observedTechniques.size === 0) {
        this.highlightedTechniques = {};
        this.parentsWithPredictedChildren = {};
        this.predictionTime = "";
        return;
      }

      this.isLoading = true;

      try {
        // Filter to only trained techniques
        const trainedObserved = new Set(
          [...this.observedTechniques].filter(t => this.trainedTechniques.has(t))
        );

        if (trainedObserved.size === 0) {
          this.highlightedTechniques = {};
          this.parentsWithPredictedChildren = {};
          this.isLoading = false;
          return;
        }

        const predictions = await this.store.predictNewReport(trainedObserved);
        this.predictionTime = predictions.metadata.humanReadableTime;

        // Process predictions
        const TOP_N = 25;
        const highlighted: Record<string, { likelihood: number }> = {};
        const sortedPredictions = [...predictions.entries()]
          .sort((a, b) => a[1].rank - b[1].rank)
          .slice(0, TOP_N);

        const parentScores: Record<string, number> = {};

        for (const [id, technique] of sortedPredictions) {
          const topScore = sortedPredictions[0][1].likelihood;
          const relativeScore = topScore > 0 ? (technique.likelihood / topScore) * 100 : 0;

          if (id.includes('.')) {
            const parentId = id.split('.')[0];
            highlighted[id] = { likelihood: relativeScore };
            // Track parent scores separately for dropdown highlighting
            const existingParentScore = parentScores[parentId] || 0;
            if (relativeScore > existingParentScore) {
              parentScores[parentId] = relativeScore;
            }
          } else {
            highlighted[id] = { likelihood: relativeScore };
          }
        }

        // Store parent scores separately for dropdown highlighting (not in highlightedTechniques)
        const parentsWithPredicted: Record<string, { likelihood: number }> = {};
        for (const [parentId, score] of Object.entries(parentScores)) {
          parentsWithPredicted[parentId] = { likelihood: score };
        }
        this.parentsWithPredictedChildren = parentsWithPredicted;

        this.highlightedTechniques = highlighted;
      } catch (error) {
        console.error("Failed to update predictions:", error);
      } finally {
        this.isLoading = false;
      }
    }
  },
  async mounted() {
    await this.loadTechniques();
  },
  components: { RouterLink, EnterpriseMatrix }
});
</script>

<style lang="scss" scoped>
@use "@/assets/styles/engenuity_scaling_system.scss" as scale;

.home-page {
  display: flex;
  flex-direction: column;
  align-items: center;
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
