<script setup>
import {
  NButton,
  NCard,
  NDataTable,
  NGi,
  NGrid,
  NInput,
  NInputGroup,
  NInputGroupLabel,
  NLayout,
  NModal,
  NSlider,
  NSpace,
  NStatistic
} from "naive-ui";
import {computed, defineComponent, h, nextTick, reactive, ref, watch} from "vue";
import {utils, writeFile} from "xlsx";

const ShowOrEdit = defineComponent({
  props: {
    value: [String, Number],
    onUpdateValue: [Function, Array]
  },
  setup(props) {
    const isEdit = ref(false)
    const inputRef = ref(null)
    const inputValue = ref(props.value)

    function handleOnClick() {
      isEdit.value = true
      nextTick(() => {
        inputRef.value.focus()
      })
    }

    function handleChange() {
      props.onUpdateValue(inputValue.value)
      isEdit.value = false
    }

    return () =>
        h(
            'div',
            {
              style: 'min-height: 22px',
              onClick: handleOnClick
            },
            isEdit.value
                ? h(NInput, {
                  round: true,
                  ref: inputRef,
                  size: 'small',
                  value: inputValue.value,
                  onUpdateValue: (v) => {
                    inputValue.value = v
                  },
                  onChange: handleChange,
                  onBlur: handleChange
                })
                : props.value
        )
  }
})

function createTableColumns(tableData) {
  return [
    {
      title: '序号',
      key: 'no',
    }, {
      title: '名称',
      key: 'name',
      render(row, index) {
        return h(ShowOrEdit, {
          value: row.name,
          onUpdateValue(v) {
            tableData.data[index].name = v
          }
        })
      },
      filter(value, row) {
        return !row.name.toLowerCase().search(value.toLowerCase())
      }
    }, {
      title: '价格',
      key: 'price',
      render(row, index) {
        return h(ShowOrEdit, {
          value: row.price,
          onUpdateValue(v) {
            tableData.data[index].price = v
          }
        })
      }
    }, {
      title: '操作',
      key: 'action',
      render(row, index) {
        return h(NButton, {
          quaternary: true,
          type: 'error',
          size: "small",
          onClick(v) {
            console.log(v)
            tableData.data = tableData.data.filter((d => d.uid !== row.uid))
                // .splice(index, 1)
            tableData.data.forEach((d, i) => {
              d.no = i + 1
            })
          }
        }, {default: () => "删除"})
      }
    },
  ]
}

const table1Ex = reactive({
  caption: '表1',
  height: 200,
  data: [],
})

const table1Ref = ref(null)
const table2Ref = ref(null)
const table3Ref = ref(null)

function tableFilterName(ref, newValue) {
  ref.value.filter({
    name: [newValue]
  })
}

const table1FilterData = ref(null)
watch(table1FilterData, (newF, _) => {
  if (newF == null || newF.length === 0) {
    tableFilterName(table1Ref, null)
  }
  tableFilterName(table1Ref, newF)
})
const table2FilterData = ref(null)
watch(table2FilterData, (newF, _) => {
  if (newF == null || newF.length === 0) {
    tableFilterName(table2Ref, null)
  }
  tableFilterName(table2Ref, newF)
})
const table3FilterData = ref(null)
watch(table3FilterData, (newF, _) => {
  if (newF == null || newF.length === 0) {
    tableFilterName(table3Ref, null)
  }
  tableFilterName(table3Ref, newF)
})


const table2Ex = reactive({
  caption: '表2',
  height: 200,
  data: [],
})

const table3Ex = reactive({
  caption: '中间表',
  height: 200,
  data: [],
})


const table1Columns = createTableColumns(table1Ex)
const table2Columns = createTableColumns(table2Ex)
const table3Columns = [
  {
    title: '序号',
    key: 'no',
  }, {
    title: '名称',
    key: 'name',
    render(row, index) {
      return h(ShowOrEdit, {
        value: row.name,
        onUpdateValue(v) {
          table3Ex.data[index].name = v
        }
      })
    },
    filter(value, row) {
      return !row.name.toLowerCase().search(value.toLowerCase())
    }
  },
  {
    title: '操作',
    key: 'action',
    render(row, index) {
      return h(NButton, {
        quaternary: true,
        type: 'error',
        size: "small",
        onClick(v) {
          table3Ex.data.splice(index, 1)
          table3Ex.data.forEach((d, i) => {
            d.no = i + 1
          })
        }
      }, {default: () => "删除"})
    }
  },
]


