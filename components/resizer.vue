<template lang="pug">
#app-wrapper(
  v-bind:style="appStyles",
  @mousemove="mouseMove",
  @mouseup="deregisterMouseMove",
  @mouseleave="deregisterMouseMove"
)
  #tab-resize-guide(v-bind:style="guideStyles")
  .left-resize-container
    slot(name="left")
  #tab-resize(
    @mousedown.left="registerMouseMove",
    v-show="isTabActive"
  )
    .tab-close(v-on:click="$emit('close-tab')")
      i.fas.fa-caret-right
  .right-resize-container(
    v-bind:style="rightContainerStyles",
    v-if="isTabActive"
  )
    slot(name="right")
</template>

<script>
const DRAG_BAR_WIDTH = 13.25;
const SCROLL_BAR_WIDTH = 17;
const GUIDE_BAR_WIDTH = 2;

const throttledEvent = (delay, handler) => {
  let lastCalled = 0;
  return (...args) => {
    if (Date.now() - lastCalled > delay) {
      lastCalled = Date.now();
      handler(...args);
    }
  };
};

export default {
  props: {
    isTabActive: true,
  },

  data() {
    return {
      guideWidth: (0.5 * window.innerWidth - (GUIDE_BAR_WIDTH / 2)) / window.innerWidth,
      flexWidth: 0.5,
      isResizing: false,
    };
  },

  methods: {
    registerMouseMove() {
      this.isResizing = true;
    },

    deregisterMouseMove() {
      this.isResizing = false;
      this.flexWidth = (this.guideWidth * window.innerWidth + (GUIDE_BAR_WIDTH / 2))
        / window.innerWidth;
    },
  },

  computed: {
    appStyles() {
      return this.isResizing
        ? 'user-select: none; cursor: col-resize;'
        : '';
    },

    guideStyles() {
      return this.isResizing
        ? `display: block; right: ${this.guideWidth * 100}%;`
        : '';
    },

    rightContainerStyles() {
      return `flex: 0 0 ${this.flexWidth * 100}%;`;
    },

    mouseMove() {
      if (this.isResizing) {
        return throttledEvent(25, (event) => {
          this.guideWidth = (
            Math.min(
                Math.max(
                    window.innerWidth - event.clientX,
                    SCROLL_BAR_WIDTH + DRAG_BAR_WIDTH,
                ),
                window.innerWidth - SCROLL_BAR_WIDTH,
            )
            - (GUIDE_BAR_WIDTH / 2)
          ) / window.innerWidth;
        });
      }
      return () => {};
    },
  },
}
</script>

<style lang="scss" scoped>
#app-wrapper {
  display: flex;
  height: 100vh;
  left: 0;
  position: absolute;
  top: 0;
  width: 100%;
  z-index: z-index('app-wrapper');

  #tab-resize {
    background-color: mui-color('black');
    cursor: col-resize;
    height: 100%;
    width: 13.250px;
    z-index: z-index('center-divider');
  }

  .tab-close {
    cursor: pointer;
    position: relative;
    top: calc(50vh - 3rem);
    z-index: z-index('center-divider');

    svg {
      background-color: mui-color('grey', '300');
      color: mui-color('black');
      padding: 3rem .25rem;
    }
  }

  .left-resize-container {
    flex-grow: 1;
    height: 100%;
    min-width: 0;
  }

  .right-resize-container {
    flex: 0 0 50%;
    height: 100%;
    min-width: 0;
  }

  #tab-resize-guide {
    $tabs-width: 50%;

    background-color: mui-color('black');
    cursor: col-resize;
    display: none;
    height: 100%;
    position: fixed;
    right: $tabs-width;
    width: 2px;
    z-index: z-index('center-divider');
  }

  #tabs-wrapper {
    height: 100%;
    overflow-y: scroll;
    position: relative;

    .tab-content {
      display: flex;

      .tab-panes {
        text-align: left;
        width: 100%;
      }
    }
  }
}
</style>
