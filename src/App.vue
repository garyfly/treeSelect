<template>
  <div id="app">
    <h1>混合模式树形选择组件</h1>
    <p>当前选中的所有 ID: {{ selectedIds }}</p>
    <TreeSelect v-model="selectedIds" :options="treeData" :multiple="true" placeholder="请选择..." />
  </div>
</template>

<script setup>
import { ref } from 'vue';
import TreeSelect from './components/TreeSelect/TreeSelect.vue';

// v-model 始终绑定到一个数组
const selectedIds = ref(['2-1-1']); // 可以设置初始值

const treeData = ref([
  {
    id: '1',
    label: '选择操作系统 (此分支为单选)',
    selectionMode: 'single',
    children: [
      { id: '1-1', label: 'Windows' },
      { id: '1-2', label: 'macOS' },
      { id: '1-3', label: 'Linux' }
    ]
  },
  {
    id: '2',
    label: '选择前端框架 (此分支为多选)',
    selectionMode: 'multiple',
    children: [
      {
        id: '2-1',
        label: '主流框架',
        children: [
          { id: '2-1-1', label: 'Vue' },
          { id: '2-1-2', label: 'React' },
        ],
      },
      { id: '2-2', label: '新兴框架' },
    ],
  },
  {
    id: '3',
    label: '选择后端语言 (无子节点)',
    selectionMode: 'multiple',
  }
]);
</script>

<style>
</style>