# 电子书上传

## 创建上传页面组件

电子书上传过程分为新增电子书和编辑电子书，新增：

```html
<template>
  <detail :is-edit="false" />
</template>

<script>
import Detail from './components/Detail'

export default {
  name: 'CreateBook',
  components: { Detail }
}
</script>
```

编辑：

```html
<template>
  <article-detail :is-edit />
</template>

<script>
import Detail from './components/Detail'

export default {
  name: 'EditBook',
  components: { Detail }
}
</script>
```

Detail 组件比较复杂，我们逐步实现，首先实现 Detail 的大体布局，包括一个 el-form 和 sticky 导航栏，sticky 在内容较多时会产生吸顶效果：

```html
<div class="detail">
    <el-form ref="postForm" :model="postForm" :rules="rules" class="form-container">
      <sticky :z-index="10" :class-name="'sub-navbar ' + postForm.status">
        <el-button v-if="!isEdit" @click.prevent.stop="showGuide">显示帮助</el-button>
        <el-button v-loading="loading" style="margin-left: 10px;" type="success" @click="submitForm">
          {{ isEdit ? '编辑电子书' : '新增电子书' }}
        </el-button>
      </sticky>
      <div class="detail-container">
        <el-row>
          <Warning />
          <el-col :span="24">
            <!-- 编写具体表单控件 -->          
          </el-col>
          <el-col :span="24">
            <!-- 编写具体表单控件 -->          
          </el-col>
        </el-row>
      </div>
    </el-form>
</div>

<style lang="scss" scoped>
  @import "~@/styles/mixin.scss";

  .detail {
    position: relative;
    .detail-container {
      padding: 40px 45px 20px 50px;
      .preview-img {
        width: 200px;
        height: 270px;
      }
      .contents-wrapper {
        padding: 5px 0;
      }
    }
  }
</style>
```




## 上传组件开发

