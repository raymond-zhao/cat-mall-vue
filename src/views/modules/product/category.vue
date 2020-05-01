<template>
  <div>
    <el-tree :data="menus" :props="defaultProps" :expand-on-click-node="false"
             show-checkbox node-key="catId" :default-expanded-keys="expandedKey">
      <span class="custom-tree-node" slot-scope="{ node, data }">
          <span>{{ node.label }}</span>
          <span>
            <el-button type="text"
                       size="mini"
                       @click="() => edit(data)">
              Edit
            </el-button>
            <el-button v-if="node.level <= 2"
              type="text"
              size="mini"
              @click="() => append(data)">
              Append
            </el-button>
            <el-button v-if="node.childNodes.length == 0"
              type="text"
              size="mini"
              @click="() => remove(node, data)">
              Delete
            </el-button>
          </span>
        </span>
    </el-tree>
    <el-dialog
      :title="title"
      :visible.sync="dialogVisible"
      width="30%" :close-on-click-modal="false">
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input v-model="category.productUnit" autocomplete="off"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
      <el-button @click="dialogVisible = false">取 消</el-button>
      <el-button type="primary" @click="submit">确 定</el-button>
    </span>
    </el-dialog>
  </div>
</template>

<script>
  export default {
    data () {
      return {
        title: '',
        dialogType: '', // edit,add
        category: {
          name: '',
          parentCid: 0,
          catLevel: 0,
          showStatus: 1,
          sort: 0,
          productUnit: '',
          icon: '',
          catId: null
        },
        dialogVisible: false,
        expandedKey: [],
        menus: [],
        defaultProps: {
          children: 'children',
          label: 'name'
        }
      }
    },
    methods: {
      submit () {
        if (this.dialogType === 'add') {
          this.addCategory()
        }
        if (this.dialogType === 'edit') {
          this.editCategory()
        }
      },
      append (data) {
        this.dialogType = 'add'
        this.title = '添加分类'
        this.dialogVisible = true
        this.category.parentCid = data.catId
        this.category.catLevel = data.catLevel * 1 + 1
        this.category.catId = null
        this.category.name = ''
        this.category.icon = ''
        this.category.productUnit = ''
        this.category.sort = 0
        this.category.showStatus = 1
      },
      edit (data) {
        this.dialogType = 'edit'
        this.title = '编辑分类'
        this.dialogVisible = true
        // 发送请求获取当前节点最新的数据
        this.$http({
          url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
          method: 'get'
        }).then(({ data }) => {
          // 请求成功
          this.category.name = data.data.name
          this.category.catId = data.data.catId
          this.category.icon = data.data.icon
          this.category.productUnit = data.data.productUnit
        })
      },
      // 编辑分类
      editCategory () {
        const {catId, name, icon, productUnit} = this.category
        this.$http({
          url: this.$http.adornUrl('/product/category/update'),
          method: 'post',
          data: this.$http.adornData({ catId, name, icon, productUnit }, false)
        }).then(({ data }) => {
          this.$message({
            message: '菜单修改成功',
            type: 'success'
          })
          // 关闭对话框
          this.dialogVisible = false
          // 刷新出新的菜单
          this.getMenus()
          // 设置需要默认展开的菜单
          this.expandedKey = [this.category.parentCid]
        })
      },
      // 添加分类
      addCategory () {
        this.$http({
          url: this.$http.adornUrl('/product/category/save'),
          method: 'post',
          data: this.$http.adornData(this.category, false)
        }).then(({data}) => {
          this.$message({
            message: '添加成功',
            type: 'success'
          })
          this.dialogVisible = false
          this.getMenus()
          this.expandedKey = [this.category.parentCid]
        })
      },
      // 删除分类
      remove (node, data) {
        this.$confirm(`确认删除[${data.name}]？`, '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {
          const ids = [data.catId]
          this.$http({
            url: this.$http.adornUrl('/product/category/delete'),
            method: 'post',
            data: this.$http.adornData(ids, false)
          }).then(({data}) => {
            this.$message({
              message: '删除成功',
              type: 'success'
            })
            // 刷新菜单
            this.getMenus()
            // 展开 key
            this.expandedKey = [node.parent.data.catId]
          })
        }).catch(() => {

        })
      },
      getMenus () {
        this.$http({
          url: this.$http.adornUrl('/product/category/list/tree'),
          method: 'get'
        }).then(({data}) => {
          this.menus = data.data
        })
      }
    },
    created () {
      this.getMenus()
    }
  }
</script>

