<template>
  <div class="tree-select-container" ref="containerRef">
    <div class="selected-value-wrapper" @click="toggleDropdown">
      <span v-if="!hasSelection" class="placeholder">{{ placeholder }}</span>
      <span v-else class="selected-label">{{ selectedLabel }}</span>
      <span class="dropdown-icon" :class="{ open: isOpen }">▲</span>
    </div>

    <Transition name="fade">
      <div v-if="isOpen" class="dropdown-panel">
        <input type="text" v-model="searchTerm" placeholder="Search..." class="search-input" @click.stop />
        <div class="tree-view">
          <TreeNode 
            v-for="option in options" 
            :key="option.id" 
            :node="option" 
            :modelValue="modelValue"
            :multiple="multiple"
            :searchTerm="searchTerm" 
            @select="handleNodeSelect" 
            @update="handleNodeUpdate" />
        </div>
      </div>
    </Transition>
  </div>
</template>

<script setup>
import { ref, computed, defineProps, defineEmits, onMounted, onBeforeUnmount } from 'vue';
import TreeNode from './TreeNode.vue';

// --- Props and Emits ---
const props = defineProps({
  options: { type: Array, required: true },
  modelValue: { type: [Array, String, Number], default: null },
  multiple: { type: Boolean, default: false },
  placeholder: { type: String, default: 'Select...' },
});

const emit = defineEmits(['update:modelValue']);

// --- State ---
const isOpen = ref(false);
const searchTerm = ref('');
const containerRef = ref(null);

// --- Computed Properties ---
const hasSelection = computed(() => {
  if (props.multiple) {
    return props.modelValue && Array.isArray(props.modelValue) && props.modelValue.length > 0;
  } else {
    return props.modelValue != null;
  }
});

const optionsMap = computed(() => {
  const flat = {};
  const recurse = (node) => {
    flat[node.id] = node.label;
    if (node.children) node.children.forEach(recurse);
  };
  props.options.forEach(recurse);
  return flat;
});

const selectedLabel = computed(() => {
  if (!hasSelection.value) return '';
  if (props.multiple) {
    const count = props.modelValue.length;
    return `${count} item${count > 1 ? 's' : ''} selected`;
  }
  return optionsMap.value[props.modelValue] || '';
});

// --- Methods ---
const toggleDropdown = () => isOpen.value = !isOpen.value;
const closeDropdown = () => {
  isOpen.value = false;
  searchTerm.value = ''; // Reset search on close
};

const handleClickOutside = (event) => {
  if (containerRef.value && !containerRef.value.contains(event.target)) {
    closeDropdown();
  }
};

const handleNodeSelect = (payload) => {
  console.log('handleNodeSelect', payload);

  if (props.multiple) {
    const { ids, isSelected } = payload;
    let currentSelection = new Set(Array.isArray(props.modelValue) ? props.modelValue : []);


    if (isSelected) {
      ids.forEach(id => currentSelection.add(id));
    } else {
      ids.forEach(id => currentSelection.delete(id));
    }
    emit('update:modelValue', Array.from(currentSelection));
  } else {
    emit('update:modelValue', payload.id);
    closeDropdown();
  }
};

const handleNodeUpdate = (payload) => {
  console.log('handleNodeUpdate', payload);
  
  if (props.multiple) {
    const { add = [], remove = [] } = payload;
    let currentSelection = new Set(Array.isArray(props.modelValue) ? props.modelValue : []);
    
    // 添加需要选中的项
    add.forEach(id => currentSelection.add(id));
    
    // 移除需要取消选中的项
    remove.forEach(id => currentSelection.delete(id));
    
    emit('update:modelValue', Array.from(currentSelection));
  } else {
    // 单选模式下，直接设置选中值
    if (payload.add && payload.add.length > 0) {
      emit('update:modelValue', payload.add[0]);
      closeDropdown();
    }
  }
};

// --- Lifecycle Hooks ---
onMounted(() => document.addEventListener('click', handleClickOutside));
onBeforeUnmount(() => document.removeEventListener('click', handleClickOutside));
</script>

<style scoped>
.tree-select-container {
  position: relative;
  width: 300px;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
}

.selected-value-wrapper {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px 12px;
  border: 1px solid #ccc;
  border-radius: 6px;
  cursor: pointer;
  background-color: #fff;
  transition: border-color 0.2s;
}

.selected-value-wrapper:hover {
  border-color: #888;
}

.placeholder {
  color: #aaa;
}

.selected-label {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.dropdown-icon {
  color: #888;
  transition: transform 0.3s ease;
  transform: rotate(180deg);
}

.dropdown-icon.open {
  transform: rotate(0deg);
}

.dropdown-panel {
  position: absolute;
  top: calc(100% + 4px);
  left: 0;
  right: 0;
  background-color: #fff;
  border: 1px solid #ccc;
  border-radius: 6px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1);
  z-index: 1000;
  max-height: 280px;
  display: flex;
  flex-direction: column;
}

.search-input {
  padding: 10px 12px;
  border: none;
  border-bottom: 1px solid #eee;
  outline: none;
  width: 100%;
  box-sizing: border-box;
}

.tree-view {
  padding: 8px;
  overflow-y: auto;
  flex-grow: 1;
}

/* Fade transition for the dropdown */
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.2s ease, transform 0.2s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
  transform: translateY(-5px);
}
</style>