<template>
  <div class="file-tree-container" ref="scrollableContainer">
    <ul class="file-tree">
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
        class="file-tree-item"
      >
        <div
          v-if="dragging && dragOverIndex === index"
          class="placeholder"
        ></div>
        <div v-else>
          {{ item.position }} - {{ item.name }}
          <div
            class="drop-zone-container"
            @dragenter="onDragEnter($event, index, item.id)"
            @dragleave="onDragLeave($event, index, item.id)"
          >
            <div
              class="drop-zone"
              v-show="dropZonesVisible[index] && draggedItemId !== item.id"
              @dragover="onDragOver($event)"
              @drop="onDropUnder($event, item.id)"
            >
              Move to next position
            </div>
            <ul v-if="item.children" class="file-tree-children">
              <FileTree
                :fileTree="item.children"
                :rootFileTree="rootFileTree"
                :isTopLevel="false"
                @draggedItemChanged="emitDraggedItemChanged"
                @itemMoved="emitItemMoved"
                @itemDeleted="emitItemDeleted"
                @resetDropZones="resetDropZones"
              />
            </ul>
          </div>
        </div>
      </li>
      <li
        v-if="dragging && dragOverIndex === fileTree.length"
        class="placeholder"
      ></li>
      <li
        class="drop-zone"
        v-show="dragging && dropZonesVisible['end']"
        @dragover="onDragOver($event)"
        @drop="onDropAtEnd($event)"
      >
        Move to next position
      </li>
    </ul>
  </div>
</template>

<script setup>
import {
  defineProps,
  defineEmits,
  reactive,
  toRefs,
  watch,
  ref,
  computed,
} from "vue";

const props = defineProps({
  fileTree: {
    type: Array,
    required: true,
  },
  rootFileTree: {
    type: Array,
    required: true,
  },
  isTopLevel: {
    type: Boolean,
    default: true,
  },
});

const emit = defineEmits([
  "draggedItemChanged",
  "itemMoved",
  "itemDeleted",
  "resetDropZones",
]);

const { fileTree, rootFileTree, isTopLevel } = toRefs(reactive(props));

let dragOverItemId = null;
let draggedItemId = null;
let dragging = false;
let dragOverIndex = null;
const dropZonesVisible = reactive({});
const dropZoneCounters = reactive({});
const scrollableContainer = ref(null);
let autoScrollInterval = null;

const draggedItem = ref(null);

const canDeleteDraggedItem = computed(() => {
  return (
    draggedItem.value &&
    draggedItem.value.deletable &&
    (!draggedItem.value.children || draggedItem.value.children.length === 0)
  );
});

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
    if (item.children && item.children.some((child) => child.id === id)) {
      return item;
    }
    if (item.children) stack.push(...item.children);
  }
  return null;
}

function updatePositions(tree, parentPosition = "") {
  tree.forEach((item, index) => {
    item.position = parentPosition
      ? `${parentPosition}-${index + 1}`
      : `${index + 1}`;
    if (item.children) {
      updatePositions(item.children, item.position);
    }
  });
}

watch(
  rootFileTree,
  () => {
    updatePositions(rootFileTree.value);
  },
  { immediate: true }
);

function isDescendant(parentId, childId) {
  const parentItem = findItemById(rootFileTree.value, parentId);
  if (!parentItem || !parentItem.children) {
    return false;
  }
  const stack = [...parentItem.children];
  while (stack.length) {
    const item = stack.pop();
    if (item.id === childId) {
      return true;
    }
    if (item.children) {
      stack.push(...item.children);
    }
  }
  return false;
}

function onDragStart(event, itemId) {
  console.log("Drag start:", itemId);
  const item = findItemById(rootFileTree.value, itemId);
  if (!item) {
    event.preventDefault();
    return;
  }
  event.stopPropagation();
  draggedItemId = itemId;
  draggedItem.value = item;
  dragging = true;
  event.dataTransfer.setData("text/plain", itemId);
  emitDraggedItemChanged(item);
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
  const container = scrollableContainer.value;
  if (!container) return;
  const rect = container.getBoundingClientRect();
  const offset = 20; // Distance from the edge to start scrolling

  if (event.clientY < rect.top + offset) {
    startAutoScroll(-5); // Scroll up
  } else if (event.clientY > rect.bottom - offset) {
    startAutoScroll(5); // Scroll down
  } else {
    stopAutoScroll();
  }
}

function onDrop(event, targetItemId) {
  event.preventDefault();
  event.stopPropagation();
  const draggedItemId = event.dataTransfer.getData("text/plain");
  if (
    draggedItemId !== targetItemId &&
    !isDescendant(draggedItemId, targetItemId)
  ) {
    moveItemAsChild(draggedItemId, targetItemId);
  }
  resetDragState();
}

function onDropInside(event, targetItemId) {
  event.preventDefault();
  event.stopPropagation();
  const draggedItemId = event.dataTransfer.getData("text/plain");
  if (
    draggedItemId !== targetItemId &&
    !isDescendant(draggedItemId, targetItemId)
  ) {
    moveItemAsChild(draggedItemId, targetItemId);
  }
  resetDragState();
}

