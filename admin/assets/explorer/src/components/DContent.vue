<template>
    <div class="d-content" 
        v-loading="loading">
        <div class="d-content-header">
            <div class="header-toolbar" v-if="table">
                <div class="header-toolbar-col" style="margin-right: 10px; flex-grow: 1;">
                    <el-button 
                        size="small"
                        type="success"
                        @click="openDialogInsert">Insertar</el-button>
                    <el-button 
                        size="small" 
                        type="danger"
                        @click="openDialogDelete">Delete</el-button>
                </div>
                <div class="header-toolbar-col filter-bar">
                    <div class="filter-col" style="margin-right: 5px; display: flex;">
                        <el-select size="small" clearable v-model="filterCol">
                            <el-option 
                                v-for="col in filterColumns" 
                                :key="col"
                                :value="col" >{{col}}</el-option>
                        </el-select>
                        <el-select size="small" v-model="filterMethod">
                            <el-option :key="'equal'" :value="'equal'"  :label="'='"></el-option>
                            <el-option :key="'like_both'" :value="'like_both'" :label="'like %...%'"></el-option>
                            <el-option :key="'like_start'" :value="'like_start'" :label="'like ...%'"></el-option>
                            <el-option :key="'like_end'" :value="'like_end'" :label="'like %...'"></el-option>
                        </el-select>
                        <el-input size="small" placeholder="valor de columna" v-model="filterValue" />
                    </div>
                    <div class="filter-col">
                        <el-button size="small" type="primary" @click="doFilter">Filter</el-button>
                        <el-button size="small" @click="clearFilter">Clear</el-button>
                    </div>
                </div>
            </div>
            
        </div>
        <div class="d-content-inner" :style="{height, overflow: loading ? 'hidden' : 'auto'}">
            <table class="d-content-table">
                <thead>
                    <tr>
                        <th v-for="column in columns" :key="column.name">{{column.name}}</th>
                    </tr>
                </thead>
                <tbody>
                    <tr v-for="(item, row) in items" 
                        :key="row" 
                        :class="{selected: activeRow === row}"
                        @click="selectRow(row)">
                        <td v-for="(value, column) in item" 
                            :key="column"
                            :class="{selected: activeNode === (row + column + '')}"
                            @click="selectNode(row + column + '', item, column)">{{value ? ('' + value).substring(0, 100) : ''}}</td>
                    </tr>
                </tbody>
            </table>
        </div>
        <div style="margin-top: 5px; text-align: right;">
            <el-pagination
                layout="jumper, prev, pager, next"
                :page-size="50"
                :hide-on-single-page="true"
                :current-page.sync="currentPage"
                :total="total"
                @current-change="changePage">
            </el-pagination>
        </div>
        <el-dialog
            width="60%"
            :append-to-body="true"
            :close-on-click-modal="false"
            :close-on-press-escape="!saving"
            :show-close="!saving"
            :destroy-on-close="true"
            :custom-class="'dialog-edit'"
            :title="activeColumn + ' - Editor'"
            :visible.sync="dialogEdit">
            <textarea 
                v-model="dialogValue" 
                style="width: 100%; box-sizing: border-box;" 
                rows="15"
                :disabled="saving"></textarea>
            <span slot="footer" class="dialog-footer">
                <el-button 
                    size="small" 
                    :disabled="saving"
                    @click="dialogEdit = false">Close</el-button>
                <el-button 
                    size="small" 
                    type="primary" 
                    :loading="saving"
                    @click="saveNode">Save</el-button>
            </span>
        </el-dialog>
        <el-dialog
            width="60%"
            :append-to-body="true"
            :close-on-click-modal="false"
            :close-on-press-escape="!saving"
            :show-close="!saving"
            :destroy-on-close="true"
            :custom-class="'dialog-insert'"
            :title="table + ' - Insertar registro'"
            :visible.sync="dialogInsert">
            <form class="form-insert" v-loading="saving">
                <table>
                    <tr v-for="column in cloneColumns" :key="column.name">
                        <th>{{column.name}}</th>
                        <td>{{column.type}}</td>
                        <td>
                            <span v-if="column.extra === 'auto_increment'">Auto Increment</span>
                            <span v-else-if="column.default === 'CURRENT_TIMESTAMP'">Current Timestamp</span>
                            <input 
                                type="text"
                                v-else 
                                v-model="column.default"
                                :name="column.name">
                        </td>
                    </tr>
                </table>
            </form>
            <span slot="footer" class="dialog-footer">
                <el-button 
                    size="small" 
                    :disabled="saving"
                    @click="dialogInsert = false">Close</el-button>
                <el-button 
                    size="small" 
                    type="primary" 
                    :loading="saving"
                    @click="insertRecord">Insertar</el-button>
            </span>
        </el-dialog>
        <el-dialog
            width="50%"
            :append-to-body="true"
            :close-on-click-modal="false"
            :close-on-press-escape="!saving"
            :show-close="!saving"
            :destroy-on-close="true"
            :custom-class="'dialog-insert'"
            :title="'Table `' + table + '`: Do you want to delete this record?'"
            :visible.sync="dialogDelete">
            <table>
                <tr v-for="(value, key) in activeItem" :key="key">
                    <th>{{key}}</th>
                    <td>{{value ? ('' + value).substring(0, 100) : ''}}</td>
                </tr>
            </table>
            <span slot="footer" class="dialog-footer">
                <el-button 
                    size="small" 
                    :disabled="saving"
                    @click="dialogDelete = false">Close</el-button>
                <el-button 
                    size="small" 
                    type="primary" 
                    :loading="saving"
                    @click="deleteRecord">Delete</el-button>
            </span>
        </el-dialog>
    </div>
