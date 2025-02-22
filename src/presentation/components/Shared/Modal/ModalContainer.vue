<template>
  <div
    v-if="isRendered"
    class="modal-container"
  >
    <ModalOverlay
      @transitionedOut="onOverlayTransitionedOut"
      @click="onBackgroundOverlayClick"
      :show="isOpen"
    />
    <ModalContent
      class="modal-content"
      :show="isOpen"
      @transitionedOut="onContentTransitionedOut"
    >
      <slot />
    </ModalContent>
  </div>
</template>

<script lang="ts">
import {
  defineComponent, ref, watchEffect, nextTick,
} from 'vue';
import ModalOverlay from './ModalOverlay.vue';
import ModalContent from './ModalContent.vue';
import { useLockBodyBackgroundScroll } from './Hooks/UseLockBodyBackgroundScroll';
import { useCurrentFocusToggle } from './Hooks/UseCurrentFocusToggle';
import { useEscapeKeyListener } from './Hooks/UseEscapeKeyListener';
import { useAllTrueWatcher } from './Hooks/UseAllTrueWatcher';

export default defineComponent({
  components: {
    ModalOverlay,
    ModalContent,
  },
  emits: {
    /* eslint-disable @typescript-eslint/no-unused-vars */
    input: (isOpen: boolean) => true,
    /* eslint-enable @typescript-eslint/no-unused-vars */
  },
  props: {
    value: {
      type: Boolean,
      required: true,
    },
    closeOnOutsideClick: {
      type: Boolean,
      default: true,
    },
  },
  setup(props, { emit }) {
    const isRendered = ref(false);
    const isOpen = ref(false);
    const overlayTransitionedOut = ref(false);
    const contentTransitionedOut = ref(false);

    useLockBodyBackgroundScroll(isOpen);
    useCurrentFocusToggle(isOpen);
    useEscapeKeyListener(() => handleEscapeKeyUp());

    const {
      onAllConditionsMet: onModalFullyTransitionedOut,
      resetAllConditions: resetTransitionStatus,
    } = useAllTrueWatcher(overlayTransitionedOut, contentTransitionedOut);

    onModalFullyTransitionedOut(() => {
      isRendered.value = false;
      resetTransitionStatus();
      if (props.value) {
        emit('input', false);
      }
    });

    watchEffect(() => {
      if (props.value) {
        open();
      } else {
        close();
      }
    });

    function onOverlayTransitionedOut() {
      overlayTransitionedOut.value = true;
    }

    function onContentTransitionedOut() {
      contentTransitionedOut.value = true;
    }

    function handleEscapeKeyUp() {
      close();
    }

    function close() {
      if (!isRendered.value) {
        return;
      }

      isOpen.value = false;

      if (props.value) {
        emit('input', false);
      }
    }

    function open() {
      if (isRendered.value) {
        return;
      }

      isRendered.value = true;

      nextTick(() => { // Let the modal render first
        isOpen.value = true;
      });

      if (!props.value) {
        emit('input', true);
      }
    }

    function onBackgroundOverlayClick() {
      if (props.closeOnOutsideClick) {
        close();
      }
    }

    return {
      isRendered,
      onBackgroundOverlayClick,
      onOverlayTransitionedOut,
      onContentTransitionedOut,
      isOpen,
    };
  },
});
</script>

<style scoped lang="scss">
.modal-container {
  position: fixed;
  box-sizing: border-box;
  left: 0;
  top: 0;
  width: 100%;
  height: 100vh;
  z-index: 999;
}
</style>