const editModelRef = reactive({
  showEditModel: false,
  previewEditColumns: null,
  editDataSplit: ',',
  editRawData: null,
  targetTableEx: null
})

const previewEditData = computed(() => {
  const rawData = editModelRef.editRawData || ''
  if (rawData == null || rawData.length === 0) {
    return []
  }

  return rawData.split('\n')
      .map(line => line.trim())
      .filter(line => line.length > 0)
      .map((line, i) => {
        const split = line.split(new RegExp(editModelRef.editDataSplit))
        const no = i + 1
        const name = split[0]
        let price = ''
        if (split.length > 1) {
          price = split[1]
        }

        return {no: no, name: name, price: price}
      })
})

/**
 * 获取2个字符串的相似度
 * @param {string} str1 字符串1
 * @param {string} str2 字符串2
 * @returns {number} 相似度
 */
function getSimilarity(str1, str2) {
  let sameNum = 0
  //寻找相同字符
  for (let i = 0; i < str1.length; i++) {
    for(let j =0;j<str2.length;j++){
      if(str1[i]===str2[j]){
        sameNum ++
        break
      }
    }
  }
  // console.log(str1,str2);
  // console.log("相似度",(sameNum/str1.length) * 100);
  //判断2个字符串哪个长度比较长
  let length = str1.length > str2.length ? str1.length : str2.length
  return (sameNum/length) * 100 || 0
}


function editTable1() {
  editModelRef.targetTableEx = table1Ex
  editModelRef.editRawData = null
  editModelRef.previewEditColumns = table1Columns.filter(d => d.key !== 'action').map(d => {
    return {title: d.title, key: d.key}
  })
  editModelRef.showEditModel = true
}

function editTable2() {
  editModelRef.targetTableEx = table2Ex
  editModelRef.editRawData = null
  editModelRef.previewEditColumns = table2Columns.filter(d => d.key !== 'action').map(d => {
    return {title: d.title, key: d.key}
  })
  editModelRef.showEditModel = true
}

function editTable3() {
  editModelRef.targetTableEx = table3Ex
  editModelRef.editRawData = null
  editModelRef.previewEditColumns = table3Columns.filter(d => d.key !== 'action').map(d => {
    return {title: d.title, key: d.key}
  })
  editModelRef.showEditModel = true
}

function editTableConfirm() {
  const targetEx = editModelRef.targetTableEx
  // 数据保存
  const oldData = targetEx.data
  console.log(oldData)
  const newData = previewEditData.value
  console.log(newData)

  const finalData = [...oldData]
  newData.forEach((d, i) => {
    const no = i + 1 + oldData.length
    const d0 = {...d}
    d0.no = no
    d0.uid = crypto.randomUUID()
    finalData.push(d0)
  })

  targetEx.data = finalData
  editModelRef.targetTableEx = null
  editModelRef.editRawData = null
  editModelRef.previewEditColumns = []
  editModelRef.showEditModel = false
}

const mergedTableColumns = reactive([
  {
    title: '序号',
    key: 'no',
  }, {
    title: '名称',
    key: 'name',
  }, {
    get title() {
      return table1Ex.caption + ':名称'
    },
    key: 't1name',
  }, {
    get title() {
      return table1Ex.caption + ':匹配度'
    },
    key: 't1source',
  }, {
    get title() {
      return table2Ex.caption + ':名称'
    },
    key: 't2name',
  }, {
    get title() {
      return table2Ex.caption + ':匹配度'
    },
    key: 't2source',
  }, {
    get title() {
      return table1Ex.caption + ':价格'
    },
    key: 't1price',
  }, {
    get title() {
      return table2Ex.caption + ':价格'
    },
    key: 't2price',
  }
])

const mergedTableEx = reactive({
  height: 200
})

