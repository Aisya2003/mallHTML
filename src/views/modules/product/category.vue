<template>
  <div>
    <el-switch
      v-model="draggable"
      active-text="开启拖拽"
      inactive-text="关闭拖拽"
    ></el-switch>
    <el-button v-if="draggable" @click="saveDrag()">保存</el-button>
    <el-button @click="deleteSelected()" type="danger">批量删除</el-button>
    <el-tree
      show-checkbox
      node-key="catId"
      :data="menu"
      :props="defaultProps"
      :expand-on-click-node="false"
      :default-expanded-keys="expandedKey"
      :draggable="draggable"
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
      ref="menuTree"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="node.level <= 2"
            type="text"
            size="mini"
            @click="() => append(data)"
          >
            Append
          </el-button>
          <el-button type="text" size="mini" @click="() => edit(data)">
            Edit
          </el-button>
          <el-button
            v-if="node.childNodes.length == 0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >
            Delete
          </el-button>
        </span>
      </span>
    </el-tree>
    <el-dialog
      :title="title"
      :visible.sync="dialogVisible"
      width="30%"
      :close-on-click-modal="false"
    >
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input
            v-model="category.productUnit"
            autocomplete="off"
          ></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="submitEvent()">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>
<script>
export default {
  data() {
    return {
      //拖拽后的展开信息
      parentId: [],
      //开启拖拽
      draggable: false,
      //拖拽之后更新的节点ID
      updateNodes: [],
      //最大深度
      maxLevel: 0,
      //对话框提示信息
      title: "",
      //分类信息实体
      category: {
        catId: null,
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        productUnit: "",
        icon: "",
      },
      //对话框
      dialogVisible: false,
      //点击事件类别
      dialogType: "",
      //分类数据
      menu: [],
      //展开菜单
      expandedKey: [],
      defaultProps: {
        children: "children",
        label: "name",
      },
    };
  },
  methods: {
    /**
     * 批量删除选中
     */
    deleteSelected() {
      let catIds = [];
      let catNames = [];
      let selected = this.$refs.menuTree.getCheckedNodes();
      for (let i = 0; i < selected.length; i++) {
        catIds.push(selected[i].catId);
        catNames.push(selected[i].name)
      }
      this.$confirm(`是否批量删除分类${catNames}?`, "提示", {
        confirmButtonText: "确认",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          if(catIds == null || catIds.length <=0 ){
            this.$message({
              message: "没有选中任何分类！",
              type: "warning",
            });
            return;
          }
          this.$http({
            url: this.$http.adornUrl("/product/category/deleteBatch"),
            method: "post",
            data: this.$http.adornData(catIds, false),
          }).then(({ data }) => {
            this.$message({
              message: "删除成功！",
              type: "success",
            });
             this.getMenu();
          });
        })
        .catch(() => {});
    },
    /**
     * 保存拖拽信息
     */
    saveDrag() {
      this.$http({
        url: this.$http.adornUrl("/product/category/updateBatch"),
        method: "post",
        data: this.$http.adornData(this.updateNodes, false),
      }).then(({ data }) => {
        this.$message({
          message: "修改成功！",
          type: "success",
        });
      });
      this.getMenu();
      this.expandedKey = this.parentId;
      this.updateNodes = [];
      this.maxLevel = 0;
      // this.parentId = [];
      this.draggable = false;
    },
    /**
     * 拖拽成功事件
     */
    handleDrop(draggingNode, dropNode, dropType, ev) {
      let parentId = 0;
      let sibling = null;
      if (dropType == "before" || dropType == "after") {
        parentId =
          dropNode.parent.data.catId == undefined
            ? 0
            : dropNode.parent.data.catId;
        sibling = dropNode.parent.childNodes;
      } else {
        parentId = dropNode.data.catId;
        sibling = dropNode.childNodes;
      }
      this.parentId.push(parentId);

      for (let i = 0; i < sibling.length; i++) {
        if (sibling[i].data.catId == draggingNode.data.catId) {
          let level = draggingNode.level;
          if (sibling[i].level != draggingNode.level) {
            level = sibling[i].level;
            this.updateLevel(sibling[i]);
          }

          this.updateNodes.push({
            catId: sibling[i].data.catId,
            sort: i,
            parentCid: parentId,
            catLevel: level,
          });
        } else {
          this.updateNodes.push({ catId: sibling[i].data.catId, sort: i });
        }
      }
    },
    /**
     * 修改节点层次
     */
    updateLevel(node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          let child = node.childNodes[i].data;
          this.updateNodes.push({
            catId: child.catId,
            catLevel: node.childNodes[i].level,
          });
          this.updateLevel(node.childNodes[i]);
        }
      }
    },
    /**
     * 拖拽事件
     */
    allowDrop(draggingNode, dropNode, type) {
      this.maxLevel = 0;
      this.getLevel(draggingNode);
      let currentLevel = Math.abs(this.maxLevel - draggingNode.level + 1);
      if (type == "inner") {
        return currentLevel + dropNode.level <= 3;
      } else {
        return currentLevel + dropNode.parent.level <= 3;
      }
    },
    /**
     * 获取节点的最大分类层数
     */
    getLevel(node) {
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level;
          }
          this.getLevel(node.childNodes[i]);
        }
      }
    },
    /**
     * 修改分类信息
     */
    edit(data) {
      this.title = "修改信息";
      this.dialogType = "edit";
      this.dialogVisible = true;

      //获取最新数据
      this.$http({
        url: this.$http.adornUrl(`/product/category/${data.catId}`),
        method: "get",
        params: this.$http.adornParams({}),
      }).then(({ data }) => {
        this.category.name = data.data.name;
        this.category.catId = data.data.catId;
        this.category.productUnit = data.data.productUnit;
        this.category.icon = data.data.icon;
        this.category.parentCid = data.data.parentCid;
      });
    },

    /**
     * 获取分类
     */
    getMenu() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        this.menu = data.data;
      });
    },
    /**
     * 添加分类信息
     */
    append(data) {
      this.title = "添加信息";
      this.dialogType = "append";
      this.dialogVisible = true;
      this.category.catId = null;
      this.category.name = "";
      this.category.parentCid = data.catId;
      this.category.catLevel = data.catLevel * 1 + 1;
      this.category.productUnit = "";
      this.category.icon = "";
    },
    /**
     * 删除分类
     */
    remove(node, data) {
      var ids = [data.catId];
      this.$confirm(`是否删除【${data.name}】菜单`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      }).then(() => {
        this.$http({
          url: this.$http.adornUrl("/product/category/deleteBatch"),
          method: "post",
          data: this.$http.adornData(ids, false),
        }).then(({ data }) => {
          this.getMenu();
          this.$message({
            message: `删除成功！`,
            type: "success",
          });
          this.expandedKey = [node.parent.data.catId];
        });
      });
    },
    /**
     * 添加分类至数据库
     */
    appendCategory() {
      this.$http({
        url: this.$http.adornUrl("/product/category/append"),
        method: "post",
        data: this.$http.adornData(this.category, false),
      }).then(({ data }) => {
        this.$message({
          message: `保存成功！`,
          type: "success",
        });
      });
      this.dialogVisible = false;
      this.getMenu();
      this.expandedKey = [this.category.parentCid];
      this.category.name = "";
    },
    /**
     * 修改分类信息
     */
    editCategory() {
      var { catId, name, icon, productUnit } = this.category;
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData({ catId, name, icon, productUnit }, false),
      }).then(({ data }) => {
        this.$message({
          message: "修改成功!",
          type: "success",
        });
        this.getMenu();
        this.dialogVisible = false;
        this.dialogType = "";
        this.expandedKey = [this.category.parentCid];
      });
    },
    /**
     * 按下确认触发的事件
     */
    submitEvent() {
      if (this.dialogType == "append") {
        this.appendCategory();
      } else if (this.dialogType == "edit") {
        this.editCategory();
      }
    },
  },
  /**
   * 页面加载完成，查询分类
   */
  created() {
    this.getMenu();
  },
};
</script>
<style>
.custom-tree-node {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: space-between;
  font-size: 14px;
  padding-right: 8px;
}
</style>