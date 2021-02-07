# 电子书编辑

## 表单规则校验

创建表单规则校验逻辑：

```js
const validateRequire = (rule, value, callback) => {
    if (value === '') {
      this.$message({
        message: rule.field + '为必传项',
        type: 'error'
      })
      callback(new Error(rule.field + '为必传项'))
    } else {
      callback()
    }
  }
  const validateSourceUri = (rule, value, callback) => {
    if (value) {
      if (validURL(value)) {
        callback()
      } else {
        this.$message({
          message: '外链url填写不正确',
          type: 'error'
        })
        callback(new Error('外链url填写不正确'))
      }
    } else {
      callback()
    }
  }
  return {
    postForm: Object.assign({}, defaultForm),
    rules: {
      image_uri: [{ validator: validateRequire }],
      title: [{ validator: validateRequire }],
      content: [{ validator: validateRequire }],
      source_uri: [{ validator: validateSourceUri, trigger: 'blur' }]
    }
  }
}
```

## 提交表单

提交表单时需要提供两个接口，`createBook` 和 `updateBook`

```js
submitForm() {
    this.$refs.postForm.validate(valid => {
      if (valid) {
        this.loading = true
        const book = Object.assign({}, this.postForm)
        delete book.contents
        if (!this.isEdit) {
          createBook(book).then(response => {
            this.loading = false
            this.$notify({
              title: '成功',
              message: response.msg,
              type: 'success',
              duration: 2000
            })
            this.toDefault()
          }).catch(() => {
            this.loading = false
          })
        } else {
          updateBook(book).then(response => {
            console.log('updateBook', response)
            this.loading = false
            this.$notify({
              title: '成功',
              message: response.msg,
              type: 'success',
              duration: 2000
            })
          }).catch(() => {
            this.loading = false
          })
        }
      } else {
        return false
      }
    })
  }
```

# 电子书删除

## 删除逻辑

```js
async function removeBook(book) {
  if (book) {
    book.reset()
    if (book.fileName) {
      const removeBookSql = `delete from book where fileName='${book.fileName}'`
      const removeContentsSql = `delete from contents where fileName='${book.fileName}'`
      await db.querySql(removeBookSql)
      await db.querySql(removeContentsSql)
    }
  }
}
```

# 电子书列表

## 获取分类接口

查询分类视图时错误：

- sql_mode 为 only_full_group_by，具体错误原因参考：https://www.cnblogs.com/wxw7blog/p/10021563.html
- 视图创建者修改

## 电子书列表

