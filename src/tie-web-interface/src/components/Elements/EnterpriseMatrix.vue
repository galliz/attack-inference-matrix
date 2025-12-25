<template>
  <div class="enterprise-matrix-wrapper">
    <div class="matrix-header">
      <div class="header-left">
        <h3>ATT&CK Enterprise Matrix</h3>
        <div class="search-container">
          <input
            type="text"
            class="search-input"
            placeholder="Search techniques..."
            :value="searchQuery"
            @input="$emit('search-change', ($event.target as HTMLInputElement).value)"
          />
          <span v-if="searchQuery" class="search-count">{{ matchingTechniqueCount }} matches</span>
        </div>
      </div>
      <div class="header-right">
        <div class="selection-info" v-if="selectedCount > 0">
          <span class="selection-count">{{ selectedCount }} selected</span>
          <button class="clear-btn" @click="$emit('clear-selection')">Clear</button>
        </div>
        <div class="matrix-legend">
          <div class="legend-item">
            <span class="legend-box selected-box"></span>
            <span class="legend-label">Selected</span>
          </div>
          <div class="legend-item">
            <span class="legend-gradient"></span>
            <span class="legend-label">Predicted Likelihood</span>
          </div>
        </div>
      </div>
    </div>
    <div class="matrix-container p-3">
      <div class="overflow-x-auto matrix-scroll-box pb-3" ref="matrixScrollBox">
          <table class="matrix side">
            <thead>
              <tr>
                <td class="tactic name" v-for="tactic in tactics" :key="tactic.id">
                  <a :href="'https://attack.mitre.org/tactics/' + tactic.id" target="_blank" :title="tactic.id">
                    {{ tactic.name }}
                  </a>
                </td>
              </tr>
              <tr>
                <td class="tactic count" v-for="tactic in tactics" :key="tactic.id + '-count'">
                  {{ getTechniqueCount(tactic.id) }}&nbsp;techniques
                </td>
              </tr>
            </thead>
            <tbody>
              <tr>
                <td class="tactic" v-for="tactic in tactics" :key="tactic.id + '-techniques'">
                  <table class="techniques-table">
                    <tbody>
                      <template
                        v-for="technique in getParentTechniquesForTactic(tactic.id)"
                        :key="technique.id"
                      >
                        <!-- Parent technique row -->
                        <tr class="technique-row" :class="{ 'search-hidden': !matchesSearch(technique) && searchQuery }">
                          <td>
                            <div class="technique-wrapper">
                              <div
                                class="technique-cell"
                                :class="[getTechniqueClasses(technique.id), { 'has-subtechniques': technique.subtechniques && technique.subtechniques.length > 0, 'search-match': matchesSearch(technique) && searchQuery }]"
                                :data-technique-id="technique.id"
                                @click="$emit('technique-toggle', technique)"
                              >
                                <span class="technique-name">{{ technique.name }}</span>
                                <span class="technique-id">{{ technique.id }}</span>
                              </div>
                              <!-- Subtechnique toggle button -->
                              <div
                                v-if="technique.subtechniques && technique.subtechniques.length > 0"
                                class="subtechnique-toggle"
                                :class="{ 'expanded': expandedTechniques.has(technique.id), 'has-selected-child': parentsWithSelectedChildren[technique.id], 'has-search-match-child': hasSubtechniqueSearchMatch(technique) }"
                                @click="toggleSubtechniques(technique.id)"
                              >
                                <span class="toggle-count">({{ technique.subtechniques.length }})</span>
                                <span class="toggle-arrow">▼</span>
                              </div>
                            </div>
                          </td>
                        </tr>
                        <!-- Subtechniques dropdown -->
                        <tr
                          v-if="technique.subtechniques && technique.subtechniques.length > 0 && expandedTechniques.has(technique.id)"
                          class="subtechniques-row"
                        >
                          <td>
                            <div class="subtechniques-container">
                              <div
                                v-for="sub in technique.subtechniques"
                                :key="sub.id"
                                class="technique-cell subtechnique"
                                :class="[getTechniqueClasses(sub.id), { 'search-match': matchesSearchSub(sub) && searchQuery, 'search-hidden': !matchesSearchSub(sub) && searchQuery }]"
                                :data-technique-id="sub.id"
                                @click="$emit('technique-toggle', sub)"
                              >
                                <span class="technique-name">{{ sub.name }}</span>
                                <span class="technique-id">{{ sub.id }}</span>
                              </div>
                            </div>
                          </td>
                        </tr>
                      </template>
                    </tbody>
                  </table>
                </td>
              </tr>
            </tbody>
          </table>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, type PropType } from "vue";