const mergedTableData = computed(() => {
  return table3Ex.data.map((r, i) => {
    let t1v = table1Ex.data.find(d => d.name === r.name)
    if (t1v == null) {
      const dataCp = [...table1Ex.data]
      const sorted = dataCp.sort((a, b) => getSimilarity(r.name, a.name) - getSimilarity(r.name, b.name))
      t1v = sorted[sorted.length - 1]
      // t1v = table1Ex.data.find(d => getSimilarity(d.name, r.name) > 30)
    }
    let t2v = table2Ex.data.find(d => d.name === r.name)
    if (t2v == null) {
      const dataCp = [...table2Ex.data]
      const sorted = dataCp.sort((a, b) => getSimilarity(r.name, a.name) - getSimilarity(r.name, b.name))
      t2v = sorted[sorted.length - 1]
    }

    const source1 = t1v != null ? getSimilarity(t1v.name, r.name) : null
    const source2 = t2v != null ? getSimilarity(t2v.name, r.name) : null

    return {
      no: i + 1,
      name: r.name,
      t1source: source1,
      t1name: t1v?.name,
      t2source: source2,
      t2name: t2v?.name,
      t1price: t1v?.price,
      t2price: t2v?.price,
    }

  })
})

function exportMergedDataToExcel() {
  const dataList = mergedTableData.value.map((v) => {
    const r = {}
    r[mergedTableColumns.find(c => c.key === 'no').title] = v.no
    r[mergedTableColumns.find(c => c.key === 'name').title] = v.name
    r[mergedTableColumns.find(c => c.key === 't1name').title] = v.t1name
    r[mergedTableColumns.find(c => c.key === 't1source').title] = v.t1source
    r[mergedTableColumns.find(c => c.key === 't2name').title] = v.t2name
    r[mergedTableColumns.find(c => c.key === 't2source').title] = v.t2source
    r[mergedTableColumns.find(c => c.key === 't1price').title] = v.t1price
    r[mergedTableColumns.find(c => c.key === 't2price').title] = v.t2price
    return r
  })
  const data = utils.json_to_sheet(dataList)
  const wb = utils.book_new()
  utils.book_append_sheet(wb, data, 'data')
  writeFile(wb, 'export.xlsx')
}

function clearAll(tableEx) {
  tableEx.data = []
}

</script>

