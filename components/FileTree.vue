<template>
  <div>
    <ul>
      <li
        v-for="(item, index) in fileTree"
        :key="item.id"
        :draggable="true"
        @dragstart="onDragStart($event, item.id)"
        @dragover="onDragOver($event, item.id)"
        @drop="onDrop($event, item.id)"
        :class="{ 'drag-over': dragOverItemId === item.id }"
        role="treeitem"
        :aria-expanded="!!item.children"
        :aria-grabbed="draggedItemId === item.id"
        tabindex="0"
      >
        {{ item.position }} - {{ item.name }}
        <button @click="deleteItem(item.id)" :disabled="!item.deletable || (item.children && item.children.length > 0)">Delete</button>
        <ul v-if="item.children">
          <FileTree 
            :fileTree="item.children" 
            :rootFileTree="rootFileTree" 
            :isTopLevel="false" 
            @itemMoved="emitItemMoved" 
            @itemDeleted="emitItemDeleted" 
          />
        </ul>
      </li>
    </ul>
    <div
      v-if="isTopLevel"
      class="empty-slot"
      @dragover="onDragOver($event)"
      @drop="onDropToParent($event)"
    >
      Drop here to move up one level
    </div>
  </div>
</template>

<script setup>
import { defineProps, defineEmits, reactive, toRefs, watch } from 'vue';

const props = defineProps({
  fileTree: {
    type: Array,
    required: true
  },
  rootFileTree: {
    type: Array,
    required: true
  },
  isTopLevel: {
    type: Boolean,
    default: true
  }
});

const emit = defineEmits(['itemMoved', 'itemDeleted']);

const { fileTree, rootFileTree, isTopLevel } = toRefs(reactive(props));

let dragOverItemId = null;
let draggedItemId = null;

function findItemById(tree, id) {
  const stack = [...tree];
  while (stack.length) {
    const item = stack.pop();
    if (item.id === id) return item;
    if (item.children) stack.push(...item.children);
  }
  return null;
}

function findParent(tree, id) {
  const stack = [...tree];
  while (stack.length) {
    const item = stack.pop();
    if (item.children && item.children.some(child => child.id === id)) {
      return item;
    }
    if (item.children) stack.push(...item.children);
  }
  return null;
}

function updatePositions(tree, parentPosition = '') {
  tree.forEach((item, index) => {
    item.position = parentPosition ? `${parentPosition}-${index + 1}` : `${index + 1}`;
    if (item.children) {
      updatePositions(item.children, item.position);
    }
  });
}

watch(rootFileTree, () => {
  updatePositions(rootFileTree.value);
}, { immediate: true });

function onDragStart(event, itemId) {
  const item = findItemById(rootFileTree.value, itemId);
  if (!item) {
    event.preventDefault();
    return;
  }
  event.stopPropagation();
  draggedItemId = itemId;
  event.dataTransfer.setData("text/plain", itemId);
}

function onDragOver(event, itemId) {
  event.preventDefault();
  dragOverItemId = itemId;
}

function onDrop(event, targetItemId) {
  event.preventDefault();
  event.stopPropagation();
  const draggedItemId = event.dataTransfer.getData("text/plain");
  if (draggedItemId !== targetItemId) {
    moveItem(draggedItemId, targetItemId);
  }
  dragOverItemId = null;
}

function onDropToParent(event) {
  event.preventDefault();
  event.stopPropagation();
  const draggedItemId = event.dataTransfer.getData("text/plain");
  moveItemUpOneLevel(draggedItemId);
  dragOverItemId = null;
}

function moveItem(draggedItemId, targetItemId) {
  const draggedItem = findItemById(rootFileTree.value, draggedItemId);
  if (!draggedItem) {
    return;
  }

  let parent = findParent(rootFileTree.value, draggedItemId);
  if (parent) {
    parent.children = parent.children.filter(item => item.id !== draggedItemId);
  } else {
    const rootIndex = rootFileTree.value.findIndex(item => item.id === draggedItemId);
    if (rootIndex !== -1) {
      rootFileTree.value.splice(rootIndex, 1);
    }
  }

  const targetItem = findItemById(rootFileTree.value, targetItemId);
  if (!targetItem.children) {
    targetItem.children = [];
  }
  targetItem.children.push(draggedItem);

  updatePositions(rootFileTree.value);

  emit('itemMoved', { draggedItemId, targetItemId, parentId: parent ? parent.id : null });
}

function moveItemUpOneLevel(draggedItemId) {
  const draggedItem = findItemById(rootFileTree.value, draggedItemId);
  if (!draggedItem) {
    return;
  }

  let parent = findParent(rootFileTree.value, draggedItemId);
  if (!parent) {
    return;
  }

  let grandParent = findParent(rootFileTree.value, parent.id);
  if (grandParent) {
    parent.children = parent.children.filter(item => item.id !== draggedItemId);
    grandParent.children.push(draggedItem);
  } else {
    parent.children = parent.children.filter(item => item.id !== draggedItemId);
    rootFileTree.value.push(draggedItem);
  }

  updatePositions(rootFileTree.value);

  emit('itemMoved', { draggedItemId, targetItemId: grandParent ? grandParent.id : null, parentId: parent.id });
}

function deleteItem(itemId) {
  const item = findItemById(rootFileTree.value, itemId);
  if (item && item.deletable && (!item.children || item.children.length === 0)) {
    let parent = findParent(rootFileTree.value, itemId);
    if (parent) {
      parent.children = parent.children.filter(child => child.id !== itemId);
    } else {
      const index = rootFileTree.value.findIndex(item => item.id === itemId);
      if (index !== -1) {
        rootFileTree.value.splice(index, 1);
      }
    }
    updatePositions(rootFileTree.value);
    emit('itemDeleted', { itemId, parentId: parent ? parent.id : null });
  }
}

function emitItemMoved(event) {
  emit('itemMoved', event);
}

function emitItemDeleted(event) {
  emit('itemDeleted', event);
}
</script>

<style>
.empty-slot {
  margin-top: 10px;
  padding: 10px;
  border: 2px dashed #ccc;
  text-align: center;
}

.drag-over {
  background-color: #f0f8ff;
  border: 1px dashed #1e90ff; /* Clear visual indicator */
}
</style>