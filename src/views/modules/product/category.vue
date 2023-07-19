<template>
  <div>
    <el-switch
      v-model="draggable"
      active-text="开启拖拽"
      inactive-text="关闭拖拽"
    >
    </el-switch>
    <el-button
      type="danger"
      icon="el-icon-delete"
      circle
      @click="batchDelete"
    ></el-button>
    <el-tree
      :data="data"
      :props="defaultProps"
      @node-click="handleNodeClick"
      @node-drop="handleDrop"
      :expand-on-click-node="false"
      :draggable="draggable"
      show-checkbox
      node-key="catId"
      :default-expanded-keys="expandedKey"
      :allow-drop="allowDrop"
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
            Update
          </el-button>
        </span>
      </span>
    </el-tree>

    <el-dialog
      title="新增分类"
      :visible.sync="addDialogVisible"
      width="30%"
      :close-on-click-modal="false"
    >
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="分类图标">
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
        <el-button @click="addDialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="addCategory">新 增</el-button>
      </span>
    </el-dialog>

    <el-dialog
      title="修改分类"
      :visible.sync="editDialogVisible"
      width="30%"
      :close-on-click-modal="false"
    >
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="分类图标">
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
        <el-button @click="editDialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="editCategory">修 改</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
export default {
  data() {
    return {
      data: [],
      defaultProps: {
        children: "children",
        label: "name",
      },
      category: {
        catId: null,
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        icon: "",
        productUnit: "",
      },
      expandedKey: [],
      // checkedNodeKeys: []
      addDialogVisible: false,
      editDialogVisible: false,
      maxLevel: 0,
      updateNodes: [],
      draggable: false,
    };
  },
  methods: {
    getMenu() {
      this.dataListLoading = true;
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        this.data = data.data;
        this.dataListLoading = false;
      });
    },
    handleNodeClick(data) {
      // console.log(data)
      // console.log(this.$refs.tree.getCheckedKeys());
      // this.checkedNodeKeys.push(data.catId);
    },
    editCategory() {
      var { catId, name, icon, productUnit } = this.category;
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData({ catId, name, icon, productUnit }, false),
      }).then(({ data }) => {
        this.editDialogVisible = false;
        if (data && data.code === 0) {
          this.$message({
            message: "修改成功",
            type: "success",
            duration: 1500,
            onClose: () => {
              this.getMenu();
              this.expandedKey = [this.category.parentCid];
            },
          });
        } else {
          this.$message.error(data.msg);
        }
      });
    },
    addCategory() {
      // console.log(this.category);
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false),
      }).then(({ data }) => {
        this.addDialogVisible = false;
        if (data && data.code === 0) {
          this.$message({
            message: "保存成功",
            type: "success",
            duration: 1500,
            onClose: () => {
              this.getMenu();
              this.expandedKey = [this.category.parentCid];
            },
          });
        } else {
          this.$message.error(data.msg);
        }
      });
    },
    append(data) {
      // console.log(data);
      this.addDialogVisible = true;
      this.category.catId = null;
      this.category.parentCid = data.catId;
      this.category.name = "";
      this.category.catLevel = data.catLevel * 1 + 1;
      this.category.icon = "";
      this.category.productUnit = "";
    },
    edit(data) {
      // console.log("edit:", data);
      this.editDialogVisible = true;
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
      }).then(({ data }) => {
        this.category.catId = data.data.catId;
        this.category.parentCid = data.data.parentCid;
        this.category.name = data.data.name;
        this.category.catLevel = data.data.catLevel;
        this.category.icon = data.data.icon;
        this.category.productUnit = data.data.productUnit;
      });
    },
    allowDrop(draggingNode, dropNode, type) {
      // console.log(draggingNode, dropNode, type);
      // 判断拖动后的结构层数不超过 3
      this.maxLevel = 0;
      this.countNodeLevel(draggingNode.data);
      let deep = this.maxLevel - dropNode.data.catLevel + 1;
      if (type == "inner") {
        return deep + dropNode.level <= 3;
      } else {
        return deep + dropNode.parent.level <= 3;
      }
    },
    countNodeLevel(node) {
      if (node.children != null && node.children.length > 0) {
        for (let i = 0; i < node.children.length; i++) {
          if (node.children[i].catLevel > this.maxLevel) {
            this.maxLevel = node.children[i].catLevel;
          }
          this.countNodeLevel(node.children[i]);
        }
      }
    },
    handleDrop(draggingNode, dropNode, dropType, ev) {
      // console.log("tree drop: ", dropNode.label, dropType);
      // 抽取 被拖拽节点的最新父节点 id， 以及当前所在节点（已包含本身）的最新排序，以及此节点及其子节点当前所在层级
      let newParentId = 0;
      let siblings = null;
      if (dropType == "before" || dropType == "after") {
        newParentId =
          dropNode.parent.data.catId == undefined
            ? 0
            : dropNode.parent.data.catId;
        siblings = dropNode.parent.childNodes;
      } else {
        newParentId = dropNode.data.catId;
        siblings = dropNode.childNodes;
      }

      for (let i = 0; i < siblings.length; i++) {
        // 遍历到被拖拽节点时要判断是否有父子关系的变化
        if (siblings[i].data.catId == draggingNode.data.catId) {
          // 判断节点层级变化
          let catLevel = draggingNode.level;
          if (siblings[i].level != draggingNode.level) {
            catLevel = siblings[i].level;
            // 修改当前遍历节点子节点的层级
            this.updateChildNodeLevel(siblings[i]);
          }
          this.updateNodes.push({
            catId: siblings[i].data.catId,
            sort: i,
            parentCid: newParentId,
            catLevel: catLevel,
          });
        } else {
          this.updateNodes.push({ catId: siblings[i].data.catId, sort: i });
        }
      }

      // console.log(this.updateNodes);
      this.$http({
        url: this.$http.adornUrl(`/product/category/update/updateByArray`),
        method: "post",
        data: this.$http.adornData(this.updateNodes, false),
      }).then(({ data }) => {
        this.$message({
          message: "菜单更新成功",
          type: "success",
        });
        this.getMenu();
        this.expandedKey = [newParentId];
        this.updateNodes.length = 0;
      });
    },
    updateChildNodeLevel(node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          var currentNode = node.childNodes[i].data;
          this.updateNodes.push({
            catId: currentNode.catId,
            catLevel: node.childNodes[i].level,
          });
          this.updateChildNodeLevel(node.childNodes[i]);
        }
      }
    },
    batchDelete() {
      let checkedNodesKeys = this.$refs.menuTree.getCheckedKeys(false, false);
      // console.log(checkedNodes)
      this.$confirm(
        `确定对[id=${ids.join(",")}]进行[${
          ids.length == 1 ? "删除" : "批量删除"
        }]操作?`,
        "提示",
        {
          confirmButtonText: "确定",
          cancelButtonText: "取消",
          type: "warning",
        }
      )
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(checkedNodesKeys, false),
          }).then(({ data }) => {
            if (data && data.code === 0) {
              this.$message({
                message: "删除成功",
                type: "success",
                duration: 1500,
                onClose: () => {
                  this.getMenu();
                  this.expandedKey = [];
                },
              });
            } else {
              this.$message.error(data.msg);
            }
          });
        })
        .catch(() => {});
    },
  },
  created() {
    this.getMenu();
  },
};
</script>