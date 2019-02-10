<template>
    <div>
  <my-breadcrumb
    level2="权限管理"
    level3="角色列表"></my-breadcrumb>
  <el-row>
    <el-col :span="24">
      <el-button @click="addRoleFormVisible = true;">添加角色</el-button>
    </el-col>
  </el-row>
  <!-- 角色列表 -->
  <el-table
    v-loading="loading"
    :data="tableData"
    :border="true"
    style="width: 100%">
    <el-table-column type="expand">
      <template slot-scope="scope">
        <!-- 一级权限 -->
        <el-row class="level1-row" v-for="level1 in scope.row.children" :key="level1.id" >
          <el-col :span="4">
            <el-tag
              closable
              @close="handleDeleteRights(scope.row, level1)"
              type="">
              {{ level1.authName }}
            </el-tag>
            <i class="el-icon-arrow-right"></i>
          </el-col>
          <el-col :span="20">
            <!-- 二级权限 -->
            <el-row class="level2-row" v-for="level2 in level1.children" :key="level2.id">
              <el-col :span="5">
                <el-tag
                  closable
                  @close="handleDeleteRights(scope.row, level2)"
                  type="success">
                  {{ level2.authName }}
                </el-tag>
                <i class="el-icon-arrow-right"></i>
              </el-col>
              <el-col :span="19">
                <!-- 三级菜单 -->
                <el-tag
                  class="level3-row"
                  v-for="level3 in level2.children"
                  :key="level3.id"
                  @close="handleDeleteRights(scope.row, level3)"
                  closable
                  type="warning">
                  {{ level3.authName }}
                </el-tag>
              </el-col>
            </el-row>
          </el-col>
        </el-row>
        <el-row v-if="scope.row.children.length === 0">
          <el-col :span="24">
            没有分配权限
          </el-col>
        </el-row>
      </template>
    </el-table-column>
    <el-table-column
      type="index">
    </el-table-column>
    <el-table-column
      label="角色名称"
      prop="roleName">
    </el-table-column>
    <el-table-column
      label="角色描述"
      prop="roleDesc">
    </el-table-column>
    <el-table-column
      label="操作">
      <template slot-scope="scope">
          <el-button type="primary" 
          size="mini" 
          icon="el-icon-edit"
          @click="showEditRoleDialog(scope.row)"
          plain>
        </el-button>
        <el-button type="danger"
          size="mini"
          @click="handleDelete(scope.row)"
          icon="el-icon-delete"
          plain>
        </el-button>
        <el-button type="warning"
          size="mini"
          @click="showRightsDialog(scope.row)"
          icon="el-icon-check"
          plain>
        </el-button>
      </template>
    </el-table-column>
  </el-table>
  <!-- 添加角色的弹出层 -->
  <el-dialog title="添加角色" :visible.sync="addRoleFormVisible">
    <el-form
      ref="addRoleForm"
      :rules="rules"
      :model="addRoleFormData"
      label-position="right"
      label-width="100px">
      <el-form-item prop="roleName" label="角色名称">
        <el-input v-model="addRoleFormData.roleName" auto-complete="off"></el-input>
      </el-form-item>
      <el-form-item prop="roleDesc" label="角色描述">
        <el-input v-model="addRoleFormData.roleDesc" auto-complete="off"></el-input>
      </el-form-item>
    </el-form>
    <div slot="footer" class="dialog-footer">
      <el-button @click="addRoleFormVisible = false">取 消</el-button>
      <el-button type="primary" @click="handleAddRole">确 定</el-button>
    </div>
  </el-dialog>
  <!-- 修改角色的弹出层 -->
  <el-dialog title="编辑角色" :visible.sync="editRoleFormVisible">
    <el-form
      ref="editRoleForm"
      :rules="rules"
      :model="addRoleFormData"
      label-position="right"
      label-width="100px">
      <el-form-item prop="roleName" label="角色名称">
        <el-input v-model="addRoleFormData.roleName" auto-complete="off"></el-input>
      </el-form-item>
      <el-form-item prop="roleDesc" label="角色描述">
        <el-input v-model="addRoleFormData.roleDesc" auto-complete="off"></el-input>
      </el-form-item>
    </el-form>
    <div slot="footer" class="dialog-footer">
      <el-button @click="editRoleFormVisible = false">取 消</el-button>
      <el-button type="primary" @click="handleEidtRole">确 定</el-button>
    </div>
  </el-dialog>
  <!-- 权限分配 -->
  <el-dialog title="权限分配" :visible.sync="editRightsDialog">
    <div>
      <el-tree
        :data="rightsData"
        ref="tree"
        default-expand-all
        :default-checked-keys="checkedKeys"
        show-checkbox
        node-key="id"
        :props="defaultProps">
      </el-tree>
    </div>
    <div slot="footer" class="dialog-footer">
      <el-button @click="editRightsDialog = false">取 消</el-button>
      <el-button type="primary" @click="handleRights">确 定</el-button>
    </div>
  </el-dialog>
</div>
</template>

