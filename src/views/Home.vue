<template lang="pug">
  .page.page--home
    v-container
      v-row
        v-col(cols="12" lg="9" xl="10")
          v-row: v-col
            v-card(outlined)
              v-toolbar(flat)
                v-btn(tag="label" for="fileinput" text) Choose files
                v-btn(tag="label" for="fileinput" icon): v-icon mdi-folder-search-outline
                v-btn(icon disabled): v-icon mdi-dropbox
                v-btn(icon disabled): v-icon mdi-google-drive
                v-btn(icon disabled): v-icon mdi-link
              v-card-text
                label.v-card.v-card--flat.v-sheet.theme--light.grey.lighten-4(for="fileinput" v-ripple)
                  v-card-text
                    .d-flex.justify-center.align-center(style="min-height:350px")
                      span Drop files or click here to upload
                v-file-input#fileinput.d-none(
                  v-model="model.files"
                  multiple
                  hide-input
                )
          v-row: v-col
            v-card(outlined)
              v-form
                v-row
                  v-col(cols="12" md="6" xl="3")
                    v-subheader Select target file format
                    v-card-text: v-select(
                      dense solo flat hide-details
                      v-model="model.filetype"
                      :items="state.filetypes"
                      label="Click to select"
                      background-color="grey lighten-4"
                    )
                  v-col(cols="12" md="6" xl="3")
                    v-subheader Select quality level
                    v-card-text: v-slider(
                      thumb-label
                      min="0" max="100"
                      append-icon="mdi-quality-high"
                      prepend-icon="mdi-quality-low"
                    )
                  v-col(cols="12" md="6" xl="3")
                  v-col(cols="12" md="6" xl="3")
              v-toolbar(flat)
                v-checkbox(label="I accept the terms of use" hide-details)
                v-spacer
                v-btn(depressed color="primary" :disabled="!state.items.length" @click="convertAll") Convert all files
          v-row: v-col
            v-card(outlined)
              v-toolbar(flat)
                v-btn(text :disabled="!model.table.length") Download
                v-btn(text :disabled="!model.table.length" @click="deleteSelected") Delete
              v-card-text
                v-data-table(
                  v-model="model.table"
                  :headers="state.headers"
                  :items="state.items"
                  :items-per-page="-1"
                  item-key="id"
                  show-select
                  disable-pagination
                  hide-default-footer
                )
                  template(v-slot:item.progress="{ item }")
                    v-progress-linear(v-if="typeof item.status === 'number'" :value="item.status" color="green" rounded)
                    v-chip(v-else-if="item.status === 'ready'" small label) Ready to convert
                    v-chip.white--text(v-else-if="item.status === 'error'" small label color="red") Cannot convert
                    v-chip.white--text(v-else-if="item.status === 'done'" small label color="green") Converted
                  template(v-slot:item.action="{ item }")
                    v-btn(icon :disabled="item.status !== 'done'"): v-icon mdi-download
                    v-btn(icon @click="deleteItem(item.id)"): v-icon mdi-delete
        v-col(cols="12" lg="3" xl="2")
          v-row(style="position:sticky;top:88px"): v-col
            v-card(outlined)
              v-card-text
                
</template>

<script lang="ts">
import { computed, defineComponent, reactive, ref, watch } from '@vue/composition-api'
import filesize from 'filesize'
import moment from 'moment'

interface FileListItem {
  id: number;
  file: File;
  status: string | number;
  converted?: HTMLImageElement;
}

const convert = (
  image: CanvasImageSource,
  format: string
): HTMLImageElement | undefined => {
  const canvas = document.createElement('canvas')

  canvas.width = image.width as number
  canvas.height = image.height as number

  const context = canvas.getContext('2d')

  if (context) {
    context.drawImage(image, 0, 0)

    const newImage = new Image()
    newImage.src = canvas.toDataURL(format)

    return newImage
  } else {
    alert('Could not convert image')
  }
}

const urlToFile = (
  url: string,
  filename: string,
  mimeType: string
): Promise<File> => (
  fetch(url)
    .then(res => res.arrayBuffer())
    .then(buf => new File([buf], filename, { type: mimeType }))
)

const downloadFile = (filename: string, content: string) => {
  const element = document.createElement('a')
  element.setAttribute('href', content)
  element.setAttribute('download', filename)

  element.style.display = 'none'
  document.body.appendChild(element)
  element.click()
  document.body.removeChild(element)
}

export default defineComponent({
  setup() {
    const id = ref(0)

    const model = reactive({
      files: [],
      table: [],
      filetype: ''
    })

    const items: {
      list: FileListItem[];
    } = reactive({
      list: []
    })

    const state = reactive({
      filetypes: [
        { value: 'image/jpg', text: 'JPG' },
        { value: 'image/png', text: 'PNG' }
      ],
      headers: [
        { text: '#', value: 'id' },
        { text: 'File name', value: 'name' },
        { text: 'File format', value: 'format' },
        { text: 'File size', value: 'size' },
        { text: 'Last modified', value: 'modified' },
        { text: 'Progress', value: 'progress', sortable: false },
        { text: 'Action', value: 'action', sortable: false, align: 'end' }
      ],
      items: computed(() => items.list.map(
        (item: FileListItem) => ({
          id: item.id,
          status: item.status,
          name: item.file.name,
          format: item.file.type,
          size: filesize(item.file.size),
          modified: moment(item.file.lastModified).calendar()
        })
      ))
    })

    const deleteSelected = () => {
      const ids = model.table.map((row: any) => row.id)
      ids.forEach(id => { items.list = items.list.filter(item => item.id !== id) })
      model.table = []
    }

    const deleteItem = (id: number) => {
      items.list = items.list.filter(item => item.id !== id)
    }

    const convertAll = () => {
      items.list.forEach(async item => {
        item.status = 25
        const imageFile = new Image()
        const reader = new FileReader()
        reader.readAsDataURL(item.file)
        reader.onload = () => {
          item.status = 75
          imageFile.src = reader.result as string
          item.converted = convert(imageFile, model.filetype)
          item.status = 'done'
        }
      })
    }

    const download = (id: number) => {
      const item = items.list.find(item => item.id === id)

      if (item && item.converted) {
        downloadFile(item.file.name, item.converted.src)
      }
    }

    watch(() => model.files, newFiles => {
      newFiles.forEach(file => {
        id.value++

        items.list.push({
          id: id.value,
          file,
          status: 'ready'
        })
      })

      model.files.length = 0
    })

    return { model, state, deleteSelected, deleteItem, convertAll }
  }
})
</script>