<template>
  <n-modal v-model:show="editModelRef.showEditModel">
    <n-card
        style="max-width: 600px"
        title="编辑数据"
        :bordered="false"
        size="huge"
        role="dialog"
        aria-modal="true"
    >

      <n-space vertical>
        <n-input-group>
          <n-input-group-label>每行数据切割符:</n-input-group-label>
          <n-input placeholder="正则切割符" v-model:value="editModelRef.editDataSplit"/>
        </n-input-group>

        <n-input
            v-model:value="editModelRef.editRawData"
            type="textarea"
            placeholder="csv原数据"
        />

        数据预览:

        <n-data-table
            :columns="editModelRef.previewEditColumns"
            :data="previewEditData"
            :max-height="400"
            virtual-scroll
        />
      </n-space>

      <template #footer>
        <n-button type="success" @click="editTableConfirm">
          确认
        </n-button>
      </template>
    </n-card>
  </n-modal>


  <!--  <n-space>-->
  <n-layout :native-scrollbar="false">
    <n-layout-header>
      <n-grid x-gap="20" :cols="2" style="padding-bottom: 25px;">
        <n-gi>
          <n-statistic label="两表数据量" :value="table1Ex.data.length + ' / ' + table2Ex.data.length"/>
        </n-gi>
        <n-gi>
          <n-statistic label="中间表数据量" :value="table3Ex.data.length"/>
        </n-gi>
      </n-grid>
    </n-layout-header>
    <n-layout-content content-style="padding: 30px; ">
      <n-grid x-gap="20" y-gap="20" :cols="1" style="padding-bottom: 30px; ">
        <n-gi>
          <n-card hoverable :bordered="false">
            <n-space vertical>
              <n-input size="small" round v-model:value="table1Ex.caption"/>
              <n-grid :cols="3" x-gap="10" y-gap="10">
                <n-gi>
                  <n-slider
                      v-model:value="table1Ex.height"
                      :min="100"
                      :max="1500"
                      :step="100"
                      style="max-width: 180px"
                  />
                </n-gi>
                <n-gi>
                  <n-input size="small" round clearable v-model:value="table1FilterData" placeholder="查找名称"/>
                </n-gi>
              </n-grid>
              <n-data-table
                  virtual-scroll
                  ref="table1Ref"
                  :columns="table1Columns"
                  :data="table1Ex.data"
                  :style="{ height: `${table1Ex.height}px` }"
                  flex-height
              />
            </n-space>
            <template #action>
              <n-space justify="end">
                <n-button type="info" @click="editTable1">
                  编辑
                </n-button>
                <n-button type="error" @click="clearAll(table1Ex)">
                  全部清除
                </n-button>
              </n-space>
            </template>
          </n-card>
        </n-gi>
        <n-gi>
          <n-card hoverable :bordered="false">
            <n-space vertical>
              <n-input size="small" round v-model:value="table2Ex.caption"/>
              <n-grid :cols="3" x-gap="10" y-gap="10">
                <n-gi>
                  <n-slider
                      v-model:value="table2Ex.height"
                      :min="100"
                      :max="1500"
                      :step="100"
                      style="max-width: 180px"
                  />
                </n-gi>
                <n-gi>
                  <n-input size="small" round clearable v-model:value="table2FilterData" placeholder="查找名称"/>
                </n-gi>
              </n-grid>
              <n-data-table
                  virtual-scroll
                  ref="table2Ref"
                  :columns="table2Columns"
                  :data="table2Ex.data"
                  :style="{ height: `${table2Ex.height}px` }"
                  flex-height
              />
            </n-space>
            <template #action>
              <n-space justify="end">
                <n-button type="info" @click="editTable2">
                  编辑
                </n-button>
                <n-button type="error" @click="clearAll(table2Ex)">
                  全部清除
                </n-button>
              </n-space>
            </template>
          </n-card>

        </n-gi>
        <n-gi>
          <n-card hoverable :bordered="false">
            <n-space vertical>
              <h3>{{ table3Ex.caption }}</h3>
              <n-grid :cols="3" x-gap="10" y-gap="10">
                <n-gi>
                  <n-slider
                      v-model:value="table3Ex.height"
                      :min="100"
                      :max="1500"
                      :step="100"
                      style="max-width: 180px"
                  />
                </n-gi>
                <n-gi>
                  <n-input size="small" round clearable v-model:value="table3FilterData" placeholder="查找名称"/>
                </n-gi>
              </n-grid>
              <n-data-table
                  virtual-scroll
                  ref="table3Ref"
                  :columns="table3Columns"
                  :data="table3Ex.data"
                  :style="{ height: `${table3Ex.height}px` }"
                  flex-height
              />
            </n-space>
            <template #action>
              <n-space justify="end">
                <n-button type="info" @click="editTable3">
                  编辑
                </n-button>
                <n-button type="error" @click="clearAll(table3Ex)">
                  全部清除
                </n-button>
              </n-space>
            </template>
          </n-card>
        </n-gi>
      </n-grid>

      <n-layout content-style="padding: 20px;">
        <n-layout-content>
          <n-card title="结果表" hoverable :bordered="false">
            <template #action>
              <n-button type="info" @click="exportMergedDataToExcel">
                导出为Excel
              </n-button>
            </template>
            <n-space vertical>
              <n-slider
                  v-model:value="mergedTableEx.height"
                  :min="100"
                  :max="1500"
                  :step="100"
                  style="max-width: 180px"
              />
              <n-data-table
                  virtual-scroll
                  :columns="mergedTableColumns"
                  :data="mergedTableData"
                  :style="{ height: `${mergedTableEx.height}px` }"
                  flex-height
              />
            </n-space>
          </n-card>
        </n-layout-content>
      </n-layout>

    </n-layout-content>

    <n-layout-footer></n-layout-footer>
  </n-layout>
  <!--  </n-space>-->

</template>

<style scoped>


</style>