// Define tactic structure
interface Tactic {
  id: string;
  name: string;
}

// Define technique structure
interface Technique {
  id: string;
  name: string;
  tactics: string[];
  subtechniques?: Technique[];
  likelihood?: number;
}

// Technique with grouped subtechniques
interface TechniqueWithSubs extends Technique {
  subtechniques: Technique[];
}

// Enterprise ATT&CK Tactics in order (with names for matching)
const ENTERPRISE_TACTICS: Tactic[] = [
  { id: "TA0043", name: "Reconnaissance" },
  { id: "TA0042", name: "Resource Development" },
  { id: "TA0001", name: "Initial Access" },
  { id: "TA0002", name: "Execution" },
  { id: "TA0003", name: "Persistence" },
  { id: "TA0004", name: "Privilege Escalation" },
  { id: "TA0005", name: "Defense Evasion" },
  { id: "TA0006", name: "Credential Access" },
  { id: "TA0007", name: "Discovery" },
  { id: "TA0008", name: "Lateral Movement" },
  { id: "TA0009", name: "Collection" },
  { id: "TA0011", name: "Command and Control" },
  { id: "TA0010", name: "Exfiltration" },
  { id: "TA0040", name: "Impact" }
];

// Create a map of tactic names to IDs for matching
const TACTIC_NAME_TO_ID: Record<string, string> = {};
ENTERPRISE_TACTICS.forEach(t => {
  TACTIC_NAME_TO_ID[t.name] = t.id;
});

