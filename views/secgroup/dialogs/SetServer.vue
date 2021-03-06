<template>
  <base-dialog @cancel="cancelDialog">
    <div slot="header">关联云服务器</div>
    <div slot="body">
      <dialog-selected-tips :count="params.data.length" action="关联云服务器" />
      <vxe-grid class="mb-2" :data="params.data" :columns="params.columns.slice(0, 3)" />
      <a-form :form="form.fc" hideRequiredMark>
        <a-form-item label="主机" v-bind="formItemLayout">
          <base-select
            class="w-100"
            @update:options="onServerSucceed"
            filterable
            remote
            v-decorator="decorators.sergroups"
            searchKey="search"
            resource="servers"
            :mapper="mapperServers"
            :params="requestParams"
            :select-props="{ allowClear: true, placeholder: '请选择主机', mode: 'tags', defaultValue }" />
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
import DialogMixin from '@/mixins/dialog'
import WindowsMixin from '@/mixins/windows'

export default {
  name: 'SetServerDialog',
  mixins: [DialogMixin, WindowsMixin],
  data () {
    // const validateServers = (rule, value, callback) => {
    //   let minError = '最少关联一个'
    //   if (!value || value.length < 1) {
    //     return callback(minError)
    //   }
    //   return callback()
    // }
    return {
      loading: false,
      scope: this.$store.getters.scope,
      form: {
        fc: this.$form.createForm(this),
      },
      decorators: {
        sergroups: [
          'sergroups',
          {
            validateFirst: true,
            rules: [
              // { validator: validateServers, trigger: 'change' },
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
      bindServers: [],
      bindServersLoaded: false,
      defaultValue: [],
    }
  },
  computed: {
    ...mapGetters([
      'isAdminMode',
      'userInfo',
    ]),
    requestParams () {
      if (this.isAdminMode) {
        // if (this.requestParams.tenant) {
        //   delete this.requestParams.tenant
        // }
        return {
          search: '',
          limit: 0,
          offset: 0,
          filter: 'hypervisor.notin(container, baremetal, esxi)',
          project_domain: this.userInfo.projectDomain,
        }
      } else {
        // if (this.requestParams.scope) {
        //   return {
        //     search: '',
        //     limit: 20,
        //     offset: 0,
        //     filter: 'hypervisor.notin(container, baremetal, esxi)',
        //   }
        // }
        return {
          search: '',
          limit: 0,
          offset: 0,
          filter: 'hypervisor.notin(container, baremetal, esxi)',
          tenant: this.params.data[0]['tenant_id'],
        }
      }
    },
  },
  created () {
    // this.fetchBindedServers()
  },
  methods: {
    mapperServers (data) {
      data = data.concat(this.bindServers)
      data = R.uniqBy(item => item.id, data)
      return data
    },
    onServerSucceed () {
      this.fetchBindedServers()
    },
    async fetchBindedServers () {
      const manager = new this.$Manager('servers')
      try {
        const params = {
          scope: this.scope,
          filter: 'hypervisor.notin(container, baremetal, esxi)',
          limit: 0,
          secgroup: this.params.data[0]['id'],
        }
        if (this.isAdminMode) {
          params['project_domain'] = this.userInfo.projectDomain
          delete params.scope
        }
        const { data: { data = [] } } = await manager.list({ params })
        this.bindServers = data
        this.bindServersLoaded = true
        this.$nextTick(() => {
          this.form.fc.setFieldsValue({ 'sergroups': data.map(item => item.id) })
        })
      } catch (error) {
        console.log(error)
      }
    },
    async handleConfirm () {
      this.loading = true
      try {
        const values = await this.form.fc.validateFields()
        let ids = []
        const bindServers = []
        let removeServers = []
        const data = {
          'secgrp.0': this.params.data[0].id,
        }
        ids = JSON.parse(JSON.stringify(values.sergroups))
        this.bindServers.forEach((item, idx) => {
          bindServers.push(item.id)
        })
        for (var i = 0; i < bindServers.length; i++) {
          if (!R.includes(bindServers[i], ids)) {
            removeServers.push(bindServers[i])
          }
          for (var j = 0; j < ids.length; j++) {
            if (bindServers[i] === ids[j]) {
              ids.splice(j, 1)
            }
          }
        }
        if (removeServers.length > 0) {
          await new this.$Manager('servers').batchPerformAction({
            ids: removeServers,
            action: 'revoke-secgroup',
            data,
          })
        }
        if (ids.length > 0) {
          await new this.$Manager('servers').batchPerformAction({
            ids,
            action: 'add-secgroup',
            data,
          })
        }
        this.loading = false
        this.cancelDialog()
        this.params.list.refresh()
      } finally {
        this.loading = false
      }
    },
  },
}
</script>
