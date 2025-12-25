<template>
  <div class="enterprise-matrix-wrapper">
    <div class="matrix-header">
      <h3>ATT&CK Enterprise Matrix</h3>
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
                        <tr class="technique-row">
                          <td>
                            <div class="technique-wrapper">
                              <div
                                class="technique-cell"
                                :class="[getTechniqueClasses(technique.id), { 'has-subtechniques': technique.subtechniques && technique.subtechniques.length > 0 }]"
                                :data-technique-id="technique.id"
                                @click="$emit('technique-click', technique)"
                              >
                                <a
                                  :href="'https://attack.mitre.org/techniques/' + technique.id"
                                  target="_blank"
                                  :title="technique.id"
                                  @click.stop
                                >
                                  {{ technique.name }}
                                </a>
                              </div>
                              <!-- Subtechnique toggle button -->
                              <div
                                v-if="technique.subtechniques && technique.subtechniques.length > 0"
                                class="subtechnique-toggle"
                                :class="{ 'expanded': expandedTechniques.has(technique.id) }"
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
                                :class="getTechniqueClasses(sub.id)"
                                :data-technique-id="sub.id"
                                @click="$emit('technique-click', sub)"
                              >
                                <a
                                  :href="'https://attack.mitre.org/techniques/' + sub.id.replace('.', '/')"
                                  target="_blank"
                                  :title="sub.id"
                                  @click.stop
                                >
                                  {{ sub.name }}
                                </a>
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
    }
  },
  emits: ["technique-click"],
  data() {
    return {
      tactics: ENTERPRISE_TACTICS,
      expandedTechniques: new Set<string>()
    };
  },
  computed: {
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

  h3 {
    margin: 0;
    color: var(--engenuity-white, #fff);
    font-size: 1rem;
  }
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

  &.has-subtechniques {
    border-right: none;
  }

  &:hover {
    background: var(--engenuity-navy-light, #1a3a5c);
    border-left-color: var(--engenuity-accent, #c63f1f);
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
