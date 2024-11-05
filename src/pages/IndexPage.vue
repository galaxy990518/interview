<template>
  <q-page class="row q-py-md">
    <!-- 左側新增區域 -->
    <div class="col-4 q-px-md">
      <q-form ref="formRef">
        <q-input
          v-model="tempData.name"
          label="姓名"
          :rules="[(val) => !!val || '姓名不可為空']"
          lazy-rules
        />
        <q-input
          v-model="tempData.age"
          label="年齡"
          type="number"
          :rules="[
            (val) => !!val || '年齡不可為空',
            (val) => (val && val > 0) || '請輸入正確年齡(應為正整數)',
          ]"
          lazy-rules
        />
        <q-btn
          :color="isEditing ? 'secondary' : 'primary'"
          class="q-mt-md"
          @click="handleSubmit"
        >
          {{ isEditing ? '更新' : '新增' }}
        </q-btn>
        <q-btn
          v-if="isEditing"
          color="grey"
          class="q-mt-md q-ml-sm"
          @click="cancelEdit"
        >
          取消
        </q-btn>
      </q-form>
    </div>

    <!-- 右側表格區域 -->
    <div class="col-8 q-px-md table-container">
      <q-table
        flat
        bordered
        ref="tableRef"
        :rows="blockData"
        :columns="(tableConfig as QTableProps['columns'])"
        row-key="id"
        hide-pagination
        separator="cell"
        :rows-per-page-options="[0]"
        :filter="filter"
        style="max-height: 100%"
      >
        <template v-slot:top>
          <div class="row full-width items-center q-mb-md">
            <q-input
              v-model="filter"
              placeholder="搜尋..."
              dense
              outlined
              class="col-3"
            >
              <template v-slot:append>
                <q-icon name="search" />
              </template>
            </q-input>
          </div>
        </template>

        <template v-slot:header="props">
          <q-tr :props="props">
            <q-th
              v-for="col in props.cols"
              :key="col.name"
              :props="props"
              :class="{ 'cursor-pointer': col.sortable }"
            >
              {{ col.label }}
            </q-th>
          </q-tr>
        </template>

        <template v-slot:body="props">
          <q-tr :props="props">
            <q-td
              v-for="col in props.cols"
              :key="col.name"
              :props="props"
              style="min-width: 120px"
            >
              <template v-if="col.name === 'actions'">
                <q-btn
                  v-for="(btn, index) in tableButtons"
                  :key="index"
                  @click="handleClickOption(btn, props.row)"
                  size="sm"
                  :color="btn.color"
                  round
                  dense
                  :icon="btn.icon"
                  class="q-ml-md"
                  padding="5px 5px"
                >
                  <q-tooltip
                    transition-show="scale"
                    transition-hide="scale"
                    anchor="top middle"
                    self="bottom middle"
                    :offset="[10, 10]"
                  >
                    {{ btn.label }}
                  </q-tooltip>
                </q-btn>
              </template>
              <template v-else>
                {{ col.value }}
              </template>
            </q-td>
          </q-tr>
        </template>
        <template v-slot:no-data="{ icon }">
          <div
            class="full-width row flex-center items-center text-primary q-gutter-sm"
            style="font-size: 18px"
          >
            <q-icon size="2em" :name="icon" />
            <span> 無相關資料 </span>
          </div>
        </template>
      </q-table>
    </div>
  </q-page>
</template>

<script setup lang="ts">
import axios from 'axios';
import { QTableProps, useQuasar, QForm } from 'quasar';
import { ref, onMounted, nextTick, watch } from 'vue';

const $q = useQuasar();
const isEditing = ref(false);
const editingId = ref(null);

interface btnType {
  label: string;
  icon: string;
  status: string;
  color: string;
}

interface DataType {
  id: number;
  name: string | null;
  age: number | null;
}

