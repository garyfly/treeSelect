<template>
  <div v-if="isVisible" class="tree-node">
    <div class="node-item" :class="{ 'is-selected': isChecked }" @click.stop="handleLabelClick">
      <span @click.stop="toggleExpand" class="toggle-icon-wrapper">
        <span v-if="hasChildren" class="toggle-icon">
          {{ isExpanded ? "▼" : "►" }}
        </span>
        <span v-else class="toggle-icon-placeholder"></span>
      </span>

      <input v-if="currentMode === 'multiple'" type="checkbox" :checked="isChecked" :indeterminate="isIndeterminate"
        @click.stop="handleSelect" class="node-control" />
      <input v-if="currentMode === 'single'" type="radio" :checked="isChecked" @click.stop="handleSelect"
        class="node-control" />

      <label class="node-label">
        {{ node.label }}
      </label>
    </div>

    <div v-if="isExpanded && hasChildren" class="node-children">
      <TreeNode v-for="child in node.children" :key="child.id" :node="child" :modelValue="modelValue"
        :multiple="currentMode === 'multiple'"
        :searchTerm="searchTerm" :parentMode="currentMode" :siblings="node.children"
        @update="emitUpdate" @select="emitSelect" />
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, defineProps, defineEmits } from "vue";

// 定义节点类型
interface TreeNode {
  id: any;
  label: string;
  children?: TreeNode[];
  selectionMode?: string;
}

const props = defineProps({
  node: { type: Object as () => TreeNode, required: true },
  modelValue: { type: [Array, String, Number], default: null },
  multiple: { type: Boolean, default: false },
  searchTerm: { type: String, default: "" },
  // 从父节点继承选择模式
  parentMode: { type: String, default: "multiple" },
  // 传入同级节点，用于处理单选逻辑
  siblings: { type: Array as () => TreeNode[], default: () => [] },
});

const emit = defineEmits(["update", "select"]);

const isExpanded = ref(true);

// --- 计算属性 ---

// 确定当前节点的选择模式，如果自身未定义，则继承父节点
const currentMode = computed(() => {
  // 如果节点自身定义了 selectionMode，则使用它
  if (props.node.selectionMode) {
    return props.node.selectionMode;
  }
  // 如果没有定义，则根据 multiple 属性判断
  return props.multiple ? "multiple" : props.parentMode;
});

const hasChildren = computed(
  () => props.node.children && props.node.children.length > 0
);

const allDescendantIds = computed(() => {
  const ids: any[] = [];
  function collectIds(node: TreeNode) {
    ids.push(node.id);
    if (node.children) node.children.forEach(collectIds);
  }
  if (hasChildren.value && props.node.children)
    props.node.children.forEach(collectIds);
  return ids;
});

// 节点是否被选中
const isChecked = computed(() => {
  if (Array.isArray(props.modelValue)) {
    return props.modelValue.includes(props.node.id);
  } else {
    return props.modelValue === props.node.id;
  }
});

// 多选模式下的半选状态
const isIndeterminate = computed(() => {
  if (
    currentMode.value !== "multiple" ||
    !hasChildren.value ||
    isChecked.value
  ) {
    return false;
  }

  // 确保 modelValue 是数组格式再处理
  const modelValueArray = Array.isArray(props.modelValue)
    ? props.modelValue
    : [];
  const selectedChildren = allDescendantIds.value.filter((id) =>
    modelValueArray.includes(id)
  );
  return (
    selectedChildren.length > 0 &&
    selectedChildren.length < allDescendantIds.value.length
  );
});

const isVisible = computed(() => {
  if (!props.searchTerm) return true;
  const searchLower = props.searchTerm.toLowerCase();
  const isSelfVisible = props.node.label.toLowerCase().includes(searchLower);
  if (isSelfVisible) return true;
  const isChildVisible = (node: TreeNode) =>
    node.label.toLowerCase().includes(searchLower) ||
    (node.children ? node.children.some(isChildVisible) : false);
  return hasChildren.value && props.node.children?.some(isChildVisible);
});

// --- 方法 ---

const toggleExpand = () => {
  if (hasChildren.value) isExpanded.value = !isExpanded.value;
};

const handleLabelClick = () => handleSelect();

const handleSelect = () => {
  const toAdd = new Set();
  const toRemove = new Set();

  if (currentMode.value === "single") {
    // 单选逻辑
    if (!isChecked.value) {
      // 只有在未选中时才操作
      toAdd.add(props.node.id);
      // 移除所有同级节点的选中状态
      props.siblings.forEach((sibling) => {
        if (sibling.id !== props.node.id) {
          toRemove.add(sibling.id);
        }
      });

      // 发送 select 事件用于单选模式
      emit("select", {
        id: props.node.id,
        ids: [props.node.id],
        isSelected: true,
      });
    }
  } else {
    // 多选逻辑
    const idsToUpdate = [props.node.id, ...allDescendantIds.value];
    const shouldSelect = !isChecked.value;

    if (shouldSelect) {
      idsToUpdate.forEach((id) => toAdd.add(id));
    } else {
      idsToUpdate.forEach((id) => toRemove.add(id));
    }

    // 发送 select 事件用于多选模式
    emit("select", {
      ids: idsToUpdate,
      isSelected: shouldSelect,
    });
  }

  // 向上层发出带有 add 和 remove 列表的事件
  emit("update", {
    add: Array.from(toAdd),
    remove: Array.from(toRemove),
  });
};

const emitUpdate = (payload: any) => {
  // 将事件向上传递
  emit("update", payload);
};

const emitSelect = (payload: any) => {
  // 将 select 事件向上传递
  emit("select", payload);
};
</script>

<style scoped>
/* 样式与前一版类似，可以微调 */
.node-control {
  margin-right: 8px;
  width: 15px;
  height: 15px;
}

.node-item.is-selected {
  background-color: #e0efff;
}

/* 其他样式保持不变 */
.node-item {
  display: flex;
  align-items: center;
  padding: 5px 8px;
  border-radius: 4px;
  cursor: pointer;
  user-select: none;
  transition: background-color 0.2s;
}

.node-item:hover {
  background-color: #f0f7ff;
}

.toggle-icon-wrapper {
  padding: 2px 6px 2px 2px;
}

.toggle-icon {
  color: #666;
  font-size: 0.75em;
}

.toggle-icon-placeholder {
  display: inline-block;
  width: 1em;
  margin-right: 6px;
}

.node-label {
  flex-grow: 1;
  cursor: pointer;
}

.node-children {
  padding-left: 24px;
}
</style>
