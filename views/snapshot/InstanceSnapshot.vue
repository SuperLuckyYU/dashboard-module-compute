<template>
  <div>
    <page-header title="主机快照" />
    <page-body>
      <snapshot-list :list="list" type="instance" />
    </page-body>
  </div>
</template>

<script>
import SnapshotList from './components/List'

export default {
  name: 'DiskIndex',
  components: {
    SnapshotList,
  },
  data () {
    return {
      list: this.$list.createList(this, {
        id: 'VmInstanceSnapshotsList',
        resource: 'instance_snapshots',
        getParams: {
          details: true,
        },
        filterOptions: {
          name: {
            label: '名称',
            filter: true,
            formatter: val => {
              return `name.contains(${val})`
            },
          },
        },
      }),
    }
  },
}
</script>
