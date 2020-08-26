<template>
  <div>
    <!-- 面包屑导航区域 -->
    <el-breadcrumb separator="/">
      <el-breadcrumb-item :to="{ path: '/' }">首页</el-breadcrumb-item>
      <el-breadcrumb-item>权限管理</el-breadcrumb-item>
      <el-breadcrumb-item>角色列表</el-breadcrumb-item>
    </el-breadcrumb>

    <el-card>
      <el-row>
        <el-col>
          <el-button type="primary" @click="showAddRoleDialog">添加角色</el-button>
        </el-col>
      </el-row>
      <el-table :data="rolesList" stripe border>
        <!-- 展开列 -->
        <el-table-column type="expand">
          <template slot-scope="scope">
            <el-row
              v-for="(item1,l1) in scope.row.children"
              :key="item1.id"
              :class="['bdbottom', l1 === 0 ? 'bdtop':'','vcenter' ]"
            >
              <!-- 一级权限 -->
              <el-col :span="5">
                <el-tag closable @close="deleteRightsById(scope.row,item1.id)">{{item1.authName}}</el-tag>
                <i class="el-icon-caret-right"></i>
              </el-col>
              <!-- 二级和三级 -->
              <el-col :span="19">
                <el-row
                  v-for="(item2, l2) in item1.children"
                  :key="item2.id"
                  :class="[l2 === 0 ? '' : 'bdtop', 'vcenter']"
                >
                  <el-col :span="5">
                    <el-tag
                      type="success"
                      closable
                      @close="deleteRightsById(scope.row,item2.id)"
                    >{{item2.authName}}</el-tag>
                    <i class="el-icon-caret-right"></i>
                  </el-col>
                  <el-col :span="19">
                    <el-tag
                      type="warning"
                      closable
                      v-for="item3 in item2.children"
                      :key="item3.id"
                      class="vcenter"
                      @close="deleteRightsById(scope.row,item3.id)"
                    >{{item3.authName}}</el-tag>
                  </el-col>
                </el-row>
              </el-col>
            </el-row>
          </template>
        </el-table-column>
        <el-table-column label="序号" type="index"></el-table-column>
        <el-table-column label="角色名称" prop="roleName"></el-table-column>
        <el-table-column label="角色描述" prop="roleDesc"></el-table-column>
        <el-table-column label="操作">
          <template slot-scope="scope">
            <el-button
              type="primary"
              icon="el-icon-edit"
              size="mini"
              @click="showEditRoleDialog(scope.row)"
            ></el-button>
            <el-button
              type="danger"
              icon="el-icon-delete"
              size="mini"
              @click="deleteRole(scope.row)"
            ></el-button>
            <el-tooltip
              class="item"
              effect="dark"
              content="分配权限"
              placement="top"
              :enterable="false"
            >
              <el-button
                type="warning"
                icon="el-icon-setting"
                size="mini"
                @click="showSetRightsDialog(scope.row)"
              ></el-button>
            </el-tooltip>
          </template>
        </el-table-column>
      </el-table>
    </el-card>
    <!-- 分配权限区域 -->
    <el-dialog
      title="分配权限"
      :visible.sync="setRightsDialogVisible"
      width="50%"
      @close="setRightsDialogClosed"
    >
      <el-tree
        :data="rightsTree"
        show-checkbox
        node-key="id"
        :props="defaultProps"
        default-expand-all
        :default-checked-keys="defKeys"
        ref="setRightsDialogRef"
      ></el-tree>
      <span slot="footer" class="dialog-footer">
        <el-button @click="setRightsDialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="editRightsDialog">确 定</el-button>
      </span>
    </el-dialog>
    <!-- 编辑角色 -->
    <el-dialog title="修改角色" :visible.sync="editRoleDialogVisible" width="50%">
      <el-form
        ref="editRoleFormRef"
        :model="editRoleForm"
        :rules="editRoleFormRules"
        label-width="80px"
      >
        <el-form-item label="角色名称" prop="roleName">
          <el-input v-model="editRoleForm.roleName"></el-input>
        </el-form-item>
        <el-form-item label="角色描述">
          <el-input v-model="editRoleForm.roleDesc"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="editRoleDialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="editRole">确 定</el-button>
      </span>
    </el-dialog>
    <!-- 添加角色 -->
    <el-dialog title="添加角色" :visible.sync="addRoleDialogVisible" @close="addRoleDialogClose" width="50%">
      <el-form
        ref="addRoleFormRef"
        :model="addRoleForm"
        :rules="addRoleFormRules"
        label-width="80px"
      >
        <el-form-item label="角色名称" prop="roleName">
          <el-input v-model="addRoleForm.roleName"></el-input>
        </el-form-item>
        <el-form-item label="角色描述">
          <el-input v-model="addRoleForm.roleDesc" @change="addRole"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="addRoleDialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="addRole">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
