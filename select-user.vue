<template>
	<el-dialog :close-on-click-modal="false" :title="dioTit" :visible.sync="isDio" width="1422px" @close="closeDio">
		<div class="prisoner-manage">
			<div class="transfer-area">
				<div class="transfer-content">
					<div class="transfer-search">
						<el-row :gutter="24">
							<el-col
								:span="12"
								style="display: flex; align-items: center; justify-content: space-between"
							>
								<div style="flex: 5">待选区({{ unSelect.length }})</div>
								<el-cascader
									:options="orgs"
									ref="casc"
									v-model="currentId"
									:show-all-levels="false"
									:props="cascProps"
									size="small"
									style="width: 180px; flex: 5"
									@change="casChange"
								>
								</el-cascader>
							</el-col>
							<el-col :span="12">
								<el-input
									clearable
									size="small"
									style="width: 300px; font-size: 12px"
									placeholder="请输入警号或姓名、姓名全拼、姓名拼音首字母"
									suffix-icon="el-icon-search"
									v-model="unSelectFilterValueOne"
									@input="getOrgsByFilter(unSelectFilterValueOne, false)"
								></el-input>
							</el-col>
						</el-row>
					</div>
					<div class="transfer-list" v-if="unSelect.length">
						<div
							class="transfer-list__item"
							:class="{ 'transfer-list__item--active': item.checked }"
							v-for="item in unSelect"
							:key="item.id"
							@click="handleChange(item)"
						>
							<div class="prisoner-name">{{ item.name }}</div>
							<div class="prisoner-num">{{ item.code }}</div>
							<div class="prisoner-area">{{ item.orgFullName }}</div>
						</div>
					</div>
					<div class="transfer-list--empty" v-if="!unSelect.length">
						<el-empty description="暂无数据"></el-empty>
					</div>
				</div>
				<div class="transfer-btn">
					<div>
						<el-button
							type="primary"
							icon="el-icon-d-arrow-left"
							@click="handleTransferAllToLeft"
						></el-button>
					</div>
					<div>
						<el-button type="primary" icon="el-icon-arrow-left" @click="handleTransferToLeft"></el-button>
					</div>
					<div>
						<el-button type="primary" icon="el-icon-arrow-right" @click="handleTransferToRight"></el-button>
					</div>
					<div>
						<el-button
							type="primary"
							icon="el-icon-d-arrow-right"
							@click="handleTransferAllToRight"
						></el-button>
					</div>
				</div>
				<div class="transfer-content">
					<div class="transfer-search">
						<el-row :gutter="24">
							<el-col :span="12">
								<span style="line-height: 32px">已选区({{ selected.length }})</span>
                                <span v-if="isDraggableDes" style="font-size:12px"> 如需调整值班人员顺序，可拖动排序</span>
							</el-col>
							<el-col :span="12">
								<el-input
									clearable
									size="small"
									style="width: 300px; font-size: 12px"
									placeholder="请输入警号或姓名、姓名全拼、姓名拼音首字母"
									suffix-icon="el-icon-search"
									v-model="selectFilterValueTwo"
									@input="getOrgsByFilter(selectFilterValueTwo, true)"
								></el-input>
							</el-col>
						</el-row>
					</div>
					<div  v-if="selected.length">
						<draggable v-model="selected" class="transfer-list">
							<div
								class="transfer-list__item"
								:class="{ 'transfer-list__item--active': item.checked }"
								v-for="item in selected"
								:key="item.id"
								@click="handleChange(item)"
							>
								<div class="prisoner-name">{{ item.name }}</div>
								<div class="prisoner-num">{{ item.code }}</div>
								<div class="prisoner-area">{{ item.orgFullName }}</div>
							</div>
						</draggable>
					</div>
					<div class="transfer-list--empty" v-if="!selected.length">
						<el-empty description="暂无数据"></el-empty>
					</div>
				</div>
			</div>
		</div>
		<div slot="footer" class="flex-jcfe">
			<el-button size="small" @click="closeDio">关闭</el-button>
			<el-button type="primary" size="small" @click="handleSubmit">提交</el-button>
		</div>
	</el-dialog>
</template>

