<template>
  <v-app>
    <FileTreeWrapper 
      :fileTree="fileTree" 
      @itemMoved="handleItemMoved" 
      @itemDeleted="handleItemDeleted" 
    />
  </v-app>
</template>

<script setup>
import { reactive } from 'vue';
import FileTreeWrapper from '@/components/FileTreeWrapper.vue';

const fileTree = reactive([
  { id: '1', name: "Folder 1", children: [], deletable: true, position: "1" },
  { id: '2', name: "Folder 2", children: [], deletable: true, position: "2" },
  {
    id: '3',
    name: "Folder 3",
    children: [
      { id: '4', name: "File 1", children:[
        { id: '6', name: "File 3", children:[], deletable: true, position: "3-1-1" },
        { id: '7', name: "File 4", children:[], deletable: true, position: "3-1-2" }
      ], deletable: true, position: "3-1" },
      { id: '5', name: "File 2", children:[], deletable: false, position: "3-2" }
    ],
    deletable: true,
    position: "3"
  },
  { id: '8', name: "Folder 4", children: [
    { id: '9', name: "File 5", children:[], deletable: true, position: "4-1" },
    { id: '10', name: "File 6", children:[], deletable: true, position: "4-2" },
    { id: '11', name: "File 7", children:[], deletable: true, position: "4-3" },
    { id: '12', name: "File 8", children:[
      { id: '13', name: "File 9", children:[], deletable: true, position: "4-4-1" },
      { id: '14', name: "File 10", children:[
        { id: '15', name: "File 11", children:[], deletable: true, position: "4-4-2-1" },
        { id: '16', name: "File 12", children:[
          { id: '17', name: "File 13", children:[], deletable: true, position: "4-4-2-2-1" },
          { id: '18', name: "File 14", children:[
            { id: '19', name: "File 15", children:[], deletable: true, position: "4-4-2-2-2-1" },
            { id: '20', name: "File 16", children:[
              { id: '21', name: "File 17", children:[], deletable: true, position: "4-4-2-2-2-2-1" },
              { id: '22', name: "File 18", children:[], deletable: true, position: "4-4-2-2-2-2-2" }
            ], deletable: true, position: "4-4-2-2-2-2" }
          ], deletable: true, position: "4-4-2-2-2" }
        ], deletable: true, position: "4-4-2-2" }
      ], deletable: true, position: "4-4-2" }
    ], deletable: true, position: "4-4" }
  ], deletable: true, position: "4" }
]);

function handleItemMoved(event) {
  console.log('Item moved:', event);
}

function handleItemDeleted(event) {
  console.log('Item deleted:', event);
}
</script>