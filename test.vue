<template>
  <BasicModal
    v-bind="$attrs"
    :title="getTitle"
    :width="700"
    @register="registerModal"
    @ok="handleOk"
  >
    <BasicForm @register="registerForm">
      <template #applySubnetSlot="{ field, model }">
        <SubnetFormList
          ref="subnetFormListRef"
          :disabled="model['type'] === 'approval'"
          :extra="model[field]"
          :subnetObj="subnetObj"
          @check="handleCheck"
          @record-netSystem="recordNetSystem"
          @record-subnet="recordSubnet"
          @open-address="openAddress"
        />
      </template>
      <template #distributeSubnetSlot="{ field, model }">
        <DistributeSubnetFormList :distributionList="model[field]" />
      </template>
    </BasicForm>
    <SubnetModal @register="registerAddressModal" @success="handleSuccess" />
  </BasicModal>
</template>

<script lang="ts" setup>
  import { ref } from 'vue';
  import { message } from 'ant-design-vue';
  import { useModal, useModalInner, BasicModal } from '/@/components/Modal';
  import { useForm, BasicForm, FormSchema } from '/@/components/Form';
  import SubnetFormList from './ApplySubnetFormList.vue';
  import DistributeSubnetFormList from './DistributeSubnetFormList.vue';
  import { addOrder, auditOrder } from '/@/api/ipam/approval/index';
  import SubnetModal from './SubnetModal.vue';

  const formSchema: FormSchema[] = [
    {
      field: 'id',
      label: 'id',
      component: 'Input',
      show: false,
      ifShow: ({ values }) => !!values.id,
    },
    {
      field: 'type',
      label: 'type',
      component: 'Input',
      show: false,
    },
    {
      field: '',
      component: 'Divider',
      colProps: { span: 24 },
      label: '申请信息',
    },
    {
      field: 'person',
      label: '申请人',
      component: 'Input',
      colProps: { span: 24 },
      required: true,
      componentProps: {
        placeholder: '请输入申请人',
        disabled: true,
      },
    },
    {
      field: 'userId',
      label: 'userId',
      component: 'InputNumber',
      colProps: { span: 24 },
      show: false,
    },
    {
      field: 'purpose',
      label: '用途',
      component: 'Input',
      colProps: { span: 24 },
      required: true,
      componentProps: {
        placeholder: '请输入用途',
      },
      dynamicDisabled: ({ values }) => values.type === 'approval',
    },
    {
      field: 'extra',
      label: '申请子网',
      component: 'Input',
      colSlot: 'applySubnetSlot',
      colProps: { span: 24 },
      defaultValue: [
        {
          networkSystemId: '',
          networkSystemName: '',
          disable: '',
          address: '',
          addressId: '',
          name: '',
          num: '',
          maskLength: '',
          children: '',
          gensubnets: '',
        },
      ],
    },
    {
      field: 'remark',
      label: '备注',
      component: 'InputTextArea',
      colProps: { span: 24 },
      componentProps: {
        placeholder: '请输入备注',
        rows: 2,
      },
      dynamicDisabled: ({ values }) => values.type === 'approval',
    },
    {
      field: '',
      component: 'Divider',
      colProps: { span: 24 },
      label: '分配信息',
      ifShow: ({ values }) => values.type === 'approval',
    },
    {
      field: 'distributionList',
      label: '分配子网',
      component: 'Input',
      colSlot: 'distributeSubnetSlot',
      colProps: { span: 24 },
      defaultValue: [{ name: '测试子网1', subnet: '12313213212' }],
      ifShow: ({ values }) => values.type === 'approval',
    },
  ];

  const getTitle = ref();
  const networkSystem = ref();
  const subnetObj = ref();
  const subnetFormListRef = ref();
  const emit = defineEmits(['success', 'register']);

  const [registerForm, { resetFields, setFieldsValue, validate, validateFields, getFieldsValue }] =
    useForm({
      labelWidth: 70,
      colon: true,
      schemas: formSchema,
      showActionButtonGroup: false,
    });

  const [registerAddressModal, { openModal: openAddressModal }] = useModal();

  const openAddress = (index) => {
    openAddressModal(true, { index });
  };

  const [registerModal, { closeModal, setModalProps }] = useModalInner(({ type, record }) => {
    resetFields();

    if (type === 'apply') {
      getTitle.value = '子网申请';
      setFieldsValue({
        type: 'apply',
        ...record,
      });
    } else {
      getTitle.value = '子网审核';
      console.log({ record });
      setFieldsValue({
        type: 'approval',
        ...record,
      });
    }
  });

  const handleAddOrder = async (values) => {
    const { person, userId, purpose, remark, extra } = values;

    if (!Array.isArray(extra)) return;

    const extra1 = extra.map(({ children, gensubnets, ...rest }) => {
      return rest;
    });

    const gensubnets = extra.map((ele) => {
      const { gensubnets, addressId, disable, networkSystemId, networkSystemName } = ele;
      const list = gensubnets.split(',');
      return list.map((item) => {
        return {
          name: item,
          subnetAddress: item,
          status: 3,
          remark: '',
          pid: addressId,
          disable,
          networkSystemId,
          networkSystemName,
        };
      });
    });

    console.log('gensubnets', gensubnets);

    const data = gensubnets.flat();

    const params = {
      orderType: 0,
      person,
      userId,
      purpose,
      remark,
      extra: extra1,
      data,
    };
    console.log('params', params);

    try {
      setModalProps({ loading: true });
      await addOrder(params);
      message.success('提交成功');
      emit('success');
      closeModal();
    } finally {
      setModalProps({ loading: false });
    }
  };

  const handleApproval = async (values) => {
    const { id, status, remark, distributionList } = values;
    const params = {
      id,
      status,
      orderType: 0,
      person,
      userId,
      purpose,
      remark,
      data: distributionList,
    };

    try {
      setModalProps({ loading: true });
      await auditOrder(params);
      message.success('提交成功');
      emit('success');
      closeModal();
    } finally {
      setModalProps({ loading: false });
    }
  };

  const handleCheck = (index, name) => {
    console.log('handleCheck', index, name);
    validateFields([['extra', index, name]]);
  };

  const recordNetSystem = (value) => {
    networkSystem.value = value;
  };

  const recordSubnet = (value) => {
    subnetObj.value = value;
  };

  const handleSuccess = (index, value) => {
    //更新第几个子网
    // setFieldsValue({
    //   [('extra', index, 'address')]: value,
    // });
    console.log('handleSuccess', index, value);
    const values = getFieldsValue();
    const { extra } = values;
    console.log('extra', extra);
    const { address, addressId, name, disable, networkSystemId, networkSystemName, children } =
      value;
    extra[index].address = address;
    extra[index].addressId = addressId;
    extra[index].name = name;
    extra[index].disable = disable;
    extra[index].networkSystemId = networkSystemId;
    extra[index].networkSystemName = networkSystemName;
    extra[index].children = children;
    setFieldsValue({
      extra,
    });
  };

  const handleOk = async () => {
    // @TODO
    const values = await validate();
    console.log('handleOk', values);

    if (values.type === 'apply') {
      handleAddOrder(values);
    } else {
      handleApproval(values);
    }
  };
</script>
