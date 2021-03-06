<template>
  <el-dialog :fullscreen="fullscreen" :width="width" :title="showTitle" :visible.sync="visible">
    <div class="dialog-content">
      <el-form ref="formEditor" :model="model" :label-position="labelPosition" @submit.native.prevent>
        <el-row :gutter="20">
          <el-col :md="span" v-for="(item, key) in datas" :key="key">
            <el-form-item :label="item.label + ':'" :prop="item.name" :required="item.required" :label-width="labelWidth">
               <!-- link组件 -->
              <el-link type="primary" v-if="item.type == 'link'" @click="sampleDownload(item)">{{ item.name||$t('importer.sample')}}</el-link>
              <!-- 开关组件 -->
              <el-switch v-model="model[item.name]" :readonly="item.readonly" v-else-if="item.type == 'switch'" />
              <!-- 下拉组件 -->
              <el-select v-model="model[item.name]" :placeholder="item.placeholder" :disabled="item.readonly" v-else-if="item.type == 'select'" :filterable="item.filterable" :multiple="item.multiple" >
               <template v-if="item.group">
                 <el-option-group  v-for="group in item.options" :key="group.label" :label="group.label">
                    <el-option v-for="(j, i) in group.options" :key="i" :label="j.label" :value="j.value" :disabled="j.disabled" />
                 </el-option-group>
               </template>
                <el-option v-for="(j, i) in item.options" :key="i" :label="j.label" :value="j.value" :disabled="j.disabled" v-else/>
              </el-select>
              <!-- 级联选择 -->
              <el-cascader v-model="model[item.name]" :placeholder="item.placeholder" :disabled="item.readonly" v-else-if="item.type == 'cascader'" collapse-tags filterable :clearable="item.clearable" :options="item.options" :props="item.props" />
              <!-- 单选组件 -->
              <el-radio-group v-model="model[item.name]" :disabled="item.readonly" v-else-if="item.type == 'radio'">
                <el-radio v-for="(j, i) in item.options" :key="i" :label="j.value">{{ j.label }}</el-radio>
              </el-radio-group>
              <!-- 多选组件 -->
              <el-checkbox-group v-model="model[item.name]" :disabled="item.readonly" v-else-if="item.type == 'checkbox'">
                <el-checkbox v-for="(j, i) in item.options" :key="i" :label="j.value">{{ j.label }}</el-checkbox>
              </el-checkbox-group>
              <!-- 上传组件 -->
              <el-upload ref="upload" :class="{'uploader':!item.fileList}" :show-file-list="item.fileList" :limit="1" :before-upload="beforeUpload" :on-success="upload"
                :action="item.action" :headers="{ Authorization: 'Bearer ' + item.token }" :http-request="uploadFile" v-else-if="item.type == 'upload'">
                <el-image lazy v-if="model[item.url]" :src="model[item.url]" fit="cover" />
                <i class="el-icon-plus" v-else-if="!model[item.url]&&!item.fileList"/>
                <el-button plain v-else size="mini" icon="el-icon-upload2">{{ $t('import') }}</el-button>
              </el-upload>
              <!-- 日期组件 -->
              <el-date-picker v-model="model[item.name]" :readonly="item.readonly" type="date" v-else-if="item.type == 'date'" />
              <!-- 多行文本 -->
              <el-input type="textarea" v-model="model[item.name]" :placeholder="item.placeholder" :readonly="item.readonly" v-else-if="item.type == 'textarea'" />
              <!-- 文本组件 -->
              <el-input v-model="model[item.name]" :placeholder="item.placeholder" :readonly="item.readonly" @keyup.enter.native="submit('form')" v-else/>
            </el-form-item>
          </el-col>
        </el-row>
      </el-form>
    </div>
    <div slot="footer">
      <el-button @click="cancel">{{ $t('cancel') }}</el-button>
      <el-button @click="submit" type="primary">{{ $t('save') }}</el-button>
    </div>
  </el-dialog>
</template>

<script>
export default {
  props: {
    show: { type: Boolean, default: false },
    title: String,
    width: { type: String, default: '800px' },
    items: Array,
    model: Object,
    columns: { type: Number, default: 1 },
    labelPosition: String,
    labelWidth: String,
    fullscreen: Boolean,
    importFile:{ type: Boolean, default: false },
  },
  data() {
    return {
      showTitle: this.title || this.$t('edit'),
      visible: false,
    }
  },
  computed: {
    datas() {
      return this.items.map(j => {
        if (j.type == 'switch') {
          this.model[j.name] = this.model[j.name] == 0 ? false : true
        } else if (j.type == 'upload') { // 上传组件, 网址替换模板变量
          let matched = j.url.match(/\{([^}]+)\}/)
          j.action = matched ? j.url.replace(matched[0], this.model[matched[1]]) : j.url
        }
        return j
      })
    },
    span() {
      return 24 / this.columns
    }
  },
  watch: {
    show() {
      this.visible = this.show
    },
    visible(val) {
      if (!val) this.close()
    }
  },
  methods: {
    // 关闭窗口
    close() {
      this.$emit("close")
    },
    // 重置表单
    cancel() {
      this.$refs['formEditor'].resetFields()      
      this.$emit("cancel")
      this.close()
    },
    // 提交表单
    submit(formData) {
      this.$refs['formEditor'].validate((valid) => {
        if (!valid) return false
        this.$emit("submit",formData)
      })
    },
    // 上传图片
    upload(ret) {
      this.$emit("upload", ret)
    },
    // 校验上传
    beforeUpload(file) {
      if (this.importFile) return true
      const isType = file.type === 'image/jpeg' || file.type === 'image/png'
      const isLt2M = file.size / 1024 / 1024 < 2
      if (!isType) this.$message.error('上传头像图片只能是 JPG/PNG 格式!')
      if (!isLt2M) this.$message.error('上传头像图片大小不能超过 2MB!')
      return isType && isLt2M;
    },
    // 自定义上传行为
    uploadFile( params ) {
      this.model.file = params.file
    },
    // 下载模板
    sampleDownload( value ) {
      let a = document.createElement( 'a' )
      a.href = value.url
      a.click()
    },
  }
}
</script>

<style scoped>
.is-fullscreen .dialog-content { margin: 0 -10px; padding: 0 10px; height: calc(100vh - 185px); overflow: scroll; }
.el-select, .el-cascader, .el-date-editor { width: 100%; }
.uploader { height: 120px; }
.uploader >>> .el-upload { position: relative; overflow: hidden; cursor: pointer; }
.uploader >>> .el-upload:hover { border-color: #409EFF; }
.uploader .el-icon-plus { width: 118px; height: 118px; line-height: 118px; text-align: center; font-size: 16px; color: #999; border: 1px dashed #eee; }
.uploader .el-image { width: 120px; height: 120px; }
</style>
