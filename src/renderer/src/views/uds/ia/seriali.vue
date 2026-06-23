<template>
  <div style="display: relative">
    <VxeGrid ref="xGrid" v-bind="gridOptions" class="sequenceTable" @cell-click="ceilClick">
      <template #default_send="{ row, rowIndex }">
        <el-button
          v-if="row.trigger.type == 'manual'"
          type="primary"
          size="small"
          plain
          style="width: 70px"
          :disabled="!globalStart"
          @click="sendFrame(rowIndex)"
        >
          <Icon :icon="sendIcon" />
        </el-button>
        <el-button
          v-else
          :type="periodTimer[rowIndex] ? 'danger' : 'primary'"
          size="small"
          plain
          style="width: 70px"
          :disabled="!globalStart"
          @click="sendFrame(rowIndex)"
        >
          <Icon :icon="periodTimer[rowIndex] ? stopIcon : sendIcon" />
        </el-button>
      </template>
      <template #default_trigger="{ row, rowIndex }">
        <span class="lr">
          <span>
            {{ row.trigger.type.toUpperCase() }}
            <span v-if="row.trigger.type == 'periodic'" style="padding: 0 5px">
              ({{ row.trigger.period || 100 }}ms)
            </span>
          </span>
          <el-button
            :ref="(e) => (buttonRef[rowIndex] = e)"
            link
            style="float: right"
            @click="openPr(rowIndex)"
          >
            <el-icon><ArrowDown /></el-icon>
          </el-button>
        </span>
      </template>
      <template #default_channel="{ row }">
        {{ devices[row.channel]?.name }}
      </template>
      <template #edit_channel="{ row }">
        <el-select v-model="row.channel" size="small" style="width: 100%" clearable>
          <el-option
            v-for="key in dataBase.ia[editIndex].devices"
            :key="key"
            :value="key"
            :label="devices[key]?.name"
          ></el-option>
        </el-select>
      </template>
      <template #default_canId="{ row }">
        {{ row.canId ? '0x' + row.canId.toUpperCase().padStart(3, '0') : '-' }}
      </template>
      <template #default_data="{ row }">
        {{ row.data.join(' ') }}
      </template>
      <template #toolbar>
        <div
          style="
            justify-content: flex-start;
            display: flex;
            align-items: center;
            gap: 2px;
            margin-left: 5px;
          "
        >
          <el-button-group>
            <el-tooltip effect="light" content="Edit Connect" placement="bottom">
              <el-button type="primary" link @click="editConnect">
                <Icon :icon="linkIcon" style="rotate: -45deg; font-size: 18px" />
              </el-button>
            </el-tooltip>
            <el-tooltip effect="light" content="Add Frame" placement="bottom">
              <el-button link @click="addFrame">
                <Icon :icon="addIcon" style="font-size: 18px" />
              </el-button>
            </el-tooltip>
            <el-tooltip effect="light" content="Edit Frame" placement="bottom">
              <el-button link type="success" :disabled="popoverIndex < 0" @click="editFrame">
                <Icon :icon="editIcon" style="font-size: 18px" />
              </el-button>
            </el-tooltip>
            <el-tooltip effect="light" content="Delete Frame" placement="bottom">
              <el-button
                link
                type="danger"
                :disabled="popoverIndex < 0 || periodTimer[popoverIndex] == true"
                @click="deleteFrame"
              >
                <Icon :icon="deleteIcon" style="font-size: 18px" />
              </el-button>
            </el-tooltip>
          </el-button-group>
        </div>
      </template>
    </VxeGrid>

    <el-popover width="250" :virtual-ref="ppRef" trigger="click" virtual-triggering>
      <el-row v-if="dataBase.ia[editIndex]?.action[popoverIndex]" style="padding: 10px">
        <el-col :span="24">
          <el-radio-group
            v-model="dataBase.ia[editIndex].action[popoverIndex].trigger.type"
            :disabled="periodTimer[popoverIndex]"
          >
            <el-radio value="manual">Manual</el-radio>
            <el-radio value="periodic">Periodic</el-radio>
          </el-radio-group>
        </el-col>
        <el-col :span="12">
          <div>Period (ms)</div>
          <el-input-number
            v-model="dataBase.ia[editIndex].action[popoverIndex].trigger.period"
            size="small"
            style="width: 100%"
            controls-position="right"
            :min="1"
            :disabled="
              dataBase.ia[editIndex].action[popoverIndex].trigger.type != 'periodic' ||
              periodTimer[popoverIndex]
            "
            placeholder="100"
          />
        </el-col>
      </el-row>
    </el-popover>

    <!-- Device connect dialog -->
    <el-dialog
      v-if="connectV"
      v-model="connectV"
      title="Device Connect"
      width="590"
      align-center
      :append-to="`#win${editIndex}_ia`"
    >
      <div style="text-align: center; padding: 10px; width: 570px; height: 250px; overflow: auto">
        <el-transfer
          v-model="dataBase.ia[editIndex].devices"
          class="canit"
          style="text-align: left; display: inline-block"
          :data="allDeviceLabel"
          :titles="['Available', 'Assigned']"
        />
      </div>
    </el-dialog>

    <!-- Edit frame dialog -->
    <el-dialog
      v-if="editV && formData"
      v-model="editV"
      :title="`Edit: ${formData.name}`"
      width="500"
      align-center
      :append-to="`#win${editIndex}_ia`"
    >
      <div style="padding: 10px">
        <el-form :model="formData" label-width="80px" size="small">
          <el-form-item label="Name">
            <el-input v-model="formData.name" />
          </el-form-item>
          <el-form-item label="CAN ID">
            <el-input
              v-model="formData.canId"
              placeholder="e.g. 7DF"
              style="width: 100%"
              @input="(v) => onCanIdInput(v)"
            >
              <template #prepend>0x</template>
            </el-input>
          </el-form-item>
          <el-form-item label="Channel">
            <el-select v-model="formData.channel" style="width: 100%" clearable>
              <el-option
                v-for="key in dataBase.ia[editIndex].devices"
                :key="key"
                :value="key"
                :label="devices[key]?.name"
              ></el-option>
            </el-select>
          </el-form-item>
        </el-form>
        <div style="margin: 10px 0; font-size: 12px; color: var(--el-text-color-secondary)">
          Data (hex bytes, click to edit):
        </div>
        <div
          style="
            display: flex;
            flex-wrap: wrap;
            gap: 4px;
            padding: 8px;
            border: 1px solid var(--el-border-color);
            border-radius: 4px;
            min-height: 60px;
          "
        >
          <el-input
            v-for="(byte, i) in formData.data"
            :key="i"
            v-model="formData.data[i]"
            size="small"
            style="width: 40px; font-family: monospace"
            maxlength="2"
            @input="(v) => onByteInput(i, v)"
          />
          <el-button size="small" type="primary" plain @click="addByte">+</el-button>
          <el-button
            size="small"
            type="danger"
            plain
            :disabled="formData.data.length == 0"
            @click="removeByte"
          >
            -
          </el-button>
        </div>
      </div>
      <template #footer>
        <el-button size="small" @click="editV = false">Cancel</el-button>
        <el-button size="small" type="primary" @click="saveEdit">OK</el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script lang="ts" setup>
