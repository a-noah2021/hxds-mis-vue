<template>
	<div v-if="isAuth(['ROOT', 'COMMENT:SELECT'])">
		<el-form :inline="true" :model="dataForm" :rules="dataRule" ref="dataForm">
			<el-form-item prop="orderId">
				<el-input v-model="dataForm.orderId" placeholder="订单编号" size="medium" clearable="clearable" />
			</el-form-item>
			<el-form-item prop="driverId">
				<el-input v-model="dataForm.driverId" placeholder="司机编号" size="medium" clearable="clearable" />
			</el-form-item>
			<el-form-item prop="customerId">
				<el-input v-model="dataForm.customerId" placeholder="乘客编号" size="medium" clearable="clearable" />
			</el-form-item>
			<el-form-item>
				<el-select v-model="dataForm.rate" class="input" placeholder="评价" size="medium" clearable="clearable">
					<el-option label="好评" value="好评" />
					<el-option label="中评" value="中评" />
					<el-option label="差评" value="差评" />
				</el-select>
			</el-form-item>
			<el-form-item>
				<el-select
					v-model="dataForm.status"
					class="input"
					placeholder="状态"
					size="medium"
					clearable="clearable"
				>
					<el-option label="未申诉" value="1" />
					<el-option label="已申诉" value="2" />
					<el-option label="申诉失败" value="3" />
					<el-option label="申诉成功" value="4" />
				</el-select>
			</el-form-item>
			<el-form-item>
				<el-date-picker
					v-model="dataForm.date"
					type="daterange"
					range-separator="~"
					start-placeholder="开始日期"
					end-placeholder="结束日期"
					size="medium"
				></el-date-picker>
			</el-form-item>
			<el-form-item>
				<el-button size="medium" type="primary" @click="searchHandle()">查询</el-button>
			</el-form-item>
		</el-form>
		<el-table
            :data="dataList"
            border
            v-loading="dataListLoading"
            cell-style="padding: 4px 0"
            style="width: 100%;"
            size="medium"
            @expand-change="expand"
            :row-key="getRowKeys"
            :expand-row-keys="expands"
        >
            <el-table-column prop="content" header-align="center" align="center" type="expand">
                <template #default="scope">
                    <div class="order-table">
                        <div class="row">
                            <div class="th">【乘客评价】</div>
                            <div class="td">{{ scope.row.remark }}</div>
                        </div>
                        <div class="row" v-show="scope.row.status != '未申诉'">
                            <div class="th">【申诉理由】</div>
                            <div class="td">{{ scope.row.reason }}</div>
                        </div>
                        <div class="row" v-show="scope.row.status == '申诉失败'">
                            <div class="th">【处理意见】</div>
                            <div class="td">{{ scope.row.note }}</div>
                        </div>
                    </div>
                </template>
            </el-table-column>
            <el-table-column type="index" header-align="center" align="center" width="100" label="序号">
                <template #default="scope">
                    <span>{{ (pageIndex - 1) * pageSize + scope.$index + 1 }}</span>
                </template>
            </el-table-column>
            <el-table-column prop="orderId" header-align="center" align="center" min-width="200" label="订单编号" />
            <el-table-column prop="driverId" header-align="center" align="center" min-width="200" label="司机编号" />
            <el-table-column prop="customerId" header-align="center" align="center" min-width="200" label="乘客编号" />
            <el-table-column prop="rate" header-align="center" align="center" min-width="80" label="评分">
                <template #default="scope">
                    <span v-show="scope.row.rate == '好评'" class="comment-rate-green">好评</span>
                    <span v-show="scope.row.rate == '中评'" class="comment-rate-orange">中评</span>
                    <span v-show="scope.row.rate == '差评'" class="comment-rate-red">差评</span>
                </template>
            </el-table-column>
            <el-table-column prop="status" header-align="center" align="center" min-width="120" label="状态" />
            <el-table-column prop="acceptTime" header-align="center" align="center" min-width="180" label="日期" />
            <el-table-column prop="userName" header-align="center" align="center" min-width="120" label="申诉受理人" />
            <el-table-column header-align="center" align="center" width="180" label="操作">
                <template #default="scope">
                    <el-button
                        type="text"
                        size="medium"
                        v-if="isAuth(['ROOT', 'COMMENT:APPROVAL'])"
                        @click="showAcceptModel(scope.row.commentId)"
                        :disabled="!(scope.row.status == '已申诉' && scope.row.userId == null)"
                    >
                        受理
                    </el-button>

                    <el-button
                        type="text"
                        size="medium"
                        v-if="isAuth(['ROOT', 'COMMENT:APPROVAL'])"
                        @click="showHandleModel('no', scope.row.commentId, scope.row.instanceId)"
                        :disabled="!(scope.row.status == '已申诉' && scope.row.userId != null && scope.row.handler)"
                    >
                        驳回
                    </el-button>
                    <el-button
                        type="text"
                        size="medium"
                        v-if="isAuth(['ROOT', 'COMMENT:APPROVAL'])"
                        @click="showHandleModel('yes', scope.row.commentId, scope.row.instanceId)"
                        :disabled="!(scope.row.status == '已申诉' && scope.row.userId != null && scope.row.handler)"
                    >
                        通过
                    </el-button>
                </template>
            </el-table-column>
        </el-table>
		<el-pagination
			@size-change="sizeChangeHandle"
			@current-change="currentChangeHandle"
			:current-page="pageIndex"
			:page-sizes="[10, 20, 50]"
			:page-size="pageSize"
			:total="totalCount"
			layout="total, sizes, prev, pager, next, jumper"
		></el-pagination>
		<el-dialog v-model="acceptVisible" title="提示信息" width="350px" center>
			<div class="notice">你确定受理该订单的评价申诉？</div>
			<template #footer>
				<span class="dialog-footer">
					<el-button size="medium" @click="acceptVisible = false">取消</el-button>
					<el-button type="primary" size="medium" @click="acceptHandle()">确定</el-button>
				</span>
			</template>
		</el-dialog>
		<el-dialog v-model="handleVisible" title="提示信息" width="350px" center>
			<div class="notice" v-if="handleMode == 'no'">你是否确定驳回代驾司机对乘客评价的申诉？</div>
			<div class="notice" v-if="handleMode == 'yes'">你是否确定通过代驾司机对乘客评价的申诉？</div>
			<el-input
				v-model="note"
				:rows="3"
				type="textarea"
				placeholder="输入驳回申诉的原因"
				v-if="handleMode == 'no'"
			/>
			<template #footer>
				<span class="dialog-footer">
					<el-button size="medium" @click="handleVisible = false">取消</el-button>
					<el-button type="primary" size="medium" @click="handleAppeal">确定</el-button>
				</span>
			</template>
		</el-dialog>
	</div>