```html
<template>
  <div class="app-container">
    <div class="filter-container">
      <el-input
        v-model="listQuery.title"
        clearable
        placeholder="书名"
        style="width: 200px;"
        class="filter-item"
        @keyup.enter.native="handleFilter"
        @clear="handleFilter"
        @blur="handleFilter"
      />
      <el-input
        v-model="listQuery.author"
        clearable
        placeholder="作者"
        style="width: 200px;"
        class="filter-item"
        @keyup.enter.native="handleFilter"
        @clear="handleFilter"
        @blur="handleFilter"
      />
      <el-select
        v-model="listQuery.category"
        placeholder="分类"
        clearable
        class="filter-item"
        @change="handleFilter"
      >
        <el-option v-for="item in categoryList" :key="item.value" :label="item.label" :value="item.label" />
      </el-select>
      <el-button
        v-waves
        class="filter-item"
        type="primary"
        icon="el-icon-search"
        style="margin-left: 10px"
        @click="forceRefresh"
      >
        查询
      </el-button>
      <el-button
        class="filter-item"
        type="primary"
        icon="el-icon-edit"
        style="margin-left: 5px"
        @click="handleCreate"
      >
        新增
      </el-button>
      <el-checkbox
        v-model="showCover"
        class="filter-item"
        style="margin-left:5px;"
        @change="changeShowCover"
      >
        显示封面
      </el-checkbox>
    </div>
    <el-table
      :key="tableKey"
      v-loading="listLoading"
      :data="list"
      border
      fit
      highlight-current-row
      style="width: 100%;"
      @sort-change="sortChange"
    >
      <el-table-column
        label="ID"
        prop="id"
        sortable="custom"
        align="center"
        width="80"
        :class-name="getSortClass('id')"
      />
      <el-table-column label="书名" width="150" align="center">
        <template slot-scope="{ row: { titleWrapper }}">
          <span v-html="titleWrapper" />
        </template>
      </el-table-column>
      <el-table-column label="作者" width="150" align="center">
        <template slot-scope="{ row: { authorWrapper }}">
          <span v-html="authorWrapper" />
        </template>
      </el-table-column>
      <el-table-column label="出版社" prop="publisher" width="150" align="center" />
      <el-table-column label="分类" prop="categoryText" width="100" align="center" />
      <el-table-column label="语言" prop="language" align="center" />
      <el-table-column v-if="showCover" label="封面图片" width="150" align="center">
        <template slot-scope="scope">
          <a :href="scope.row.cover" target="_blank">
            <img
              :src="scope.row.cover"
              style="width:120px;height:180px"
            >
          </a>
        </template>
      </el-table-column>
      <el-table-column label="文件名" prop="fileName" width="100" align="center" />
      <el-table-column label="文件路径" width="100" align="center">
        <template slot-scope="{ row: { filePath }}">
          <span>{{ filePath | valueFilter }}</span>
        </template>
      </el-table-column>
      <el-table-column label="封面路径" width="100" align="center">
        <template slot-scope="{ row: { coverPath }}">
          <span>{{ coverPath | valueFilter }}</span>
        </template>
      </el-table-column>
      <el-table-column label="解压路径" width="100" align="center">
        <template slot-scope="{ row: { unzipPath }}">
          <span>{{ unzipPath | valueFilter }}</span>
        </template>
      </el-table-column>
      <el-table-column label="上传人" width="100" align="center">
        <template slot-scope="scope">
          <span>{{ scope.row.createUser | valueFilter }}</span>
        </template>
      </el-table-column>
      <el-table-column label="上传时间" width="100" align="center">
        <template slot-scope="scope">
          <span>{{ scope.row.createDt | timeFilter }}</span>
        </template>
      </el-table-column>
      <el-table-column label="操作" align="center" width="120" fixed="right">
        <template slot-scope="{ row }">
          <PreviewDialog title="电子书信息" :data="row">
            <el-button type="text" icon="el-icon-view" />
          </PreviewDialog>
          <el-button type="text" icon="el-icon-edit" @click="handleUpdate(row)" />
          <el-button type="text" icon="el-icon-delete" style="color:#f56c6c" @click="handleDelete(row)" />
        </template>
      </el-table-column>
    </el-table>
    <pagination
      v-show="total > 0"
      :total="total"
      :page.sync="listQuery.page"
      :limit.sync="listQuery.pageSize"
      @pagination="refresh"
    />
  </div>
</template>
```



## 预览按钮

```html
<template>
  <span style="padding: 0 10px">
    <el-dialog :title="title" :visible.sync="visible" :modal="false">
      <el-form
        :model="formData"
        label-position="left"
        label-width="70px"
        style="width: 400px; margin-left:50px;"
      >
        <el-form-item label="书名">
          <el-input v-model="formData.title" disabled />
        </el-form-item>
        <el-form-item label="作者">
          <el-input v-model="formData.author" disabled />
        </el-form-item>
        <el-form-item label="出版社">
          <el-input v-model="formData.publisher" disabled />
        </el-form-item>
        <el-form-item label="语言">
          <el-input v-model="formData.language" disabled />
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button type="primary" @click="visible = false">
          关闭
        </el-button>
      </div>
    </el-dialog>
    <span @click="visible = true">
      <slot />
    </span>
  </span>
</template>

<script>
  export default {
    props: {
      title: {
        type: String,
        default: ''
      },
      data: {
        type: Object,
        default() {
          return {}
        }
      }
    },
    data() {
      return {
        visible: false,
        formData: {}
      }
    },
    created() {
      this.formData = Object.assign({}, this.data)
    }
  }
</script>

<style lang="scss" scoped>
</style>
```