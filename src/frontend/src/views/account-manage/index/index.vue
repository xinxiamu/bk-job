<!--
 * Tencent is pleased to support the open source community by making BK-JOB蓝鲸智云作业平台 available.
 *
 * Copyright (C) 2021 THL A29 Limited, a Tencent company.  All rights reserved.
 *
 * BK-JOB蓝鲸智云作业平台 is licensed under the MIT License.
 *
 * License for BK-JOB蓝鲸智云作业平台:
 *
 *
 * Terms of the MIT License:
 * ---------------------------------------------------
 * Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated
 * documentation files (the "Software"), to deal in the Software without restriction, including without limitation
 * the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and
 * to permit persons to whom the Software is furnished to do so, subject to the following conditions:
 *
 * The above copyright notice and this permission notice shall be included in all copies or substantial portions of
 * the Software.
 *
 * THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT
 * THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
 * AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF
 * CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS
 * IN THE SOFTWARE.
-->

<template>
    <div>
        <list-action-layout>
            <auth-button
                style="width: 120px;"
                auth="account/create"
                theme="primary"
                @click="handleAdd">
                {{ $t('account.新建') }}
            </auth-button>
            <template #right>
                <jb-search-select
                    ref="search"
                    :data="searchSelect"
                    :placeholder="$t('account.搜索账号别名，名称，更新人...')"
                    style="width: 420px;"
                    @on-change="handleSearch" />
            </template>
        </list-action-layout>
        <render-list
            ref="list"
            :data-source="dataSource"
            :size="tableSize"
            :search-control="() => $refs.search">
            <bk-table-column
                v-if="allRenderColumnMap.id"
                label="ID"
                prop="id"
                key="id"
                sortable
                width="120"
                align="left" />
            <bk-table-column
                v-if="allRenderColumnMap.alias"
                :label="$t('account.账号别名.colHead')"
                prop="alias"
                key="alias"
                sortable
                align="left" />
            <bk-table-column
                v-if="allRenderColumnMap.account"
                :label="$t('account.账号名称.colHead')"
                prop="account"
                key="account"
                sortable
                align="left" />
            <bk-table-column
                v-if="allRenderColumnMap.categoryName"
                :label="$t('account.账号用途.colHead')"
                prop="categoryName"
                key="categoryName"
                sortable
                align="left" />
            <bk-table-column
                v-if="allRenderColumnMap.typeName"
                :label="$t('account.账号类型.colHead')"
                prop="typeName"
                key="typeName"
                sortable
                align="left" />
            <bk-table-column
                v-if="allRenderColumnMap.creator"
                :label="$t('account.创建人')"
                prop="creator"
                key="creator"
                align="left" />
            <bk-table-column
                v-if="allRenderColumnMap.createTime"
                :label="$t('account.创建时间')"
                prop="createTime"
                key="createTime"
                align="left" />
            <bk-table-column
                v-if="allRenderColumnMap.lastModifyUser"
                :label="$t('account.更新人.colHead')"
                prop="lastModifyUser"
                key="lastModifyUser"
                align="left" />
            <bk-table-column
                v-if="allRenderColumnMap.lastModifyTime"
                :label="$t('account.更新时间')"
                prop="lastModifyTime"
                key="lastModifyTime"
                align="left" />
            <bk-table-column
                :label="$t('account.操作')"
                :resizable="false"
                key="action"
                width="120">
                <template slot-scope="{ row }">
                    <auth-button
                        class="mr10"
                        theme="primary"
                        :permission="row.canManage"
                        auth="account/edit"
                        :resource-id="row.id"
                        text
                        @click="handleEdit(row)">
                        {{ $t('account.编辑') }}
                    </auth-button>
                    <jb-popover-confirm
                        :title="$t('account.确定删除该账号？')"
                        :content="$t('account.删除后不可恢复，请谨慎操作！')"
                        :confirm-handler="() => handleDelete(row.id)">
                        <auth-button text
                            :permission="row.canManage"
                            auth="account/delete"
                            :resource-id="row.id">
                            {{ $t('account.删除') }}
                        </auth-button>
                    </jb-popover-confirm>
                </template>
            </bk-table-column>
            <bk-table-column type="setting">
                <bk-table-setting-content
                    :fields="tableColumn"
                    :selected="selectedTableColumn"
                    :size="tableSize"
                    @setting-change="handleSettingChange" />
            </bk-table-column>
        </render-list>
        <jb-sideslider :is-show.sync="showOperation" v-bind="operationSidesliderInfo" :width="540">
            <operation :data="editData" @on-change="handleOperationSubmit" />
        </jb-sideslider>
    </div>
