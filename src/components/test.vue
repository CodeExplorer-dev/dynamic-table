<template>
<div class="title">
  <h1>Dynamic Table</h1>
</div>
<div class="operate">
  <div class="search-container">
    <el-input
      v-model="searchValue"
      placeholder="请输入ID"
      clearable
      @input="handleSearchByID"
      @keyup.enter="handleSearchByID"
      style="width: 200px; margin-right: 10px"
    />
    <el-button @click="handleSearchByID">搜索</el-button>
  </div>

  <div class="filter-container">
    <el-form
      :inline="true"
      class="form"
    >
      <el-select
        v-model="filterContent.field"
        placeholder="选择筛选字段"
        style="width: 130px; margin-right: 10px"
      >
        <el-option v-for="option in filterOptions" :key="option.value" :label="option.label" :value="option.value" />
      </el-select>
      <el-select
        v-model="filterContent.operator"
        placeholder="选择条件"
        style="width: 70px; margin-right: 10px"
      >
        <el-option label=">" value=">" />
        <el-option label=">=" value=">=" />
        <el-option label="==" value="==" />
        <el-option label="<" value="<" />
        <el-option label="<=" value="<=" />
        <el-option label="!=" value="!=" />
      </el-select>
      <el-input v-model="filterContent.refer" placeholder="输入数值" style="width: 130px; margin-right: 10px"></el-input>
    </el-form>
  </div>
</div>
<div class="table-container">
  <el-table 
    :data="paginatedData" 
    border 
    height="435"
    @sort-change="handleSort"
  >
    <el-table-column fixed prop="ID" label="ID" sortable="custom"/>
    <el-table-column v-for="(item, index) in filterOptions" :key="index" :prop="item.value" :label="item.label" sortable="custom"/>
  </el-table>
</div>
<div class="pagination-container">
  <el-pagination
    background
    class="pagination"
    layout="sizes, prev, pager, next"
    :total="filteredData.length"
    :page-sizes="[10, 20, 50]"
    :page-size="pageSize"
    @size-change="handleSizeChange"
    @current-change="handleCurrentChange"
  />
</div>
</template>

<script setup>
import { ElMessage } from 'element-plus';
import { ref, onMounted, computed, reactive } from 'vue';
import { read, utils } from 'xlsx';
 
const tableData = ref([]) // 表格数据
const originalData = ref([]) // 原始数据
const filteredData = ref([]) // 过滤后的数据
const currentPage = ref(1) // 当前页码
const pageSize = ref(10) // 每页显示条数
const searchValue = ref('') 
const filterOptions = ref([])
const filterContent = reactive({
  field: '',
  operator: '>',
  refer: ''
})


// 计算当前页的数据
const paginatedData = computed(() => {
  const start = (currentPage.value - 1) * pageSize.value
  const end = start + pageSize.value
  return filteredData.value.slice(start, end)
})
 
// 切换每页显示数量
const handleSizeChange = (newSize) => {
  console.log(newSize)
  pageSize.value = newSize
  currentPage.value = 1 // 回到第一页
};
 
// 翻页
const handleCurrentChange = (newPage) => {
  console.log(newPage)
  currentPage.value = newPage;
};

// 搜索
const handleSearchByID = () => {
  if (!searchValue.value) {
    filteredData.value = [...tableData.value]
    currentPage.value = 1
    return
  }
  const searchID = searchValue.value.toLowerCase()
  filteredData.value = tableData.value.filter(item => 
    item.ID && item.ID.toString().toLowerCase().includes(searchID)
  )
  currentPage.value = 1
}

const handleFilter = () => {
  const field = filterContent.field
  const operator = filterContent.operator
  const refer = filterContent.refer

  console.log('字段:', field)
  console.log('条件:', operator)
  console.log('数值:', refer)

  if (!field || !operator || refer === '') {
    ElMessage.warning('请填写完整的筛选条件');
    return;
  }

  switch (operator) {
    case '>':
      filteredData.value = [...filteredData.value].filter(item => item[field] > refer)
      break
    case '>=':
      filteredData.value = [...filteredData.value].filter(item => item[field] >= refer)
      break
    case '==':
      filteredData.value = [...filteredData.value].filter(item => item[field] == refer)
      break
    case '<':
      filteredData.value = [...filteredData.value].filter(item => item[field] < refer)
      break
    case '<=':
      filteredData.value = [...filteredData.value].filter(item => item[field] <= refer)
      break
    case '!=':
      filteredData.value = [...filteredData.value].filter(item => item[field] != refer)
      break
    default:
      ElMessage.warning('请选择正确的条件')
  }
}

const handleReset = () => {
  filteredData.value = [...tableData.value]
  currentPage.value = 1
  filterContent.field = ''
  filterContent.operator = '>'
  filterContent.refer = ''
}


const getTableData = async (file) => {
  try {
    const data = await file.arrayBuffer()
    const workbook = read(data)
    console.log(workbook)
    const sheet1 = workbook.Sheets[workbook.SheetNames[0]]
    originalData.value = utils.sheet_to_json(sheet1)
    tableData.value = utils.sheet_to_json(sheet1)
    filteredData.value = utils.sheet_to_json(sheet1)
    filterOptions.value = Object.keys(tableData.value[0]).map(key => ({ label: key, value: key })).filter(option => option.value !== 'ID')
    console.log(filterOptions.value);
    
    currentPage.value = 1
  } catch (error) {
    console.error("Error loading file:", error)
    throw error
  }
}

const loadDefaultFile = async () => {
  try {
    // 获取Excel文件，并转换为File对象
    const response = await fetch('/front-end-dynamic-table.xlsx')
    console.log(response)
    if (response.status !== 200) throw new Error("File not found")
    ElMessage.success('加载默认文件成功')
    const blob = await response.blob() // 将响应转换为Blob对象（二进制大对象）
    console.log(blob)
    const file = new File([blob], 'front-end-dynamic-table.xlsx', { 
      type: 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet' 
    })
    console.log(file)
    await getTableData(file)
  } catch (error) {
    console.warn("Failed to load default file:", error.message)
    ElMessage.error(`加载默认文件失败: ${error.message}`)
    currentPage.value = 1
  }
}

const handleSort = ({ prop, order }) => {
  console.log('排序字段:', prop)
  console.log('排序顺序:', order)

  if (!prop || !order) {
    filteredData.value = [...originalData.value]
    return
  }

  filteredData.value = [...filteredData.value].sort((a, b) => {
    if (order === 'ascending') {
      return a[prop] > b[prop] ? 1 : -1
    } else if (order === 'descending') {
      return a[prop] < b[prop] ? 1 : -1
    } else {
      return 0
    }
  })
}
 
onMounted(() => {
  loadDefaultFile()
})
</script>