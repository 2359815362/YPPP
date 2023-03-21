<script setup>
import Home from './components/Home.vue'
import {NDialogProvider, NMessageProvider} from "naive-ui";</script>

<template>
  <n-message-provider>
    <n-dialog-provider>
      <Home/>
    </n-dialog-provider>
  </n-message-provider>
</template>

<style scoped>
</style>
