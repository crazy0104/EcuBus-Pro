<template>
  <el-form
    ref="ruleFormRef"
    :model="data"
    label-width="130px"
    size="small"
    :disabled="globalStart"
    :rules="rules"
    class="hardware"
    hide-required-asterisk
  >
    <el-form-item label="Address Name" required prop="name">
      <el-input v-model="data.name" />
    </el-form-item>
    <el-divider content-position="left">ISO-TP Parameters</el-divider>
    <el-form-item label-width="0">
      <el-col :span="12">
        <el-form-item label="N_As (ms)" prop="nAs">
          <el-input-number v-model.number="data.nAs" :min="0" controls-position="right" />
        </el-form-item>
      </el-col>
      <el-col :span="12">
        <el-form-item label="N_Ar (ms)" prop="nAr">
          <el-input-number v-model.number="data.nAr" :min="0" controls-position="right" />
        </el-form-item>
      </el-col>
    </el-form-item>
    <el-form-item label-width="0">
      <el-col :span="12">
        <el-form-item label="N_Bs (ms)" prop="nBs">
          <el-input-number v-model.number="data.nBs" :min="0" controls-position="right" />
        </el-form-item>
      </el-col>
      <el-col :span="12">
        <el-form-item label="N_Br (ms)" prop="nBr">
          <el-input-number v-model.number="data.nBr" :min="0" controls-position="right" />
        </el-form-item>
      </el-col>
    </el-form-item>
    <el-form-item label-width="0">
      <el-col :span="12">
        <el-form-item label="N_Cs (ms)" prop="nCs">
          <el-input-number v-model.number="data.nCs" :min="0" controls-position="right" />
        </el-form-item>
      </el-col>
      <el-col :span="12">
        <el-form-item label="N_Cr (ms)" prop="nCr">
          <el-input-number v-model.number="data.nCr" :min="0" controls-position="right" />
        </el-form-item>
      </el-col>
    </el-form-item>
    <el-form-item label-width="0">
      <el-col :span="12">
        <el-form-item label="STmin (ms)" prop="stMin">
          <el-input-number
            v-model.number="data.stMin"
            :min="0"
            :max="127"
            controls-position="right"
          />
        </el-form-item>
      </el-col>
      <el-col :span="12">
        <el-form-item label="BlockSize" prop="bs">
          <el-input-number v-model.number="data.bs" :min="0" :max="255" controls-position="right" />
        </el-form-item>
      </el-col>
    </el-form-item>
    <el-form-item label-width="0">
      <el-col :span="12">
        <el-form-item label="Max WTF" prop="maxWTF">
          <el-input-number v-model.number="data.maxWTF" :min="0" controls-position="right" />
        </el-form-item>
      </el-col>
      <el-col :span="12">
        <el-form-item label="Frame Size" prop="maxFrameSize">
          <el-input-number
            v-model.number="data.maxFrameSize"
            :min="7"
            :max="63"
            controls-position="right"
          />
        </el-form-item>
      </el-col>
    </el-form-item>
    <el-form-item label="Padding">
      <el-switch v-model="data.padding" />
      <el-input-number
        v-if="data.padding"
        v-model="data.paddingValue"
        :min="0"
        :max="255"
        style="margin-left: 10px; width: 120px"
        controls-position="right"
        placeholder="0xCC"
      />
    </el-form-item>
  </el-form>
</template>

<script lang="ts" setup>
import { ref, onMounted } from 'vue'
import { type FormRules, type FormInstance } from 'element-plus'
import { useGlobalStart } from '@r/stores/runtime'
import type { SerialAddr } from 'nodeCan/serial'
import { UdsAddress } from 'nodeCan/uds'

const ruleFormRef = ref<FormInstance>()
const globalStart = useGlobalStart()

const data = defineModel<SerialAddr>({
  required: true
})

defineProps<{
  index: number
  addrs: UdsAddress[]
}>()

const rules: FormRules = {
  name: [{ required: true, message: 'Please enter address name', trigger: 'blur' }]
}

onMounted(() => {
  ruleFormRef.value?.validate().catch(null)
})

async function dataValid() {
  await ruleFormRef.value?.validate()
}

defineExpose({
  dataValid
})
</script>

<style scoped>
.hardware {
  margin: 20px;
}
</style>
