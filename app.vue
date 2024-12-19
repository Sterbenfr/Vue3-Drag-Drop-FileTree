<template>
  <div>
    <FileTree :fileTree="fileTree" :rootFileTree="fileTree" :isTopLevel="true" @itemMoved="handleItemMoved" @itemDeleted="handleItemDeleted" />
  </div>
</template>

<script setup>
import { reactive } from 'vue';
import FileTree from '@/components/FileTree.vue';

const fileTree = reactive([
  { id: '1', name: "Folder 1", children: [], deletable: true, position: "1" },
  { id: '2', name: "Folder 2", children: [], deletable: true, position: "2" },
  {
    id: '3',
    name: "Folder 3",
    children: [
      { id: '4', name: "File 1",children:[
        { id: '6', name: "File 3", children:[], deletable: true, position: "3-1-1" },
        { id: '7', name: "File 4", children:[], deletable: true, position: "3-1-2" }
      ], deletable: true, position: "3-1" },
      { id: '5', name: "File 2", children:[],deletable: false, position: "3-2" }
    ],
    deletable: true,
    position: "3"
  }
]);

function handleItemMoved(event) {
  console.log('Item moved:', event);
}

function handleItemDeleted(event) {
  console.log('Item deleted:', event);
}
</script>