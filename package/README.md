# el-table-editabled

一个用来实现el-table复杂编辑并包含验证场景的组件

[点击查看在线demo](https://code-farmer-i.github.io/el-table-editabled/dist/)

## Install
```shell
npm install el-table-editabled -S
```

## webpack.base.conf.js
因代码中使用了es6的语法 所以需要添加babel配置
``` javascript
'use strict'
const path = require('path')
const utils = require('./utils')
const config = require('../config')
const vueLoaderConfig = require('./vue-loader.conf')
// 此处为添加的配置
const fs = require('fs')

function resolve (dir) {
  return path.join(__dirname, '..', dir)
}

//此处为添加的配置
let dirsName = fs.readdirSync(resolve('node_modules')).filter(dirName => /el-table-editabled/.test(dirName))
const includesDirs = dirsName.map(dir => resolve(`node_modules/${dir}/src`))

module.exports = {
  ... //省略的代码
  module: {
    rules: [
      // 省略代码...
      // 此处有添加babel-loader配置 ‘...includesDirs‘
      {
        test: /.js$/,
        loader: 'babel-loader',
        include: [resolve('src'), resolve('test'), resolve('node_modules/webpack-dev-server/client'), ...includesDirs]
      },
      {
        test: /.(png|jpe?g|gif|svg)(?.*)?$/,
        loader: 'url-loader',
        options: {
          limit: 10000,
          name: utils.assetsPath('img/[name].[hash:7].[ext]')
        }
      },
```

## Quick Start
``` javascript
import Vue from 'vue'
import ElTableEditabled from 'el-table-editabled'

Vue.use(ElTableEditabled)
```

### 基础用法
```html
<template>
  <div>
    <div style="margin-bottom: 12px">
      <el-button @click="handleValidate">表格验证</el-button>
      <el-button @click="handleNewRows">新增行</el-button>
      <el-button @click="handleDelRows">删除行</el-button>
      <el-button @click="handleCheckData">打印编辑后的表格数据</el-button>
    </div>
    <el-table-editabled
      v-model="tableData"
      :columns="['date', 'name']"
      :validators="validators"
      ref="elTableEditabled"
    >
      <el-table
        :data="tableData"
        style="width: 100%"
        @selection-change="selection = arguments[0]"
      >
        <el-table-column
          type="selection"
        ></el-table-column>
        <el-table-column
          fixed
          prop="date"
          label="日期"
        >
          <template slot-scope="{ row }">
            <el-table-editabled-cell
              :row="row"
              prop="date"
            >
              <template slot-scope="{ rowStates, validateOwn }">
                <span v-show="!rowStates.editing">{{row.date}}</span>
                <el-date-picker
                  v-show="rowStates.editing"
                  v-model="row.date"
                  type="date"
                  value-format="yyyy-MM-dd"
                  placeholder="选择日期"
                  clearable
                  @change="validateOwn"
                >
                </el-date-picker>
              </template>
            </el-table-editabled-cell>
          </template>
        </el-table-column>
        <el-table-column
          prop="name"
          label="姓名"
        >
          <template slot-scope="{ row }">
            <el-table-editabled-cell
              :row="row"
              prop="name"
            >
              <template slot-scope="{ rowStates, validateOwn }">
                <span v-show="!rowStates.editing">{{row.name}}</span>
                <el-input
                  v-show="rowStates.editing"
                  v-model="row.name"
                  clearable
                  @blur="validateOwn"
                >
                </el-input>
              </template>
            </el-table-editabled-cell>
          </template>
        </el-table-column>
        <el-table-column
          prop="zip"
          label="邮编"
        >
        </el-table-column>
        <el-table-column
          fixed="right"
          label="操作"
          width="120">
          <template slot-scope="{ row }">
            <el-table-editabled-cell
              :row="row"
            >
              <template slot-scope="{ rowStates }">
                <el-button
                  v-show="!rowStates.editing"
                  @click="handleEdit(row)"
                  type="text"
                  size="small">
                  编辑
                </el-button>
                <div v-show="rowStates.editing">
                  <el-button
                    @click="handleCanelRow(row)"
                    type="text"
                    size="small">
                    取消
                  </el-button>
                  <el-button
                    @click="handleSave(row)"
                    type="text"
                    size="small">
                    保存
                  </el-button>
                </div>
              </template>
            </el-table-editabled-cell>
          </template>
        </el-table-column>
      </el-table>
    </el-table-editabled>
  </div>
</template>

<script>
export default {
  data () {
    return {
      tableData: [{
        date: '2016-05-03',
        name: '王小虎',
        zip: 200333
      }, {
        date: '2016-05-02',
        name: '王小虎',
        zip: 200333
      }, {
        date: '2016-05-04',
        name: '王小虎',
        zip: 200333
      }],
      validators: {
        date ({ date }, next) {
          if (!date) {
            next('请选择日期')
          } else {
            next('')
          }
        },
        name ({ name }, next) {
          if (!name) {
            next('请选择日期')
          } else {
            next('')
          }
        }
      }
    }
  },
  methods: {
    handleCheckData () {
      console.log(this.tableData)
    },
    getNewRowData () {
      return {
        date: '',
        name: '',
        zip: ''
      }
    },
    handleEdit (row) {
      this.$refs.elTableEditabled.editRows([row])
    },
    handleCanelRow (row) {
      this.$refs.elTableEditabled.cancelRows([row])
    },
    handleSave (row) {
      // 验证一些正在编辑的行
      this.$refs.elTableEditabled.validateRows([row], valid => {
        // valid 为布尔值 代表表格正在编辑的所有字段是否验证通过
        if (valid) {
          // 第二个参数为 是否回退数据 若传false 则只取消编辑状态 编辑的数据会保存
          this.$refs.elTableEditabled.cancelRows([row], false)
        }
      })
    },
    handleNewRows () {
      const newRow = this.getNewRowData()

      this.$refs.elTableEditabled.newRows([newRow])
    },
    handleDelRows () {
      // 删除选中的数据
      this.$refs.elTableEditabled.delRows(this.selection)
    },
    handleValidate () {
      // 验证表格中所有正在编辑的字段
      this.$refs.elTableEditabled.validate(valid => {
        // valid 为布尔值 代表表格正在编辑的所有字段是否验证通过
        console.log(valid)
      })
    }
  }
}
</script>
```

## API

### El-Table-Editabled Props:

属性  |  说明  |  类型  |  默认值
:-------: | -------  |  :-------:  |  :-------:
v-model  |  绑定el-table的表格数据  |  Array  |  --
columns  |  需要编辑的列的字段名  |  Array  |  --
rowStates  |  可以给行添加一些状态  |  Function(row)  |  --
cellStates  | 可以给每一个单元格添加一些状态 |  Object  |  --
validators  | 给需要编辑的列添加验证规则  |  --
default-editing  |  表格是否默认为正在编辑状态  |  Boolean  | false
validate-msg-tooltip  |  表格验证的提示信息是否使用tooltip展示  |  Boolean  | false

### El-Table-Editabled Methods:

方法  |  说明  |  参数
:-------: | -------  |  :-------:
editRows  |  使某些行进入编辑状态  |  Function(rows: Array&lt;row&gt;)
editColumns | 是某些行的某些字段进入编辑状态 | Function(rows: Array&lt;row&gt;, columns: Array&lt;column&gt;)
delRows  |  删除某些行  |  Function(rows: Array&lt;row&gt;)
newRows  |  新增行  |  Function(newDatas: Array&lt;newRowData&gt;)
insertRowsBeforeRow | 在某一行之前插入新数据 | Function(row: Object, newDatas: Array&lt;newRowData&gt;)
insertRowsAfterRow | 在某一行之后插入新数据 | Function(row: Object, newDatas: Array&lt;newRowData&gt;)
cancelRows | 撤销某些行的编辑状态和数据 | Function(rows: Array&lt;row&gt;, isCancelData: Boolean(是否回滚数据，默认为true))
cancelCells | 撤销某些单元格的编辑状态和数据 | Function(rows: Array&lt;row&gt;, cells: Array&lt;cell&gt;, isCancelData: Boolean(是否回滚数据，默认为true))
validateRows | 对某些正在编辑的行数据进行校验 | Function(rows: Array&lt;row&gt;, callback: Function(valid))
validate | 对整个表格正在编辑的数据进行校验 | Function(callback: Function(valid))

### El-Table-Editabled-Cell Props:

属性  |  说明  |  类型  |  默认值
:-------: | -------  |  :-------:  |  :-------:
row  |  当前单元格所在的行数据  | Object  |  --
prop  |  字段名(对应columns数组里面的配置)  |  String  |  --
dependencies  |  可以通过该属性来监听rowStates，row，cellStates上的一些数据的变化来触发某些动作  |  Object  |  --

### El-Table-Editabled-Cell Scoped Slot:

插槽名  |  说明  |  scope
:-------: | -------  |  :-------:
--  |  当前单元格的内容  | { rowStates: { editing }, cellStates, ownStates: { editing, validateMsg }, validateOwn }

作者wx: ckang1229

