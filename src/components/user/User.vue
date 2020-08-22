<template>
  <div>
    <!-- 面包屑导航区域 -->
    <el-breadcrumb separator="/">
      <el-breadcrumb-item :to="{ path: '/home' }">首页</el-breadcrumb-item>
      <el-breadcrumb-item>用户管理</el-breadcrumb-item>
      <el-breadcrumb-item>用户列表</el-breadcrumb-item>
    </el-breadcrumb>
    <!-- 卡片视图区域 -->
    <el-card>
      <el-row :gutter="20">
        <el-col :span="7">
          <el-input v-model="queryInfo.query" placeholder="请输入查询内容" @change="getUserList" clearable>
            <el-button slot="append" icon="el-icon-search" @click="getUserList"></el-button>
          </el-input>
        </el-col>
        <el-col :span="4">
          <el-button type="primary" @click="addDialogVisible = true">添加用户</el-button>
        </el-col>
      </el-row>
    </el-card>
    <!-- 表格区域 -->
    <el-table :data="userList" stripe border>
      <el-table-column type="index" label="序号"></el-table-column>
      <el-table-column prop="username" label="姓名"></el-table-column>
      <el-table-column prop="email" label="邮箱"></el-table-column>
      <el-table-column prop="mobile" label="电话"></el-table-column>
      <el-table-column prop="role_name" label="角色"></el-table-column>
      <el-table-column label="状态">
        <template slot-scope="scope">
          <el-switch
            v-model="scope.row.mg_state"
            active-color="#13ce66"
            inactive-color="#ff4949"
            @change="userStateChanged(scope.row)"
          ></el-switch>
        </template>
      </el-table-column>
      <el-table-column label="操作">
        <template slot-scope="scope">
          <!-- 修改 -->
          <el-button
            type="primary"
            icon="el-icon-edit"
            size="mini"
            @click="showEditDialog(scope.row.id)"
          ></el-button>
          <!-- 删除 -->
          <el-button
            type="danger"
            icon="el-icon-delete"
            size="mini"
            @click="deleteUserById(scope.row.id)"
          ></el-button>
          <!-- 分配角色 -->
          <el-tooltip effect="dark" content="分配角色" placement="top" :enterable="false">
            <el-button
              type="warning"
              icon="el-icon-setting"
              size="mini"
              @click="showSetUserRoleDialog(scope.row)"
            ></el-button>
          </el-tooltip>
        </template>
      </el-table-column>
    </el-table>
    <!-- 分页区域 -->
    <el-pagination
      @size-change="handleSizeChange"
      @current-change="handleCurrentChange"
      :current-page="queryInfo.pagenum"
      :page-sizes="[5, 10, 15]"
      :page-size="queryInfo.pagesize"
      layout="total, sizes, prev, pager, next, jumper"
      :total="totalSize"
      background
    ></el-pagination>
    <!-- 添加用户区域 -->
    <el-dialog title="添加用户" :visible.sync="addDialogVisible" width="50%" @close="addDialogClosed">
      <!-- 内容主体区域 -->
      <el-form :model="addForm" :rules="addFormRules" ref="addFormRef" label-width="70px">
        <el-form-item prop="username" label="用户名">
          <el-input v-model="addForm.username"></el-input>
        </el-form-item>
        <el-form-item prop="password" label="密码">
          <el-input v-model="addForm.password" type="password"></el-input>
        </el-form-item>
        <el-form-item prop="email" label="邮箱">
          <el-input v-model="addForm.email" type="email"></el-input>
        </el-form-item>
        <el-form-item prop="mobile" label="电话">
          <el-input v-model="addForm.mobile" type="mobile"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer">
        <el-button @click="addDialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="addUser">确 定</el-button>
      </span>
    </el-dialog>
    <!-- 修改用户区域 -->
    <el-dialog title="修改用户" :visible.sync="editDialogVisible" width="50%" @close="editDialogClosed">
      <!-- 内容主体区域 -->
      <el-form :model="editForm" :rules="editFormRules" ref="editFormRef" label-width="70px">
        <el-form-item prop="username" label="用户名">
          <el-input v-model="editForm.username" disabled></el-input>
        </el-form-item>
        <el-form-item prop="email" label="邮箱">
          <el-input v-model="editForm.email" type="email"></el-input>
        </el-form-item>
        <el-form-item prop="mobile" label="电话">
          <el-input v-model="editForm.mobile" type="mobile"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer">
        <el-button @click="editDialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="editUser">确 定</el-button>
      </span>
    </el-dialog>
    <!-- 分配角色 -->
    <el-dialog title="分配角色" :visible.sync="setUserRoleDialogVisible" width="50%" @close="clearSetUserRoleDialog">
      <p>当前用户: {{user.username}}</p>
      <p>当前角色: {{user.role_name}}</p>
      <p>
        分配角色:
        <el-select v-model="selectedRoleId" placeholder="请选择">
          <el-option
            v-for="item in roleList"
            :key="item.id"
            :label="item.roleName"
            :value="item.id"
          ></el-option>
        </el-select>
      </p>
      <span slot="footer" class="dialog-footer">
        <el-button @click="setUserRoleDialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="saveUserRole">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