import { ref, computed, toRef, watch } from 'vue'
import { VxeGrid } from 'vxe-table'
import type { VxeGridProps } from 'vxe-table'
import { Icon } from '@iconify/vue'
import { ArrowDown } from '@element-plus/icons-vue'
import { v4 } from 'uuid'
import { cloneDeep } from 'lodash'
import { useDataStore } from '@r/stores/data'
import { useGlobalStart } from '@r/stores/runtime'
import { useRuntimeStore } from '@r/stores/runtime'
import type { SerialBaseInfo } from 'nodeCan/serial'
import type { SerialInterAction } from 'src/preload/data'
import sendIcon from '@iconify/icons-material-symbols/send'
import stopIcon from '@iconify/icons-material-symbols/stop'
import linkIcon from '@iconify/icons-material-symbols/add-link'
import addIcon from '@iconify/icons-material-symbols/add-circle-outline'
import editIcon from '@iconify/icons-material-symbols/edit-square-outline'
import deleteIcon from '@iconify/icons-material-symbols/delete'

const props = defineProps<{
  height: number
  editIndex: string
}>()

const editIndex = toRef(props, 'editIndex')
const dataBase = useDataStore()
const globalStart = useGlobalStart()
const runtime = useRuntimeStore()

const xGrid = ref()
const connectV = ref(false)
const editV = ref(false)
const popoverIndex = ref(-1)
const buttonRef = ref<Record<number, any>>({})
const ppRef = computed(() => buttonRef.value[popoverIndex.value])
const formData = ref<SerialInterAction | null>(null)

const periodTimer = computed<Record<number, boolean>>(() => {
  const result: Record<number, boolean> = {}
  for (const [key, value] of Object.entries(runtime.serialPeriods)) {
    const a = key.split('-')
    const item = a.slice(0, -1).join('-')
    const index = Number(a[a.length - 1])
    if (item === editIndex.value) {
      result[index] = value
    }
  }
  return result
})

const devices = computed(() => {
  const dd: Record<string, SerialBaseInfo> = {}
  for (const d in dataBase.devices) {
    if (dataBase.devices[d]?.type == 'serial' && dataBase.devices[d].serialDevice) {
      dd[d] = dataBase.devices[d].serialDevice
    }
  }
  return dd
})