<script>
import { apiUrl } from '@/assets/js/api';
import draggable from 'vuedraggable';
import { treeDataTranslate } from '@/utils/utils';
import qs from 'qs';
// 上次选中项
let lastSelect = [];
// 当前机构以及所有子机构的机构id集合，用于把人员从已选区左移到待选区时，是否展示出来。
let orgIds = new Set();
export default {
	name: 'register-prisonerManage',
	props: {
		// 是否通过监区机构查监狱机构
		getAllByPrisonArea: {
			type: Boolean,
			default: false,
		},
        isDraggableDes: {
            type: Boolean,
			default: false,
        },
		provinceOffice: {
			type: Boolean,
			default: false,
		},
		isShowPrisonType: {
			type: Boolean,
			default: false,
		},
		isDefaultTopOrg: {
			type: Boolean,
			default: false,
		},
		checkNum: {
			type: Number,
			default: 99999,
		},
	},
	components: {
		draggable,
	},
	data() {
		return {
			isDio: false, // 弹窗类型
			dioTit: '选择人员', // 弹窗标题
			unSelectFilterValueOne: '',
			selectFilterValueTwo: '',
			currentId: '',
			orgs: [],
			list: [],
			cascProps: {
				checkStrictly: true, // 是否单选
				emitPath: false,
				label: 'orgFullName',
				value: 'id',
				children: 'children',
			},
		};
	},
	computed: {
		unSelect() {
			return this.list.filter(
				item =>
					item.isSelected === false &&
					item.isFiltered === false &&
					(orgIds.size == 0 || orgIds.has(item.orgid || item.orgId))
			);
		},
		selected: {
			get() {
				return this.list.filter(item => item.isSelected === true && item.isFiltered === false);
			},
            // 拖拽修改排序
			set: function (newValue) {
				this.list = this.removeDuplicates([...newValue, ...this.list]);
			},
		},
	},
	methods: {
		// 初始化
		async init(defaultData = []) {
			this.isDio = true;
			this.getSecurityOrgList(defaultData);
		},
		// 获取当前机构的所有子机构id
		getOrgIds(node) {
			orgIds.add(node.value);
			if (node.hasChildren) {
				node.children.forEach(item => {
					this.getOrgIds(item);
				});
			}
		},
		// 选择父级机构
		casChange(orgId) {
			this.currentId = orgId || this.orgs[0].id;
			const nodes = this.$refs.casc.getCheckedNodes();
			orgIds.clear();
			this.getOrgIds(nodes[0]);
			lastSelect = this.selected.filter(item => item.isSelected === true);
			this.unSelectFilterValueOne = '';
			this.selectFilterValueTwo = '';
			this.getList(this.currentId);
		},
		getOrgsByFilter(val, type) {
			if (val.indexOf('，') != -1) {
				val = val.split('，');
			} else {
				val = val.split(',');
			}
			this.list.forEach(item => {
				if (item.isSelected == type) {
					for (let i = 0; i < val.length; i++) {
						if (val[i] == '' && val.length != 1) continue;
						const check =
							item.name.indexOf(val[i]) != -1 ||
							item.code.indexOf(val[i]) != -1 ||
							(item.pyName && item.pyName.indexOf(val[i]) != -1) ||
							(item.pyShortName && item.pyShortName.indexOf(val[i]) != -1);
						if (!check) {
							item.isFiltered = true;
						} else {
							item.isFiltered = false;
							break;
						}
					}
				}
			});
		},
		async getSecurityOrgList(defaultData) {
			const res = await apiUrl.getSecurityOrgList(
				qs.stringify({
					// orgId:
					// 	this.getAllByPrisonArea && this.$cookie.get('orgDutyType') == '2'
					// 		? this.$cookie.get('unitId')
					// 		: this.$cookie.get('orgId'),
					// userId: this.$cookie.get('userId'),
					orgId: this.$cookie.get('unitId'),
					prisonCode: this.$cookie.get('prisonCode'),
				})
			);
			if (res?.code === 0) {
				if (this.provinceOffice) {
					res.list = res.list.filter(item => item.orgDutyType == 0);
				}
				if (this.isShowPrisonType) {
					res.list = res.list.filter(
						item =>
							(item.orgType == 3 || item.orgType == 4 || item.orgType == 5) &&
							item.orgName.indexOf('会见室') === -1 &&
							item.orgName.indexOf('AB门') === -1
					);
				}
				this.orgs = treeDataTranslate(res.list, 'id', 'orgParentId');
				this.currentId = this.isDefaultTopOrg ? this.orgs[0].id : this.$cookie.get('orgId');

				this.getList(this.currentId, defaultData);
			} else {
				this.$message.error(res.msg);
			}
		},
		async getList(orgId = '', defaultData = []) {
			const res = await apiUrl.selectAcceptUserName(
				qs.stringify({
					deptId: orgId,
				})
			);
			if (res?.code === 0) {
				let list = [...lastSelect, ...this.addMark(res.data, false)];
				if (defaultData.length > 0) {
					list = [...this.addMark(defaultData, true), ...list];
					const nodes = this.$refs.casc.getCheckedNodes();
					// 获取当前机构的所有子机构
					this.getOrgIds(nodes[0]);
				}
				this.list = this.removeDuplicates(list);
			} else {
				this.$message.error(res.msg);
			}
		},
		handleChange(item) {
			item.checked = !item.checked;
		},
		isLimit(all) {
			return (
				this.unSelect.reduce((pre, cur) => {
					if (cur.checked || all) return pre + 1;
					return pre + 0;
				}, 0) + this.selected.length
			);
		},
		// 转移到右边
		handleTransferToRight() {
			if (this.isLimit(false) <= this.checkNum) {
				this.list.forEach(item => {
					item.checked && ((item.checked = false), (item.isSelected = true));
				});
			} else {
				this.$message.warning(`系统设置只允许选择${this.checkNum}个用户`);
			}
		},
		handleTransferAllToRight() {
			if (this.isLimit(true) <= this.checkNum) {
				this.list.forEach(item => {
					item.isSelected = true;
					item.checked = false;
				});
			} else {
				this.$message.warning(`系统设置只允许选择${this.checkNum}个用户`);
			}
		},
		// 转移到左边
		async handleTransferToLeft() {
			this.list.forEach(item => {
				item.checked && ((item.checked = false), (item.isSelected = false));
			});
		},
		handleTransferAllToLeft() {
			this.list.forEach(item => {
				item.isSelected = false;
				item.checked = false;
			});
		},
        // 标记
		addMark(data, selectType = false) {
			let result;
			if (Array.isArray(data)) {
				result = data.map(item => {
					item.checked = false; // 是否被点击
					item.isSelected = selectType; // 是否被选择
					item.isFiltered = false; // 是否被过滤
					return {
						...item,
					};
				});
			}
			return result;
		},
		// 提交
		handleSubmit() {
			if (this.selected.length === 0) return this.$message.warning('请先把用户移动到选中区！');
			this.$emit(
				'subUserOrg',
				this.selected.map(item => item.id)
			);
			this.$emit('subUserOrgObj', this.selected);
			this.closeDio();
		},
        // 去重
		removeDuplicates(data) {
			let result = [];
			let obj = {};
			for (var i = 0; i < data.length; i++) {
				if (!obj[data[i].id]) {
					result.push(data[i]);
					obj[data[i].id] = true;
				}
			}
			return result;
		},
		// 关闭弹窗
		closeDio() {
			this.isDio = false;
			this.unSelectFilterValueOne = '';
			this.selectFilterValueTwo = '';
			this.currentId = '';
			this.list = [];
			lastSelect = [];
			orgIds = new Set();
		},
	},
};
</script>