这里我基于 el-upload 封装了上传组件 [EbookUpload](http://www.youbaobao.xyz/admin-docs/guide/extra/upload.html)，

```
<template>
  <div class="singleImageUpload2 upload-container">
    <el-upload
      :action="action"
      :headers="headers"
      :multiple="false"
      :limit="1"
      :before-upload="beforeUpload"
      :on-success="onSuccess"
      :on-error="onError"
      :on-remove="onRemove"
      :file-list="fileList"
      :on-exceed="onExceed"
      :disabled="disabled"
      drag
      show-file-list
      class="image-uploader"
      accept="application/epub+zip"
    >
      <i class="el-icon-upload" />
      <div v-if="fileList.length === 0" class="el-upload__text">
        请将电子书拖入或
        <em>点击上传</em>
      </div>
      <div v-else class="el-upload__text">
        图书已上传
      </div>
    </el-upload>
  </div>
</template>

<script>
  import { getToken } from '@/utils/auth'

  export default {
    name: 'EbookUpload',
    props: {
      fileList: {
        type: Array,
        default() {
          return []
        }
      },
      disabled: {
        type: Boolean,
        default: false
      }
    },
    data() {
      return {
        action: `${process.env.VUE_APP_BASE_API}book/upload`
      }
    },
    computed: {
      headers() {
        return {
          Authorization: `Bearer ${getToken()}`
        }
      }
    },
    methods: {
      onRemove() {
        this.$message({
          message: '电子书删除成功',
          type: 'success'
        })
        this.$emit('onRemove')
      },
      onExceed() {
        this.$message({
          message: '每次只能上传一本电子书',
          type: 'warning'
        })
      },
      beforeUpload(file) {
        this.$emit('beforeUpload', file)
      },
      onSuccess(response, file) {
        console.log('onSuccess', response, file)
        const { code, msg, data } = response
        if (code !== 0) {
          this.$message({
            message: (msg && `上传失败，失败原因：${msg}`) || '上传失败',
            type: 'error'
          })
          this.$emit('onError', data)
        } else {
          this.$message({
            message: '上传成功',
            type: 'success'
          })
          this.$emit('onSuccess', data)
        }
      },
      onError(err) {
        const errMsg = (err.message && JSON.parse(err.message)) || '上传失败'
        this.$message({
          message: (errMsg.msg && `上传失败，失败原因：${errMsg.msg}`) || '上传失败',
          type: 'error'
        })
        this.$emit('onError', err)
      }
    }
  }
</script>

<style lang="scss" scoped>
  .upload-container {
    width: 100%;
    height: 100%;
    position: relative;

    .image-uploader {
      height: 100%;
    }
  }
</style>
```

基于 EbookUpload 我们再实现上传组件就非常容易了：

```html
<el-form-item prop="image_uri" style="margin-bottom: 0">
  <Upload
    v-model="postForm.image_uri"
    :file-list="fileList"
    :disabled="isEdit"
    @onSuccess="onUploadSuccess"
    @onRemove="onUploadRemove"
  />
</el-form-item>
```



## 上传图书表单

图书表单中包括以下信息：

- 书名
- 作者
- 出版社
- 语言
- 根文件
- 文件路径
- 解压路径
- 封面路径
- 文件名称
- 封面
- 目录

```html
<el-col :span="24">
    <el-form-item style="margin-bottom: 40px;" prop="title">
      <MDinput v-model="postForm.title" :maxlength="100" name="name" required>
        书名
      </MDinput>
    </el-form-item>
    <div>
      <el-row>
        <el-col :span="12" class="form-item-author">
          <el-form-item :label-width="labelWidth" label="作者：">
            <el-input
              v-model="postForm.author"
              placeholder="作者"
              style="width: 100%"
            />
          </el-form-item>
        </el-col>
        <el-col :span="12">
          <el-form-item :label-width="labelWidth" label="出版社：">
            <el-input
              v-model="postForm.publisher"
              placeholder="出版社"
              style="width: 100%"
            />
          </el-form-item>
        </el-col>
      </el-row>
      <el-row>
        <el-col :span="12">
          <el-form-item :label-width="labelWidth" label="语言：">
            <el-input
              v-model="postForm.language"
              placeholder="语言"
              style="width: 100%"
            />
          </el-form-item>
        </el-col>
        <el-col :span="12">
          <el-form-item :label-width="labelWidth" label="根文件：">
            <el-input
              v-model="postForm.rootFile"
              placeholder="根文件"
              style="width: 100%"
              disabled
            />
          </el-form-item>
        </el-col>
      </el-row>
      <el-row>
        <el-col :span="12">
          <el-form-item :label-width="labelWidth" label="文件路径：">
            <el-input
              v-model="postForm.filePath"
              placeholder="文件路径"
              style="width: 100%"
              disabled
            />
          </el-form-item>
        </el-col>
        <el-col :span="12">
          <el-form-item :label-width="labelWidth" label="解压路径：">
            <el-input
              v-model="postForm.unzipPath"
              placeholder="解压路径"
              style="width: 100%"
              disabled
            />
          </el-form-item>
        </el-col>
      </el-row>
      <el-row>
        <el-col :span="12">
          <el-form-item :label-width="labelWidth" label="封面路径：">
            <el-input
              v-model="postForm.coverPath"
              placeholder="封面路径"
              style="width: 100%"
              disabled
            />
          </el-form-item>
        </el-col>
        <el-col :span="12">
          <el-form-item :label-width="labelWidth" label="文件名称：">
            <el-input
              v-model="postForm.fileName"
              placeholder="文件名称"
              style="width: 100%"
              disabled
            />
          </el-form-item>
        </el-col>
      </el-row>
      <el-row>
        <el-col :span="24">
          <el-form-item label-width="60px" label="封面：">
            <a v-if="postForm.cover" :href="postForm.cover" target="_blank">
              <img :src="postForm.cover" class="preview-img">
            </a>
            <span v-else>无</span>
          </el-form-item>
        </el-col>
      </el-row>
      <el-row>
        <el-col :span="24">
          <el-form-item label-width="60px" label="目录：">
            <div
              v-if="postForm.contents && postForm.contents.length > 0"
              class="contents-wrapper"
            >
              <el-tree :data="contentsTree" @node-click="onContentClick" />
            </div>
            <span v-else>无</span>
          </el-form-item>
        </el-col>
      </el-row>
    </div>
</el-col>
```



## 上传 API 开发

指定目的 nginx 上传路径，这样做的好处是一旦电子书拷贝到指定目录下后，就可以通过 nginx 生成下载链接：

```js
const { env } = require('./env')
const UPLOAD_PATH = env === 'dev' ?
  '/Users/sam/upload/admin-upload-ebook' :
  '/root/upload/admin-upload-ebook'
```

开发电子书模型 Book 和电子书逻辑参考：电子书解析

安装 multer：

```js
const multer = require('multer')
```

下载 API：

```js
router.post(
  '/upload',
  multer({ dest: `${UPLOAD_PATH}/book` }).single('file'),
  function(req, res, next) {
    if (!req.file || req.file.length === 0) {
      new Result('上传电子书失败').fail(res)
    } else {
      const book = new Book(req.file)
      book.parse()
        .then(book => {
          new Result(book.toJson(), '上传成功').success(res)
        })
        .catch((err) => {
          console.log('/book/upload', err)
          next(boom.badImplementation(err))
          book.reset()
        })
    }
  })
```



## 前端逻辑

### 上传成功事件

上传成功时，会将解析的电子书内容填入表单

```js
onUploadSuccess(data) {
  this.setData(data)
}
```

setData 方法实现如下：

```js
setData(data) {
    const {
      title,
      author,
      publisher,
      language,
      rootFile,
      cover,
      originalName,
      url,
      contents,
      contentsTree,
      fileName,
      coverPath,
      filePath,
      unzipPath
    } = data
    this.postForm = {
      title,
      author,
      publisher,
      language,
      rootFile,
      cover,
      url,
      originalName,
      contents,
      fileName,
      coverPath,
      filePath,
      unzipPath
    }
    this.fileList = [{ name: originalName, url }]
    this.contentsTree = contentsTree
}
```

### 删除电子书事件

删除电子书时，会将电子书表单内容复原

```js
onUploadRemove() {
  this.toDefault()
}
```

toDefault 方法源码如下：

```js
toDefault() {
    this.postForm = Object.assign({}, defaultForm)
    this.fileList = []
    this.contentsTree = []
}
```

我们需要将表单复原，所以需要表单的默认值：

```js
const defaultForm = {
    title: '', // 书名
    author: '', // 作者
    publisher: '', // 出版社
    language: '', // 语种
    rootFile: '', // 根文件路径
    cover: '', // 封面图片URL
    coverPath: '', // 封面图片路径
    fileName: '', // 文件名
    originalName: '', // 文件原始名称
    filePath: '', // 文件所在路径
    unzipPath: '', // 解压文件所在路径
    contents: [] // 目录
}
```



### 目录点击事件

```js
onContentClick(data) {
    const { text } = data
    if (text) {
      window.open(text)
    }
}
```

## 用户引导

### 创建 step

```js
const steps = [
  {
    element: '.el-upload-dragger',
    popover: {
      title: '上传电子书',
      description: '点击上传本地的epub电子书',
      position: 'bottom'
    }
  },
  {
    element: '.form-item-author',
    popover: {
      title: '电子书信息',
      description: '电子书上传后会自动解析电子书的信息并填入表单，此时可以对电子书的信息进行编辑',
      position: 'top'
    }
  },
  {
    element: '.submit-btn',
    popover: {
      title: '提交信息',
      description: '将电子书的信息保存到数据库',
      position: 'left'
    }
  }
]

export default steps
```



### 引入 Driver

```js
import Driver from 'driver.js'
import 'driver.js/dist/driver.min.css'
import steps from './steps'
```



### 创建 Driver

```js
mounted() {
  this.driver = new Driver({
    neevBtnText: '上一个',
    closeBtnText: '关闭',
    doneBtnText: '完成'
  })
}
```



### 显示引导

```js
showGuide() {
  this.driver.defineSteps(steps)
  this.driver.start()
}
```



# 电子书解析

## 构建函数

Book 对象分为两种场景，第一种是直接从电子书文件中解析出 Book 对象，第二种是从 data 对象中生成 Book 对象：

```js
constructor(file, data) {
    if (file) {
      this.createBookFromFile(file)
    } else if (data) {
      this.createBookFromData(data)
    }
}
```

## 从文件创建 Book 对象

从文件读取电子书后，初始化 Book 对象

```js
createBookFromFile(file) {
    const {
      destination: des, // 文件本地存储目录
      filename, // 文件名称
      mimetype = MIME_TYPE_EPUB // 文件资源类型
    } = file
    const suffix = mimetype === MIME_TYPE_EPUB ? '.epub' : ''
    const oldBookPath = `${des}/${filename}`
    const bookPath = `${des}/${filename}${suffix}`
    const url = `${UPLOAD_URL}/book/${filename}${suffix}`
    const unzipPath = `${UPLOAD_PATH}/unzip/${filename}`
    const unzipUrl = `${UPLOAD_URL}/unzip/${filename}`
    if (!fs.existsSync(unzipPath)) {
      fs.mkdirSync(unzipPath, { recursive: true }) // 创建电子书解压后的目录
    }
    if (fs.existsSync(oldBookPath) && !fs.existsSync(bookPath)) {
      fs.renameSync(oldBookPath, bookPath) // 重命名文件
    }
    this.fileName = filename // 文件名
    this.path = `/book/${filename}${suffix}` // epub文件路径
    this.filePath = this.path // epub文件路径
    this.url = url // epub文件url
    this.title = '' // 标题
    this.author = '' // 作者
    this.publisher = '' // 出版社
    this.contents = [] // 目录
    this.cover = '' // 封面图片URL
    this.category = -1 // 分类ID
    this.categoryText = '' // 分类名称
    this.language = '' // 语种
    this.unzipPath = `/unzip/${filename}` // 解压后的电子书目录
    this.unzipUrl = unzipUrl // 解压后的电子书链接
    this.originalName = file.originalname
}
```



## 从数据创建 Book 对象

从表单对象中创建 Book 对象：

```js
createBookFromData(data) {
    this.fileName = data.fileName
    this.cover = data.coverPath
    this.title = data.title
    this.author = data.author
    this.publisher = data.publisher
    this.bookId = data.fileName
    this.language = data.language
    this.rootFile = data.rootFile
    this.originalName = data.originalName
    this.path = data.path || data.filePath
    this.filePath = data.path || data.filePath
    this.unzipPath = data.unzipPath
    this.coverPath = data.coverPath
    this.createUser = data.username
    this.createDt = new Date().getTime()
    this.updateDt = new Date().getTime()
    this.updateType = data.updateType === 0 ? data.updateType : UPDATE_TYPE_FROM_WEB
    this.contents = data.contents
}
```



## 电子书解析

初始化后，可以调用 Book 实例的 parse 方法解析电子书，这里我们使用了 epub 库，我们直接将 epub 库源码集成到项目中：

### epub 库集成

epub 库源码：https://github.com/julien-c/epub，我们直接将 epub.js 拷贝到 `/utils/epub.js`

### epub 库获取图片逻辑修改

修改获取图片的源码：

```js
getImage(id, callback) {
    if (this.manifest[id]) {
      if ((this.manifest[id]['media-type'] || '').toLowerCase().trim().substr(0, 6) != 'image/') {
        return callback(new Error('Invalid mime type for image'))
      }
      this.getFile(id, callback)
    } else {
      const coverId = Object.keys(this.manifest).find(key => (
        this.manifest[key].properties === 'cover-image'))
      if (coverId) {
        this.getFile(coverId, callback)
      } else {
        callback(new Error('File not found'))
      }
    }
};
```



### 使用 epub 库解析电子书

```js
parse() {
    return new Promise((resolve, reject) => {
      const bookPath = `${UPLOAD_PATH}${this.path}`
      if (!this.path || !fs.existsSync(bookPath)) {
        reject(new Error('电子书路径不存在'))
      }
      const epub = new Epub(bookPath)
      epub.on('error', err => {
        reject(err)
      })
      epub.on('end', err => {
        if (err) {
          reject(err)
        } else {
          let {
            title,
            language,
            creator,
            creatorFileAs,
            publisher,
            cover
          } = epub.metadata
          // title = ''
          if (!title) {
            reject(new Error('图书标题为空'))
          } else {
            this.title = title
            this.language = language || 'en'
            this.author = creator || creatorFileAs || 'unknown'
            this.publisher = publisher || 'unknown'
            this.rootFile = epub.rootFile
            const handleGetImage = (error, imgBuffer, mimeType) => {
              if (error) {
                reject(error)
              } else {
                const suffix = mimeType.split('/')[1]
                const coverPath = `${UPLOAD_PATH}/img/${this.fileName}.${suffix}`
                const coverUrl = `${UPLOAD_URL}/img/${this.fileName}.${suffix}`
                fs.writeFileSync(coverPath, imgBuffer, 'binary')
                this.coverPath = `/img/${this.fileName}.${suffix}`
                this.cover = coverUrl
                resolve(this)
              }
            }
            try {
              this.unzip() // 解压电子书
              this.parseContents(epub)
                .then(({ chapters, chapterTree }) => {
                  this.contents = chapters
                  this.contentsTree = chapterTree
                  epub.getImage(cover, handleGetImage) // 获取封面图片
                })
                .catch(err => reject(err)) // 解析目录
            } catch (e) {
              reject(e)
            }
          }
        }
      })
      epub.parse()
      this.epub = epub
    })
}
```



### 电子书目录解析

电子书解析过程中我们需要自定义电子书目录解析，第一步需要解压电子书：

```js
unzip() {
    const AdmZip = require('adm-zip')
    const zip = new AdmZip(Book.genPath(this.path)) // 解析文件路径
    zip.extractAllTo(
      /*target path*/Book.genPath(this.unzipPath),
      /*overwrite*/true
    )
}
```

genPath 是 Book 的一个属性方法，我们可以使用 es6 的 static 属性来实现：

```js
static genPath(path) {
    if (path.startsWith('/')) {
      return `${UPLOAD_PATH}${path}`
    } else {
      return `${UPLOAD_PATH}/${path}`
    }
}
```

电子书目录解析算法：

```js
parseContents(epub) {
    function getNcxFilePath() {
      const manifest = epub && epub.manifest
      const spine = epub && epub.spine
      const ncx = manifest && manifest.ncx
      const toc = spine && spine.toc
      return (ncx && ncx.href) || (toc && toc.href)
    }

    /**
     * flatten方法，将目录转为一维数组
     *
     * @param array
     * @returns {*[]}
     */
    function flatten(array) {
      return [].concat(...array.map(item => {
        if (item.navPoint && item.navPoint.length) {
          return [].concat(item, ...flatten(item.navPoint))
        } else if (item.navPoint) {
          return [].concat(item, item.navPoint)
        } else {
          return item
        }
      }))
    }

    /**
     * 查询当前目录的父级目录及规定层次
     *
     * @param array
     * @param level
     * @param pid
     */
    function findParent(array, level = 0, pid = '') {
      return array.map(item => {
        item.level = level
        item.pid = pid
        if (item.navPoint && item.navPoint.length) {
          item.navPoint = findParent(item.navPoint, level + 1, item['$'].id)
        } else if (item.navPoint) {
          item.navPoint.level = level + 1
          item.navPoint.pid = item['$'].id
        }
        return item
      })
    }

    if (!this.rootFile) {
      throw new Error('目录解析失败')
    } else {
      const fileName = this.fileName
      return new Promise((resolve, reject) => {
        const ncxFilePath = Book.genPath(`${this.unzipPath}/${getNcxFilePath()}`) // 获取ncx文件路径
        const xml = fs.readFileSync(ncxFilePath, 'utf-8') // 读取ncx文件
        // 将ncx文件从xml转为json
        xml2js(xml, {
          explicitArray: false, // 设置为false时，解析结果不会包裹array
          ignoreAttrs: false  // 解析属性
        }, function(err, json) {
          if (!err) {
            const navMap = json.ncx.navMap // 获取ncx的navMap属性
            if (navMap.navPoint) { // 如果navMap属性存在navPoint属性，则说明目录存在
              navMap.navPoint = findParent(navMap.navPoint)
              const newNavMap = flatten(navMap.navPoint) // 将目录拆分为扁平结构
              const chapters = []
              epub.flow.forEach((chapter, index) => { // 遍历epub解析出来的目录
                // 如果目录大于从ncx解析出来的数量，则直接跳过
                if (index + 1 > newNavMap.length) {
                  return
                }
                const nav = newNavMap[index] // 根据index找到对应的navMap
                chapter.text = `${UPLOAD_URL}/unzip/${fileName}/${chapter.href}` // 生成章节的URL
                // console.log(`${JSON.stringify(navMap)}`)
                if (nav && nav.navLabel) { // 从ncx文件中解析出目录的标题
                  chapter.label = nav.navLabel.text || ''
                } else {
                  chapter.label = ''
                }
                chapter.level = nav.level
                chapter.pid = nav.pid
                chapter.navId = nav['$'].id
                chapter.fileName = fileName
                chapter.order = index + 1
                chapters.push(chapter)
              })
              const chapterTree = []
              chapters.forEach(c => {
                c.children = []
                if (c.pid === '') {
                  chapterTree.push(c)
                } else {
                  const parent = chapters.find(_ => _.navId === c.pid)
                  parent.children.push(c)
                }
              }) // 将目录转化为树状结构
              resolve({ chapters, chapterTree })
            } else {
              reject(new Error('目录解析失败，navMap.navPoint error'))
            }
          } else {
            reject(err)
          }
        })
      })
    }
}
```

## Book 对象的其他方法

```js
toJson() {
    return {
      path: this.path,
      url: this.url,
      title: this.title,
      language: this.language,
      author: this.author,
      publisher: this.publisher,
      cover: this.cover,
      coverPath: this.coverPath,
      unzipPath: this.unzipPath,
      unzipUrl: this.unzipUrl,
      category: this.category,
      categoryText: this.categoryText,
      contents: this.contents,
      contentsTree: this.contentsTree,
      originalName: this.originalName,
      rootFile: this.rootFile,
      fileName: this.fileName,
      filePath: this.filePath
    }
}

toDb() {
    return {
      fileName: this.fileName,
      cover: this.cover,
      title: this.title,
      author: this.author,
      publisher: this.publisher,
      bookId: this.bookId,
      updateType: this.updateType,
      language: this.language,
      rootFile: this.rootFile,
      originalName: this.originalName,
      filePath: this.path,
      unzipPath: this.unzipPath,
      coverPath: this.coverPath,
      createUser: this.createUser,
      createDt: this.createDt,
      updateDt: this.updateDt,
      category: this.category || 99,
      categoryText: this.categoryText || '自定义'
    }
}

getContents() {
    return this.contents
}

reset() {
    if (this.path && Book.pathExists(this.path)) {
      fs.unlinkSync(Book.genPath(this.path))
    }
    if (this.filePath && Book.pathExists(this.filePath)) {
      fs.unlinkSync(Book.genPath(this.filePath))
    }
    if (this.coverPath && Book.pathExists(this.coverPath)) {
      fs.unlinkSync(Book.genPath(this.coverPath))
    }
    if (this.unzipPath && Book.pathExists(this.unzipPath)) {
      // 注意node低版本将不支持第二个属性
      fs.rmdirSync(Book.genPath(this.unzipPath), { recursive: true })
    }
}
  
static pathExists(path) {
    if (path.startsWith(UPLOAD_PATH)) {
      return fs.existsSync(path)
    } else {
      return fs.existsSync(Book.genPath(path))
    }
}

static genCoverUrl(book) {
    console.log('genCoverUrl', book)
    if (Number(book.updateType) === 0) {
      const { cover } = book
      if (cover) {
        if (cover.startsWith('/')) {
          return `${OLD_UPLOAD_URL}${cover}`
        } else {
          return `${OLD_UPLOAD_URL}/${cover}`
        }
      } else {
        return null
      }
    } else {
      if (book.cover) {
        if (book.cover.startsWith('/')) {
          return `${UPLOAD_URL}${book.cover}`
        } else {
          return `${UPLOAD_URL}/${book.cover}`
        }
      } else {
        return null
      }
    }
}
```