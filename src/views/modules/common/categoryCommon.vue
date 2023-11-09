<template>
  <div>
    <el-input placeholder="输入关键字进行过滤" v-model="filterText"></el-input>
    <el-tree
      :data="menu"
      node-key="catId"
      :props="defaultProps"
      ref="menuTree"
      @node-click="treeClick"
      :filter-node-method="filterNode"
      :highlight-current="true"
    >
    </el-tree>
  </div>
</template>

<script>
export default {
  data() {
    return {
      filterText: "",
      menu: [],
      //展开菜单
      expandedKey: [],
      defaultProps: {
        children: "children",
        label: "name",
      },
    };
  },
  watch: {
    filterText(val) {
      this.$refs.menuTree.filter(val);
    },
  },
  methods: {
    filterNode(value, data) {
      if (!value) return true;
      return data.name.indexOf(value) !== -1;
    },
    getMenu() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        this.menu = data.data;
      });
    },
    treeClick(data, node, component) {
      this.$emit("tree-node-click", data, node, component);
    },
  },
  created() {
    this.getMenu();
  },
};
</script>

<style>
</style>