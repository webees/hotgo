<template>
  <div>
    <n-card :bordered="false" class="proCard" title="岗位管理">
      <BasicForm
        @register="register"
        @submit="handleSubmit"
        @reset="handleReset"
        @keyup.enter="handleSubmit"
        ref="searchFormRef"
      >
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
        :resizeHeightOffset="-10000"
        :scroll-x="1090"
      >
        <template #tableTitle>
          <n-button type="primary" @click="addTable">
            <template #icon>
              <n-icon>
                <PlusOutlined />
              </n-icon>
            </template>
            添加岗位
          </n-button>
          &nbsp;
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

      <n-modal
        v-model:show="showModal"
        :show-icon="false"
        preset="dialog"
        :title="formParams?.id > 0 ? '编辑岗位 #' + formParams?.id : '添加岗位'"
      >
        <n-form
          :model="formParams"
          ref="formRef"
          label-placement="left"
          :label-width="80"
          class="py-4"
        >
          <n-form-item label="岗位名称" path="name">
            <n-input placeholder="请输入岗位名称" v-model:value="formParams.name" />
          </n-form-item>

          <n-form-item label="岗位编码" path="code">
            <n-input placeholder="请输入岗位编码" v-model:value="formParams.code" />
          </n-form-item>

          <n-form-item label="排序" path="sort">
            <n-input-number v-model:value="formParams.sort" clearable style="width: 100%" />
          </n-form-item>

          <n-form-item label="状态" path="status">
            <n-radio-group v-model:value="formParams.status" name="status">
              <n-radio-button
                v-for="status in statusOptions"
                :key="status.value"
                :value="status.value"
                :label="status.label"
              />
            </n-radio-group>
          </n-form-item>

          <n-form-item label="备注" path="remark">
            <n-input type="textarea" placeholder="请输入备注" v-model:value="formParams.remark" />
          </n-form-item>
        </n-form>

        <template #action>
          <n-space>
            <n-button @click="() => (showModal = false)">取消</n-button>
            <n-button type="info" :loading="formBtnLoading" @click="confirmForm">确定</n-button>
          </n-space>
        </template>
      </n-modal>
    </n-card>
  </div>
</template>

<script lang="ts" setup>
  import { h, reactive, ref } from 'vue';
  import { useDialog, useMessage } from 'naive-ui';
  import { BasicTable, TableAction } from '@/components/Table';
  import { BasicForm, FormSchema, useForm } from '@/components/Form/index';
  import { Delete, Edit, getPostList } from '@/api/org/post';
  import { columns } from './columns';
  import { DeleteOutlined, PlusOutlined } from '@vicons/antd';
  import { statusOptions } from '@/enums/optionsiEnum';
  import { defRangeShortcuts } from '@/utils/dateUtil';
  import { cloneDeep } from 'lodash-es';

  const params = ref<any>({
    pageSize: 10,
    name: '',
    code: '',
    status: null,
  });

  const schemas: FormSchema[] = [
    {
      field: 'name',
      component: 'NInput',
      label: '岗位名称',
      componentProps: {
        placeholder: '请输入岗位名称',
        onUpdateValue: (e: any) => {
          console.log(e);
        },
      },
      rules: [{ message: '请输入岗位名称', trigger: ['blur'] }],
    },
    {
      field: 'code',
      component: 'NInput',
      label: '岗位编码',
      componentProps: {
        placeholder: '请输入岗位编码',
        showButton: false,
        onUpdateValue: (e: any) => {
          console.log(e);
        },
      },
    },
    {
      field: 'status',
      component: 'NSelect',
      label: '状态',
      defaultValue: null,
      componentProps: {
        placeholder: '请选择类型',
        options: statusOptions,
        onUpdateValue: (e: any) => {
          console.log(e);
        },
      },
    },
    {
      field: 'createdAt',
      component: 'NDatePicker',
      label: '创建时间',
      componentProps: {
        type: 'datetimerange',
        clearable: true,
        shortcuts: defRangeShortcuts(),
        onUpdateValue: (e: any) => {
          console.log(e);
        },
      },
    },
  ];

  const message = useMessage();
  const actionRef = ref();
  const dialog = useDialog();
  const showModal = ref(false);
  const formBtnLoading = ref(false);
  const searchFormRef = ref<any>({});
  const formRef = ref<any>({});
  const batchDeleteDisabled = ref(true);
  const checkedIds = ref([]);

  const resetFormParams = {
    id: 0,
    code: '',
    name: '',
    remark: '',
    sort: null,
    status: 1,
  };
  const formParams = ref<any>(resetFormParams);
  const actionColumn = reactive({
    width: 150,
    title: '操作',
    key: 'action',
    fixed: 'right',
    render(record) {
      return h(TableAction as any, {
        style: 'button',
        actions: [
          {
            label: '编辑',
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

  const [register, {}] = useForm({
    gridProps: { cols: '1 s:1 m:2 l:3 xl:4 2xl:4' },
    labelWidth: 80,
    schemas,
  });

  function addTable() {
    showModal.value = true;
    formParams.value = cloneDeep(resetFormParams);
  }

  const loadDataTable = async (res) => {
    return await getPostList({ ...params.value, ...res, ...searchFormRef.value?.formModel });
  };

  function onCheckedRow(rowKeys) {
    console.log(rowKeys);
    batchDeleteDisabled.value = rowKeys.length <= 0;
    checkedIds.value = rowKeys;
  }

  function reloadTable() {
    actionRef.value.reload();
  }

  function confirmForm(e) {
    e.preventDefault();
    formBtnLoading.value = true;
    formRef.value.validate((errors) => {
      if (!errors) {
        Edit(formParams.value).then((_res) => {
          message.success('操作成功');
          setTimeout(() => {
            showModal.value = false;
            reloadTable();
            formParams.value = ref(resetFormParams);
          });
        });
      } else {
        message.error('请填写完整信息');
      }
      formBtnLoading.value = false;
    });
  }

  function handleEdit(record: Recordable) {
    showModal.value = true;
    formParams.value = record;
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
    });
  }

  function handleSubmit(values: Recordable) {
    console.log(values);
    params.value = values;
    reloadTable();
  }

  function handleReset(values: Recordable) {
    params.value = values;
    reloadTable();
  }
</script>

<style lang="less" scoped></style>
