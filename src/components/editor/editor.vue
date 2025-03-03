<template>
      <div :id="props.containerId" :style="{ height: `${height}px` }"></div>
</template>
<script setup lang="ts">
import * as monaco from 'monaco-editor';
import editorWorker from 'monaco-editor/esm/vs/editor/editor.worker?worker'
import { createSQLToken } from "@/components/editor/impl"
import { onMounted, onUnmounted } from '@vue/runtime-core';
import { format } from 'sql-formatter';
import { ref } from 'vue';
import { Request } from '@/apis/orderPostApis';
import { AxiosResponse } from 'axios';
import { Res } from '@/config/request';
import { CaretRightOutlined } from '@ant-design/icons-vue';
import { IDisposable } from 'monaco-editor';

self.MonacoEnvironment = {
      getWorker (_, label) {
            return new editorWorker()
      }
}

interface Props {
      containerId: string,
      readonly?: boolean,
      isQuery?: boolean
}

const props = withDefaults(defineProps<Props>(), {
      containerId: "",
      readonly: false,
      isQuery: false
})

const emit = defineEmits(['getValues', 'changeContent'])

let model = {} as monaco.editor.IStandaloneCodeEditor

const request = new Request

let completionProvider: any

const beautyFunc: monaco.editor.IActionDescriptor = {
      id: 'ms-beauty',
      label: 'SQL美化',
      keybindings: [
            monaco.KeyMod.CtrlCmd | monaco.KeyCode.KEY_B,
      ],
      contextMenuGroupId: 'navigation',
      contextMenuOrder: 1.5,
      run: function (ed: monaco.editor.ICodeEditor) {
            ed.setValue(format(ed.getValue()))
      }
}

const mergeDDL: monaco.editor.IActionDescriptor = {
      id: 'ms-merge',
      label: "多条alter语句合并",
      keybindings: [
            monaco.KeyMod.CtrlCmd | monaco.KeyCode.KEY_M,
      ],
      contextMenuGroupId: 'navigation',
      contextMenuOrder: 1.5,
      run: function (ed: monaco.editor.ICodeEditor) {
            let s = ed.getModel() as monaco.editor.ITextModel
            let sel = s.getValueInRange(ed.getSelection() as monaco.Selection)
            let sqls = ""
            if (sel !== "") {
                  sqls = sel
            } else {
                  sqls = ed.getValue()
            }
            request.Merge(sqls).then((res: AxiosResponse<Res<string>>) => ed.setValue(res.data.payload))
      }
}

const GetValueFunc: monaco.editor.IActionDescriptor = {
      id: 'ms-test',
      label: props.isQuery ? "查询" : "SQL检测",
      keybindings: [
            monaco.KeyMod.CtrlCmd | monaco.KeyCode.KEY_E,
      ],
      contextMenuGroupId: 'navigation',
      contextMenuOrder: 1.5,
      run: function (ed: monaco.editor.ICodeEditor) {
            let s = ed.getModel() as monaco.editor.ITextModel
            let sel = s.getValueInRange(ed.getSelection() as monaco.Selection)
            if (!props.isQuery) {
                  emit("getValues", ed.getValue())
                  return
            }
            if (sel !== "") {
                  emit("getValues", sel)
            } else {
                  emit("getValues", ed.getValue())
            }
      }
}

const RunEditor = (highlight: { [key: string]: string }[]) => {
      completionProvider !== undefined ? completionProvider.dispose() : null
      completionProvider = monaco.languages.registerCompletionItemProvider('sql', {
            provideCompletionItems: (model, position): monaco.languages.ProviderResult<monaco.languages.CompletionList> => {
                  let word = model.getWordUntilPosition(position);
                  let range = {
                        startLineNumber: position.lineNumber,
                        endLineNumber: position.lineNumber,
                        startColumn: word.startColumn,
                        endColumn: word.endColumn
                  };
                  return {
                        suggestions: createSQLToken(range, highlight)
                  }
            },
            triggerCharacters: ['.']

      });
      model.focus()
}



const ChangeEditorText = (sql: string) => {
      model.onDidChangeModelLanguageConfiguration(() => {

      })
      model.setValue(sql)
}

const GetValue = () => {
      return model.getValue()
}

const height = ref(250)

onMounted(() => {
      model = monaco.editor.create(document.getElementById(props.containerId) as HTMLElement, {
            language: "sql",
            fontSize: 16,
            theme: localStorage.getItem("theme") === "light" ? "vs-light" : "vs-dark",
            automaticLayout: true,
            readOnly: props.readonly,
            accessibilityHelpUrl: "https://next.yearning.io",
      });
      model.addAction(beautyFunc)
      model.addAction(GetValueFunc)
      props.isQuery ? null : model.addAction(mergeDDL)
      model.focus()
      window.onresize = () => {
            height.value = document.body.clientHeight - 600 > 150 ? document.body.clientHeight - 600 : 150
      }
      model.onDidChangeModelContent((e) => {
            emit("changeContent")
      })
})

onUnmounted(() => {
      model.dispose()
      completionProvider !== undefined ? completionProvider.dispose() : null
})

defineExpose({
      RunEditor,
      ChangeEditorText,
      GetValue
})

</script> 