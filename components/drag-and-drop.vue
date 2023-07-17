<script setup>
import { reactive } from "vue";
const event = reactive({ drag: false, hover: false });

const dragover = (e) => {
  e.preventDefault();
  event.drag = true;
};
const dragleave = (e) => {
  e.preventDefault();
  event.drag = false;
};
const mouseover = (e) => {
  e.preventDefault();
  event.hover = true;
};
const mouseleave = (e) => {
  e.preventDefault();
  event.hover = false;
};
const drop = (e) => {
  event.drag = false;
};
</script>
<template>
  <div
    class="d-flex row"
    @dragover="dragover"
    @dragleave="dragleave"
    @mouseover="mouseover"
    @mouseleave="mouseleave"
    @drop="drop"
  >
    <slot v-bind="event"></slot>
  </div>
</template>
a
