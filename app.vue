<script setup>
const showFiles = (val) => {
  console.log("from select emit", val);
};
const clear = (val) => {
  console.log(val);
};
const preloaded = ref([
  "https://images.unsplash.com/photo-1682685796063-d2604827f7b3?ixlib=rb-4.0.3&ixid=M3wxMjA3fDF8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=1170&q=80",
  "https://plus.unsplash.com/premium_photo-1688114984765-308599ec1e13?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=687&q=80",
  "https://images.unsplash.com/photo-1689464090276-50bed9a6798f?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=880&q=80",
]);
</script>
<template>
  <div>
    <Fileupload
      name="fileupload"
      :maxFileSize="1000000"
      :fileLimit="3"
      accept="image/*,.pdf"
      @select="showFiles"
      @clear="clear"
      :init="preloaded"
    />
  </div>
</template>