function onDropUnder(event, targetItemId) {
  event.preventDefault();
  event.stopPropagation();
  stopAutoScroll();
  const draggedItemId = event.dataTransfer.getData("text/plain");
  if (
    draggedItemId !== targetItemId &&
    !isDescendant(draggedItemId, targetItemId)
  ) {
    moveItemUnder(draggedItemId, targetItemId);
  }
  resetDragState();
}

function moveItem(draggedItemId, targetItemId) {
  if (isDescendant(draggedItemId, targetItemId)) {
    return;
  }

  const draggedItem = findItemById(rootFileTree.value, draggedItemId);
  if (!draggedItem) {
    return;
  }

  let parent = findParent(rootFileTree.value, draggedItemId);
  if (parent) {
    parent.children = parent.children.filter(
      (item) => item.id !== draggedItemId
    );
  } else {
    const rootIndex = rootFileTree.value.findIndex(
      (item) => item.id === draggedItemId
    );
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

  emit("itemMoved", {
    draggedItemId,
    targetItemId,
    parentId: parent ? parent.id : null,
  });
}

function moveItemAsChild(draggedItemId, targetItemId) {
  if (isDescendant(draggedItemId, targetItemId)) {
    return;
  }

  const draggedItem = findItemById(rootFileTree.value, draggedItemId);
  if (!draggedItem) {
    return;
  }

  let parent = findParent(rootFileTree.value, draggedItemId);
  if (parent) {
    parent.children = parent.children.filter(
      (item) => item.id !== draggedItemId
    );
  } else {
    const rootIndex = rootFileTree.value.findIndex(
      (item) => item.id === draggedItemId
    );
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

  emit("itemMoved", {
    draggedItemId,
    targetItemId,
    parentId: parent ? parent.id : null,
  });
}

function moveItemUnder(draggedItemId, targetItemId) {
  if (isDescendant(draggedItemId, targetItemId)) {
    return;
  }

  const draggedItem = findItemById(rootFileTree.value, draggedItemId);
  if (!draggedItem) {
    return;
  }

  let parent = findParent(rootFileTree.value, draggedItemId);
  if (parent) {
    parent.children = parent.children.filter(
      (item) => item.id !== draggedItemId
    );
  } else {
    const rootIndex = rootFileTree.value.findIndex(
      (item) => item.id === draggedItemId
    );
    if (rootIndex !== -1) {
      rootFileTree.value.splice(rootIndex, 1);
    }
  }

  let targetParent = findParent(rootFileTree.value, targetItemId);
  if (!targetParent) {
    targetParent = { children: rootFileTree.value };
  }

  const targetIndex = targetParent.children.findIndex(
    (item) => item.id === targetItemId
  );
  targetParent.children.splice(targetIndex + 1, 0, draggedItem);

  updatePositions(rootFileTree.value);

  emit("itemMoved", {
    draggedItemId,
    targetItemId,
    parentId: targetParent ? targetParent.id : null,
  });
}

function deleteItem(itemId) {
  const item = findItemById(rootFileTree.value, itemId);
  if (
    item &&
    item.deletable &&
    (!item.children || item.children.length === 0)
  ) {
    let parent = findParent(rootFileTree.value, itemId);
    if (parent) {
      parent.children = parent.children.filter((child) => child.id !== itemId);
    } else {
      const index = rootFileTree.value.findIndex((item) => item.id === itemId);
      if (index !== -1) {
        rootFileTree.value.splice(index, 1);
      }
    }
    updatePositions(rootFileTree.value);
    emit("itemDeleted", { itemId, parentId: parent ? parent.id : null });
  }
}

function deleteDraggedItem() {
  if (canDeleteDraggedItem.value) {
    deleteItem(draggedItem.value.id);
    draggedItem.value = null;
    resetDragState();
  }
}

function resetDragState() {
  dragOverItemId = null;
  draggedItemId = null;
  draggedItem.value = null;
  dragging = false;
  dragOverIndex = null;
  Object.keys(dropZonesVisible).forEach((key) => {
    dropZonesVisible[key] = false;
  });
  emit("resetDropZones");
}

function resetDropZones() {
  Object.keys(dropZonesVisible).forEach((key) => {
    dropZonesVisible[key] = false;
  });
  emit("resetDropZones");
}

function emitDraggedItemChanged(item) {
  emit("draggedItemChanged", item);
}

function emitItemMoved(event) {
  emit("itemMoved", event);
}

function emitItemDeleted(event) {
  emit("itemDeleted", event);
}

function startAutoScroll(speed) {
  if (autoScrollInterval) {
    clearInterval(autoScrollInterval);
  }
  autoScrollInterval = setInterval(() => {
    if (scrollableContainer.value) {
      scrollableContainer.value.scrollTop += speed;
    }
  }, 20);
}

function stopAutoScroll() {
  if (autoScrollInterval) {
    clearInterval(autoScrollInterval);
    autoScrollInterval = null;
  }
}
</script>

<style>
.file-tree {
  list-style-type: none;
  padding-left: 20px;
}

.file-tree-item {
  margin: 5px 0;
}

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

.file-tree-container {
  max-height: 100%;
  overflow-y: auto;
}
</style>