</template>

<script>
import Vue from 'vue';
import debounce from 'lodash/debounce';

export default {
    data() {
        return {
            loading: false,
            columns: [],
            cloneColumns: [],
            total: 0,
            items: [],
            height: '0px',
            currentPage: 1,
            activeNode: '',
            activeColumn: '',
            activeRow: -1,
            activeItem: {},
            dialogValue: '',
            dialogEdit: false,
            saving: false,
            dialogInsert: false,
            dialogDelete: false,
            filterCol: '',
            filterValue: '',
            filterMethod: 'equal',
        }
    },

    mounted() {
        setTimeout(() => {
            this.setContentHeight();
        });

        jQuery(window).on('resize.ffexplorer', () => {
            this.setContentHeight();
        });
    },

    computed: {
        table() {
            return this.$store.state.activeTable;
        },

        filterColumns() {
            return this.columns.map(col => col.name);
        },
    },

    watch: {
        table(name) {
            this.currentPage = 1;

            this.resetFilter();
            this.resetActiveNode();
            this.initTable(name);
            this.resetScrollPosition();
        },
    },

    methods: {
        clearFilter() {
            this.currentPage = 1;
            this.resetActiveNode();
            this.resetFilter();
            this.resetScrollPosition();
            this.initTable();
        },

        doFilter() {
            this.currentPage = 1;
            this.initTable();
        },

        openDialogDelete() {
            if (this.activeRow === -1) {
                return alert('Seleccione un registro para eliminar');
            }

            this.dialogDelete = true;
        },

        deleteRecord() {
            this.saving = true;

            const cols = this.getConditionColumns();
            const condition = {};
            cols.forEach(col => {
                condition[col] = this.activeItem[col]
            });

            this.$ajax({
                task: 'db.deleteRecord',
                table: this.table,
                condition: JSON.stringify(condition),
            })
            .then(res => {
                if (res.error) {
                    return alert(res.error);
                }

                if (res.success) {
                    this.resetActiveNode();

                    this.$message({
                        type: 'success',
                        message: 'delete succesfully'
                    });
                    return this.initTable(this.table, this.currentPage);
                }
            })
            .catch(error => {
                alert('delete error');
            })
            .finally(() => {
                this.saving = false;
                this.dialogDelete = false;
            });
        },

        openDialogInsert() {
            Vue.set(this, 'cloneColumns', JSON.parse(JSON.stringify(this.columns)));
            this.dialogInsert = true;
        },
        
        insertRecord() {
            this.saving = true;

            const data = jQuery('form.form-insert').serializeArray();

            this.$ajax({
                task: 'db.insertRecord',
                table: this.table,
                data: JSON.stringify(data),
            })
            .then(res => {
                if (res.error) {
                    return alert(res.error);
                }

                if (res.success) {
                    this.$message({
                        type: 'success',
                        message: 'insert succesfully'
                    });
                    return this.initTable(this.table, this.currentPage);
                }
            })
            .catch(error => {
                alert('insert error');
            })
            .finally(() => {
                this.saving = false;
                this.dialogInsert = false;
            });
        },

        saveNode() {
            this.saving = true;

            const cols = this.getConditionColumns();
            const condition = {};
            cols.forEach(col => {
                condition[col] = this.activeItem[col];
            });

            this.$ajax({
                task: 'db.saveNode',
                table: this.table,
                condition: JSON.stringify(condition),
                column: this.activeColumn,
                value: this.dialogValue,
            })
            .then(res => {
                if (res.error) {
                    return alert(res.error);
                }

                this.$message({
                    type: 'success',
                    message: 'Save successfully',
                });

                return this.initTable(this.table, this.currentPage);
            })
            .catch(error => {
                alert('save error');
            })
            .finally(() => {
                this.saving = false;
            });
        },

        getConditionColumns() {
            const priCols = this.columns.filter(col => {
                return col.key === 'PRI';
            })
            .map(col => {
                return col.name;
            });

            if (priCols.length) {
                return priCols;
            } else {
                return this.columns.map(col => col.name);
            }
        },

        resetActiveNode() {
            this.activeNode = '';
            this.activeColumn = '';
            this.activeRow = -1;
            Vue.set(this, 'activeItem', {});
        },

        resetFilter() {
            this.filterCol = '';
            this.filterValue = '';
            this.filterMethod = 'equal';
        },

        changePage(page) {
            this.resetActiveNode();
            this.initTable(this.table, page).then(() => {
                this.resetScrollPosition();
            });
        },

        selectRow(row) {
            this.activeRow = row;
        },

        selectNode(nodeId, item, column) {
            if (this.activeNode !== nodeId) {
                this.activeNode = nodeId;
                this.activeColumn = column;
                this.dialogValue = item[column];
                Vue.set(this, 'activeItem', item);
            } else {
                this.dialogEdit = true;
            }
        },

        setContentHeight: debounce(function() {
            const $ = jQuery;
            const wHeight = $(window).height();
            if ($('.d-content-inner').length) {
                this.height = (wHeight - $('.d-content-inner').offset().top - 50) + 'px';
            }
        }, 100),

        initTable(name, page) {
            return new Promise((resolve, reject) => {
                name = name ? name : this.table;
                page = page ? page : 1;

                this.loading = true;
                const {filterCol, filterValue, filterMethod} = this;
                this.$ajax({
                    task: 'db.initTable',
                    name,
                    page,
                    filterCol,
                    filterValue,
                    filterMethod,
                })
                .then(res => {
                    if (res.error) {
                        return alert(res.error);
                    }

                    if (res.data) {
                        this.total = +res.data.total;
                        Vue.set(this, 'columns', res.data.columns);
                        Vue.set(this, 'items', res.data.items);
                        this.setContentHeight();
                    }
                })
                .catch(error => {
                    alert('init table error');
                })
                .finally(() => {
                    this.loading = false;
                    resolve();
                });
            });
        },

        resetScrollPosition() {
            const $inner = this.$el.querySelector('.d-content-inner');
            $inner.scrollTop = 0;
            $inner.scrollLeft = 0;
        }
    },
}
</script>

