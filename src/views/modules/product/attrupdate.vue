<template>
  <div>
    <el-row :gutter="20">
      <el-col :span="16">
        <el-card class="box-card">
          <el-tabs tab-position="left" style="width: 100%">
            <el-tab-pane
              v-for="(group, gidx) in dataResp.attrGroups"
              :label="group.attrGroupName"
              :key="group.attrGroupId"
            >
              <!-- 遍历属性,每个tab-pane对应一个表单，每个属性是一个表单项  spu.baseAttrs[0] = [{attrId:xx,val:}]-->
              <el-form ref="form" :model="dataResp">
                <el-form-item v-for="(attr, aidx) in group.attrs" :label="attr.attrName" :key="attr.attrId">
                  <el-input v-model="dataResp.baseAttrs[gidx][aidx].attrId" type="hidden" v-show="false"></el-input>
                  <el-select
                    v-model="dataResp.baseAttrs[gidx][aidx].attrValues"
                    :multiple="attr.valueType == 1"
                    filterable
                    allow-create
                    default-first-option
                    placeholder="请选择或输入值"
                  >
                    <el-option
                      v-for="(val, vidx) in attr.valueSelect.split(';')"
                      :key="vidx"
                      :label="val"
                      :value="val"
                    ></el-option>
                  </el-select>
                  <el-checkbox
                    v-model="dataResp.baseAttrs[gidx][aidx].showDesc"
                    :true-label="1"
                    :false-label="0"
                    >快速展示</el-checkbox
                  >
                </el-form-item>
              </el-form>
            </el-tab-pane>
          </el-tabs>
          <el-button type="success" style="float: right" @click="submitSpuAttrs">确认修改</el-button>
        </el-card>
      </el-col>
    </el-row>
  </div>
</template>

<script>
export default {
  components: {},
  props: {},
  data() {
    return {
      spuId: "",
      catalogId: "",
      dataResp: {
        // 后台返回的所有数据
        attrGroups: [],
        baseAttrs: [],
      },
      spuAttrsMap: {}
    };
  },
  computed: {},
  methods: {
    clearData() {
      this.dataResp.attrGroups = [];
      this.dataResp.baseAttrs = [];
      this.spuAttrsMap = {};
    },

    // 从路由中获取参数
    getQueryParams() {
      this.spuId = this.$route.query.spuId;
      this.catalogId = this.$route.query.catalogId;
    },

    // 获取 SPU 的基础属性信息
    getSpuBaseAttrs() {
      this.$http({
        url: this.$http.adornUrl(`/product/attr/base/listforspu/${this.spuId}`),
        method: "get",
      }).then(({ data }) => {
        data.data.forEach((item) => {
          this.spuAttrsMap["" + item.attrId] = item;
        })
      });
    },

    showBaseAttrs() {
      let _this = this;
      this.$http({
        url: this.$http.adornUrl(
          `/product/attrgroup/${this.catalogId}/withattr`
        ),
        method: "get",
        params: this.$http.adornParams({}),
      }).then(({data}) => {
        // 先对表单的 baseAttrs 进行初始化
        data.data.forEach((item) => {
          let attrArray = [];
          item.attrs.forEach((attr) => {
            let v = "";
            if (_this.spuAttrsMap["" + attr.attrId]) {
              v = _this.spuAttrsMap["" + attr.attrId].attrValue.split(";"); // 字符串数组
              if (v.length === 1) { // 如果数组只包含单个元素，则将其中唯一的那个元素转为字符串。
                v = v[0] + "";
              }
            }
            attrArray.push({
              attrId: attr.attrId,
              attrName: attr.attrName,
              attrValues: v,
              showDesc: _this.spuAttrsMap["" + attr.attrId]
                ? _this.spuAttrsMap["" + attr.attrId].quickShow
                : attr.showDesc
            });
          });
          this.dataResp.baseAttrs.push(attrArray);
        });
        this.dataResp.attrGroups = data.data;
      });
    },
    // 确认修改按钮 点击事件
    submitSpuAttrs() {
      let submitData = [];
      this.dataResp.baseAttrs.forEach((item) => {
        item.forEach((attr) => {
          let val = "";
          console.log(attr.attrValues);
          if (Array.isArray(attr.attrValues)) {
            val = attr.attrValues.join(";");
          } else {
            val = attr.attrValues;
          }
          if (val !== "") {
            submitData.push({
              attrId: attr.attrId,
              attrName: attr.attrName,
              attrValue: val,
              quickShow: attr.showDesc,
            });
          }
        });
      });
      this.$confirm("修改商品规格信息, 是否继续?", "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl(`/product/attr/update/${this.spuId}`),
            method: "post",
            data: this.$http.adornData(submitData, false),
          }).then(({ data }) => {
            this.$message({
              type: "success",
              message: "属性修改成功!",
            });
          });
        })
        .catch((e) => {
          this.$message({
            type: "info",
            message: "已取消修改 " + e,
          });
        });
    },
  },
  activated() {
    this.clearData();
    this.getQueryParams();
    if (this.spuId && this.catalogId) {
      this.getSpuBaseAttrs();
      this.showBaseAttrs();
    }
  }
};
</script>
<style scoped>
</style>