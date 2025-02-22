<template>
  <SizeObserver
    v-on:sizeChanged="sizeChanged()"
    v-non-collapsing>
    <div
      :id="editorId"
      class="code-area"
    />
  </SizeObserver>
</template>

<script lang="ts">
import {
  defineComponent, onUnmounted, onMounted, inject,
} from 'vue';
import { useCollectionStateKey } from '@/presentation/injectionSymbols';
import { ICodeChangedEvent } from '@/application/Context/State/Code/Event/ICodeChangedEvent';
import { IScript } from '@/domain/IScript';
import { ScriptingLanguage } from '@/domain/ScriptingLanguage';
import { IReadOnlyCategoryCollectionState } from '@/application/Context/State/ICategoryCollectionState';
import { CodeBuilderFactory } from '@/application/Context/State/Code/Generation/CodeBuilderFactory';
import SizeObserver from '@/presentation/components/Shared/SizeObserver.vue';
import { NonCollapsing } from '@/presentation/components/Scripts/View/Cards/NonCollapsingDirective';
import ace from './ace-importer';

export default defineComponent({
  props: {
    theme: {
      type: String,
      default: undefined,
    },
  },
  components: {
    SizeObserver,
  },
  directives: {
    NonCollapsing,
  },
  setup(props) {
    const { onStateChange, currentState, events } = inject(useCollectionStateKey)();

    const editorId = 'codeEditor';
    let editor: ace.Ace.Editor | undefined;
    let currentMarkerId: number | undefined;

    onUnmounted(() => {
      destroyEditor();
    });

    onMounted(() => { // allow editor HTML to render
      onStateChange((newState) => {
        handleNewState(newState);
      }, { immediate: true });
    });

    function handleNewState(newState: IReadOnlyCategoryCollectionState) {
      destroyEditor();
      editor = initializeEditor(
        props.theme,
        editorId,
        newState.collection.scripting.language,
      );
      const appCode = newState.code;
      const innerCode = appCode.current || getDefaultCode(newState.collection.scripting.language);
      editor.setValue(innerCode, 1);
      events.unsubscribeAll();
      events.register(appCode.changed.on((code) => updateCode(code)));
    }

    function updateCode(event: ICodeChangedEvent) {
      removeCurrentHighlighting();
      if (event.isEmpty()) {
        const defaultCode = getDefaultCode(currentState.value.collection.scripting.language);
        editor.setValue(defaultCode, 1);
        return;
      }
      editor.setValue(event.code, 1);
      if (event.addedScripts?.length > 0) {
        reactToChanges(event, event.addedScripts);
      } else if (event.changedScripts?.length > 0) {
        reactToChanges(event, event.changedScripts);
      }
    }

    function sizeChanged() {
      editor?.resize();
    }

    function destroyEditor() {
      editor?.destroy();
      editor = undefined;
    }

    function removeCurrentHighlighting() {
      if (!currentMarkerId) {
        return;
      }
      editor.session.removeMarker(currentMarkerId);
      currentMarkerId = undefined;
    }

    function reactToChanges(event: ICodeChangedEvent, scripts: ReadonlyArray<IScript>) {
      const positions = scripts
        .map((script) => event.getScriptPositionInCode(script));
      const start = Math.min(
        ...positions.map((position) => position.startLine),
      );
      const end = Math.max(
        ...positions.map((position) => position.endLine),
      );
      scrollToLine(end + 2);
      highlight(start, end);
    }

    function highlight(startRow: number, endRow: number) {
      const AceRange = ace.require('ace/range').Range;
      currentMarkerId = editor.session.addMarker(
        new AceRange(startRow, 0, endRow, 0),
        'code-area__highlight',
        'fullLine',
      );
    }

    function scrollToLine(row: number) {
      const column = editor.session.getLine(row).length;
      editor.gotoLine(row, column, true);
    }

    return {
      editorId,
      sizeChanged,
    };
  },
});

function initializeEditor(
  theme: string | undefined,
  editorId: string,
  language: ScriptingLanguage,
): ace.Ace.Editor {
  theme = theme || 'github';
  const editor = ace.edit(editorId);
  const lang = getLanguage(language);
  editor.getSession().setMode(`ace/mode/${lang}`);
  editor.setTheme(`ace/theme/${theme}`);
  editor.setReadOnly(true);
  editor.setAutoScrollEditorIntoView(true);
  editor.setShowPrintMargin(false); // hides vertical line
  editor.getSession().setUseWrapMode(true); // So code is readable on mobile
  return editor;
}

function getLanguage(language: ScriptingLanguage) {
  switch (language) {
    case ScriptingLanguage.batchfile: return 'batchfile';
    case ScriptingLanguage.shellscript: return 'sh';
    default:
      throw new Error('unknown language');
  }
}

function getDefaultCode(language: ScriptingLanguage): string {
  return new CodeBuilderFactory()
    .create(language)
    .appendCommentLine('privacy.sexy — Now you have the choice.')
    .appendCommentLine(' 🔐 Enforce privacy & security best-practices on Windows, macOS and Linux.')
    .appendLine()
    .appendCommentLine('-- 🤔 How to use')
    .appendCommentLine(' 📙 Start by exploring different categories and choosing different tweaks.')
    .appendCommentLine(' 📙 On top left, you can apply predefined selections for privacy level you\'d like.')
    .appendCommentLine(' 📙 After you choose any tweak, you can download or copy to execute your script.')
    .appendCommentLine(' 📙 Come back regularly to apply latest version for stronger privacy and security.')
    .appendLine()
    .appendCommentLine('-- 🧐 Why privacy.sexy')
    .appendCommentLine(' ✔️ Rich tweak pool to harden security & privacy of the OS and other software on it.')
    .appendCommentLine(' ✔️ No need to run any compiled software on your system, just run the generated scripts.')
    .appendCommentLine(' ✔️ Have full visibility into what the tweaks do as you enable them.')
    .appendCommentLine(' ✔️ Open-source and free (both free as in beer and free as in speech).')
    .toString();
}

</script>

<style scoped lang="scss">
@use "@/presentation/assets/styles/main" as *;

::v-deep .code-area {
  min-height: 200px;
  width: 100%;
  height: 100%;
  overflow: auto;
  &__highlight {
    background-color: $color-secondary-light;
    position: absolute;
  }
}
</style>
