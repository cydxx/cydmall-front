<template>
  <div>
    <el-switch v-model="dragFlag" active-text="开启拖拽" inactive-text="关闭拖拽"></el-switch>
    <el-button v-if="dragFlag" @click="saveBatch">批量保存</el-button>
    <el-tree
      :data="menus"
      :props="defaultProps"
      show-checkbox
      node-key="catId"
      :expand-on-click-node="false"
      :default-expanded-keys="checkedKeys"
      @node-drop="handleDrop"
      :draggable="dragFlag"
      :allow-drop="allowDrop"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button v-if="node.level <=2 " type="text" size="mini" @click="() => append(data)">新增</el-button>
          <el-button type="text" size="mini" @click="() => edit(data)">编辑</el-button>
          <el-button
            v-if="node.childNodes.length==0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >删除</el-button>
        </span>
      </span>
    </el-tree>
    <el-dialog :title="title" :visible.sync="dialogVisible" width="30%">
      <el-form :model="cataform">
        <el-form-item label="分类名称">
          <el-input v-model="cataform.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="cataform.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input v-model="cataform.productUnit" autocomplete="off"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="addOrUpdate">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
export default {
  data() {
    return {
      pCid:[],
      dragFlag: false,
      updateNodes: [],
      maxLevel: 0,
      dialogType: "",
      title: "",
      dialogVisible: false,
      cataform: {
        name: "",
        icon: "",
        productUnit: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
      },
      menus: [],
      checkedKeys: [],
      defaultProps: {
        children: "children",
        label: "name",
      },
    };
  },

  methods: {
    getMenus() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        if (data && data.code === 0) {
          console.log(data);
          this.menus = data.data;
        }
      });
    },
    append(data) {
      this.dialogType = "add";
      (this.title = "新增分类"), (this.dialogVisible = true);
      this.cataform.name = "";
      this.cataform.productUnit = "";
      this.cataform.icon = "";
      this.cataform.parentCid = data.catId;
      this.cataform.catLevel = data.catLevel * 1 + 1;
    },
    addCata() {
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.cataform, false),
      }).then(({ data }) => {
        this.$message({
          type: "sucess",
          message: "保存成功！",
        });
        this.dialogVisible = false;
        this.getMenus();
        this.checkedKeys = [this.cataform.parentCid];
      });
    },
    edit(data) {
      this.dialogType = "edit";
      (this.title = "编辑分类"), (this.dialogVisible = true);
      console.log("data---->", data);
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
      }).then(({ data }) => {
        console.log("data---->", data);
        this.cataform.catId = data.data.catId;
        this.cataform.productUnit = data.data.productUnit;
        this.cataform.icon = data.data.icon;
        this.cataform.name = data.data.name;
        this.cataform.parentCid = data.data.parentCid;
      });
    },
    addOrUpdate() {
      if (this.dialogType == "add") {
        this.addCata();
      }
      if (this.dialogType == "edit") {
        this.editCat();
      }
    },
    editCat() {
      var { name, icon, productUnit, catId } = this.cataform;
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData({ name, icon, productUnit, catId }, false),
      }).then(({ data }) => {
        this.$message({
          type: "success",
          message: "编辑成功！",
        });
        this.dialogVisible = false;
        this.getMenus();
        this.checkedKeys = [this.cataform.parentCid];
      });
    },
    remove(node, data) {
      var ids = [data.catId];
      this.$confirm(`是否删除当前数据【${data.name}】？`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false),
          }).then(({ data }) => {
            if (data && data.code === 0) {
              this.$message({
                type: "success",
                message: "删除成功！",
              });
              this.getMenus();
              this.checkedKeys = [node.parent.data.catId];
            }
          });
        })
        .catch(() => {
          this.$message({
            type: "info",
            message: "已取消删除",
          });
        });
    },
    saveBatch() {
      this.$http({
        url: this.$http.adornUrl("/product/category/update/sort"),
        method: "post",
        data: this.$http.adornData(this.updateNodes, false),
      }).then(({ data }) => {
        this.$message({
          type: "success",
          message: "更新成功！",
        });
        this.getMenus();
        this.checkedKeys = this.pCid;
        this.updateNodes = [];
        this.maxLevel = 0;
      });
    },
    allowDrop(draggingNode, dropNode, type) {
      this.countNodeLevel(draggingNode);

      let deep = this.maxLevel - draggingNode.level + 1;
      if (type == "inner") {
        return deep + dropNode.level <= 3;
      } else {
        return deep + dropNode.parent.level <= 3;
      }
    },
    countNodeLevel(node) {
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > this.maxLevel)
            this.maxLevel = node.childNodes[i].level;
        }
        countNodeLevel(node.childNodes[i]);
      }
    },
    handleDrop(draggingNode, dropNode, dropType, ev) {
      console.log("tree drop: ", dropNode.label, dropType);
      let pCid = 0;
      let sibling = null;
      if (dropType == "before" || dropType == "after") {
        pCid = dropNode.parent.catId;
        sibling = dropNode.parent.childNodes;
      } else {
        pCid = dropNode.data.catId;
        sibling = dropNode.childNodes;
      }
      for (let i = 0; i < sibling.length; i++) {
        if (sibling[i].data.catId == draggingNode.data.catId) {
          let catlevel = draggingNode.level;
          if (sibling[i].level != draggingNode.level) {
            catlevel = sibling[i].level;
            this.updateChildNodeLevel(sibling[i]);
          }
          this.updateNodes.push({
            catId: sibling[i].data.catId,
            sort: i,
            parentCid: pCid,
          });
        } else {
          this.updateNodes.push({ catId: sibling[i].data.catId, sort: i });
        }
      }
      this.pCid.push(pCid)
    },
    updateChildNodeLevel(node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          var cNode = node.childNodes[i].data;
          this.updateNodes.push({
            catId: cNode,
            catLevel: node.childNodes[i].level,
          });
          this.upupdateChildNodeLevel(node.childNodes[i]);
        }
      }
    }
  },
  created() {
    this.getMenus();
  },
};
</script>

<style>
</style>