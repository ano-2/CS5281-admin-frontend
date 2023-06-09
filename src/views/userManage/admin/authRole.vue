<template>
    <div>
        <el-form :inline="true" :model="query" class="query-form" size="mini">
            <el-form-item class="query-form-item">
                <el-input v-model="query.name" placeholder="角色名称"></el-input>
            </el-form-item>
            <el-form-item class="query-form-item">
                <el-select v-model="query.status" placeholder="状态">
                    <el-option label="全部" value=""></el-option>
                    <el-option label="禁用" value="0"></el-option>
                    <el-option label="正常" value="1"></el-option>
                </el-select>
            </el-form-item>
            <el-form-item>
                <el-button type="primary" icon="el-icon-refresh" @click="getList"></el-button>
            </el-form-item>
            <el-form-item>
                <el-button type="primary" icon="search" @click="onSubmit">查询</el-button>
            </el-form-item>
            <el-form-item>
                <el-button type="primary" @click.native="handleForm(null, null)">新增</el-button>
            </el-form-item>
        </el-form>
        <el-table
            v-loading="loading"
            :data="list"
            style="width: 100%;"
            max-height="500">
            <el-table-column
                label="角色 ID"
                prop="id">
            </el-table-column>
            <el-table-column
                label="角色名称"
                prop="name">
            </el-table-column>
            <el-table-column
                label="状态">
                <template slot-scope="scope">
                    <el-tag :type="scope.row.status | statusFilterType">{{scope.row.status | statusFilterName}}</el-tag>
                </template>
            </el-table-column>
            <el-table-column
                label="描述"
                :show-overflow-tooltip="true">
                <template slot-scope="scope">
                    <span>{{ scope.row.remark }}</span>
                </template>
            </el-table-column>
            <el-table-column
                label="操作"
                fixed="right"
                width="200">
                <template slot-scope="scope">
                    <el-button type="text" size="small" @click.native="handleAuth(scope.row.id)">授权</el-button>
                    <el-button type="text" size="small" @click.native="handleForm(scope.$index, scope.row)">编辑</el-button>
                    <el-button type="text" size="small" @click.native="handleDel(scope.$index, scope.row)" :loading="deleteLoading">删除</el-button>
                </template>
            </el-table-column>
        </el-table>


        <!--授权界面-->
        <el-dialog
            title="授权"
            :visible.sync="authFormVisible"
            :close-on-click-modal="false"
            class="dialog">
            <el-tree
                style="max-height: 75vh;overflow-y: auto;"
                :data="authList"
                show-checkbox=""
                default-expand-all=""
                node-key="id"
                ref="tree"
                :props="defaultProps">
            </el-tree>
            <div slot="footer" class="dialog-footer">
                <el-button @click.native="authFormVisible = !authFormVisible">取消</el-button>
                <el-button type="primary" @click.native="authSubmit()" :loading="authLoading">提交
                </el-button>
            </div>
        </el-dialog>

        <!--表单-->
        <el-dialog
            :title="formMap[formName]"
            :visible.sync="formVisible"
            :before-close="hideForm"
            width="85%"
            top="5vh">
            <el-form :model="formData" :rules="formRules" ref="dataForm">
                <el-form-item label="角色名称" prop="name">
                    <el-input v-model="formData.name" auto-complete="off"></el-input>
                </el-form-item>
                <el-form-item label="排序" prop="listorder">
                    <el-input type="" v-model="formData.listorder" auto-complete="off"></el-input>
                </el-form-item>
                <el-form-item label="状态" prop="status">
                    <el-radio-group v-model="formData.status">
                        <el-radio label="0">禁用</el-radio>
                        <el-radio label="1">正常</el-radio>
                    </el-radio-group>
                </el-form-item>
                <el-form-item label="描述">
                    <el-input type="textarea" v-model="formData.remark"></el-input>
                </el-form-item>
            </el-form>
            <div slot="footer" class="dialog-footer">
                <el-button @click.native="hideForm">取消</el-button>
                <el-button type="primary" @click.native="formSubmit()" :loading="formLoading">提交</el-button>
            </div>
        </el-dialog>
    </div>

</template>

