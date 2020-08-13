<template>
  <div>
    <el-tree
      :data="menus"
      :props="defaultProps"
      show-checkbox
      node-key="catId"
      :expand-on-click-node="false"
      :default-expanded-keys="checkedKeys"
      draggable
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
    allowDrop(draggingNode, dropNode, type) {
      this.countNodeLevel(draggingNode.data);

      let deep = (this.maxLevel - draggingNode.data.catLevel)+1
      if(type == 'inner'){
        return (deep + dropNode.level) <= 3
      }else{
         return (deep + dropNode.parent.level) <= 3
      }

    },
    countNodeLevel(node) {
      if (node.children != null && node.children.length > 0) {
        for (let i = 0; i < node.children.length; i++) {
          if (node.children[i].catLevel > this.maxLevel)
            this.maxLevel = node.children.catLevel;
        }
        countNodeLevel(node.children[i]);
      }
    },
  },
  created() {
    this.getMenus();
  },
};
</script>

<style>
</style>