<style lang="scss">
.d-content {
    flex: 1;
    margin-left: 10px;
    overflow: hidden;

    .el-loading-mask {
        z-index: 1000;
    }

    .d-content-header {
        min-height: 38px;
        display: flex;

        .header-toolbar {
            flex: 1;
            display: flex;
            flex-wrap: wrap;

            .header-toolbar-col {
                margin-bottom: 5px;
            }

            .filter-bar {
                display: flex;
                flex-wrap: wrap;

                .filter-col {
                    margin-bottom: 5px;
                }
            }

            select {
                width: unset;
                margin: 0;
            }

            input {
                margin: 0;
                width: 150px;
            }
        }
    }

    .d-content-inner {
        overflow: auto;
        border: solid 1px #ddd;
    }

    .d-content-table {
        position: relative;

        th {
            text-align: unset;
            max-width: 300px;
            padding: 3px 10px;
            word-break: normal;
            position: sticky;
            top: 0;
            background-color: #eee;
            box-shadow: 0 2px 2px -1px rgba(0, 0, 0, 0.4);
            border-right: solid 1px #ddd;
            border-bottom: solid 1px #ddd;
        }

        tr {
            transition: background-color 300ms;

            &.selected {
                background-color: rgba(0, 0, 0, 0.1);
            }

            > td {
                max-width: 300px;
                padding: 3px 10px;
                overflow: hidden;
                white-space: nowrap;
                text-overflow: ellipsis;
                user-select: none;
                transition: background-color 300ms;
                border-right: solid 1px #ddd;
                border-bottom: solid 1px #ddd;

                &.selected {
                    background-color: rgba(103, 186, 224, 0.73);
                    outline: dashed 1px #3a8ee6;
                }
            }
        }
    }

    .el-select {
        .el-input__inner {
            background-color: unset;
        }
    }

    .el-button + .el-button {
        margin-left: 0;
    }
}

.dialog-edit {
    .el-dialog__body {
        padding: 10px;
    }
    
    .el-dialog__footer {
        padding: 10px;
    }
}

.dialog-insert {
    table {
        width: 100%;

        > tr {
            border-bottom: dashed 1px #ddd;

            th {
                text-align: unset;
                padding: 5px;
                white-space: nowrap;
            }

            td {
                padding: 5px;
                overflow: hidden;
                text-overflow: ellipsis;
                white-space: nowrap;
                max-width: 300px;

                input {
                    margin: 0;
                }
            }
        }
    }
}
</style>
