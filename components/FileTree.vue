<template>
  <div>
    <ul>
      <li
        v-for="(item, index) in fileTree"
        :key="item.id"
        :draggable="true"
        @dragstart="onDragStart($event, item.id)"
        @dragenter="onDragEnter($event, index, item.id)"
        @dragleave="onDragLeave($event, index, item.id)"
        @dragover="onDragOver($event)"
        @drop="onDrop($event, item.id)"
        role="treeitem"
        :aria-expanded="!!item.children"
        :aria-grabbed="draggedItemId === item.id"
        tabindex="0"
      >
        <div v-if="dragging && dragOverIndex === index" class="placeholder"></div>
        <div v-else>
          {{ item.position }} - {{ item.name }}
          <button @click="deleteItem(item.id)" :disabled="!item.deletable || (item.children && item.children.length > 0)">Delete</button>
          <div
            class="drop-zone-container"
            @dragenter="onDragEnter($event, index, item.id)"
            @dragleave="onDragLeave($event, index, item.id)"
          >
            <div
              class="drop-zone"
              v-show="dropZonesVisible[index] && draggedItemId !== item.id"
              @dragover="onDragOver($event)"
              @drop="onDropInside($event, item.id)"
            >
              Drop here to move inside
            </div>
            <ul v-if="item.children">
              <FileTree 
                :fileTree="item.children" 
                :rootFileTree="rootFileTree" 
                :isTopLevel="false" 
                @itemMoved="emitItemMoved" 
                @itemDeleted="emitItemDeleted" 
                @resetDropZones="resetDropZones"
              />
            </ul>
            <div
              class="drop-zone"
              v-show="dropZonesVisible[index] && draggedItemId !== item.id"
              @dragover="onDragOver($event)"
              @drop="onDropUnder($event, item.id)"
            >
              Drop here to move under
            </div>
          </div>
        </div>
      </li>
      <li v-if="dragging && dragOverIndex === fileTree.length" class="placeholder"></li>
    </ul>
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

const emit = defineEmits(['itemMoved', 'itemDeleted', 'resetDropZones']);

const { fileTree, rootFileTree, isTopLevel } = toRefs(reactive(props));

let dragOverItemId = null;
let draggedItemId = null;
let dragging = false;
let dragOverIndex = null;
const dropZonesVisible = reactive({});
const dropZoneCounters = reactive({});

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
  console.log('Drag start:', itemId);
  const item = findItemById(rootFileTree.value, itemId);
  if (!item) {
    event.preventDefault();
    return;
  }
  event.stopPropagation();
  draggedItemId = itemId;
  dragging = true;
  event.dataTransfer.setData("text/plain", itemId);
}

function onDragEnter(event, index, itemId) {
  event.preventDefault();
  if (draggedItemId !== itemId) {
    dropZoneCounters[index] = (dropZoneCounters[index] || 0) + 1;
    dropZonesVisible[index] = true;
  }
}

function onDragLeave(event, index, itemId) {
  event.preventDefault();
  if (draggedItemId !== itemId) {
    dropZoneCounters[index] = (dropZoneCounters[index] || 1) - 1;
    if (dropZoneCounters[index] <= 0) {
      dropZonesVisible[index] = false;
    }
  }
}

function onDragOver(event) {
  event.preventDefault();
}

function onDrop(event, targetItemId) {
  event.preventDefault();
  event.stopPropagation();
  const draggedItemId = event.dataTransfer.getData("text/plain");
  if (draggedItemId !== targetItemId) {
    moveItemAsChild(draggedItemId, targetItemId);
  }
  resetDragState();
}

function onDropInside(event, targetItemId) {
  event.preventDefault();
  event.stopPropagation();
  const draggedItemId = event.dataTransfer.getData("text/plain");
  if (draggedItemId !== targetItemId) {
    moveItemAsChild(draggedItemId, targetItemId);
  }
  resetDragState();
}

function onDropUnder(event, targetItemId) {
  event.preventDefault();
  event.stopPropagation();
  const draggedItemId = event.dataTransfer.getData("text/plain");
  if (draggedItemId !== targetItemId) {
    moveItemUnder(draggedItemId, targetItemId);
  }
  resetDragState();
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

function moveItemAsChild(draggedItemId, targetItemId) {
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

function moveItemUnder(draggedItemId, targetItemId) {
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

  let targetParent = findParent(rootFileTree.value, targetItemId);
  if (!targetParent) {
    targetParent = { children: rootFileTree.value };
  }

  const targetIndex = targetParent.children.findIndex(item => item.id === targetItemId);
  targetParent.children.splice(targetIndex + 1, 0, draggedItem);

  updatePositions(rootFileTree.value);

  emit('itemMoved', { draggedItemId, targetItemId, parentId: targetParent ? targetParent.id : null });
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

function resetDragState() {
  dragOverItemId = null;
  dragging = false;
  dragOverIndex = null;
  Object.keys(dropZonesVisible).forEach(key => {
    dropZonesVisible[key] = false;
  });
  emit('resetDropZones');
}

function resetDropZones() {
  Object.keys(dropZonesVisible).forEach(key => {
    dropZonesVisible[key] = false;
  });
  emit('resetDropZones');
}

function emitItemMoved(event) {
  emit('itemMoved', event);
}

function emitItemDeleted(event) {
  emit('itemDeleted', event);
}
</script>

<style>
.placeholder {
  height: 20px;
  background-color: #f0f8ff;
  border: 1px dashed #1e90ff;
  margin-bottom: 10px;
}

.drop-zone-container {
  display: flex;
  flex-direction: column;
}

.drop-zone {
  margin: 5px 0;
  padding: 5px;
  border: 2px dashed #ccc;
  text-align: center;
  background-color: #f9f9f9;
}
</style>