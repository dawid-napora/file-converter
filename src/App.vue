<template lang="pug">
  v-app
    v-app-bar(app flat dark)
      v-app-bar-nav-icon
      v-toolbar-title File Converter
      v-spacer
      v-btn(href="https://github.com/dawnap/file-converter" target="_blank" icon): v-icon mdi-github
    v-main
      router-view
</template>

<script lang="ts">
import { defineComponent } from '@vue/composition-api'

export default defineComponent({})
</script>
