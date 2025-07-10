<template>
  <div class="version-control-container">
    <header class="version-control-header">
      <div class="header-left">
        <span class="title">版本控制</span>
        <link-button :href="docsUrl" class="link-button"></link-button>
      </div>

      <div class="header-right">
        <close-icon @close="close"></close-icon>
      </div>
    </header>

    <main class="version-control-content"></main>
  </div>
</template>

<script>
import { LinkButton, CloseIcon } from '@opentiny/tiny-engine-common'
import { useHelp } from '@opentiny/tiny-engine-controller'

export default {
  components: {
    LinkButton,
    CloseIcon
  },
  emits: ['close'],
  setup(props, { emit }) {
    const docsUrl = useHelp().getDocsUrl('script')

    const close = () => {
      emit('close')
    }

    return {
      docsUrl,
      close
    }
  }
}
</script>

<style lang="less" scoped>
.version-control-container {
  width: 50vw;
  height: 100%;
  background-color: var(--ti-lowcode-plugin-version-control-bg);
  box-shadow: 6px 0px 3px 0px rgba(0, 0, 0, 0.05);

  .version-control-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    border-bottom: 1px solid var(--ti-lowcode-plugin-version-control-head-border-bottom-color);
    padding: 10px 0;

    .header-left {
      padding-left: 15px;
      display: flex;
      align-items: center;

      .title {
        color: var(--ti-lowcode-plugin-panel-title-color);
        font-weight: var(--ti-lowcode-plugin-panel-title-font-weight);
      }

      .link-button {
        display: inline-block;
        width: 24px;
        height: 24px;
        margin-left: 8px;
        cursor: pointer;
        color: var(--ti-lowcode-plugin-version-control-help-link-color);
        &:hover {
          text-decoration: underline;
        }
      }
    }

    .header-right {
      margin-right: 20px;
      display: flex;
      align-items: center;
    }
  }
}
</style>
