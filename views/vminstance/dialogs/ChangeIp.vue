<template>
  <base-dialog @cancel="cancelDialog">
    <div slot="header">更换IP</div>
    <div slot="body">
      <dialog-selected-tips :count="params.data.length" action="更换IP" />
      <vxe-grid class="mb-2" :data="params.data" :columns="params.columns.slice(0, 3)" />
      <a-form :form="form.fc" hideRequiredMark>
        <a-form-item label="IP子网" v-bind="formItemLayout" class="mb-0">
          <a-row :gutter="20">
            <a-col :span="14">
              <a-form-item>
                <base-select
                  class="w-100"
                  v-decorator="decorators.network"
                  resource="networks"
                  :params="networkParams"
                  :item.sync="form.fi.network"
                  remote
                  :label-format="item => `${item.name}(${item.guest_ip_start} - ${item.guest_ip_end}, vlan=${item.vlan_id})`"
                  :remote-fn="q => ({ filter: `name.contains(${q})` })"
                  :select-props="{ placeholder: '请选择IP子网' }" />
              </a-form-item>
            </a-col>
            <a-col :span="10">
              <a-form-item>
                <a-input
                  class="w-100"
                  v-decorator="decorators.ip"
                  placeholder="请输入子网内的IP地址，非必填" />
              </a-form-item>
            </a-col>
          </a-row>
        </a-form-item>
      </a-form>
    </div>
    <div slot="footer">
      <a-button type="primary" @click="handleConfirm" :loading="loading">{{ $t('dialog.ok') }}</a-button>
      <a-button @click="cancelDialog">{{ $t('dialog.cancel') }}</a-button>
    </div>
  </base-dialog>
</template>

<script>
import * as R from 'ramda'
import { mapGetters } from 'vuex'
import { isWithinRange } from '@/utils/validate'
import DialogMixin from '@/mixins/dialog'
import WindowsMixin from '@/mixins/windows'

export default {
  name: 'VmChangeIpDialog',
  mixins: [DialogMixin, WindowsMixin],
  data () {
    const validateIp = (rule, value, callback) => {
      let msg
      if (!R.isNil(value) && !R.isEmpty(value)) {
        if (R.isEmpty(this.form.fi.network)) {
          msg = '请选择子网'
          return callback(msg)
        }
        if (!isWithinRange(value, this.form.fi.network.guest_ip_start, this.form.fi.network.guest_ip_end)) {
          msg = '输入的IP不在子网内'
          return callback(msg)
        }
      }
      return callback()
    }
    return {
      loading: false,
      form: {
        fc: this.$form.createForm(this),
        fi: {
          network: {},
        },
      },
      decorators: {
        network: [
          'network',
          {
            rules: [
              { required: true, message: '请选择子网' },
            ],
          },
        ],
        ip: [
          'ip',
          {
            rules: [
              { validator: validateIp },
            ],
          },
        ],
      },
      formItemLayout: {
        wrapperCol: {
          span: 21,
        },
        labelCol: {
          span: 3,
        },
      },
    }
  },
  computed: {
    ...mapGetters(['scope']),
    networkParams () {
      return {
        zone: this.params.zone,
        scope: this.scope,
      }
    },
  },
  methods: {
    async handleConfirm () {
      this.loading = true
      let manager = new this.$Manager('servers')
      try {
        const values = await this.form.fc.validateFields()
        const data = {
          ip_addr: this.params.data[0]['ip_addr'],
          net_desc: values.ip ? `${values.network}:${values.ip}` : values.network,
        }
        await manager.performAction({
          id: this.params.resId,
          action: 'change-ipaddr',
          data,
        })
        this.params.list.refresh()
        this.cancelDialog()
      } finally {
        this.loading = false
        manager = null
      }
    },
  },
}
</script>
