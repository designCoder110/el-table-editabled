<template>
  <div class="el-table-editabled">
    <slot></slot>
  </div>
</template>

<script>
  import {
    getEmptyArray,
    getEmptyObject,
    isEmpty,
    deepCopy,
    observable
  } from "./utils"
  import TabelStore from './table-store'

  export default {
    name: 'ElTableEditabled',
    provide () {
      return {
        editValidator: this
      }
    },
    model: {
      prop: 'tableData',
      event: 'table-data-change'
    },
    props: {
      tableData: {
        type: Array,
        default: getEmptyArray
      },
      columns: {
        type: Array,
        default: getEmptyArray
      },
      rowStates: {
        type: Function,
        default: getEmptyObject
      },
      cellStates: {
        type: Object,
        default: () => {
          return {
            default: getEmptyObject
          }
        }
      },
      validators: {
        type: Object,
        default: getEmptyObject
      },
      defaultEditing: {
        type: Boolean,
        default: false
      },
      validateMsgTooltip: {
        type: Boolean,
        default: false
      }
    },
    data () {
      return {
        store: new Map()
      }
    },
    watch: {
      tableData: {
        immediate: true,
        handler (newVal, oldVal) {
          if (!this.byOwnerAction && newVal !== oldVal) {
            this.init()
          }
          this.byOwnerAction = false
        }
      }
    },
    methods: {
      init () {
        this.tableCacheData = new WeakMap()
        const cellStatesCreator = {}

        this.columns.forEach(cell => {
          const initCellStates = this.cellStates[cell]

          cellStatesCreator[cell] = row => {
            const ownStates = Object.assign({
              editing: this.defaultEditing,
              validateMsg: '',
              hovering: false
            }, initCellStates && initCellStates(row))

            return observable(ownStates)
          }
        })

        this.store = new TabelStore({
          tableData: this.tableData,
          columns: this.columns,
          rowStatesCreator: row => {
            const rowStates = Object.assign({
              editing: this.defaultEditing
            }, this.rowStates(row))

            return observable(rowStates)
          },
          cellStatesCreator,
          onInitLoop: (row) => {
            // 在初始化表格状态遍历表格数据的时候去初始化表格缓存数据
            this.updateTableCache([row], 'add')
          }
        })
      },

      editRows (rows) {
        this.store.setRowsStates(rows, {
          editing: true
        })
      },

      updateTableCache (rows, action) {
        const {
          tableCacheData
        } = this

        rows.forEach(row => {
          if ((action === 'add' && !tableCacheData.has(row)) || action === 'update') {
            tableCacheData.set(row, deepCopy(row))
          } else if (action === 'delete') {
            tableCacheData.delete(row)
          }
        })
      },

      saveRows (rows) {
        this.saveCells(rows, this.columns)
      },

      saveCells (rows, cells) {
        this.cancelCells(rows, cells, false)
      },

      cancelRows (rows) {
        this.cancelCells(rows, this.columns)
      },

      cancelCells (rows, cells, cancelData = true) {
        const {
          tableCacheData,
          store
        } = this

        rows.forEach(row => {
          const rowCacheData = tableCacheData.get(row)

          cells.forEach(cell => {
            // 数据回滚
            if (cancelData && rowCacheData) {
              if (cell in rowCacheData) {
                row[cell] = rowCacheData[cell]
              } else {
                delete row[cell]
              }
            }

            // 取消单元格编辑状态并重置表单验证状态
            store.setCellStates([row], [cell], {
              editing: false,
              validateMsg: ''
            })
          })

          // 更新缓存 防止字段的值是引用类型 导致修改数据的时候缓存也会被修改
          this.updateTableCache([row], 'update')

          // 取消行编辑状态
          store.setRowsStates([row], {
            editing: false
          })
        })
      },

      editColumns (rows, cells) {
        this.store.setCellStates(rows, cells, {
          editing: true
        })
      },

      delRows (rows) {
        this.byOwnerAction = true
        this.updateTableCache(rows, 'delete')

        this.$emit('table-data-change', this.tableData.filter(row => !rows.includes(row)))

        this.store.delStates(rows)
      },

      newRows (rows) {
        const firstRow = this.tableData[0]

        this.insertRowsBeforeRow(firstRow, rows)
      },

      insertRowsBeforeRow (row, rows) {
        this.insertRows('before', row, rows)
      },

      insertRowsAfterRow (row, rows) {
        this.insertRows('after', row, rows)
      },

      insertRows (type, row, rows) {
        let insertIdx

        insertIdx = this.tableData.findIndex((_row) => _row === row)
        if (insertIdx === -1) {
          insertIdx = 0
        } else if (type === 'after'){
          insertIdx++
        }
        
        this.byOwnerAction = true
        this.tableData.splice(insertIdx, 0, ...rows)
        this.store.addStates(rows)
        this.updateTableCache(rows, 'add')
        this.editRows(rows)
      },

      validateRows (rows, cb) {
        this.validateCells(rows, this.columns, cb)
      },

      validate (cb) {
        this.validateRows(this.tableData, cb)
      },

      async validateCells (rows, cells, cb) {
        let validatePromiseStacks = []
        let valid

        rows.forEach(row => {
          const cellStates = this.store.getStates(row)
          const rowStates = cellStates._states

          cells.forEach(cell => {
            const validator = this.validators[cell]
            const ownStates = cellStates ? cellStates[cell] : {}

            if ((ownStates.editing || rowStates.editing) && validator) {
              const validPropPromise = new Promise((resolve, reject) => {
                const next = (errorMsg) => {
                  if (ownStates.editing || rowStates.editing) {
                    ownStates.validateMsg = errorMsg

                    if (errorMsg) {
                      reject()
                    } else {
                      resolve()
                    }
                  }
                }

                validator(row, next, rowStates, cellStates)
              })

              validatePromiseStacks.push(validPropPromise)
            }
          })
        })

        try {
          await Promise.all(validatePromiseStacks)

          valid = true
        } catch(e) {
          valid = false
        }

        typeof cb === 'function' && cb(valid)
      },

      getTableEditState () {
        const {
          tableData,
          columns,
          store
        } = this
        let tableEditing = false

        for (let rowIdx = 0;rowIdx < tableData.length;rowIdx++) {
          const row = tableData[rowIdx]
          const cellStates = store.getStates(row)
          const rowStates = cellStates._states
          
          // if current row is editing
          if (rowStates.editing) return true

          for (let columnIdx = 0;columnIdx < columns.length;columnIdx++) {
            const cell = columns[columnIdx]
            const cellState = cellStates[cell]

            // if current prop is editing
            if (cellState.editing) return true
          }
        }
        
        return tableEditing
      }
    }
  }
</script>
