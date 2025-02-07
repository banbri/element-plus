<template>
  <div
    :class="[
      ppNs.b(),
      drpNs.b(),
      {
        'has-sidebar': Boolean($slots.sidebar) || hasShortcuts,
      },
    ]"
  >
    <div :class="ppNs.e('body-wrapper')">
      <slot name="sidebar" :class="ppNs.e('sidebar')" />
      <div v-if="hasShortcuts" :class="ppNs.e('sidebar')">
        <button
          v-for="(shortcut, key) in shortcuts"
          :key="key"
          type="button"
          :class="ppNs.e('shortcut')"
          @click="handleShortcutClick(shortcut)"
        >
          {{ shortcut.text }}
        </button>
      </div>
      <div :class="ppNs.e('body')">
        <div :class="[ppNs.e('content'), drpNs.e('content')]" class="is-left">
          <div :class="drpNs.e('header')">
            <button
              type="button"
              :class="ppNs.e('icon-btn')"
              class="d-arrow-left"
              @click="leftPrevYear"
            >
              <el-icon><d-arrow-left /></el-icon>
            </button>
            <button
              v-if="unlinkPanels"
              type="button"
              :disabled="!enableYearArrow"
              :class="[
                ppNs.e('icon-btn'),
                { [ppNs.is('disabled')]: !enableYearArrow },
              ]"
              class="d-arrow-right"
              @click="leftNextYear"
            >
              <el-icon><d-arrow-right /></el-icon>
            </button>
            <div>{{ leftLabel }}</div>
          </div>
          <month-table
            selection-mode="range"
            :date="leftDate"
            :min-date="minDate"
            :max-date="maxDate"
            :range-state="rangeState"
            :disabled-date="disabledDate"
            @changerange="handleChangeRange"
            @pick="handleRangePick"
            @select="onSelect"
          />
        </div>
        <div :class="[ppNs.e('content'), drpNs.e('content')]" class="is-right">
          <div :class="drpNs.e('header')">
            <button
              v-if="unlinkPanels"
              type="button"
              :disabled="!enableYearArrow"
              :class="[ppNs.e('icon-btn'), { 'is-disabled': !enableYearArrow }]"
              class="d-arrow-left"
              @click="rightPrevYear"
            >
              <el-icon><d-arrow-left /></el-icon>
            </button>
            <button
              type="button"
              :class="ppNs.e('icon-btn')"
              class="d-arrow-right"
              @click="rightNextYear"
            >
              <el-icon><d-arrow-right /></el-icon>
            </button>
            <div>{{ rightLabel }}</div>
          </div>
          <month-table
            selection-mode="range"
            :date="rightDate"
            :min-date="minDate"
            :max-date="maxDate"
            :range-state="rangeState"
            :disabled-date="disabledDate"
            @changerange="handleChangeRange"
            @pick="handleRangePick"
            @select="onSelect"
          />
        </div>
      </div>
    </div>
  </div>
</template>

<script lang="ts" setup>
import { computed, inject, ref, toRef, useAttrs, useSlots, watch } from 'vue'
import dayjs from 'dayjs'
import ElIcon from '@element-plus/components/icon'
import { useLocale, useNamespace } from '@element-plus/hooks'
import { isArray, isFunction } from '@element-plus/utils'
import { DArrowLeft, DArrowRight } from '@element-plus/icons-vue'
import { ROOT_PICKER_INJECTION_KEY } from '@element-plus/tokens'
import {
  panelMonthRangeEmits,
  panelMonthRangeProps,
} from '../props/panel-month-range'
import { useMonthRangeHeader } from '../composables/use-month-range-header'
import MonthTable from './basic-month-table.vue'

import type { SetupContext } from 'vue'
import type { Dayjs } from 'dayjs'
import type { RangeState } from '../props/shared'

defineOptions({
  name: 'DatePickerMonthRange',
})

const props = defineProps(panelMonthRangeProps)
const emit = defineEmits(panelMonthRangeEmits)
const attrs = useAttrs()
const slots = useSlots()

const { pickerNs: ppNs } = inject(ROOT_PICKER_INJECTION_KEY)!
const drpNs = useNamespace('date-range-picker')
const { lang } = useLocale()
const leftDate = ref(dayjs().locale(lang.value))
const rightDate = ref(dayjs().locale(lang.value).add(1, 'year'))

const hasShortcuts = computed(() => !!shortcuts.length)

// FIXME: extract this to `date-picker.ts`
type Shortcut = {
  text: string
  value: [Date, Date] | (() => [Date, Date])
  onClick?: (
    ctx: Omit<SetupContext<typeof panelMonthRangeEmits>, 'expose'>
  ) => void
}