export default {
  data() {
    // 自定义邮箱规则
    var checkEmail = (rule, value, callback) => {
      const regEmail = /^\w+@\w+(\.\w+)+$/
      if (regEmail.test(value)) {
        // 合法邮箱
        return callback()
      }
      callback(new Error('请输入合法邮箱'))
    }
    // 自定义手机号规则
    var checkMobile = (rule, value, callback) => {
      const regMobile = /^1[34578]\d{9}$/
      if (regMobile.test(value)) {
        return callback()
      }
      // 返回一个错误提示
      callback(new Error('请输入合法的手机号码'))
    }
    return {
      queryInfo: {
        query: '',
        pagenum: 1,
        pagesize: 10,
      },
      userList: [],
      totalSize: 0,
      //控制添加用户对话框的显示和隐藏
      addDialogVisible: false,
      //控制修改用户对话框的显示和隐藏
      editDialogVisible: false,
      //控制分配角色dialog窗口显示与关闭
      setUserRoleDialogVisible: false,
      //角色列表
      roleList: [],
      //即将分配角色的user
      user: {},
      //选中的角色id
      selectedRoleId: '',
      //添加用户表单数据
      addForm: {
        username: '',
        password: '',
        email: '',
        mobile: '',
      },
      editForm: {},

      //添加用户表单验证规则
      addFormRules: {
        username: [
          { required: true, message: '用户名', trigger: 'blur' },
          {
            min: 3,
            max: 12,
            message: '长度在 3 到 12 个字符',
            trigger: 'blur',
          },
        ],
        password: [
          { required: true, message: '请输入密码', trigger: 'blur' },
          {
            min: 6,
            max: 13,
            message: '长度在 6 到 13 个字符',
            trigger: 'blur',
          },
        ],
        email: [
          {
            validator: checkEmail,
            trigger: ['blur', 'change'],
          },
        ],
        mobile: [
          {
            validator: checkMobile,
            trigger: ['blur', 'change'],
          },
        ],
      },
      editFormRules: {
        username: [
          { required: true, message: '用户名', trigger: 'blur' },
          {
            min: 3,
            max: 12,
            message: '长度在 3 到 12 个字符',
            trigger: 'blur',
          },
        ],
        email: [
          {
            validator: checkEmail,
            trigger: ['blur', 'change'],
          },
        ],
        mobile: [
          {
            validator: checkMobile,
            trigger: ['blur', 'change'],
          },
        ],
      },
    }
  },
  created() {
    this.getUserList()
  },
  methods: {
    async getUserList() {
      const { data: res } = await this.$http.get('users', {
        params: this.queryInfo,
      })
      if (res.meta.status !== 200) {
        return this.$message.error(res.meta.meg)
      }
      this.userList = res.data.users
      this.totalSize = res.data.total
    },
    handleSizeChange(newPageSize) {
      this.queryInfo.pagesize = newPageSize
      this.getUserList()
    },
    handleCurrentChange(newPageNum) {
      this.queryInfo.pagenum = newPageNum
      this.getUserList()
    },
    async userStateChanged(user) {
      const { data: res } = await this.$http.put(
        `users/${user.id}/state/${user.mg_state}`
      )
      if (res.meta.status !== 200) {
        user.mg_state = !user.mg_state
        return this.$message.error(res.meta.msg)
      }
      this.$message.success(res.meta.msg)
    },
    addDialogClosed() {
      this.$refs.addFormRef.resetFields()
    },
    editDialogClosed() {
      this.$refs.editFormRef.resetFields()
    },
    addUser() {
      this.$refs.addFormRef.validate(async (valid) => {
        if (!valid) return
        const { data: res } = await this.$http.post('users', this.addForm)
        if (res.meta.status !== 201) {
          this.addDialogVisible = false
          return this.$message.error(res.meta.msg)
        }
        this.addDialogVisible = false
        this.getUserList()
        this.$message.success(res.meta.msg)
      })
    },
    async showEditDialog(id) {
      const { data: res } = await this.$http.get(`users/${id}`)
      if (res.meta.status !== 200) return $message.error(res.meta.msg)
      this.editForm = res.data
      this.editDialogVisible = true
    },
    editUser() {
      this.$refs.editFormRef.validate(async (valid) => {
        if (!valid) return
        const { data: res } = await this.$http.put(
          `users/${this.editForm.id}`,
          this.editForm
        )
        if (res.meta.status !== 200) {
          this.editDialogVisible = false
          return $message.error(res.meta.msg)
        }
        this.getUserList()
        this.editDialogVisible = false
        this.$message.success(res.meta.msg)
      })
    },
    async deleteUserById(id) {
      const confirmResult = await this.$confirm(
        '此操作将永久删除该用户, 是否继续?',
        '提示',
        {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning',
        }
      ).catch((error) => error)
      if (confirmResult !== 'confirm')
        return this.$message.info('已取消了删除 !')
      const { data: res } = await this.$http.delete(`users/${id}`)
      if (res.meta.status !== 200) return this.$message.error('删除失败 !')
      this.getUserList()
      this.$message.success('删除成功 !')
    },
    async showSetUserRoleDialog(user) {
      this.user = user
      const { data: res } = await this.$http.get('roles')
      this.roleList = res.data
      this.setUserRoleDialogVisible = true
    },
    //点击按钮分配角色
    async saveUserRole() {
      if (!this.selectedRoleId) return this.$message.error('请选择所属角色')
      const { data: res } = await this.$http.put(`users/${this.user.id}/role`, {
        rid: this.selectedRoleId,
      })
      if (res.meta.status !== 200) return this.$message.error('分配角色失败 !')
      this.setUserRoleDialogVisible = false
      this.getUserList()
      this.$message.success('分配角色成功 !')
    },
    //关闭分配角色窗口清空select选框
    clearSetUserRoleDialog() {
      this.selectedRoleId = ''
    }
  },
}
</script>

<style lang="less" scope>
</style>