interface Option {
  key: string
  label: string
  disabled: boolean
}
const allDeviceLabel = computed<Option[]>(() => {
  return Object.keys(devices.value).map((key) => ({
    key,
    label: devices.value[key].name,
    disabled: false
  }))
})

watch(globalStart, (v) => {
  if (!v) {
    for (const key of Object.keys(runtime.serialPeriods)) {
      if (key.startsWith(editIndex.value + '-')) {
        runtime.removeSerialPeriod(key)
      }
    }
  }
})

function ceilClick(val: any) {
  popoverIndex.value = val.rowIndex
}

function openPr(rowIndex: number) {
  popoverIndex.value = rowIndex
}

function addFrame() {
  const channel = Object.keys(devices.value)[0] || ''
  ;(dataBase.ia[editIndex.value].action as SerialInterAction[]).push({
    uuid: v4(),
    trigger: { type: 'manual' },
    name: `Frame${dataBase.ia[editIndex.value].action.length}`,
    channel,
    canId: '7DF',
    data: ['00']
  })
}

function editFrame() {
  if (popoverIndex.value < 0) return
  formData.value = cloneDeep(
    dataBase.ia[editIndex.value].action[popoverIndex.value] as SerialInterAction
  )
  editV.value = true
}

function saveEdit() {
  if (formData.value) {
    dataBase.ia[editIndex.value].action[popoverIndex.value] = cloneDeep(formData.value)
  }
  editV.value = false
}

function deleteFrame() {
  if (popoverIndex.value >= 0) {
    dataBase.ia[editIndex.value].action.splice(popoverIndex.value, 1)
    popoverIndex.value = -1
    xGrid.value?.clearCurrentRow()
  }
}

function editConnect() {
  connectV.value = true
}

function sendFrame(index: number) {
  const frame = dataBase.ia[editIndex.value]?.action[index] as SerialInterAction
  if (!frame) return
  if (frame.trigger.type == 'manual') {
    window.electron.ipcRenderer.send('ipc-send-serial', cloneDeep(frame))
  } else {
    const key = `${editIndex.value}-${index}`
    if (runtime.serialPeriods[key]) {
      runtime.removeSerialPeriod(key)
      window.electron.ipcRenderer.send('ipc-stop-serial-period', key)
    } else {
      runtime.setSerialPeriod(key, true)
      window.electron.ipcRenderer.send('ipc-send-serial-period', key, cloneDeep(frame))
    }
  }
}

function onByteInput(index: number, v: string) {
  if (!formData.value) return
  if (v.length > 0 && !/^[0-9a-fA-F]*$/.test(v)) {
    formData.value.data[index] = v.slice(0, -1)
  }
  if (v.length > 2) {
    formData.value.data[index] = v.slice(0, 2)
  }
}

function onCanIdInput(v: string) {
  if (!formData.value) return
  const clean = v.replace(/[^0-9a-fA-F]/g, '').slice(0, 8)
  formData.value.canId = clean
}

function addByte() {
  formData.value?.data.push('00')
}

function removeByte() {
  formData.value?.data.pop()
}

const gridOptions = computed<VxeGridProps<SerialInterAction>>(() => ({
  border: true,
  size: 'mini',
  columnConfig: { resizable: true },
  height: props.height,
  showOverflow: true,
  scrollY: { enabled: true, gt: 0 },
  rowConfig: { isCurrent: true, keyField: 'uuid' },
  editConfig: {
    trigger: 'click',
    mode: 'cell',
    showIcon: false,
    beforeEditMethod({ rowIndex }) {
      return periodTimer.value[rowIndex] !== true
    }
  },
  toolbarConfig: { slots: { tools: 'toolbar' } },
  align: 'center',
  columns: [
    { type: 'seq', width: 50, title: '#', align: 'center', fixed: 'left', resizable: false },
    {
      field: 'send',
      title: 'Send',
      width: 100,
      resizable: false,
      slots: { default: 'default_send' }
    },
    {
      field: 'trigger',
      title: 'Trigger',
      width: 180,
      resizable: false,
      slots: { default: 'default_trigger' }
    },
    {
      field: 'name',
      title: 'Name',
      minWidth: 100,
      editRender: { name: 'input' }
    },
    {
      field: 'channel',
      title: 'Channel',
      minWidth: 120,
      editRender: {},
      slots: { default: 'default_channel', edit: 'edit_channel' }
    },
    {
      field: 'canId',
      title: 'CAN ID',
      width: 100,
      slots: { default: 'default_canId' }
    },
    {
      field: 'data',
      title: 'Data (Hex)',
      minWidth: 200,
      slots: { default: 'default_data' }
    }
  ],
  data:
    dataBase.ia[editIndex.value]?.type == 'serial'
      ? (dataBase.ia[editIndex.value]?.action as SerialInterAction[]) || []
      : []
}))
</script>

<style scoped>
.lr {
  display: flex;
  align-items: center;
  justify-content: space-between;
}
</style>