export default {
  data() {
    return {
      //权限列表
      rightsTree: [],
      //角色列表
      rolesList: [],
      //控制分配权限dialog显示与关闭
      setRightsDialogVisible: false,
      //树形组件数据展示
      defaultProps: {
        children: 'children',
        label: 'authName',
      },
      //默认选择的树形节点
      defKeys: [],
      //当前即将分配权限角色的id
      roleId: '',
      //控制编辑角色dialog显示与关闭
      editRoleDialogVisible: false,
      //修改角色表单提交数据
      editRoleForm: {},
      //修改角色表单提交数据验证规则
      editRoleFormRules: {
        roleName: [
          { required: true, message: '请输入角色名称', trigger: 'blur' },
        ],
      },
      //控制添加角色dialog显示与关闭
      addRoleDialogVisible: false,
      //添加角色表单提交
      addRoleForm: {},
      //添加角色数据验证规则
      addRoleFormRules: {
        roleName: [
          { required: true, message: '请输入角色名称', trigger: 'blur' },
        ],
      },
    }
  },
  created() {
    this.getRolesList()
  },
  methods: {
    //获取角色列表
    async getRolesList() {
      const { data: res } = await this.$http.get('roles')
      if (res.meta.status !== 200)
        return this.$message.error('获取角色列表失败 !')
      this.rolesList = res.data
      this.$message.success('获取角色列表成功 !')
    },
    //删除角色下指定权限
    async deleteRightsById(role, rightId) {
      const confirmResult = await this.$confirm(
        '此操作将永久删除该权限, 是否继续?',
        '提示',
        {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning',
        }
      ).catch((error) => error)
      if (confirmResult !== 'confirm') return this.$message.info('取消了删除 !')
      const { data: res } = await this.$http.delete(
        `roles/${role.id}/rights/${rightId}`
      )
      if (res.meta.status !== 200) return this.$message.error('删除权限失败 !')
      role.children = res.data
      this.$message.success('成功删除权限 !')
    },
    //显示分配权限dialog
    async showSetRightsDialog(role) {
      this.roleId = role.id
      const { data: res } = await this.$http.get('rights/tree')
      if (res.meta.status !== 200)
        return this.$message.error('获取权限列表失败')
      this.rightsTree = res.data
      this.getLeafKeys(role, this.defKeys)
      this.$message.success('获取权限列表成功 !')
      this.setRightsDialogVisible = true
    },
    //分配权限dialog窗口关闭清空defKeys
    setRightsDialogClosed() {
      this.defKeys = []
    },
    //函数递归调用
    getLeafKeys(node, arr) {
      if (!node.children) {
        return arr.push(node.id)
      }
      node.children.forEach((item) => this.getLeafKeys(item, arr))
    },
    //分配权限
    async editRightsDialog() {
      const keys = [
        ...this.$refs.setRightsDialogRef.getCheckedKeys(),
        ...this.$refs.setRightsDialogRef.getHalfCheckedKeys(),
      ]
      const idStr = keys.join(',')
      const { data: res } = await this.$http.post(
        `roles/${this.roleId}/rights`,
        { rids: idStr }
      )
      if (res.meta.status !== 200) return this.$message.error('分配权限失败 !')
      this.$message.success('分配权限成功 !')
      this.getRolesList()
      this.setRightsDialogVisible = false
    },
    //显示修改角色数据回显
    async showEditRoleDialog(role) {
      this.roleId = role.id
      const { data: res } = await this.$http.get(`roles/${this.roleId}`)
      if (res.meta.status !== 200)
        return this.$message.error('获取角色信息失败')
      this.editRoleForm = res.data
      this.editRoleDialogVisible = true
    },
    //修改角色
    editRole() {
      this.$refs.editRoleFormRef.validate(async (valid) => {
        if (!valid) return
        const { data: res } = await this.$http.put(
          `roles/${this.roleId}`,
          this.editRoleForm
        )
        if (res.meta.status != 200) return this.$message.error('修改角色失败!')
        this.$message.success('修改角色成功!')
        this.getRolesList()
        this.editRoleDialogVisible = false
      })
    },
    //删除角色
    async deleteRole(role) {
      const confirmResullt = await this.$confirm(
        '此操作将永久删除该角色, 是否继续?',
        '提示',
        {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning',
        }
      ).catch((error) => error)
      if (confirmResullt !== 'confirm')
        return this.$message.info('已取消了删除 !')
      const { data: res } = await this.$http.delete(`roles/${role.id}`)
      if (res.meta.status !== 200) return this.$message.error('删除失败!')
      this.getRolesList()
      this.$message.success('删除成功!')
    },
    //显示添加角色dialog
    showAddRoleDialog() {
      this.addRoleDialogVisible = true
    },
    //添加角色
    addRole() {
      this.$refs.addRoleFormRef.validate(async (valid) => {
        if (!valid) return
        const { data: res } = await this.$http.post('roles', this.addRoleForm)
        if (res.meta.status !== 201) return this.$message.error('添加角色失败!')
        this.getRolesList()
        this.addRoleDialogVisible = false
        this.$message.success('添加角色成功')
      })
    },
    //添加角色dialog关闭清除addRoleForm
    addRoleDialogClose(){
      this.addRoleForm = {}
    }
  },
}
</script>

<style lang="less" scope>
.el-tag {
  margin: 7px;
}
.bdtop {
  border-top: 1px solid #eee;
}
.bdbottom {
  border-bottom: 1px solid #eee;
}
.vcenter {
  display: flex;
  align-items: center;
}
</style>