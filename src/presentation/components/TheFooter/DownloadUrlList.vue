<template>
  <span
    class="container"
    v-bind:class="{
      'container-unsupported': !hasCurrentOsDesktopVersion,
      'container-supported': hasCurrentOsDesktopVersion,
    }">
    <span class="description">
      <font-awesome-icon class="description__icon" :icon="['fas', 'desktop']" />
      <span class="description__text">For desktop:</span>
    </span>
    <span class="urls">
      <span class="urls__url" v-for="os of supportedDesktops" v-bind:key="os">
        <DownloadUrlListItem :operatingSystem="os" />
      </span>
    </span>
  </span>
</template>

<script lang="ts">
import { defineComponent, inject } from 'vue';
import { OperatingSystem } from '@/domain/OperatingSystem';
import { useEnvironmentKey } from '@/presentation/injectionSymbols';
import DownloadUrlListItem from './DownloadUrlListItem.vue';

const supportedOperativeSystems: readonly OperatingSystem[] = [
  OperatingSystem.Windows,
  OperatingSystem.Linux,
  OperatingSystem.macOS,
];

export default defineComponent({
  components: {
    DownloadUrlListItem,
  },
  setup() {
    const { os: currentOs } = inject(useEnvironmentKey);
    const supportedDesktops = [
      ...supportedOperativeSystems,
    ].sort((os) => (os === currentOs ? 0 : 1));

    const hasCurrentOsDesktopVersion = supportedOperativeSystems.includes(currentOs);

    return {
      supportedDesktops,
      hasCurrentOsDesktopVersion,
    };
  },
});
</script>

<style scoped lang="scss">
@use "@/presentation/assets/styles/main" as *;

.container {
  display:flex;
  flex-wrap: wrap;
  justify-content: space-around;
  &-unsupported {
    opacity: $color-primary-light;
  }
  &-supported {
    font-size: 1em;
  }
  .description {
    &__icon {
      margin-right: 0.5em;
    }
    &__text {
      margin-right: 0.3em;
    }
  }
}
.urls {
  &__url {
    &:not(:first-child)::before {
      content: "|";
      font-size: 0.6rem;
      padding: 0 5px;
    }
  }
}
</style>