</template>
<script>
import $ from 'jquery';
export default {
	components: {},
	data() {
		return {
			dataForm: {
				orderId: null,
				driverId: null,
				customerId: null,
				rate: null,
				status: null,
				date: []
			},

			dataList: [],
			pageIndex: 1,
			pageSize: 10,
			totalCount: 0,
			dataListLoading: false,
			dataRule: {
				orderId: [{ required: false, pattern: '^[1-9]\\d{17}$', message: '订单编号格式错误' }],
				driverId: [{ required: false, pattern: '^[1-9]\\d{17}$', message: '司机编号格式错误' }],
				customerId: [{ required: false, pattern: '^[1-9]\\d{17}$', message: '客户编号格式错误' }]
			},
			expands: [],
			content: {},
			acceptVisible: false,
			handleVisible: false,
			handleMode: null,
			commentId: null,
			note: null,
			getRowKeys(row) {
				return row.orderId;
			}
		};
	},
	methods: {
    loadDataList: function() {
      let that = this;
      that.dataListLoading = true;
      let data = {
        page: that.pageIndex,
        length: that.pageSize,
        orderId: that.dataForm.orderId == '' ? null : that.dataForm.orderId,
        driverId: that.dataForm.driverId == '' ? null : that.dataForm.driverId,
        customerId: that.dataForm.customerId == '' ? null : that.dataForm.customerId,
        rate: that.dataForm.rate == '' ? null : that.dataForm.rate,
        status: that.dataForm.status == '' ? null : that.dataForm.status
      };
      if (that.dataForm.date != null && that.dataForm.date.length == 2) {
        let startDate = that.dataForm.date[0];
        let endDate = that.dataForm.date[1];
        data.startDate = dayjs(startDate).format('YYYY-MM-DD');
        data.endDate = dayjs(endDate).format('YYYY-MM-DD');
      }

      that.$http('comment/searchCommentByPage', 'POST', data, true, function(resp) {
        let result = resp.result;
        let list = result.list;
        let status = {
          '1': '未申诉',
          '2': '已申诉',
          '3': '申诉失败',
          '4': '申诉成功'
        };
        let rate = {
          '1': '差评',
          '2': '差评',
          '3': '中评',
          '4': '中评',
          '5': '好评'
        };
        for (let one of list) {
          one.status = status[one.status + ''];
          one.rate = rate[one.rate + ''];
        }
        that.dataList = list;
        that.totalCount = Number(result.totalCount);
        that.dataListLoading = false;
      });
    },
    searchHandle: function() {
      this.$refs['dataForm'].validate(valid => {
        if (valid) {
          this.$refs['dataForm'].clearValidate();
          this.loadDataList();
        } else {
          return false;
        }
      });
    },
    sizeChangeHandle: function(val) {
      this.pageSize = val;
      this.pageIndex = 1;
      this.loadDataList();
    },
    currentChangeHandle: function(val) {
      this.pageIndex = val;
      this.loadDataList();
    },
    showAcceptModel: function(id) {
      this.acceptVisible = true;
      this.commentId = id;
    },
    acceptHandle: function() {
      let that = this;
      let data = {
        commentId: that.commentId
      };
      that.$http('comment/acceptCommentAppeal', 'POST', data, true, function(resp) {
        that.acceptVisible = false;
        that.$message.success('受理成功');
        that.expands = [];
        that.loadDataList();
      });
    },
    handleAppeal: function() {
      let that = this;
      let data = {
        commentId: that.commentId,
        result: that.handleMode == 'yes' ? '同意' : '不同意',
        note: that.note != null && that.note.length > 0 ? that.note : null,
        instanceId: that.instanceId
      };
      that.$http('comment/handleCommentAppeal', 'POST', data, true, function(resp) {
        that.handleVisible = false;
        that.$message.success('执行成功');
        that.expands = [];
        that.loadDataList();
      });
    },
    expand: function(row, expandedRows) {
      let that = this;
      if (expandedRows.length > 0) {
        that.expands = [];
        if (row) {
          that.expands.push(row.orderId);
          if (row.status != '未申诉') {
            let data = {
              instanceId: row.instanceId,
              isEnd: row.status == '已申诉' ? false : true
            };
            that.$http('comment/searchAppealContent', 'POST', data, true, function(resp) {
              row.reason = resp.result.reason;
              row.note = resp.result.note;
            });
          }
        } else {
          that.expands = [];
        }
      }
    }
	},
  created: function() {
    this.loadDataList();
  }
};
</script>
<style lang="less" scoped="scoped">
@import url('comment.less');
</style>