<script>
import {
    authRoleList,
    authRoleAuthList,
    authRoleAuth,
    authRoleSave,
    authRoleDelete
} from "../../../api/auth/authRole";
const formJson = {
    id: "",
    name: "",
    status: "1",
    remark: "",
    listorder: 999
};
export default {
    data() {
        return {
            query: {
                name: "",
                status: ""
            },
            list: [],
            loading: true,
            authList: [],
            defaultProps: {
                children: "children",
                label: "title"
            },
            authLoading: false,
            authFormVisible: false,
            authFormData: {
                role_id: "",
                auth_rules: []
            },

            index: null,
            formName: null,
            formMap: {
                add: "新增",
                edit: "编辑"
            },
            formLoading: false,
            formVisible: false,
            formData: formJson,
            formRules: {
                name: [
                    { required: true, message: "请输入名称", trigger: "blur" }
                ],
                status: [
                    { required: true, message: "请选择状态", trigger: "change" }
                ]
            },
            deleteLoading: false
        };
    },
    methods: {
        onSubmit() {
            this.getList();
        },
        getList() {
            this.loading = true;
            authRoleList(this.query)
                .then(response => {
                    this.loading = false;
                    this.list = response || [];
                })
                .catch(() => {
                    this.loading = false;
                    this.list = [];
                });
        },
        // 显示授权界面
        handleAuth(roleId) {
            this.authFormData.role_id = roleId;
            this.authFormData.auth_rules = [];
            this.authList = [];
            authRoleAuthList({ id: roleId })
                .then(response => {
                    if (response.code) {
                        this.authFormVisible = false;
                        this.$message({
                            message: response.message,
                            type: "error"
                        });
                        return;
                    }
                    this.authFormVisible = true;
                    this.authList = response.auth_list || [];
                    const checkedKeys = response.checked_keys || [];
                    let tempCheckedKeys = [];
                    let id = null;
                    let node = null;
                    let getTreeNode = function(arr, id) {
                        for (let i in arr) {
                            let tempNode = arr[i];
                            if (tempNode.id === id) {
                                // 找到了，就不找了
                                node = tempNode;
                            }
                            // 如果还有子节点，再继续找
                            getTreeNode(tempNode.children, id);
                        }
                    };
                    for (let i in checkedKeys) {
                        id = checkedKeys[i];
                        getTreeNode(this.authList, id);
                        if (node && node.children.length <= 0) {
                            // 如果下面没有子节点，则加入
                            tempCheckedKeys.push(id);
                        }
                    }
                    this.$refs.tree.setCheckedKeys(tempCheckedKeys);
                })
                .catch(() => {});
        },
        authSubmit() {
            this.authLoading = true;
            let getNodeParents = function(node) {
                let arr = [];
                let id = node.data.id || null;
                if (id) {
                    arr.push(node.data.id);
                }
                if (node.parent) {
                    arr = arr.concat(getNodeParents(node.parent));
                }
                return arr;
            };
            let checkedKeys = this.$refs.tree.getCheckedKeys();
            let arr = [];
            for (let checkedKey of checkedKeys) {
                let node = this.$refs.tree.getNode(checkedKey);
                arr = arr.concat(getNodeParents(node));
            }
            let setArr = new Set(arr);
            this.authFormData.auth_rules = [...setArr];
            if (!this.authFormData) {
                this.$alert("请至少选择一个权限", "提示", {
                    confirmButtonText: "确定"
                });
                return false;
            }
            authRoleAuth(this.authFormData)
                .then(res => {
                    this.authLoading = false;
                    if (res.code) {
                        this.$message({
                            message: res.message,
                            type: "error"
                        });
                    } else {
                        this.$message({
                            message: "授权成功",
                            type: "success"
                        });
                        // 刷新表单
                        this.authFormVisible = false;
                    }
                })
                .catch(() => {
                    this.editLoading = false;
                });
        },
        // 隐藏表单
        hideForm() {
            // 更改值
            this.formVisible = !this.formVisible;
            // 清空表单
            this.$refs["dataForm"].resetFields();
            return true;
        },
        // 显示表单
        handleForm(index, row) {
            this.formVisible = true;
            this.formData = Object.assign({}, formJson);
            if (row !== null) {
                this.formData = Object.assign({}, row);
            }
            this.formData.status += ""; // 转为字符串（解决默认选中的时候字符串和数字不能比较的问题）
            this.formName = "add";
            if (index !== null) {
                this.index = index;
                this.formName = "edit";
            }
            // 清空验证信息表单
            if (this.$refs["dataForm"]) {
                this.$refs["dataForm"].clearValidate();
            }
        },
        formSubmit() {
            this.$refs["dataForm"].validate(valid => {
                if (valid) {
                    this.formLoading = true;
                    let data = Object.assign({}, this.formData);
                    authRoleSave(data, this.formName)
                        .then(response => {
                            this.formLoading = false;
                            if (response.code) {
                                this.$message({
                                    message: response.message,
                                    type: "error"
                                });
                            } else {
                                this.$message({
                                    message: "操作成功",
                                    type: "success"
                                });
                                // 刷新表单
                                this.$refs["dataForm"].resetFields();
                                this.formVisible = false;
                                if (this.formName === "add") {
                                    // 向头部添加数据
                                    this.list.unshift(response);
                                } else {
                                    this.list.splice(this.index, 1, data);
                                }
                            }
                        })
                        .catch(() => {
                            this.formLoading = false;
                        });
                }
            });
        },
        // 删除
        handleDel(index, row) {
            if (row.id) {
                this.$confirm("确认删除该记录吗?", "提示", {
                    type: "warning"
                })
                    .then(() => {
                        this.deleteLoading = true;
                        let para = { id: row.id };
                        authRoleDelete(para)
                            .then(response => {
                                this.deleteLoading = false;
                                if (response.code) {
                                    this.$message({
                                        message: response.message,
                                        type: "error"
                                    });
                                } else {
                                    this.$message({
                                        message: "删除成功",
                                        type: "success"
                                    });
                                    // 刷新数据
                                    this.list.splice(index, 1);
                                }
                            })
                            .catch(() => {
                                this.deleteLoading = false;
                            });
                    })
                    .catch(() => {
                        this.$message({
                            type: "info",
                            message: "取消删除"
                        });
                    });
            }
        }
    },
    filters: {
        statusFilterType(status) {
            const statusMap = {
                0: "gray",
                1: "success"
            };
            return statusMap[status];
        },
        statusFilterName(status) {
            const statusMap = {
                0: "禁用",
                1: "正常"
            };
            return statusMap[status];
        }
    },
    mounted() {},
    created() {
        // 加载表格数据
        this.getList();
    }
};
</script>

<style lang="scss">
</style>
