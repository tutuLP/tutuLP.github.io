

表格+自定义单选筛选

```vue
<template>
  <div>
    <el-table
      :data="tableData"
      @filter-change="handleFilterChange"
    >
      <!-- 其他列... -->
      
      <!-- 带筛选功能的列 -->
      <el-table-column
        prop="yourFieldName"
        label="列名"
        width="120"
        align="center"
        :filters="filterOptions"
        :filter-multiple="false"
        :filtered-value="selectedFilters"
        filter-placement="bottom-end"
        column-key="yourFieldName"
      >
        <template slot-scope="scope">
          {{ scope.row.yourFieldName }}
        </template>
      </el-table-column>
    </el-table>
  </div>
</template>

<script>
export default {
  data() {
    return {
      tableData: [],
      selectedFilters: [], // 存储当前选中的筛选值
      selectedValue: '' // 实际用于API请求的值
    }
  },
  computed: {
    // 定义筛选选项
    filterOptions() {
      return [
        { text: '选项1', value: 'value1' },
        { text: '选项2', value: 'value2' },
        { text: '选项3', value: 'value3' }
      ]
    }
  },
  methods: {
    handleFilterChange(filters) {
      // filters 是一个对象: { 'yourFieldName': ['value1'] }
      const filterValues = filters['yourFieldName'] || []
      
      // 单选模式：取第一个值
      this.selectedValue = filterValues.length > 0 ? filterValues[0] : ''
      this.selectedFilters = filterValues
      
      // 根据筛选值重新加载数据
      this.loadData()
    },
    async loadData() {
      // 使用 this.selectedValue 作为查询参数
      const params = {}
      if (this.selectedValue) {
        params.filterField = this.selectedValue
      }
      // 调用API...
    }
  }
}
</script>
```



