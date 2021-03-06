<template>
  <page-list
    :list="list"
    :columns="columns"
    :group-actions="groupActions"
    :single-actions="singleActions" />
</template>

<script>
import { DISK_TYPES, STORAGE_TYPES } from '../constants'
import { RollbackDiskValidate } from '../validate'
import BaseDropList from '@/sections/DropList'
import {
  getNameDescriptionTableColumn,
  getBrandTableColumn,
  getStatusTableColumn,
  getProjectTableColumn,
  getTimeTableColumn,
} from '@/utils/common/tableColumn'
import WindowsMixin from '@/mixins/windows'
import { sizestr } from '@/utils/utils'

export default {
  name: 'SnapshotList',
  mixins: [WindowsMixin],
  props: {
    list: {
      type: Object,
      default: () => ({}),
    },
    type: {
      type: String,
      default: 'disk',
      validator: val => ['disk', 'instance'].includes(val),
    },
  },
  data () {
    return {
      columns: [
        getNameDescriptionTableColumn({
          vm: this,
          hideField: true,
          slotCallback: row => {
            return (
              <side-page-trigger onTrigger={ () => this.sidePageTriggerHandle(row.id, 'SnapshotSidePage', { type: this.type }) }>{ row.name }</side-page-trigger>
            )
          },
        }),
        getBrandTableColumn({ hidden: this.type === 'instance' }),
        {
          field: 'disk_name',
          title: '硬盘',
          formatter: ({ row }) => {
            return row.disk_name
          },
          hidden: () => {
            return this.type === 'instance'
          },
        },
        {
          field: 'disk_type',
          title: '磁盘类型',
          width: 70,
          formatter: ({ row }) => {
            return DISK_TYPES[row.disk_type] || row.disk_type
          },
          hidden: () => {
            return this.type === 'instance'
          },
        },
        {
          field: 'rules',
          title: '子快照',
          slots: {
            default: ({ row }) => {
              const list = row.snapshots.map(val => ({ value: val.name }))
              const dropMsg = [
                {
                  title: `${list.length}个`,
                  list,
                },
              ]
              return [<BaseDropList drop-msg={dropMsg} />]
            },
          },
          hidden: () => {
            return this.type === 'disk'
          },
        },
        {
          field: 'size',
          title: '快照大小',
          width: 70,
          formatter: ({ row }) => {
            if (this.type === 'disk') {
              return sizestr(row.size, 'M', 1024)
            } else if (this.type === 'instance') {
              const size = row.snapshots.reduce((a, b) => a + b.size, 0)
              return sizestr(size, 'M', 1024)
            }
          },
        },
        getStatusTableColumn({ statusModule: 'snapshot' }),
        getProjectTableColumn(),
        {
          field: 'guest',
          title: '虚拟机',
          minWidth: 70,
          slots: {
            default: ({ row }, h) => {
              return [
                <div class='text-truncate'>
                  {row.guest}
                  {row.guest_status ? <status status={ row['guest_status'] } statusModule='server'/> : ''}
                </div>,
              ]
            },
          },
        },
        getTimeTableColumn(),
        {
          field: 'storage_type',
          title: '存储类型',
          width: 80,
          formatter: ({ row }) => {
            return STORAGE_TYPES[row.storage_type] || row.storage_type || '-'
          },
          hidden: () => {
            return this.type === 'instance'
          },
        },
      ],
      groupActions: [
        {
          label: '删除',
          permission: 'snapshots_delete',
          action: () => {
            this.createDialog('DeleteResDialog', {
              data: this.list.selectedItems,
              columns: this.columns,
              list: this.list,
              title: '删除',
            })
          },
          meta: () => {
            const ret = {
              validate: this.list.selected.length,
              tooltip: null,
            }
            if (this.list.selectedItems.some(item => !item.can_delete)) {
              ret.validate = false
              return ret
            }
            if (this.list.selectedItems.some(item => item.is_sub_snapshot)) {
              ret.validate = false
              ret.tooltip = '此快照为主机快照子快照，不可操作'
              return ret
            }
            return ret
          },
        },
      ],
      singleActions: [
        {
          label: '回滚硬盘',
          permission: 'disks_perform_disk_reset',
          action: obj => {
            this.createDialog('RollbackDiskDialog', {
              data: [obj],
              columns: this.columns,
              list: this.list,
            })
          },
          meta: obj => {
            const brand = obj.brand && obj.brand.toLowerCase()
            if (this.isInstanceSnapshot) {
              return { validate: false, tooltip: '不支持该操作' }
            }
            if (brand) {
              return { ...RollbackDiskValidate[brand](obj) }
            }
            return { validate: true }
          },
        },
        {
          label: '删除',
          action: obj => {
            this.createDialog('DeleteResDialog', {
              data: [obj],
              columns: this.columns,
              title: '删除',
              list: this.list,
            })
          },
          meta: obj => this.$getDeleteResult(obj),
        },
      ],
    }
  },
  computed: {
    isDiskSnapshot () {
      return this.type === 'disk'
    },
    isInstanceSnapshot () {
      return this.type === 'instance'
    },
  },
  created () {
    this.initSidePageTab('snapshot-detail')
    this.list.fetchData()
  },
}
</script>
