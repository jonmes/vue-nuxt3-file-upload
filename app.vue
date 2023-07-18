<script setup>
const showFiles = (val) => {
  console.log("from select emit", val);
};
const clear = (val) => {
  console.log(val);
};
const preloaded = ref([
  "https://images.unsplash.com/photo-1682685796063-d2604827f7b3?ixlib=rb-4.0.3&ixid=M3wxMjA3fDF8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=1170&q=80",
  "https://cdn.hahu.jobs/public/mesmer/work_premise/mesmer_MPCXXr_1689660235",
  "https://images.unsplash.com/photo-1689464090276-50bed9a6798f?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=880&q=80",
]);
</script>
<template>
  <div class="flex w-1/2 mx-auto">
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
