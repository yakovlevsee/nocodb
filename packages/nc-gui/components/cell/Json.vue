<script setup lang="ts">
import NcModal from '../nc/Modal.vue'

type ModelValueType = string | Record<string, any> | undefined | null

interface Props {
  modelValue: ModelValueType
}

interface Emits {
  (event: 'update:modelValue', model: string | null): void
}

const props = defineProps<Props>()

const emits = defineEmits<Emits>()

const { showNull } = useGlobal()

const editEnabled = inject(EditModeInj, ref(false))

const active = inject(ActiveCellInj, ref(false))

const isEditColumn = inject(EditColumnInj, ref(false))

const isForm = inject(IsFormInj, ref(false))
const isExpandedFormOpen = inject(IsExpandedFormOpenInj, ref(false))!

const readOnly = inject(ReadonlyInj, ref(false))

const vModel = useVModel(props, 'modelValue', emits)

const localValueState = ref<string | undefined | null>()

const error = ref<string | undefined>()

const _isExpanded = inject(JsonExpandInj, ref(false))

const isExpanded = ref(false)

const rowHeight = inject(RowHeightInj, ref(undefined))

const formatValue = (val: ModelValueType) => {
  return !val || val === 'null' ? null : val
}

const localValue = computed<ModelValueType>({
  get: () => localValueState.value,
  set: (val: ModelValueType) => {
    localValueState.value = formatValue(val) === null ? null : typeof val === 'object' ? JSON.stringify(val, null, 2) : val
    /** if form and not expanded then sync directly */
    if (isForm.value && !isExpanded.value) {
      vModel.value = formatValue(val) === null ? null : val
    }
  },
})

const formatJson = (json: string) => {
  try {
    return JSON.stringify(JSON.parse(json))
  } catch (e) {
    console.log(e)
    return json
  }
}

function setLocalValue(val: any) {
  try {
    localValue.value = formatValue(val) === null ? null : typeof val === 'string' ? JSON.stringify(JSON.parse(val), null, 2) : val
  } catch (e) {
    localValue.value = formatValue(val) === null ? null : val
  }
}

const clear = () => {
  error.value = undefined

  isExpanded.value = false

  editEnabled.value = false

  setLocalValue(vModel.value)
}

const onSave = () => {
  isExpanded.value = false

  editEnabled.value = false

  // avoid saving if error exists or value is same as previous
  if (error.value || localValue.value === vModel.value) return false

  vModel.value = formatValue(localValue.value) === null ? null : formatJson(localValue.value as string)
}

watch(
  vModel,
  (val) => {
    setLocalValue(val)
  },
  { immediate: true },
)

watch([localValue, editEnabled], () => {
  try {
    JSON.parse(localValue.value as string)

    error.value = undefined
  } catch (e: any) {
    if (localValue.value === undefined || localValue.value === null) return

    error.value = e
  }
})

watch(editEnabled, () => {
  isExpanded.value = false

  setLocalValue(vModel.value)
})

useSelectedCellKeyupListener(active, (e) => {
  switch (e.key) {
    case 'Enter':
      e.stopPropagation()
      if (e.shiftKey) {
        return true
      }
      if (editEnabled.value) {
        onSave()
      } else {
        editEnabled.value = true
      }
      break
  }
})

const inputWrapperRef = ref<HTMLElement | null>(null)

onClickOutside(inputWrapperRef, (e) => {
  if ((e.target as HTMLElement)?.closest('.nc-json-action')) return
  editEnabled.value = false
})

watch(isExpanded, () => {
  _isExpanded.value = isExpanded.value
})

const stopPropagation = (event: MouseEvent) => {
  event.stopPropagation()
}

watch(inputWrapperRef, () => {
  if (!isEditColumn.value) return

  // stop event propogation in edit to prevent close edit modal on clicking expanded modal overlay
  const modal = document.querySelector('.nc-json-expanded-modal') as HTMLElement

  if (isExpanded.value && modal?.parentElement) {
    modal.parentElement.addEventListener('click', stopPropagation)
    modal.parentElement.addEventListener('mousedown', stopPropagation)
    modal.parentElement.addEventListener('mouseup', stopPropagation)
  } else if (modal?.parentElement) {
    modal.parentElement.removeEventListener('click', stopPropagation)
    modal.parentElement.removeEventListener('mousedown', stopPropagation)
    modal.parentElement.removeEventListener('mouseup', stopPropagation)
  }
})