export default defineComponent({
  name: "EnterpriseMatrix",
  props: {
    techniques: {
      type: Array as PropType<Technique[]>,
      default: () => []
    },
    highlightedTechniques: {
      type: Object as PropType<Record<string, { likelihood: number }>>,
      default: () => ({})
    },
    selectedTechniques: {
      type: Object as PropType<Record<string, boolean>>,
      default: () => ({})
    },
    parentsWithSelectedChildren: {
      type: Object as PropType<Record<string, boolean>>,
      default: () => ({})
    },
    searchQuery: {
      type: String,
      default: ""
    }
  },
  emits: ["technique-toggle", "search-change", "clear-selection"],
  data() {
    return {
      tactics: ENTERPRISE_TACTICS,
      expandedTechniques: new Set<string>()
    };
  },
  computed: {
    selectedCount(): number {
      return Object.keys(this.selectedTechniques).length;
    },

    matchingTechniqueCount(): number {
      if (!this.searchQuery) return 0;
      const query = this.searchQuery.toLowerCase();
      let count = 0;
      for (const technique of this.techniques) {
        if (
          technique.name.toLowerCase().includes(query) ||
          technique.id.toLowerCase().includes(query)
        ) {
          count++;
        }
      }
      return count;
    },

    // Group techniques: parent techniques contain their subtechniques
    groupedTechniquesByTactic(): Map<string, TechniqueWithSubs[]> {
      const map = new Map<string, TechniqueWithSubs[]>();

      // Initialize empty arrays for each tactic
      for (const tactic of this.tactics) {
        map.set(tactic.id, []);
      }

      // Separate parent techniques and subtechniques
      const parentTechniques: Map<string, TechniqueWithSubs> = new Map();
      const subtechniques: Technique[] = [];

      for (const technique of this.techniques) {
        // Check if it's a subtechnique (contains a dot like T1595.001)
        if (technique.id.includes('.')) {
          subtechniques.push(technique);
        } else {
          // It's a parent technique
          parentTechniques.set(technique.id, { ...technique, subtechniques: [] });
        }
      }

      // Assign subtechniques to their parents
      for (const sub of subtechniques) {
        const parentId = sub.id.split('.')[0]; // T1595.001 -> T1595
        const parent = parentTechniques.get(parentId);
        if (parent) {
          parent.subtechniques.push(sub);
        }
      }

      // Group parent techniques by their tactics
      for (const [, technique] of parentTechniques) {
        for (const tacticName of technique.tactics) {
          const tacticId = TACTIC_NAME_TO_ID[tacticName];
          if (tacticId) {
            const tacticTechniques = map.get(tacticId);
            if (tacticTechniques) {
              tacticTechniques.push(technique);
            }
          }
        }
      }

      // Sort techniques alphabetically by name within each tactic
      for (const [tacticId, techniques] of map) {
        techniques.sort((a, b) => a.name.localeCompare(b.name));
        // Also sort subtechniques
        for (const tech of techniques) {
          tech.subtechniques.sort((a, b) => a.name.localeCompare(b.name));
        }
      }

      return map;
    },

    // Sort techniques with highlighted ones at the top, ordered by likelihood
    sortedTechniquesByTactic(): Map<string, TechniqueWithSubs[]> {
      const baseMap = this.groupedTechniquesByTactic;
      const sortedMap = new Map<string, TechniqueWithSubs[]>();

      for (const [tacticId, techniques] of baseMap) {
        const sorted = [...techniques].sort((a, b) => {
          const aHighlight = this.highlightedTechniques[a.id];
          const bHighlight = this.highlightedTechniques[b.id];

          // Both highlighted: sort by likelihood (highest first)
          if (aHighlight && bHighlight) {
            return bHighlight.likelihood - aHighlight.likelihood;
          }
          // Only a is highlighted: a comes first
          if (aHighlight && !bHighlight) {
            return -1;
          }
          // Only b is highlighted: b comes first
          if (!aHighlight && bHighlight) {
            return 1;
          }
          // Neither highlighted: sort alphabetically
          return a.name.localeCompare(b.name);
        });

        // Also sort subtechniques within each technique
        for (const tech of sorted) {
          tech.subtechniques = [...tech.subtechniques].sort((a, b) => {
            const aHighlight = this.highlightedTechniques[a.id];
            const bHighlight = this.highlightedTechniques[b.id];

            if (aHighlight && bHighlight) {
              return bHighlight.likelihood - aHighlight.likelihood;
            }
            if (aHighlight && !bHighlight) {
              return -1;
            }
            if (!aHighlight && bHighlight) {
              return 1;
            }
            return a.name.localeCompare(b.name);
          });
        }

        sortedMap.set(tacticId, sorted);
      }

      return sortedMap;
    }
  },
  methods: {
    getTechniqueCount(tacticId: string): number {
      return this.groupedTechniquesByTactic.get(tacticId)?.length || 0;
    },

    getParentTechniquesForTactic(tacticId: string): TechniqueWithSubs[] {
      return this.sortedTechniquesByTactic.get(tacticId) || [];
    },

    matchesSearch(technique: Technique): boolean {
      if (!this.searchQuery) return true;
      const query = this.searchQuery.toLowerCase();
      // Check if technique name or ID matches
      if (
        technique.name.toLowerCase().includes(query) ||
        technique.id.toLowerCase().includes(query)
      ) {
        return true;
      }
      // Check if any subtechnique matches
      const techWithSubs = technique as TechniqueWithSubs;
      if (techWithSubs.subtechniques) {
        for (const sub of techWithSubs.subtechniques) {
          if (
            sub.name.toLowerCase().includes(query) ||
            sub.id.toLowerCase().includes(query)
          ) {
            return true;
          }
        }
      }
      return false;
    },

    matchesSearchSub(sub: Technique): boolean {
      if (!this.searchQuery) return true;
      const query = this.searchQuery.toLowerCase();
      return (
        sub.name.toLowerCase().includes(query) ||
        sub.id.toLowerCase().includes(query)
      );
    },

    hasSubtechniqueSearchMatch(technique: TechniqueWithSubs): boolean {
      if (!this.searchQuery) return false;
      const query = this.searchQuery.toLowerCase();
      // Check if any subtechnique matches the search
      for (const sub of technique.subtechniques) {
        if (
          sub.name.toLowerCase().includes(query) ||
          sub.id.toLowerCase().includes(query)
        ) {
          return true;
        }
      }
      return false;
    },

    toggleSubtechniques(techniqueId: string): void {
      if (this.expandedTechniques.has(techniqueId)) {
        this.expandedTechniques.delete(techniqueId);
      } else {
        this.expandedTechniques.add(techniqueId);
      }
      // Force reactivity update
      this.expandedTechniques = new Set(this.expandedTechniques);
    },

    getTechniqueClasses(techniqueId: string): Record<string, boolean> {
      const highlighted = this.highlightedTechniques[techniqueId];
      const isSelected = this.selectedTechniques[techniqueId];

      const classes: Record<string, boolean> = {};

      // Selected techniques (user-chosen) get a special highlight
      if (isSelected) {
        classes['selected'] = true;
      }

      // Predicted techniques get heat map coloring
      if (highlighted) {
        const likelihood = highlighted.likelihood;
        classes['likelihood-high'] = likelihood >= 70;
        classes['likelihood-medium'] = likelihood >= 40 && likelihood < 70;
        classes['likelihood-low'] = likelihood > 0 && likelihood < 40;
      }

      return classes;
    }
  }
});
</script>

