<template>
  <div ref="editorContainer" style="height: 100%; width: 100%;"></div>
</template>

<script>
import { ref, onMounted, watch, onBeforeUnmount } from 'vue'
import * as monaco from 'monaco-editor'

export default {
  props: {
    modelValue: {
      type: String,
      default: ''
    },
    language: {
      type: String,
      default: 'plaintext'
    }
  },
  emits: ['update:modelValue', 'change'],
  setup(props, { emit }) {
    const editorContainer = ref(null)
    let editor
    let resizeObserver

    onMounted(() => {
      editor = monaco.editor.create(editorContainer.value, {
        value: props.modelValue,
        language: props.language,
        theme: 'vs-dark',
        automaticLayout: true
      })

      editor.onDidChangeModelContent(() => {
        const value = editor.getValue()
        emit('update:modelValue', value)
        emit('change', value)
      })

      // Create a ResizeObserver to handle layout changes
      resizeObserver = new ResizeObserver(() => {
        if (editor) {
          editor.layout()
        }
      })
      resizeObserver.observe(editorContainer.value)
    })

    watch(() => props.modelValue, (newValue) => {
      if (editor && newValue !== editor.getValue()) {
        editor.setValue(newValue)
      }
    })

    onBeforeUnmount(() => {
      if (editor) {
        editor.dispose()
      }
      if (resizeObserver) {
        resizeObserver.disconnect()
      }
    })

    return {
      editorContainer
    }
  }
}
</script>