const overlayActions = computed(() => !isExpanded.value && (isExpandedFormOpen.value || isForm.value))
</script>

<template>
  <component
    :is="isExpanded ? NcModal : 'div'"
    v-model:visible="isExpanded"
    :closable="false"
    centered
    :footer="null"
    :class="{ 'group': isExpandedFormOpen || isForm, 'nc-data-cell': isExpandedFormOpen }"
    :wrap-class-name="isExpanded ? '!z-1051 nc-json-expanded-modal' : null"
  >
    <div
      v-if="editEnabled && !readOnly"
      class="flex flex-col w-full"
      :class="{ relative: overlayActions }"
      @mousedown.stop
      @mouseup.stop
      @click.stop
    >
      <div
        class="flex flex-row items-center justify-between space-x-2 nc-json-action pb-2"
        :class="{ 'absolute -top-1 right-1 z-50': overlayActions, 'top-1': isExpandedFormOpen && overlayActions }"
        @mousedown.stop
      >
        <NcButton
          type="secondary"
          :size="isExpanded ? 'small' : 'xxsmall'"
          @click="isExpanded = !isExpanded"
        >
          <CilFullscreenExit v-if="isExpanded" class="h-2.5" />
          <component v-else :is="iconMap.maximize" class="transform group-hover:(!text-grey-800) text-gray-700 w-3 h-3" />
        </NcButton>

        <div v-if="(!isForm && !isExpandedFormOpen) || isExpanded" class="flex flex-row my-1 space-x-1">
          <NcButton type="secondary" :size="isExpanded ? 'small' : 'xxsmall'" class="!rounded-lg" @click="clear">
            <div :class="isExpanded ? 'text-xs' : 'text-[10px] p-1'">{{ $t('general.cancel') }}</div>
          </NcButton>

          <NcButton
            :type="!isExpanded ? 'text' : 'primary'"
            :size="isExpanded ? 'small' : 'xxsmall'"
            class="nc-save-json-value-btn !rounded-lg"
            :class="{
              'nc-edit-modal': !isExpanded,
            }"
            :disabled="!!error || localValue === vModel"
            @click="onSave"
          >
            <div :class="isExpanded ? 'text-xs' : 'text-[10px] p-1'">{{ $t('general.save') }}</div>
          </NcButton>
        </div>
      </div>

      <LazyMonacoEditor
        ref="inputWrapperRef"
        :model-value="localValue || ''"
        class="min-w-full w-80"
        :class="{ 'expanded-editor': isExpanded, 'editor': !isExpanded }"
        :hide-minimap="true"
        :disable-deep-compare="true"
        :auto-focus="!isForm && !isExpandedFormOpen && !isEditColumn"
        @update:model-value="localValue = $event"
        @keydown.enter.stop
        @keydown.alt.stop
      />

      <span v-if="error" class="nc-cell-field text-xs w-full py-1 text-red-500">
        {{ error.toString() }}
      </span>
    </div>

    <span v-else-if="vModel === null && showNull" class="nc-cell-field nc-null uppercase">{{ $t('general.null') }}</span>

    <LazyCellClampedText v-else :value="vModel ? stringifyProp(vModel) : ''" :lines="rowHeight" class="nc-cell-field" />
  </component>
</template>

<style scoped lang="scss">
.expanded-editor {
  min-height: min(600px, 80vh);
}

.editor {
  min-height: min(200px, 10vh);
}

.nc-save-json-value-btn {
  &.nc-edit-modal:not(:disabled) {
    @apply !text-brand-500 !hover:text-brand-600;
  }
}

.nc-data-cell:focus-within {
  @apply !border-1 !border-brand-500 !rounded-lg !shadow-none !ring-0;
  // Mimic ant's input box
  transition-property: all;
  transition-timing-function: cubic-bezier(0.4, 0, 0.2, 1);
  transition-duration: 150ms;
  box-shadow: 0px 0px 0px 2px rgba(51, 102, 255, 0.24) !important;
}
.nc-data-cell {
  @apply border-1 border-gray-200 overflow-hidden rounded-lg shadow pt-2;
  box-shadow: 0px 0px 2px 0px rgba(0, 0, 0, 0.08);
}
</style>
