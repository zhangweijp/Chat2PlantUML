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
    },
    theme: {
      type: String,
      default: 'custom-light' // 将默认主题改为 'custom-light'
    }
  },
  emits: ['update:modelValue', 'change'],
  setup(props, { emit }) {
    const editorContainer = ref(null)
    let editor

    const createCustomTheme = () => {
      monaco.editor.defineTheme('custom-light', {
        base: 'vs',
        inherit: true,
        rules: [],
        colors: {
          'editor.background': '#FFFFFF', // 白色背景
          'editor.lineHighlightBackground': '#F5F5F5', // 当前行高亮颜色
        }
      })
    }

    onMounted(() => {
      createCustomTheme()

      editor = monaco.editor.create(editorContainer.value, {
        value: props.modelValue,
        language: props.language,
        theme: 'custom-light', // 使用自定义亮色主题
        automaticLayout: true
      })

      editor.onDidChangeModelContent(() => {
        const value = editor.getValue()
        emit('update:modelValue', value)
        emit('change', value)
      })

      // 将 monaco 对象暴露给全局，以便在 App.vue 中使用
      window.monaco = monaco
    })

    watch(() => props.modelValue, (newValue) => {
      if (editor && newValue !== editor.getValue()) {
        editor.setValue(newValue)
      }
    })

    watch(() => props.theme, (newTheme) => {
      if (editor) {
        monaco.editor.setTheme(newTheme === 'vs-light' ? 'custom-light' : 'vs-dark')
      }
    })

    onBeforeUnmount(() => {
      if (editor) {
        editor.dispose()
      }
    })

    return {
      editorContainer
    }
  }
}
</script>
