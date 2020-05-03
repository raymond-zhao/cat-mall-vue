<template>
  <el-tree :data="menus" :props="defaultProps" node-key="catId" ref="menuTree" @node-click="nodeClick"></el-tree>
</template>

<script>
  export default {
    data () {
      return {
        expandedKey: [],
        menus: [],
        defaultProps: {
          children: 'children',
          label: 'name'
        }
      }
    },
    methods: {
      nodeClick (data, node, component) {
        this.$emit('tree-node-click', data, node, component)
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
