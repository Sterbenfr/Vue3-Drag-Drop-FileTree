<template>
  <v-card class="pa-4 full-height">
    <FileTree
      :fileTree="fileTree"
      :rootFileTree="fileTree"
      @draggedItemChanged="updateDraggedItem"
      @itemMoved="handleItemMoved"
      @itemDeleted="handleItemDeleted"
      @resetDropZones="resetDropZones"
    />
    <v-dialog v-model="confirmDeleteDialog" max-width="500">
      <v-card>
        <v-card-title class="headline">Confirm Deletion</v-card-title>
        <v-card-text>Are you sure you want to delete this item?</v-card-text>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn color="blue darken-1" text @click="confirmDeleteDialog = false"
            >Cancel</v-btn
          >
          <v-btn color="red darken-1" text @click="confirmDelete">Delete</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
    <div
      class="delete-zone"
      :class="{ 'delete-zone-enabled': canDeleteDraggedItem }"
      @dragover.prevent
      @drop="showConfirmDeleteDialog"
    >
      <v-icon color="black">mdi-delete</v-icon>
      <span>Delete</span>
    </div>
  </v-card>
</template>

<script setup>
import { ref, computed, defineProps, defineEmits } from "vue";
import FileTree from "./FileTree.vue";

const props = defineProps({
  fileTree: {
    type: Array,
    required: true,
  },
});

const emit = defineEmits(["itemMoved", "itemDeleted", "resetDropZones"]);

const draggedItem = ref(null);
const confirmDeleteDialog = ref(false);

const canDeleteDraggedItem = computed(() => {
  return (
    draggedItem.value &&
    draggedItem.value.deletable &&
    (!draggedItem.value.children || draggedItem.value.children.length === 0)
  );
});

function updateDraggedItem(item) {
  draggedItem.value = item;
}

function showConfirmDeleteDialog() {
  if (canDeleteDraggedItem.value) {
    confirmDeleteDialog.value = true;
  }
}

function confirmDelete() {
  deleteDraggedItem();
  confirmDeleteDialog.value = false;
}

function deleteDraggedItem() {
  if (canDeleteDraggedItem.value) {
    deleteItem(draggedItem.value.id);
    draggedItem.value = null;
  }
}

function deleteItem(itemId) {
  function findAndDeleteItem(tree, id) {
    for (let i = 0; i < tree.length; i++) {
      if (tree[i].id === id) {
        tree.splice(i, 1);
        emit("itemDeleted", { itemId: id });
        return true;
      }
      if (tree[i].children) {
        const found = findAndDeleteItem(tree[i].children, id);
        if (found) return true;
      }
    }
    return false;
  }

  findAndDeleteItem(props.fileTree, itemId);
}

function handleItemMoved(event) {
  emit("itemMoved", event);
}

function handleItemDeleted(event) {
  emit("itemDeleted", event);
}

function resetDropZones() {
  emit("resetDropZones");
}
</script>

<style scoped>
.full-height {
  height: 85vh;
  display: flex;
  flex-direction: column;
}

.delete-zone {
  margin-top: 20px;
  padding: 5px;
  text-align: center;
  border: 2px solid #ccc;
  background-color: grey;
  color: white;
  cursor: not-allowed;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 100px;
  height: 40px;
}

.delete-zone-enabled {
  background-color: red;
  cursor: pointer;
}

.delete-zone v-icon {
  margin-right: 4px;
}
</style>
