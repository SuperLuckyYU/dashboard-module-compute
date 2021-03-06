<template>
  <div>
    <div class="tag-list">
      <template v-for="item of tags">
        <span
          class="tag mb-1 text-truncate d-inline-block"
          :title="item.title"
          :key="`${item.key}${item.value}`"
          :style="{ backgroundColor: item.backgroundColor, color: item.color, borderColor: item.color }">
          {{ item.title }}<a-icon style="font-size: 12px;" class="ml-1" type="close" @click="removeTag(item)" />
        </span>
      </template>
    </div>
    <a-form-item extra="每个资源最多可绑定20个标签" class="mt-2">
      <div class="d-flex">
        <div style="line-height: 40px;">
          <tag-select v-model="checked" :params="params" button-text="已有标签" />
          <a-button class="ml-2" v-if="!showForm" @click="() => showForm = true">新建</a-button>
        </div>
        <a-form
          class="ml-2"
          layout="inline"
          :form="tagForm.fc"
          v-if="showForm">
          <a-form-item>
            <a-input v-decorator="decorators.key" placeholder="标签键" />
          </a-form-item>
          <a-form-item>
            <a-input v-decorator="decorators.value" placeholder="标签值" />
          </a-form-item>
          <a-form-item>
            <a-button @click="addTag">添加</a-button>
            <a-button @click="() => showForm = false" class="ml-2">取消</a-button>
          </a-form-item>
        </a-form>
      </div>
    </a-form-item>
  </div>
</template>

<script>
import * as R from 'ramda'
import { getTagColor, getTagTitle } from '@/utils/common/tag'
import TagSelect from '@/sections/TagSelect'

export default {
  name: 'Tag',
  components: {
    TagSelect,
  },
  data () {
    return {
      checked: {},
      showForm: false,
      tagForm: {
        fc: this.$form.createForm(this),
      },
      decorators: {
        key: [
          'key',
          {
            rules: [
              { required: true, message: '请输入标签键' },
            ],
          },
        ],
        value: [
          'value',
        ],
      },
    }
  },
  computed: {
    params () {
      return {
        resources: 'server',
      }
    },
    tags () {
      let ret = []
      R.forEachObjIndexed((value, key) => {
        if (value.length > 0) {
          for (let i = 0, len = value.length; i < len; i++) {
            ret.push(this.genTag(key, value[i]))
          }
        } else {
          ret.push(this.genTag(key, null))
        }
      }, this.checked)
      return ret
    },
  },
  watch: {
    tags (val) {
      let ret
      if (val && val.length > 0) {
        ret = {}
        for (let i = 0, len = val.length; i < len; i++) {
          ret[val[i]['key']] = val[i]['value'] || ''
        }
      }
      this.$emit('change', ret)
    },
  },
  methods: {
    async addTag () {
      try {
        const values = await this.tagForm.fc.validateFields()
        const key = `user:${values.key}`
        const newValue = { ...this.checked }
        if (newValue[key]) {
          if (values.value) {
            if (!newValue[key].includes(values.value)) {
              newValue[key] = [values.value]
            }
          }
        } else {
          if (values.value) {
            newValue[key] = [values.value]
          } else {
            newValue[key] = []
          }
        }
        this.checked = newValue
        this.tagForm.fc.resetFields()
      } catch (error) {
        throw error
      }
    },
    genTag (key, value) {
      const rgb = getTagColor(key, value, 'rgb')
      const strRgb = rgb.join(',')
      return {
        title: getTagTitle(key, value),
        color: `rgb(${strRgb})`,
        backgroundColor: `rgba(${strRgb},.1)`,
        key,
        value,
      }
    },
    removeTag (item) {
      const newValue = { ...this.checked }
      delete newValue[item.key]
      this.checked = newValue
    },
  },
}
</script>

<style lang="scss" scoped>
.tag-list {
  min-height: 65px;
  border: 2px dashed #ddd;
  padding: 8px;
}
.tag {
  max-width: 100%;
  line-height: 20px;
  margin-right: 10px;
  padding: 0 6px 0 4px;
  font-size: 12px;
  border-style: solid;
  border-width: 1px;
}
</style>