const blockData = ref<DataType[]>([]);
const originalData = ref<DataType[]>([]);
const tableConfig = ref([
  {
    label: '姓名',
    name: 'name',
    field: (row) => row.name,
    align: 'left',
    sortable: true,
    format: (val: string) => val,
  },
  {
    label: '年齡',
    name: 'age',
    field: (row) => row.age,
    align: 'left',
    sortable: true,
    format: (val: number) => val,
  },
  {
    label: '操作',
    name: 'actions',
    field: 'actions',
    align: 'center',
    sortable: false,
    style: 'width: 20px',
  },
]);

const tableButtons = ref<btnType[]>([
  {
    label: '編輯',
    icon: 'edit',
    status: 'edit',
    color: 'primary',
  },
  {
    label: '刪除',
    icon: 'delete',
    status: 'delete',
    color: 'negative',
  },
]);

const tempData = ref({
  name: null,
  age: null,
});

const formRef = ref<QForm | null>(null);

const filter = ref('');

onMounted(async () => {
  await fetchData();
});

async function fetchData() {
  try {
    const response = await axios.get(
      'https://dahua.metcfire.com.tw/api/CRUDTest/a'
    );
    originalData.value = response.data;
    blockData.value = response.data;
  } catch (error) {
    console.error('獲取數據失敗:', error);
    $q.notify({
      color: 'negative',
      message: '獲取數據失敗',
    });
  }
}

watch(filter, (newVal) => {
  if (!newVal) {
    blockData.value = [...originalData.value];
    return;
  }

  const searchText = newVal.toLowerCase();
  blockData.value = originalData.value.filter(
    (item) =>
      item.name.toLowerCase().includes(searchText) ||
      item.age.toString().includes(searchText)
  );
});

async function handleSubmit() {
  const isValid = await formRef.value.validate();
  if (!isValid) {
    return;
  }

  try {
    if (isEditing.value) {
      await axios.patch('https://dahua.metcfire.com.tw/api/CRUDTest', {
        id: editingId.value,
        ...tempData.value,
        age: Number(tempData.value.age),
      });
      $q.notify({
        color: 'positive',
        message: '更新成功',
      });
    } else {
      await axios.post('https://dahua.metcfire.com.tw/api/CRUDTest', {
        ...tempData.value,
        age: Number(tempData.value.age),
      });
      $q.notify({
        color: 'positive',
        message: '新增成功',
      });
    }
    await fetchData();
    resetForm();
  } catch (error) {
    console.error(isEditing.value ? '更新失敗:' : '新增失敗:', error);
    $q.notify({
      color: 'negative',
      message: isEditing.value ? '更新失敗' : '新增失敗',
    });
  }
}

function resetForm() {
  tempData.value = { name: null, age: null };
  isEditing.value = false;
  editingId.value = null;
  nextTick(() => {
    if (formRef.value) {
      formRef.value.resetValidation();
    }
  });
}

function cancelEdit() {
  resetForm();
}

async function handleClickOption(btn: btnType, data: DataType) {
  if (btn.status === 'edit') {
    tempData.value = {
      name: data.name,
      age: data.age,
    };
    isEditing.value = true;
    editingId.value = data.id;
  } else if (btn.status === 'delete') {
    $q.dialog({
      title: '提示',
      message: '是否確定刪除該筆資料?',
      cancel: {
        label: '取消',
        flat: true,
        color: 'grey',
      },
      ok: {
        label: '確定',
        flat: true,
        color: 'primary',
      },
      persistent: true,
    }).onOk(async () => {
      try {
        await axios.delete(
          `https://dahua.metcfire.com.tw/api/CRUDTest/${data.id}`
        );
        await fetchData();
        $q.notify({
          color: 'positive',
          message: '刪除成功',
        });
      } catch (error) {
        console.error('刪除失敗:', error);
        $q.notify({
          color: 'negative',
          message: '刪除失敗',
        });
      }
    });
  }
}
</script>

<style lang="scss" scoped>
.q-table th {
  font-size: 20px;
  font-weight: bold;
}

.q-table tbody td {
  font-size: 18px;
}

.q-page {
  height: calc(100vh - 32px);
  overflow: hidden;
}

.table-container {
  height: 100%;
  overflow: auto;
}
</style>