<script>
export default {
  data() {
    return {
      tableData: [],
      loading: true,
      addRoleFormVisible: false,
      editRoleFormVisible: false,
      addRoleFormData: {
        roleName: '',
        roleDesc: ''
      },
      rules: {
        roleName: [
          { required: true, message: '请输入角色名称', trigger: 'blur' }
        ],
        roleDesc: [
          { required: true, message: '请输入角色描述', trigger: 'blur' }
        ]
      },
      // 控制权限分配的对话框的显示隐藏
      editRightsDialog: false,
      rightsData: [],
      defaultProps: {
        children: 'children',
        label: 'authName'
      },
      // 默认选中的权限id
      checkedKeys: [],
      // 点击授权的时候记录下来当前的角色
      currentRole: {}
    };
  },
  mounted() {
    this.loadData();
  },
  methods: {
    // 加载表格数据
    async loadData() {
      const res = await this.$http.get('/roles');
      if (res.data.meta.status === 200) {
        this.tableData = res.data.data;
        this.loading = false;
      }
    },
    // 添加角色
    async handleAddRole() {
      // 表单验证
      this.$refs.addRoleForm.validate(async (valid) => {
        if (!valid) {
          // 表单验证失败，返回
          return;
        }
        // 表单验证成功，添加角色
        this.addRoleFormVisible = false;
        const res = await this.$http.post('/roles', this.addRoleFormData);
        if (res.data.meta.status === 201) {
          this.$message({
            type: 'success',
            message: '创建角色成功'
          });
          // 重新加载数据
          this.loadData();
        } else {
          this.$message({
            type: 'erroe',
            message: '创建角色失败'
          });
        }
      });
    },
    // 删除角色
    async handleDelete(role) {
      // 删除提示
      this.$confirm('确认删除该角色？', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(async () => {
        // 删除操作
        const { id: roleId } = role;
        const res = await this.$http.delete(`/roles/${roleId}`);
        if (res.data.meta.status === 200) {
          this.$message({
            type: 'success',
            message: '删除角色成功'
          });
          this.loadData();
        } else {
          this.$message({
            type: 'error',
            message: '删除角色失败'
          });
        }
      }).catch(() => {
        this.$message({
          type: 'info',
          message: '已取消删除'
        });
      });
    },
    async showEditRoleDialog(role) {
      // 显示编辑角色的对话框
      this.editRoleFormVisible = true;
      const { id: roleId } = role;
      const res = await this.$http.get(`/roles/${roleId}`);
      this.addRoleFormData = res.data.data;

      this.addOrEdit = 'edit';
    },
    // 编辑角色
    async handleEidtRole() {
      const { roleId } = this.addRoleFormData;
      const res = await this.$http.put(`/roles/${roleId}`, this.addRoleFormData);
      this.editRoleFormVisible = false;
      if (res.data.meta.status === 200) {
        this.$message({
          type: 'success',
          message: '编辑用户成功'
        });
        this.loadData();
      } else {
        this.$message({
          type: 'error',
          message: '编辑用户错误'
        });
      }
    },
    // 显示权限分配的对话框
    async showRightsDialog(role) {
      // 获取当前角色具有的权限id
      function getLevel3Ids(rightsList) {
        const arr = [];
        const fn = function (list) {
          list.forEach((item) => {
            if (!item.children) {
              arr.push(item.id);
            } else {
              fn(item.children);
            }
          });
        };
        fn(rightsList);
        return arr;
      }
      // 记录当前的角色
      this.currentRole = role;
      this.editRightsDialog = true;
      const res = await this.$http.get('/rights/tree');
      this.rightsData = res.data.data;
      // 设置选中的权限
      this.checkedKeys = getLevel3Ids(role.children);
    },
    // 分配权限
    async handleRights() {
      // 获取所有选中的权限id
      const nodes = this.$refs.tree.getCheckedNodes();
      let arr = [];
      nodes.forEach((item) => {
        // 选中的子权限id
        arr.push(item.id.toString());

        // 子权限的id 对应的父权限的id
        if (typeof (item.pid) === 'number') {
          arr.push(item.pid.toString());
        } else {
          arr = arr.concat(item.pid.split(','));
        }
      });

      // 数组去重
      const set = new Set(arr);

      const ids = [...set].join(',');

      const res = await this.$http.post(`/roles/${this.currentRole.id}/rights`, {
        rids: ids
      });
      if (res.data.meta.status === 200) {
        this.editRightsDialog = false;
        this.$message({
          type: 'success',
          message: '权限分配成功'
        });
        this.loadData();
      } else {
        this.$message({
          type: 'error',
          message: '权限分配失败'
        });
      }
    },
    // 删除权限
    async handleDeleteRights(role, rights) {
      // role 权限  rights 角色
      const res = await this.$http.delete(`/roles/${role.id}/rights/${rights.id}`);
      if (res.data.meta.status === 200) {
        this.$message({
          type: 'success',
          message: '删除权限成功'
        });
        // 重新绑定数据
        role.children = res.data.data;
      } else {
        this.$message({
          type: 'error',
          message: '删除权限失败'
        });
      }
    }

  }
};

</script>

<style>
.level1-row {
  margin-bottom: 10px;
}
/* .level2-row {
  margin-bottom: 5px;
} */
.level3-row {
  margin-right: 5px;
  margin-bottom: 5px;
}

</style>
