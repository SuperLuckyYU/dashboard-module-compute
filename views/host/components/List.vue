<template>
  <page-list
    :list="list"
    :columns="columns"
    :group-actions="groupActions"
    :single-actions="singleActions" />
</template>

<script>
import qs from 'qs'
import PasswordFetcher from '@Compute/sections/PasswordFetcher'
import { sizestr, percentstr } from '@/utils/utils'
import { getRegionTableColumn, getStatusTableColumn, getBrandTableColumn, getEnabledTableColumn, getNameDescriptionTableColumn } from '@/utils/common/tableColumn'
import WindowsMixin from '@/mixins/windows'

export default {
  name: 'HostList',
  mixins: [WindowsMixin],
  props: {
    getParams: {
      type: [Function, Object],
    },
  },
  data () {
    return {
      list: this.$list.createList(this, {
        resource: 'hosts',
        getParams: this.getParams,
        filterOptions: {
          name: {
            label: '名称',
            filter: true,
            formatter: val => {
              return `name.contains(${val})`
            },
          },
          sn: {
            label: 'SN',
          },
        },
      }),
      columns: [
        getNameDescriptionTableColumn({
          vm: this,
          hideField: true,
          slotCallback: row => {
            return (
              <side-page-trigger onTrigger={ () => this.sidePageTriggerHandle(row.id, 'HostSidePage') }>{ row.name }</side-page-trigger>
            )
          },
        }),
        getEnabledTableColumn(),
        getStatusTableColumn({ statusModule: 'host' }),
        {
          field: 'access_ip',
          title: 'IP',
          width: 120,
          slots: {
            default: ({ row }) => {
              return [
                <list-body-cell-wrap row={row} field="access_ip" copy />,
              ]
            },
          },
        },
        getStatusTableColumn({ field: 'host_status', title: '服务', statusModule: 'host_status' }),
        {
          field: 'nonsystem_guests',
          title: '#VM',
          width: 60,
          formatter: ({ cellValue }) => {
            return cellValue || '0'
          },
        },
        {
          field: 'cpu_count',
          title: 'CPU',
          width: 60,
          formatter: ({ cellValue, row }) => {
            if (cellValue) {
              return '' + cellValue + '/' + percentstr(row.cpu_commit_rate)
            } else {
              return 'N/A'
            }
          },
        },
        {
          field: 'mem_size',
          title: '内存',
          width: 60,
          formatter: ({ cellValue, row }) => {
            if (cellValue) {
              return sizestr(cellValue, 'M', 1024) + '/' + percentstr(row.mem_commit_rate)
            } else {
              return 'N/A'
            }
          },
        },
        {
          field: 'storage_size',
          title: '存储',
          width: 60,
          formatter: ({ cellValue, row }) => {
            if (cellValue) {
              return sizestr(cellValue, 'M', 1024) + '/' + percentstr(row.storage_commit_rate)
            } else {
              return 'N/A'
            }
          },
        },
        {
          field: 'sn',
          title: 'SN',
          width: 120,
          showOverflow: 'ellipsis',
        },
        getBrandTableColumn(),
        getRegionTableColumn(),
        {
          field: 'id',
          title: 'IPMI',
          width: 60,
          slots: {
            default: ({ cellValue, row }) => {
              if (!row.is_baremetal) {
                return '-'
              } else {
                return [<PasswordFetcher serverId={ row.id } resourceType='baremetals' />]
              }
            },
          },
        },
        {
          field: 'server_id',
          title: '初始账号',
          width: 70,
          slots: {
            default: ({ cellValue, row }) => {
              if (!row.is_baremetal) {
                return '-'
              } else {
                return [<PasswordFetcher serverId={ row.id } resourceType='baremetal_ssh' />]
              }
            },
          },
        },
      ],
      groupActions: [
        {
          label: '启用',
          action: () => {
            this.list.batchPerformAction('enable', null, this.list.steadyStatus)
          },
          meta: (obj) => {
            const validate = this.list.selectedItems.length && this.list.selectedItems.some(item => !item.enabled)
            return {
              validate,
              tooltip: !validate ? '请选择已经禁用的实例' : '',
            }
          },
        },
        {
          label: '禁用',
          action: () => {
            this.list.batchPerformAction('disable', null, this.list.steadyStatus)
          },
          meta: (obj) => {
            const validate = this.list.selectedItems.length && this.list.selectedItems.some(item => item.enabled)
            return {
              validate,
              tooltip: !validate ? '请选择已经启用的实例' : '',
            }
          },
        },
        {
          label: '更多',
          actions: () => {
            return [
              {
                label: '调整标签',
                action: (obj) => {
                  this.createDialog('HostsAdjustLabelDialog', {
                    data: this.list.selectedItems,
                    columns: this.columns,
                    list: this.list,
                  })
                },
                meta: () => ({
                  validate: this.list.selectedItems.length,
                }),
              },
              {
                label: '回收为物理机',
                action: () => {
                  this.list.batchPerformAction('disable', null)
                },
                meta: () => {
                  if (!this.list.selectedItems.length) {
                    return {
                      validate: false,
                      tooltip: '请选择要操作的数据',
                    }
                  }
                  for (let i = 0; i < this.list.selectedItems.length; i++) {
                    let obj = this.list.selectedItems[0]
                    if (obj.host_type !== 'hypervisor') {
                      return {
                        validate: false,
                        tooltip: '必须为KVM类型的宿主机才可回收',
                      }
                    } else if (obj.nonsystem_guests > 0) {
                      return {
                        validate: false,
                        tooltip: '虚拟化机器大于0不可回收',
                      }
                    } else if (obj.enabled) {
                      return {
                        validate: false,
                        tooltip: '已启用的宿主机不可回收',
                      }
                    }
                  }
                },
              },
              {
                label: '删除',
                permission: 'hosts_delete',
                action: () => {
                  this.createDialog('DeleteResDialog', {
                    data: this.list.selectedItems,
                    columns: this.columns,
                    title: '删除',
                    list: this.list,
                  })
                },
                meta: () => this.$getDeleteResult(this.list.selectedItems),
              },
            ]
          },
        },
      ],
      singleActions: [
        {
          label: '远程终端',
          actions: obj => {
            const ret = []
            const webconsoleM = new this.$Manager('webconsole', 'v1')
            if (obj.host_type === 'baremetal') {
              ret.push({
                label: 'SOL远程终端',
                action: () => {
                  webconsoleM.objectRpc({ methodname: 'DoBaremetalConnect', objId: obj.id }).then((res) => {
                    const connectParams = qs.parse(res.data.connect_params)
                    const query = {
                      ...connectParams,
                      session: res.data.session,
                      hypervisor: obj.host_type,
                    }
                    const href = `${this.$appConfig.webConsolePath}?${qs.stringify(query)}`
                    open(href)
                  })
                    .catch((err) => {
                      console.error(err)
                    })
                },
                meta: () => ({
                  validate: obj.status === 'running',
                }),
              })
            }
            let ips = (obj.server_ips || '').split(',').filter(item => !!item)
            if (obj.access_ip) {
              ips = [obj.access_ip, ...ips]
            }
            const actionGenerator = ip => {
              return (sshData) => {
                webconsoleM.performAction({
                  action: ip,
                  data: sshData,
                  id: 'ssh',
                }).then(({ data }) => {
                  const connectParams = qs.parse(data.connect_params)
                  const query = {
                    ...connectParams,
                    session: data.session,
                    hypervisor: obj.host_type,
                  }
                  const href = `${this.$appConfig.webConsolePath}?${qs.stringify(query)}`
                  open(href)
                })
              }
            }
            ips.forEach(ip => {
              const meta = () => ({ validate: obj.status === 'running' })
              ret.push({
                label: `SSH ${ip}`,
                action: actionGenerator(ip),
                meta,
              })
              ret.push({
                label: `SSH ${ip} 自定义端口`,
                action: () => {},
                meta,
              })
            })
            return ret
          },
        },
        {
          label: '更多',
          actions: (obj) => {
            return [
              {
                label: '禁用',
                submenus: [
                  {
                    label: '启用',
                    action: () => {
                      this.list.onManager('performAction', {
                        steadyStatus: 'running',
                        id: obj.id,
                        managerArgs: {
                          action: 'enable',
                        },
                      })
                    },
                    meta: () => ({
                      validate: !obj.enabled,
                      tooltip: obj.enabled ? '请选择已禁用的实例' : '',
                    }),
                  },
                  {
                    label: '禁用',
                    action: () => {
                      this.list.onManager('performAction', {
                        steadyStatus: 'ready',
                        id: obj.id,
                        managerArgs: {
                          action: 'disable',
                        },
                      })
                    },
                    meta: () => ({
                      validate: obj.enabled,
                      tooltip: !obj.enabled ? '请选择已启用的实例' : '',
                    }),
                  },
                ],
              },
              {
                label: '设置',
                submenus: [
                  {
                    label: '调整标签',
                    action: () => {
                      this.createDialog('HostsAdjustLabelDialog', {
                        data: [obj],
                        columns: this.columns,
                        list: this.list,
                      })
                    },
                  },
                  {
                    label: '调整超售比',
                    action: () => {
                      this.createDialog('HostAdjustOversoldRatioDialog', {
                        data: [obj],
                        columns: this.columns,
                        list: this.list,
                      })
                    },
                  },
                  {
                    label: '回收为物理机',
                    action: () => {
                      this.createDialog('HostUnconvertDialog', {
                        data: [obj],
                        columns: this.columns,
                        list: this.list,
                      })
                    },
                    meta: () => {
                      if (obj.host_type !== 'hypervisor') {
                        return {
                          validate: false,
                          tooltip: '必须为KVM类型的宿主机才可回收',
                        }
                      } else if (obj.nonsystem_guests > 0) {
                        return {
                          validate: false,
                          tooltip: '虚拟化机器大于0不可回收',
                        }
                      } else if (obj.enabled) {
                        return {
                          validate: false,
                          tooltip: '已启用的宿主机不可回收',
                        }
                      }
                      return {
                        validate: true,
                      }
                    },
                  },
                  {
                    label: '进入维护模式',
                    action: () => {
                      this.createDialog('HostMaintenanceInDialog', {
                        data: [obj],
                        columns: this.columns,
                        list: this.list,
                      })
                    },
                    meta: () => {
                      if (obj.host_type !== 'hypervisor') {
                        return {
                          validate: false,
                        }
                      }
                      return {
                        validate: ['running', 'maintain_fail'].includes(obj.status),
                        tooltip: obj.status !== 'running' ? '状态为运行中的宿主机支持该操作' : '',
                      }
                    },
                  },
                  {
                    label: '退出维护模式',
                    action: () => {
                      this.createDialog('HostMaintenanceOutDialog', {
                        data: [obj],
                        columns: this.columns,
                        list: this.list,
                      })
                    },
                    meta: () => {
                      if (obj.host_type !== 'hypervisor') {
                        return {
                          validate: false,
                        }
                      }
                      return {
                        validate: ['maintaining', 'maintain_fail'].includes(obj.status),
                      }
                    },
                  },
                ],
              },
              {
                label: '删除',
                submenus: [
                  {
                    label: '删除',
                    action: () => {
                      this.createDialog('DeleteResDialog', {
                        data: [obj],
                        columns: this.columns,
                        list: this.list,
                        title: '删除',
                      })
                    },
                    meta: () => this.$getDeleteResult(obj),
                  },
                ],
              },
            ]
          },
        },
      ],
    }
  },
  created () {
    this.initSidePageTab('host-detail')
    this.list.fetchData()
  },
}
</script>
