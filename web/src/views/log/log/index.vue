<template>
  <div>
    <div class="n-layout-page-header">
      <n-card :bordered="false" title="访问日志">
        全局访问日志记录了系统中后台人员和客户端的操作记录以及服务响应情况
      </n-card>
    </div>
    <n-card :bordered="false" class="proCard">
      <BasicForm @register="register" @submit="handleSubmit" @reset="handleReset">
        <template #statusSlot="{ model, field }">
          <n-input v-model:value="model[field]" />
        </template>
      </BasicForm>

      <BasicTable
        :openChecked="true"
        :columns="columns"
        :request="loadDataTable"
        :row-key="(row) => row.id"
        ref="actionRef"
        :actionColumn="actionColumn"
        @update:checked-row-keys="onCheckedRow"
        :scroll-x="scrollX"
        :resizeHeightOffset="-20000"
      >
        <template #tableTitle>
          <n-button type="error" @click="batchDelete" :disabled="batchDeleteDisabled">
            <template #icon>
              <n-icon>
                <DeleteOutlined />
              </n-icon>
            </template>
            批量删除
          </n-button>
        </template>
      </BasicTable>
    </n-card>
  </div>
</template>

<script lang="ts" setup>
  import { computed, h, reactive, ref } from 'vue';
  import { useDialog, useMessage } from 'naive-ui';
  import { BasicTable, TableAction } from '@/components/Table';
  import { BasicForm, FormSchema, useForm } from '@/components/Form/index';
  import { getLogList, Delete } from '@/api/log/log';
  import { columns } from './columns';
  import { useRouter } from 'vue-router';
  import { DeleteOutlined } from '@vicons/antd';
  import { adaTableScrollX } from '@/utils/hotgo';

  const dialog = useDialog();
  const batchDeleteDisabled = ref(true);
  const checkedIds = ref([]);

  const schemas: FormSchema[] = [
    {
      field: 'member_id',
      component: 'NInput',
      label: '操作人',
      componentProps: {
        placeholder: '请输入操作人ID',
        onInput: (e: any) => {
          console.log(e);
        },
      },
      rules: [{ trigger: ['blur'] }],
    },
    {
      field: 'url',
      component: 'NInput',
      label: '访问路径',
      componentProps: {
        placeholder: '请输入访问路径',
        onInput: (e: any) => {
          console.log(e);
        },
      },
    },
    {
      field: 'ip',
      component: 'NInput',
      label: '访问IP',
      componentProps: {
        placeholder: '请输入IP地址',
        onInput: (e: any) => {
          console.log(e);
        },
      },
    },
    {
      field: 'method',
      component: 'NSelect',
      label: '请求方式',
      componentProps: {
        placeholder: '请选择请求方式',
        options: [
          {
            label: 'GET',
            value: 'GET',
          },
          {
            label: 'POST',
            value: 'POST',
          },
        ],
        onUpdateValue: (e: any) => {
          console.log(e);
        },
      },
    },
    {
      field: 'created_at',
      component: 'NDatePicker',
      label: '访问时间',
      componentProps: {
        type: 'datetimerange',
        clearable: true,
        // defaultValue: [new Date() - 86400000 * 30, new Date()],
        onUpdateValue: (e: any) => {
          console.log(e);
        },
      },
    },
    {
      field: 'take_up_time',
      component: 'NSelect',
      label: '请求耗时',
      componentProps: {
        placeholder: '请选择请求耗时',
        options: [
          {
            label: '50ms内',
            value: '50',
          },
          {
            label: '100ms内',
            value: '100',
          },
          {
            label: '200ms内',
            value: '200',
          },
          {
            label: '500ms内',
            value: '500',
          },
        ],
        onUpdateValue: (e: any) => {
          console.log(e);
        },
      },
    },
    {
      field: 'error_code',
      component: 'NSelect',
      label: '状态码',
      componentProps: {
        placeholder: '请选择状态码',
        options: [
          {
            label: '0 成功',
            value: '0',
          },
          {
            label: '-1 失败',
            value: '-1',
          },
        ],
        onUpdateValue: (e: any) => {
          console.log(e);
        },
      },
    },
  ];

  const router = useRouter();
  const message = useMessage();
  const actionRef = ref();
  const formParams = ref({});

  const params = ref({
    pageSize: 10,
  });

  const actionColumn = reactive({
    width: 160,
    title: '操作',
    key: 'action',
    fixed: 'right',
    render(record) {
      return h(TableAction as any, {
        style: 'button',
        actions: [
          {
            label: '查看详情',
            onClick: handleEdit.bind(null, record),
          },
          {
            label: '删除',
            onClick: handleDelete.bind(null, record),
          },
        ],
      });
    },
  });

  const scrollX = computed(() => {
    return adaTableScrollX(columns, actionColumn.width);
  });

  const [register, {}] = useForm({
    gridProps: { cols: '1 s:1 m:2 l:3 xl:4 2xl:4' },
    labelWidth: 80,
    schemas,
  });

  function onCheckedRow(rowKeys) {
    console.log(rowKeys);
    batchDeleteDisabled.value = rowKeys.length <= 0;
    checkedIds.value = rowKeys;
  }

  function handleDelete(record: Recordable) {
    dialog.warning({
      title: '警告',
      content: '你确定要删除？',
      positiveText: '确定',
      negativeText: '取消',
      onPositiveClick: () => {
        Delete(record).then((_res) => {
          message.success('操作成功');
          reloadTable();
        });
      },
      onNegativeClick: () => {
        // message.error('取消');
      },
    });
  }

  function batchDelete() {
    dialog.warning({
      title: '警告',
      content: '你确定要删除？',
      positiveText: '确定',
      negativeText: '取消',
      onPositiveClick: () => {
        Delete({ id: checkedIds.value }).then((_res) => {
          message.success('操作成功');
          reloadTable();
        });
      },
      onNegativeClick: () => {
        // message.error('取消');
      },
    });
  }

  const loadDataTable = async (res) => {
    return await getLogList({ ...formParams.value, ...params.value, ...res });
  };

  function reloadTable() {
    actionRef.value.reload();
  }

  function handleEdit(record: Recordable) {
    router.push({ name: 'log_view', params: { id: record.id } });
  }

  function handleSubmit(values: Recordable) {
    formParams.value = values;
    reloadTable();
  }

  function handleReset(values: Recordable) {
    formParams.value = {};
    reloadTable();
  }
</script>

<style lang="less" scoped></style>