const handleShortcutClick = (shortcut: Shortcut) => {
  const shortcutValues = isFunction(shortcut.value)
    ? shortcut.value()
    : shortcut.value

  if (shortcutValues) {
    emit('pick', [
      dayjs(shortcutValues[0]).locale(lang.value),
      dayjs(shortcutValues[1]).locale(lang.value),
    ])
    return
  }
  if (shortcut.onClick) {
    shortcut.onClick({
      attrs,
      slots,
      emit,
    })
  }
}

const {
  leftPrevYear,
  rightNextYear,
  leftNextYear,
  rightPrevYear,
  leftLabel,
  rightLabel,
  leftYear,
  rightYear,
} = useMonthRangeHeader({
  unlinkPanels: toRef(props, 'unlinkPanels'),
  leftDate,
  rightDate,
})

const enableYearArrow = computed(() => {
  return props.unlinkPanels && rightYear.value > leftYear.value + 1
})

const minDate = ref<Dayjs>()
const maxDate = ref<Dayjs>()

const rangeState = ref<RangeState>({
  endDate: null,
  selecting: false,
})

const handleChangeRange = (val: RangeState) => {
  rangeState.value = val
}

type RangePickValue = {
  minDate: Dayjs
  maxDate: Dayjs
}

const handleRangePick = (val: RangePickValue, close = true) => {
  // const defaultTime = props.defaultTime || []
  // const minDate_ = modifyWithTimeString(val.minDate, defaultTime[0])
  // const maxDate_ = modifyWithTimeString(val.maxDate, defaultTime[1])
  // todo
  const minDate_ = val.minDate
  const maxDate_ = val.maxDate
  if (maxDate.value === maxDate_ && minDate.value === minDate_) {
    return
  }
  maxDate.value = maxDate_
  minDate.value = minDate_

  if (!close) return
  handleConfirm()
}

const isValidValue = (value: [Dayjs | undefined, Dayjs | undefined]) => {
  return (
    isArray(value) &&
    value &&
    value[0] &&
    value[1] &&
    value[0].valueOf() <= value[1].valueOf()
  )
}

const handleConfirm = (visible = false) => {
  if (isValidValue([minDate.value, maxDate.value])) {
    emit('pick', [minDate.value, maxDate.value], visible)
  }
}

const onSelect = (selecting: boolean) => {
  rangeState.value.selecting = selecting
  if (!selecting) {
    rangeState.value.endDate = null
  }
}

const formatToString = (days: Dayjs[]) => {
  return days.map((day) => day.format(format))
}

const getDefaultValue = () => {
  let start: Dayjs
  if (isArray(defaultValue.value)) {
    const left = dayjs(defaultValue.value[0])
    let right = dayjs(defaultValue.value[1])
    if (!props.unlinkPanels) {
      right = left.add(1, 'year')
    }
    return [left, right]
  } else if (defaultValue.value) {
    start = dayjs(defaultValue.value)
  } else {
    start = dayjs()
  }
  start = start.locale(lang.value)
  return [start, start.add(1, 'year')]
}

// pickerBase.hub.emit('SetPickerOption', ['isValidValue', isValidValue])
emit('set-picker-option', ['formatToString', formatToString])
const pickerBase = inject('EP_PICKER_BASE') as any
const { shortcuts, disabledDate, format } = pickerBase.props
const defaultValue = toRef(pickerBase.props, 'defaultValue')

watch(
  () => defaultValue.value,
  (val) => {
    if (val) {
      const defaultArr = getDefaultValue()
      leftDate.value = defaultArr[0]
      rightDate.value = defaultArr[1]
    }
  },
  { immediate: true }
)

watch(
  () => props.parsedValue,
  (newVal) => {
    if (newVal && newVal.length === 2) {
      minDate.value = newVal[0]
      maxDate.value = newVal[1]
      leftDate.value = minDate.value
      if (props.unlinkPanels && maxDate.value) {
        const minDateYear = minDate.value.year()
        const maxDateYear = maxDate.value.year()
        rightDate.value =
          minDateYear === maxDateYear
            ? maxDate.value.add(1, 'year')
            : maxDate.value
      } else {
        rightDate.value = leftDate.value.add(1, 'year')
      }
    } else {
      const defaultArr = getDefaultValue()
      minDate.value = undefined
      maxDate.value = undefined
      leftDate.value = defaultArr[0]
      rightDate.value = defaultArr[1]
    }
  },
  { immediate: true }
)
</script>
