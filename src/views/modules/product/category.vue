<template>
  <div>
    <el-switch v-model="draggable" active-text="允许拖拽" inactive-text="禁止拖拽"></el-switch>
    <el-button v-if="draggable" @click="batchSave" type="primary" round>批量保存</el-button>
    <el-button @click="batchDelete" type="danger" round>批量删除</el-button>
    <el-tree :data="menus" :props="defaultProps" :expand-on-click-node="false"
             show-checkbox node-key="catId" :default-expanded-keys="expandedKey"
             @node-drop="handleDrop" :draggable="draggable" :allow-drop="allowDrop" ref="menuTree">
      <span class="custom-tree-node" slot-scope="{ node, data }">
          <span>{{ node.label }}</span>
          <span>
            <el-button type="text" size="mini" @click="() => edit(data)">Edit</el-button>
            <el-button v-if="node.level <= 2" type="text" size="mini" @click="() => append(data)">Append</el-button>
            <el-button v-if="node.childNodes.length == 0" type="text" size="mini" @click="() => remove(node, data)">Delete</el-button>
          </span>
        </span>
    </el-tree>
    <el-dialog :title="title" :visible.sync="dialogVisible" width="30%" :close-on-click-modal="false">
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
        pCid: [],
        draggable: false,
        updateNodes: [],
        maxLevel: 0,
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
      batchDelete () {
        let catIds = []
        let checkedNodes = this.$refs.menuTree.getCheckedNodes()
        for (let i = 0; i < checkedNodes.length; i++) {
          catIds.push(checkedNodes[i].catId)
        }
        this.$confirm(`确认删除？`, '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {
          this.$http({
            url: this.$http.adornUrl('/product/category/delete'),
            method: 'post',
            data: this.$http.adornData(catIds, false)
          }).then(({data}) => {
            this.$message({
              message: '删除成功',
              type: 'success'
            })
            // 刷新菜单
            this.getMenus()
          })
        }).catch(() => {

        })
        console.log('checkedNodes: ', checkedNodes)
      },
      batchSave () {
        this.$http({
          url: this.$http.adornUrl('/product/category/update/sort'),
          method: 'post',
          data: this.$http.adornData(this.updateNodes, false)
        }).then(({data}) => {
          this.$message({
            message: '修改菜单顺序成功',
            type: 'success'
          })
        })
        // 4. 刷新菜单
        this.getMenus()
        this.expandedKey = this.pCid
        // 5. 置空 准备重新接收新数据
        this.updateNodes = []
        this.maxLevel = 0
      },
      /** 收集拖拽信息开始 **/
      handleDrop (draggingNode, dropNode, dropType, ev) {
        // 视频 55
        // 1. 当前结点最新的父结点 ID
        let pCid = 0
        let siblings = null
        if (dropType === 'before' || dropType === 'after') {
          pCid = dropNode.parent.data.catId === undefined ? 0 : dropNode.parent.data.catId
          siblings = dropNode.parent.childNodes
        } else {
          pCid = dropNode.data.catId
          siblings = dropNode.childNodes
        }
        this.pCid.push(pCid)
        // 2. 当前拖拽结点的最新顺序
        for (let i = 0; i < siblings.length; i++) {
          // siblings[i].data.catId === draggingNode.data.catId ? this.updateNodes.push({ catId: siblings[i].data.catId, sort: i, parentCid: pCid }) : this.updateNodes.push({ catId: siblings[i].data.catId, sort: i })
          if (siblings[i].data.catId === draggingNode.data.catId) {
            // 如果遍历的是当前正在拖拽的结点
            let catLevel = draggingNode.level
            if (siblings[i].level !== draggingNode.level) {
              // 当前结点的层级发生变化
              catLevel = siblings[i].level
              this.updateChildNodesLevel(siblings[i])
              // 修改子节点的层级
            }
            this.updateNodes.push({ catId: siblings[i].data.catId, sort: i, parentCid: pCid, catLevel: catLevel })
          } else {
            this.updateNodes.push({ catId: siblings[i].data.catId, sort: i })
          }
        }
        // 3. 当前拖拽结点的最新层级
      },
      updateChildNodesLevel (node) {
        if (node.childNodes.length > 0) {
          for (let i = 0; i < node.childNodes.length; i++) {
            const currentNode = node.childNodes[i].data
            this.updateNodes.push({ catId: currentNode.catId, catLevel: node.childNodes[i].level })
            this.updateChildNodesLevel(node.childNodes[i])
          }
        }
      },
      /** 收集拖拽信息结束 **/
      allowDrop (draggingNode, dropNode, type) {
        // 视频 54
        // 1. 被拖动的当前结点及其所在的父结点总层数不能大于3
        // 1.1 被拖动的当前结点总层数
        this.countNodeLevel(draggingNode)
        let deep = Math.abs(this.maxLevel - draggingNode.level) + 1
        if (type === 'inner') {
          return deep + dropNode.level <= 3
        } else {
          return deep + dropNode.parent.level <= 3
        }
      },
      countNodeLevel (node) {
        // 找到所有子结点，求出最大深度
        if (node.childNodes != null && node.childNodes.length > 0) {
          for (let i = 0; i < node.childNodes.length; i++) {
            if (node.childNodes[i].level > this.maxLevel) {
              this.maxLevel = node.childNodes[i].level
            }
            this.countNodeLevel(node.childNodes[i])
          }
        }
      },
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