</template>
<script>
    import I18n from '@/i18n';
    import AccountService from '@service/account-manage';
    import NotifyService from '@service/notify';
    import { listColumnsCache } from '@utils/cache-helper';
    import ListActionLayout from '@components/list-action-layout';
    import RenderList from '@components/render-list';
    import JbSearchSelect from '@components/jb-search-select';
    import Operation from './components/operation';

    const TABLE_COLUMN_CACHE = 'account_manage_columns';

    export default {
        components: {
            ListActionLayout,
            RenderList,
            JbSearchSelect,
            Operation,
        },
        data () {
            return {
                showOperation: false,
                editData: {},
                searchParams: [],
                searchValue: [],
                selectedTableColumn: [],
                tableSize: 'small',
            };
        },
        computed: {
            isSkeletonLoading () {
                return this.$refs.list.isLoading;
            },
            allRenderColumnMap () {
                return this.selectedTableColumn.reduce((result, item) => {
                    result[item.id] = true;
                    return result;
                }, {});
            },
            operationSidesliderInfo () {
                if (this.editData.id) {
                    return {
                        title: I18n.t('account.编辑账号'),
                        okText: I18n.t('account.保存'),
                    };
                }
                return {
                    title: I18n.t('account.新建账号'),
                    okText: I18n.t('account.提交'),
                };
            },
        },
        created () {
            this.dataSource = AccountService.fetchAccountList;
            this.searchSelect = [
                {
                    name: 'ID',
                    id: 'id',
                    description: I18n.t('account.将覆盖其它条件'),
                },
                {
                    name: I18n.t('account.账号别名.colHead'),
                    id: 'alias',
                    default: true,
                },
                {
                    name: I18n.t('account.账号名称.colHead'),
                    id: 'account',
                },
                {
                    name: I18n.t('account.创建人'),
                    id: 'creator',
                    remoteMethod: NotifyService.fetchUsersOfSearch,
                    inputInclude: true,
                },
                {
                    name: I18n.t('account.更新人.colHead'),
                    id: 'lastModifyUser',
                    remoteMethod: NotifyService.fetchUsersOfSearch,
                    inputInclude: true,
                },
            ];
            this.tableColumn = [
                {
                    id: 'id',
                    label: 'ID',
                },
                {
                    id: 'alias',
                    label: I18n.t('account.账号别名.colHead'),
                    disabled: true,
                },
                {
                    id: 'account',
                    label: I18n.t('account.账号名称.colHead'),
                    disabled: true,
                },
                {
                    id: 'categoryName',
                    label: I18n.t('account.账号用途.colHead'),
                },
                {
                    id: 'typeName',
                    label: I18n.t('account.账号类型.colHead'),
                    disabled: true,
                },
                {
                    id: 'creator',
                    label: I18n.t('account.创建人'),
                },
                {
                    id: 'createTime',
                    label: I18n.t('account.创建时间'),
                },
                {
                    id: 'lastModifyUser',
                    label: I18n.t('account.更新人.colHead'),
                },
                {
                    id: 'lastModifyTime',
                    label: I18n.t('account.更新时间'),
                },
            ];
            const columnsCache = listColumnsCache.getItem(TABLE_COLUMN_CACHE);
            if (columnsCache) {
                this.selectedTableColumn = Object.freeze(columnsCache.columns);
                this.tableSize = columnsCache.size;
            } else {
                this.selectedTableColumn = Object.freeze([
                    { id: 'alias' },
                    { id: 'account' },
                    { id: 'categoryName' },
                    { id: 'typeName' },
                    { id: 'lastModifyUser' },
                    { id: 'lastModifyTime' },
                ]);
            }
        },
        methods: {
            /**
             * @desc 获取列表数据
             */
            fetchData () {
                this.$refs.list.$emit('onFetch', this.searchParams);
            },
            /**
             * @desc 表格自定时设置
             */
            handleSettingChange ({ fields, size }) {
                this.selectedTableColumn = Object.freeze(fields);
                this.tableSize = size;
                listColumnsCache.setItem(TABLE_COLUMN_CACHE, {
                    columns: fields,
                    size,
                });
            },
            /**
             * @desc 表格自定时设置
             * @param {Object} params 搜索条件
             */
            handleSearch (params) {
                this.searchParams = params;
                this.fetchData();
            },
            /**
             * @desc 显示新建账号弹层
             */
            handleAdd () {
                this.editData = {};
                this.showOperation = true;
            },
            /**
             * @desc 编辑 账号
             * @param {Object} data 某一行账号
             */
            handleEdit (data) {
                this.editData = { ...data };
                this.showOperation = true;
            },
            /**
             * @desc 编辑 账号
             * @param {Number} id 账号id
             *
             * 编辑成功后刷新列表数据
             */
            handleDelete (id) {
                return AccountService.deleteAccount({
                    id,
                }).then(() => {
                    this.fetchData();
                    this.messageSuccess(I18n.t('account.删除成功'));
                    return true;
                });
            },
            /**
             * @desc 新建成功后刷新列表数据
             */
            handleOperationSubmit () {
                this.fetchData();
            },
        },
    };
</script>
