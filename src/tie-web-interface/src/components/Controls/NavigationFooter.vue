<template>
  <div class="navigation-footer-control">
    <div class="navigation-footer-main theme-dark">
      <div class="navigation-footer-contents">
        <div class="footer-row">
          <span class="copyright">
            Based on <a href="https://github.com/center-for-threat-informed-defense/technique-inference-engine" target="_blank">Technique Inference Engine</a>
            by <a href="https://ctid.mitre.org/" target="_blank">MITRE Engenuity Center for Threat-Informed Defense</a>
          </span>
          <span class="separator">·</span>
          <a href="https://github.com/center-for-threat-informed-defense/technique-inference-engine/blob/main/LICENSE.txt" target="_blank">Apache 2.0 License</a>
          <span class="separator">·</span>
          <span class="copyright">© 2024 MITRE</span>
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
// Dependencies
import { defineComponent } from "vue";

export default defineComponent({
  name: "NavigationFooter"
});
</script>

<style lang="scss" scoped>
@use "@/assets/styles/engenuity_scaling_system" as scale;

.navigation-footer-main {
  display: flex;
  align-items: center;
  justify-content: center;
  min-width: 0;
  padding: scale.size("m") scale.size("h");
}

.navigation-footer-contents {
  display: flex;
  flex-direction: column;
  width: 100%;
  max-width: scale.$max-width;
}

.footer-row {
  display: flex;
  align-items: center;
  justify-content: center;
  flex-wrap: wrap;
  gap: 0.5rem;
  font-size: 0.75rem;
  color: var(--engenuity-gray, #888);
}

.footer-row a {
  color: var(--engenuity-gray, #aaa);
  text-decoration: none;

  &:hover {
    color: var(--engenuity-white, #fff);
    text-decoration: underline;
  }
}

.separator {
  color: var(--engenuity-gray, #666);
}

.copyright {
  color: var(--engenuity-gray, #888);
}
</style>
