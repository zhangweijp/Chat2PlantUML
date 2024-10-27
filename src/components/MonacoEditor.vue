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
      default: 'vs-light'  // 默认使用亮色主题
    }
  },
  emits: ['update:modelValue', 'change'],
  setup(props, { emit }) {
    const editorContainer = ref(null)
    let editor
    let resizeObserver

    onMounted(() => {
      // 定义自定义亮色主题
      monaco.editor.defineTheme('custom-light', {
        base: 'vs',
        inherit: true,
        rules: [],
        colors: {
          'editor.background': '#FFFFFF',
          'editor.lineHighlightBackground': '#F5F5F5',
          'editor.foreground': '#000000'
        }
      })

      // 定义自定义暗色主题
      monaco.editor.defineTheme('custom-dark', {
        base: 'vs-dark',
        inherit: true,
        rules: [],
        colors: {
          'editor.background': '#1E1E1E',
          'editor.lineHighlightBackground': '#282828',
          'editor.foreground': '#D4D4D4'
        }
      })

      editor = monaco.editor.create(editorContainer.value, {
        value: props.modelValue,
        language: props.language,
        theme: props.theme,
        automaticLayout: true,
        minimap: {
          enabled: false  // 禁用小地图
        },
        scrollBeyondLastLine: false,
        lineNumbers: 'on',
        roundedSelection: true,
        scrollbar: {
          vertical: 'visible',
          horizontal: 'visible',
          useShadows: false,
          verticalScrollbarSize: 10,
          horizontalScrollbarSize: 10
        }
      })

      editor.onDidChangeModelContent(() => {
        const value = editor.getValue()
        emit('update:modelValue', value)
        emit('change', value)
      })

      // 创建 ResizeObserver 来处理布局变化
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

    // 监听主题变化
    watch(() => props.theme, (newTheme) => {
      if (editor) {
        monaco.editor.setTheme(newTheme)
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

<style>
/* 自定义滚动条样式 */
.monaco-editor .scrollbar .slider {
  background: rgba(100, 100, 100, 0.4) !important;
  border-radius: 10px !important;
}

.monaco-editor .scrollbar .slider:hover {
  background: rgba(100, 100, 100, 0.6) !important;
}
</style>
