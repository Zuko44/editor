<template>
  <div>
    <div class="toolbar">
      <button @click="undo">↶</button>
      <button @click="redo">↷</button>
      <button @click="setBlockFormat('h1')">H1</button>
      <button @click="setBlockFormat('p')">P</button>
      <button @click="insertImage">🖼️</button>
      <button @click="copyHTML">Скопировать HTML</button>
    </div>

    <div class="editor" ref="editor" contenteditable="true" @input="handleInput">
      Место для текста
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount } from 'vue'

const editor = ref<HTMLDivElement | null>(null)
const history = ref<string[]>([])
const historyIndex = ref<number>(-1)
let observer: MutationObserver

const saveStateToHistory = (): void => {
  if (editor.value) {
    const currentState = editor.value.innerHTML
    if (historyIndex.value === -1 || history.value[historyIndex.value] !== currentState) {
      history.value = history.value.slice(0, historyIndex.value + 1)
      history.value.push(currentState)
      historyIndex.value = history.value.length - 1
    }
  }
}

const undo = (): void => {
  if (historyIndex.value > 0) {
    historyIndex.value -= 1
    if (editor.value) {
      editor.value.innerHTML = history.value[historyIndex.value]
    }
  }
}

const redo = (): void => {
  if (historyIndex.value < history.value.length - 1) {
    historyIndex.value += 1
    if (editor.value) {
      editor.value.innerHTML = history.value[historyIndex.value]
    }
  }
}

const cleanUpEditorContent = (): void => {
  if (!editor.value) return

  editor.value.querySelectorAll('div, span').forEach((node) => {
    if (node.textContent?.trim() === '') {
      node.remove()
    } else if (node.tagName.toLowerCase() === 'div') {
      const p = document.createElement('p')
      while (node.firstChild) {
        p.appendChild(node.firstChild)
      }
      node.replaceWith(p)
    } else if (node.tagName.toLowerCase() === 'span') {
      const parent = node.parentNode
      while (node.firstChild) {
        parent?.insertBefore(node.firstChild, node)
      }
      node.remove()
    }
  })
}

const handleInput = (): void => {
  if (!editor.value) return

  const selection = window.getSelection()
  const range = selection && selection.rangeCount > 0 ? selection.getRangeAt(0) : null

  cleanUpEditorContent()

  if (range) {
    selection?.removeAllRanges()
    selection?.addRange(range)
  }

  saveStateToHistory()
  saveToLocalStorage()
}

const saveToLocalStorage = (): void => {
  if (editor.value) {
    localStorage.setItem('editorContent', editor.value.innerHTML)
  }
}

const observeEditorChanges = () => {
  if (!editor.value) return

  observer = new MutationObserver(() => {
    cleanUpEditorContent()
  })

  observer.observe(editor.value, {
    childList: true,
    subtree: true,
    characterData: true,
  })
}

onMounted(() => {
  observeEditorChanges()

  const savedContent = localStorage.getItem('editorContent')
  if (savedContent && editor.value) {
    editor.value.innerHTML = savedContent
    saveStateToHistory()
  }
})

onBeforeUnmount(() => {
  observer.disconnect()
})

const setBlockFormat = (tag: string): void => {
  if (editor.value) {
    const selection = window.getSelection()
    if (selection && selection.rangeCount > 0) {
      const range = selection.getRangeAt(0)
      const block = document.createElement(tag)
      block.textContent = range.toString()
      range.deleteContents()
      range.insertNode(block)
      selection.removeAllRanges()
      saveStateToHistory()
    }
  }
}

const insertImage = (): void => {
  const url = prompt('Введите URL изображения:')
  if (url && editor.value) {
    const img = document.createElement('img')
    img.src = url
    img.alt = 'Image'
    img.style.maxWidth = '100%'
    img.style.height = 'auto'

    img.onerror = (): void => {
      alert('Ошибка: изображение не может быть загружено. Проверьте URL.')
    }

    const selection = window.getSelection()
    if (selection && selection.rangeCount > 0) {
      const range = selection.getRangeAt(0)
      range.deleteContents()
      range.insertNode(img)
      saveStateToHistory()
    }
  } else {
    alert('Введите корректный URL-адрес изображения!')
  }
}

const copyHTML = (): void => {
  if (editor.value) {
    const rawHTML = editor.value.innerHTML
    const parser = new DOMParser()
    const doc = parser.parseFromString(rawHTML, 'text/html')
    cleanUpEditorContent()
    const cleanHTML = doc.body.innerHTML

    navigator.clipboard
      .writeText(cleanHTML)
      .then((): void => {
        alert('HTML скопирован!')
      })
      .catch((err: Error): void => {
        alert('Ошибка копирования: ' + err.message)
      })
  }
}
</script>

<style scoped>
.toolbar {
  display: flex;
  gap: 10px;
  margin-bottom: 10px;
}

.editor {
  border: 1px solid #ccc;
  padding: 10px;
  min-height: 200px;
  background: #1e1e1e;
  color: #fff;
  font-family: Arial, sans-serif;
}

button {
  padding: 5px 10px;
  cursor: pointer;
}
</style>