<style lang="scss" scoped>
.prisoner-manage {
	width: 100%;
	box-sizing: border-box;
}
.transfer-area {
	display: flex;
	justify-content: space-between;
	width: 100%;
	height: 545px;
}
.transfer-content {
	flex: 1;
	overflow-x: hidden;
	overflow-y: auto;
	max-height: 100%;
	border: 1px solid #ebeef5;
}
.transfer-search {
	padding: 10px;
	width: 100%;
	border-bottom: 1px solid #ebeef5;
	box-sizing: border-box;
	&__explain {
		display: flex;
		align-items: center;
		height: 32px;
		font-size: 14px;
		color: #474747;
	}
}
.transfer-btn {
	display: flex;
	flex-direction: column;
	justify-content: center;
	align-items: center;
	padding: 0 20px;
	div {
		margin-top: 10px;
	}
	div:first-child {
		margin: 0;
	}
}
.transfer-list {
	display: flex;
	flex-wrap: wrap;
	justify-content: flex-start;
	padding: 10px 10px 10px 0;
	width: 100%;
	box-sizing: border-box;
	.transfer-list--empty {
		display: flex;
		justify-content: center;
		align-items: center;
		height: calc(100% - 53px);
	}
	.transfer-list__item {
		display: flex;
		flex-direction: column;
		justify-content: center;
		align-items: center;
		margin: 0 0 10px 10px;
		padding: 8px;
		width: 118px;
		font-size: 12px;
		color: #474747;
		border: 1px solid #ebeef5;
		border-radius: 4px;
		box-sizing: border-box;
		&--active {
			color: #fff;
			background-color: #2d72fc;
		}
	}
	&__item:hover {
		cursor: pointer;
		color: #fff;
		background-color: #409eff;
		border-color: #fff;
	}
}
.el-input__icon:hover {
	cursor: pointer;
}
// 滚动条样式
div::-webkit-scrollbar {
	// 滚动条整体部分
	width: 8px;
	height: 8px;
	overscroll-behavior-x: contain;
	overscroll-behavior-y: contain;
}
div::-webkit-scrollbar-track {
	// 滚动条的轨道（里面装有Thumb）
	background: #eee;
	border-radius: 4px;
}
div::-webkit-scrollbar-thumb {
	// 滚动条里面的小方块，能向上向下移动（或往左往右移动，取决于是垂直滚动条还是水平滚动条）
	background: #bbb;
	border-radius: 4px;
}
div::-webkit-scrollbar-corner {
	// 边角，即两个滚动条的交汇处
	background: #179a16;
}
/deep/ .el-dialog .el-dialog__body {
	padding: 0px 0px 0px 0px;
}
</style>

<style lang="scss">
.el-input--small .el-input__icon {
	height: 32px;
	line-height: 32px;
}
</style>
