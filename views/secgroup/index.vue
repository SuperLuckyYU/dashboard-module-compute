<template>
  <div>
    <page-header title="安全组" />
    <page-body>
      <secgroup-list />
    </page-body>
  </div>
</template>
<script>
import SecgroupList from './components/List'

export default {
  name: 'SecgroupIndex',
  components: {
    SecgroupList,
  },
}
</script>