<style lang="scss" scoped>
@use "@/assets/styles/engenuity_scaling_system.scss" as scale;

.enterprise-matrix-wrapper {
  width: 100%;
  max-width: 100%;
  overflow: hidden;
}

.matrix-header {
  padding: scale.size("xs") scale.size("m");
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  flex-wrap: wrap;
  gap: 0.75rem;

  h3 {
    margin: 0;
    color: var(--engenuity-white, #fff);
    font-size: 1rem;
  }
}

.header-left {
  display: flex;
  align-items: center;
  gap: 1.5rem;
  flex-wrap: wrap;
}

.header-right {
  display: flex;
  align-items: center;
  gap: 1.5rem;
  flex-wrap: wrap;
}

.search-container {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.search-input {
  background: rgba(255, 255, 255, 0.1);
  border: 1px solid rgba(255, 255, 255, 0.2);
  border-radius: 4px;
  padding: 0.35rem 0.75rem;
  color: var(--engenuity-white, #fff);
  font-size: 0.8rem;
  width: 200px;
  transition: all 0.2s ease;

  &::placeholder {
    color: rgba(255, 255, 255, 0.5);
  }

  &:focus {
    outline: none;
    border-color: var(--engenuity-accent, #c63f1f);
    background: rgba(255, 255, 255, 0.15);
  }
}

.search-count {
  font-size: 0.7rem;
  color: var(--engenuity-gray, #888);
}

.selection-info {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  background: rgba(0, 150, 209, 0.2);
  padding: 0.3rem 0.6rem;
  border-radius: 4px;
  border: 1px solid rgba(0, 150, 209, 0.4);
}

.selection-count {
  font-size: 0.75rem;
  color: #0096d1;
  font-weight: 600;
}

.clear-btn {
  background: transparent;
  border: 1px solid rgba(0, 150, 209, 0.6);
  color: #0096d1;
  padding: 0.15rem 0.5rem;
  border-radius: 3px;
  font-size: 0.65rem;
  cursor: pointer;
  transition: all 0.2s ease;

  &:hover {
    background: rgba(0, 150, 209, 0.2);
  }
}

.matrix-legend {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.legend-item {
  display: flex;
  align-items: center;
  gap: 0.4rem;
}

.legend-box {
  width: 14px;
  height: 14px;
  border-radius: 2px;
}

.selected-box {
  background: rgba(0, 150, 209, 0.6);
  border: 1px solid #0096d1;
}

.legend-gradient {
  width: 60px;
  height: 14px;
  border-radius: 2px;
  background: linear-gradient(to right,
    rgba(198, 63, 31, 0.18) 0%,
    rgba(198, 63, 31, 0.35) 50%,
    rgba(198, 63, 31, 0.55) 100%
  );
  border: 1px solid rgba(198, 63, 31, 0.7);
}

.legend-label {
  font-size: 0.7rem;
  color: var(--engenuity-white, #ccc);
}

.matrix-container {
  padding: 0.5rem;
  background: var(--engenuity-navy, #0a1628);
  border-radius: 4px;
  overflow: hidden;
}

.matrix-scroll-box {
  overflow-x: auto;
  overflow-y: auto;
  max-height: 400px;
  padding-bottom: 0.5rem;
}

// Main table structure
table.matrix {
  border-collapse: separate;
  border-spacing: 1px;
  width: max-content;
  table-layout: fixed;

  &.side {
    width: 100%;
    min-width: max-content;
  }
}

// Tactic headers (column headers)
td.tactic {
  vertical-align: top;
  padding: 0;
  width: 110px;
  min-width: 110px;
  max-width: 110px;
  overflow: hidden;

  &.name {
    background: var(--engenuity-dark-navy, #0d1f3c);
    padding: 4px 6px;
    text-align: center;
    font-weight: 600;
    border-bottom: 2px solid var(--engenuity-accent, #c63f1f);

    a {
      color: var(--engenuity-white, #fff);
      text-decoration: none;
      font-size: 0.7rem;
      line-height: 1.2;
      word-wrap: break-word;

      &:hover {
        text-decoration: underline;
        color: var(--engenuity-accent, #c63f1f);
      }
    }
  }

  &.count {
    background: var(--engenuity-navy, #0a1628);
    padding: 2px 4px;
    text-align: center;
    font-size: 0.65rem;
    color: var(--engenuity-gray, #888);
  }
}

// Techniques table within each tactic column
.techniques-table {
  width: 100%;
  max-width: 100%;
  border-collapse: collapse;
  table-layout: fixed;
}

// Technique rows
.technique-row {
  display: table-row;

  &.search-hidden {
    opacity: 0.3;
  }

  > td {
    vertical-align: top;
    width: 100%;
    overflow: hidden;
  }
}

// Technique cell styling
.technique-wrapper {
  display: flex;
  align-items: stretch;
  margin: 1px 0;
  max-width: 100%;
  overflow: hidden;
}

.technique-cell {
  background: var(--engenuity-dark-navy, #0d1f3c);
  padding: 3px 4px;
  border-left: 2px solid transparent;
  transition: all 0.2s ease;
  cursor: pointer;
  flex: 1;
  min-width: 0;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  gap: 1px;

  &.has-subtechniques {
    border-right: none;
  }

  &:hover {
    background: var(--engenuity-navy-light, #1a3a5c);
    border-left-color: var(--engenuity-accent, #c63f1f);
  }

  &.search-match {
    box-shadow: inset 0 0 0 2px rgba(255, 193, 7, 0.6);
  }

  &.search-hidden {
    opacity: 0.3;
  }

  .technique-name {
    color: var(--engenuity-white, #eee);
    font-size: 0.65rem;
    line-height: 1.3;
    display: block;
    word-wrap: break-word;
    overflow-wrap: break-word;
    hyphens: auto;
  }

  .technique-id {
    color: var(--engenuity-gray, #888);
    font-size: 0.55rem;
  }

  a {
    color: var(--engenuity-white, #eee);
    text-decoration: none;
    font-size: 0.65rem;
    line-height: 1.3;
    display: block;
    word-wrap: break-word;
    overflow-wrap: break-word;
    hyphens: auto;

    &:hover {
      color: var(--engenuity-white, #fff);
    }

    sub {
      color: var(--engenuity-gray, #888);
      font-size: 0.6rem;
    }
  }

  // Heat map coloring for predicted techniques
  // Uses the engenuity accent color (red/orange) at varying intensities
  // Higher likelihood = more intense/visible = "hotter"

  &.likelihood-high {
    // High likelihood: intense, fully saturated
    background: rgba(198, 63, 31, 0.55);
    border-left-color: #c63f1f;

    &:hover {
      background: rgba(198, 63, 31, 0.7);
    }
  }

  &.likelihood-medium {
    // Medium likelihood: moderate intensity
    background: rgba(198, 63, 31, 0.35);
    border-left-color: rgba(198, 63, 31, 0.7);

    &:hover {
      background: rgba(198, 63, 31, 0.45);
    }
  }

  &.likelihood-low {
    // Low likelihood: faded, subtle
    background: rgba(198, 63, 31, 0.18);
    border-left-color: rgba(198, 63, 31, 0.4);

    &:hover {
      background: rgba(198, 63, 31, 0.28);
    }
  }

  // Selected techniques (user-chosen observed techniques)
  // Bright cyan/blue color to stand out from predictions
  &.selected {
    background: rgba(0, 150, 209, 0.5);
    border-left-color: #0096d1;
    border-left-width: 3px;

    &:hover {
      background: rgba(0, 150, 209, 0.65);
    }

    a {
      color: #fff;
      font-weight: 600;
    }
  }
}

// Subtechnique toggle button
.subtechnique-toggle {
  display: flex;
  align-items: center;
  justify-content: center;
  background: var(--engenuity-dark-navy, #0d1f3c);
  padding: 2px 4px;
  cursor: pointer;
  border-left: 1px solid rgba(255, 255, 255, 0.1);
  transition: all 0.2s ease;

  &:hover {
    background: var(--engenuity-navy-light, #1a3a5c);
  }

  .toggle-count {
    font-size: 0.6rem;
    color: var(--engenuity-gray, #888);
    margin-right: 2px;
  }

  .toggle-arrow {
    font-size: 0.5rem;
    color: var(--engenuity-gray, #888);
    transition: transform 0.2s ease;
  }

  &.expanded .toggle-arrow {
    transform: rotate(180deg);
  }

  // Highlight dropdown when a child subtechnique is selected
  &.has-selected-child {
    background: rgba(0, 150, 209, 0.4);
    border-left-color: #0096d1;

    .toggle-count,
    .toggle-arrow {
      color: #0096d1;
      font-weight: 600;
    }

    &:hover {
      background: rgba(0, 150, 209, 0.5);
    }
  }

  // Yellow indicator when a subtechnique matches the search
  &.has-search-match-child {
    background: rgba(255, 193, 7, 0.3);
    border-left-color: #ffc107;

    .toggle-count,
    .toggle-arrow {
      color: #ffc107;
      font-weight: 600;
    }

    &:hover {
      background: rgba(255, 193, 7, 0.4);
    }
  }

  // Selected takes priority over search match
  &.has-selected-child.has-search-match-child {
    background: rgba(0, 150, 209, 0.4);
    border-left-color: #0096d1;

    .toggle-count,
    .toggle-arrow {
      color: #0096d1;
    }
  }
}

// Subtechniques container
.subtechniques-row {
  td {
    padding: 0;
  }
}

.subtechniques-container {
  background: rgba(0, 0, 0, 0.2);
  padding: 1px 0 1px 8px;
  border-left: 2px solid var(--engenuity-accent, #c63f1f);
  margin-left: 2px;

  .technique-cell.subtechnique {
    background: var(--engenuity-navy, #0a1628);
    margin: 1px 0;
    padding: 2px 4px;
    border-left: 2px solid transparent;

    &:hover {
      background: var(--engenuity-navy-light, #1a3a5c);
      border-left-color: var(--engenuity-accent, #c63f1f);
    }

    // Heat map coloring for predicted subtechniques (must override default background)
    &.likelihood-high {
      background: rgba(198, 63, 31, 0.55);
      border-left-color: #c63f1f;

      &:hover {
        background: rgba(198, 63, 31, 0.7);
      }
    }

    &.likelihood-medium {
      background: rgba(198, 63, 31, 0.35);
      border-left-color: rgba(198, 63, 31, 0.7);

      &:hover {
        background: rgba(198, 63, 31, 0.45);
      }
    }

    &.likelihood-low {
      background: rgba(198, 63, 31, 0.18);
      border-left-color: rgba(198, 63, 31, 0.4);

      &:hover {
        background: rgba(198, 63, 31, 0.28);
      }
    }

    // Selected subtechniques (user-chosen observed techniques)
    &.selected {
      background: rgba(0, 150, 209, 0.5);
      border-left-color: #0096d1;
      border-left-width: 3px;

      &:hover {
        background: rgba(0, 150, 209, 0.65);
      }

      a {
        color: #fff;
        font-weight: 600;
      }
    }

    a {
      font-size: 0.6rem;
    }
  }
}

// Scroll indicators
</style>
