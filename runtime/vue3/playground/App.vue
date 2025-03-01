<template>
  <magic-ui-page v-if="pageConfig" :key="pageConfig.id" :config="pageConfig"></magic-ui-page>
</template>

<script lang="ts" setup>
import { computed, inject, nextTick, reactive, ref, watch } from 'vue';

import Core from '@tmagic/core';
import type { Id, MApp, MNode } from '@tmagic/schema';
import { Magic, RemoveData, UpdateData } from '@tmagic/stage';
import { getNodePath, replaceChildNode } from '@tmagic/utils';

declare global {
  interface Window {
    magic: Magic;
  }
}

const app = inject<Core | undefined>('app');

const root = ref<MApp>();
const curPageId = ref<Id>();
const selectedId = ref<Id>();

const pageConfig = computed(
  () => root.value?.items?.find((item: MNode) => item.id === curPageId.value) || root.value?.items?.[0],
);

watch(pageConfig, async () => {
  // page配置变化后，需要等dom更新
  await nextTick();
  const page = document.querySelector<HTMLElement>('.magic-ui-page');
  page && window.magic.onPageElUpdate(page);
});

window.magic?.onRuntimeReady({
  getApp() {
    return app;
  },

  /** 当编辑器的dsl对象变化时会调用 */
  updateRootConfig(config: MApp) {
    root.value = config;
    app?.setConfig(config, curPageId.value);
  },

  /** 当编辑器的切换页面时会调用 */
  updatePageId(id: Id) {
    curPageId.value = id;
    app?.setPage(id);
  },

  select(id: Id) {
    selectedId.value = id;

    if (app?.getPage(id)) {
      this.updatePageId?.(id);
    }

    const el = document.getElementById(`${id}`);
    if (el) return el;
    // 未在当前文档下找到目标元素，可能是还未渲染，等待渲染完成后再尝试获取
    return nextTick().then(() => document.getElementById(`${id}`) as HTMLElement);
  },

  /** 新增组件时调用 */
  add({ config, parentId }: UpdateData) {
    if (!root.value) throw new Error('error');
    if (!selectedId.value) throw new Error('error');
    if (!parentId) throw new Error('error');

    const parent = getNodePath(parentId, [root.value]).pop();
    if (!parent) throw new Error('未找到父节点');

    if (config.type !== 'page') {
      const parentNode = app?.page?.getNode(parent.id);
      parentNode && app?.page?.initNode(config, parentNode);
    }

    if (parent.id !== selectedId.value) {
      const index = parent.items?.findIndex((child: MNode) => child.id === selectedId.value);
      parent.items?.splice(index + 1, 0, config);
    } else {
      // 新增节点添加到配置中
      parent.items?.push(config);
    }
  },

  /** 更新组件时调用 */
  update({ config, parentId }: UpdateData) {
    if (!root.value || !app) throw new Error('error');

    const newNode = app.dataSourceManager?.compiledNode(config) || config;
    replaceChildNode(reactive(newNode), [root.value], parentId);

    const nodeInstance = app.page?.getNode(config.id);
    if (nodeInstance) {
      nodeInstance.setData(config);
    }
  },

  /** 删除组件时调用 */
  remove({ id, parentId }: RemoveData) {
    if (!root.value) throw new Error('error');

    const node = getNodePath(id, [root.value]).pop();
    if (!node) throw new Error('未找到目标元素');

    const parent = getNodePath(parentId, [root.value]).pop();
    if (!parent) throw new Error('未找到父元素');

    if (node.type === 'page') {
      app?.deletePage();
    } else {
      app?.page?.deleteNode(node.id);
    }

    const index = parent.items?.findIndex((child: MNode) => child.id === node.id);
    parent.items.splice(index, 1);
  },
});
</script>
