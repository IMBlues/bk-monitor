<!--
  - Tencent is pleased to support the open source community by making BK-LOG 蓝鲸日志平台 available.
  - Copyright (C) 2021 THL A29 Limited, a Tencent company.  All rights reserved.
  - BK-LOG 蓝鲸日志平台 is licensed under the MIT License.
  -
  - License for BK-LOG 蓝鲸日志平台:
  - -------------------------------------------------------------------
  -
  - Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated
  - documentation files (the "Software"), to deal in the Software without restriction, including without limitation
  - the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software,
  - and to permit persons to whom the Software is furnished to do so, subject to the following conditions:
  - The above copyright notice and this permission notice shall be included in all copies or substantial
  - portions of the Software.
  -
  - THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT
  - LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN
  - NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY,
  - WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE
  - SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE
  -->

<template>
  <div>
    <div class="log-cluster-table-container" v-if="!globalLoading">
      <div
        class="cluster-nav"
        v-if="exhibitAll"
        data-test-id="cluster_div_fingerOperate">
        <div class="bk-button-group">
          <bk-button
            size="small"
            v-for="(item) of clusterNavList"
            :key="item.id"
            :class="active === item.id ? 'is-selected' : ''"
            @click="handleClickNav(item.id)">
            {{item.name}}
          </bk-button>
        </div>

        <finger-operate
          v-if="isFingerNav"
          :total-fields="totalFields"
          :finger-operate-data="fingerOperateData"
          :request-data="requestData"
          @handleFingerOperate="handleFingerOperate" />
      </div>

      <bk-alert
        v-if="isFingerNav && signatureSwitch && !exhibitAll"
        :title="$t('日志聚类必需至少有一个text类型的字段，当前无该字段类型，请前往日志清洗进行设置。')"
        closable
        type="info">
      </bk-alert>

      <div v-if="exhibitAll">
        <clustering-loader
          is-loading
          v-if="tableLoading"
          :width-list="smallLoaderWidthList" />
        <div v-else>
          <ignore-table
            v-if="active === 'ignoreNumbers' || active === 'ignoreSymbol'"
            v-bind="$attrs"
            v-on="$listeners"
            :retrieve-params="retrieveParams"
            :total-fields="totalFields"
            :origin-table-list="originTableList"
            :clustering-field="clusteringField"
            :active="active" />
          <data-fingerprint
            v-if="isFingerNav"
            v-bind="$attrs"
            v-on="$listeners"
            ref="fingerTableRef"
            :cluster-switch="clusterSwitch"
            :request-data="requestData"
            :config-data="configData"
            :finger-list="fingerList"
            :is-page-over="isPageOver"
            :all-finger-list="allFingerList"
            :loader-width-list="smallLoaderWidthList"
            @paginationOptions="paginationOptions"
            @updateRequest="requestFinger"
            @handleScrollIsShow="handleScrollIsShow" />
        </div>
      </div>

      <bk-table
        v-else
        class="no-text-table"
        :data="[]">
        <div slot="empty">
          <empty-status class="empty-text" empty-type="empty" :show-text="false">
            <p v-if="indexSetItem.scenario_id !== 'log' && !isHaveAnalyzed">
              <i18n path="无分词字段 请前往 {0} 调整清洗">
                <span class="empty-leave" @click="handleLeaveCurrent">{{$t('计算平台')}}</span>
              </i18n>
            </p>
            <div v-else>
              <p>{{exhibitText}}</p>
              <span class="empty-leave" @click="handleLeaveCurrent">
                {{exhibitOperate}}
              </span>
            </div>
          </empty-status>
        </div>
      </bk-table>

      <div class="fixed-scroll-top-btn" v-show="showScrollTop" @click="scrollToTop">
        <i class="bk-icon icon-angle-up"></i>
      </div>
    </div>
    <clustering-loader
      is-loading
      v-else
      :width-list="loadingWidthList.global" />
  </div>
</template>

<script>
import DataFingerprint from './data-fingerprint';
import IgnoreTable from './ignore-table';
import ClusteringLoader from '@/skeleton/clustering-loader';
import FingerOperate from './components/finger-operate';
import { mapGetters } from 'vuex';
import EmptyStatus from '@/components/empty-status';

export default {
  components: {
    DataFingerprint,
    IgnoreTable,
    ClusteringLoader,
    FingerOperate,
    EmptyStatus,
  },
  props: {
    retrieveParams: {
      type: Object,
      required: true,
    },
    configData: {
      type: Object,
      require: true,
    },
    cleanConfig: {
      type: Object,
      require: true,
    },
    totalFields: {
      type: Array,
      require: true,
    },
    originTableList: {
      type: Array,
      required: true,
    },
    indexSetItem: {
      type: Object,
      required: true,
    },
    clusterRouteParams: {
      type: Object,
      required: true,
    },
    isChangeTableNav: {
      type: Boolean,
      default: false,
    },
    isThollteField: {
      type: Boolean,
      required: true,
    },
    fingerSearchState: {
      type: Boolean,
      required: true,
    }
  },
  data() {
    return {
      active: 'ignoreNumbers',
      isClickFingerNav: false, // 是否点击过数据指纹nav
      tableLoading: false, // 详情loading
      isShowCustomize: true, // 是否显示自定义
      clusterNavList: [{
        id: 'ignoreNumbers',
        name: this.$t('忽略数字'),
      }, {
        id: 'ignoreSymbol',
        name: this.$t('忽略符号'),
      }, {
        id: 'dataFingerprint',
        name: this.$t('button-数据指纹').replace('button-', ''),
      }],
      fingerOperateData: {
        patternSize: 0, // slider当前值
        sliderMaxVal: 0, // pattern最大值
        comparedList: [], // 同比List
        patternList: [], // pattern敏感度List
        isShowCustomize: true, // 是否显示自定义
        signatureSwitch: false, // 数据指纹开关
        groupList: [], // 缓存分组列表
        alarmObj: {}, // 是否需要告警对象
      },
      requestData: { // 数据请求
        pattern_level: '',
        year_on_year_hour: 0,
        show_new_pattern: false,
        group_by: [],
        size: 10000,
      },
      isPageOver: false,
      fingerPage: 1,
      fingerPageSize: 50,
      loadingWidthList: { // loading表头宽度列表
        global: [''],
        ignore: [60, 90, 90, ''],
        notCompared: [150, 90, 90, ''],
        compared: [150, 90, 90, 100, 100, ''],
      },
      fingerList: [],
      allFingerList: [], // 所有数据指纹List
      showScrollTop: false, // 是否展示返回顶部icon
      throttle: false, // 请求防抖
      isInitPage: true, // 是否是第一次进入数据指纹
    };
  },
  computed: {
    ...mapGetters({
      globalsData: 'globals/globalsData',
    }),
    smallLoaderWidthList() {
      if (!this.isFingerNav) return this.loadingWidthList.ignore;
      return this.requestData.year_on_year_hour > 0
        ? this.loadingWidthList.compared
        : this.loadingWidthList.notCompared;
    },
    exhibitText() {
      return this.configID ? this.$t('当前无可用字段，请前往日志清洗进行设置') : this.$t('当前索引集不支持字段提取设置');
    },
    exhibitOperate() {
      return this.configID ? this.$t('跳转到日志清洗') : '';
    },
    clusteringField() {
      // 如果有聚类字段则使用设置的
      if (this.configData?.extra?.clustering_field) return this.configData.extra.clustering_field;
      // 如果有log字段则使用log类型字段
      const logFieldItem = this.totalFields.find(item => item.field_name === 'log');
      if (logFieldItem) return logFieldItem.field_name;
      // 如果没有设置聚类字段和log字段则使用text列表里的第一项值
      const textTypeFieldList = this.totalFields.filter(item => item.is_analyzed) || [];
      if (textTypeFieldList.length) return textTypeFieldList[0].field_name;
      return  '';
    },
    bkBizId() {
      return this.$store.state.bkBizId;
    },
    isHaveAnalyzed() {
      return this.totalFields.some(item => item.is_analyzed);
    },
    signatureSwitch() { // 数据指纹开关
      return this.configData.extra?.signature_switch;
    },
    configID() {
      return this.cleanConfig.extra?.collector_config_id;
    },
    /** 日志聚类开关 */
    clusterSwitch() {
      return this.configData?.is_active;
    },
    isFingerNav() {
      return this.active === 'dataFingerprint';
    },
    exhibitAll() {
      /**
       *  无字段提取或者聚类开关没开时直接不显示聚类nav和table
       *  来源如果是数据平台并且日志聚类大开关有打开则进入text判断
       *  有text则提示去开启日志聚类 无则显示跳转计算平台
       */
      return this.totalFields.some(el => el.field_type === 'text');
    },
    globalLoading() {
      // 判断是否可以字段提取的全局loading
      return this.isThollteField;
    },
  },
  watch: {
    configData: {
      deep: true,
      immediate: true,
      handler() {
        this.isClickFingerNav = false;
        // 数据指纹开关赋值
        this.fingerOperateData.signatureSwitch = this.signatureSwitch;
        // 当前nav为数据指纹且数据指纹开启点击指纹nav则不再重复请求
        if (this.isFingerNav) {
          this.fingerList = [];
          this.allFingerList = [];
        };
      },
    },
    totalFields: {
      deep: true,
      immediate: true,
      handler(newList) {
        if (newList.length) {
          /**
           *  无字段提取或者聚类开关没开时直接不显示聚类nav和table
           *  来源如果是数据平台并且日志聚类大开关有打开则进入text判断
           *  有text则提示去开启日志聚类 无则显示跳转计算平台
           */
          // 初始化分组下拉列表
          this.filterGroupList();
          this.initTable();
        }
      },
    },
    fingerSearchState: {
      immediate: true,
      handler() {
        if (this.exhibitAll) this.requestFinger();
      }
    },
    isChangeTableNav(val) {
      // 若数据指纹开启则自动显示数据指纹
      if (val && this.signatureSwitch) {
        this.active = 'dataFingerprint';
        this.$emit('update:is-change-table-nav', false);
      };
    },
  },
  methods: {
    handleClickNav(id) {
      this.active = id;
      // 切换聚类的nav 缓存聚类params
      this.$emit('backFillClusterRouteParams', 'clustering', {
        ...this.clusterRouteParams,
        activeNav: this.active,
      });
      if (!this.isClickFingerNav && id === 'dataFingerprint') {
        this.isClickFingerNav = true;
        this.requestFinger();
      }
    },
    initTable() {
      const {
        log_clustering_level_year_on_year: yearOnYearList,
        log_clustering_level: clusterLevel,
      } = this.globalsData;
      let patternLevel;
      if (clusterLevel && clusterLevel.length > 0) {
        // 判断奇偶数来取pattern中间值
        if (clusterLevel.length % 2 === 1) {
          patternLevel = (clusterLevel.length + 1) / 2;
        } else {
          patternLevel = clusterLevel.length  / 2;
        }
      };
      const patternList = clusterLevel.sort((a, b) => Number(b) - Number(a));
      const queryRequestData = { pattern_level: clusterLevel[patternLevel - 1] };
      const hasNoRouteValue = JSON.stringify(this.clusterRouteParams) === '{}';
      // 通过路由返回的值 初始化数据指纹的操作参数
      if (this.isInitPage && !hasNoRouteValue) {
        this.active = this.clusterRouteParams.activeNav;
        const paramData = this.clusterRouteParams.requestData;
        const findIndex = clusterLevel.findIndex(item => item === String(paramData.pattern_level));
        if (findIndex >= 0) patternLevel = findIndex + 1;
        Object.assign(queryRequestData, paramData, {
          pattern_level: paramData.pattern_level ? paramData.pattern_level : clusterLevel[patternLevel - 1],
        });
      };
      Object.assign(this.fingerOperateData, {
        patternSize: patternLevel - 1,
        sliderMaxVal: clusterLevel.length - 1,
        patternList,
        comparedList: yearOnYearList,
      });
      Object.assign(this.requestData, queryRequestData);
      if (this.isInitPage) {
        // 初始化nav如果是数据指纹 且打开数据指纹 则初始化时请求一次数据指纹
        if (this.signatureSwitch && hasNoRouteValue) {
          this.active = 'dataFingerprint';
          this.isClickFingerNav = true;
        }
      }
      this.isInitPage = false;
      this.$nextTick(() => {
        this.scrollEl = document.querySelector('.result-scroll-container');
      });
    },
    /**
     * @desc: 数据指纹操作
     * @param { String } operateType 操作类型
     * @param { Any } val 具体值
     */
    handleFingerOperate(operateType, val, isQuery = false) {
      switch (operateType) {
        case 'compared': // 同比操作
          this.requestData.year_on_year_hour = val;
          break;
        case 'patternSize': // patter大小
          this.requestData.pattern_level = val;
          break;
        case 'isShowNear': // 是否展示近24小时
          this.requestData.show_new_pattern = val;
          break;
        case 'enterCustomize': // 自定义同比时常
          this.handleEnterCompared(val);
          break;
        case 'customize': // 是否展示自定义
          this.fingerOperateData.isShowCustomize = val;
          break;
        case 'group': {
          // 分组操作
          const groupIdsStr = this.requestData.group_by.join(',');
          const operateIDs = val.join(',');
          if (operateIDs !== groupIdsStr) this.requestData.group_by = val;
        }
          break;
        case 'getNewStrategy': // 获取新类告警状态
          this.fingerOperateData.alarmObj = val;
          break;
        case 'editAlarm': { // 更新新类告警请求
          const { alarmObj: { strategy_id: strategyID } } = this.fingerOperateData;
          if (strategyID) {
            this.$refs.fingerTableRef.policyEditing(strategyID);
          }
        }
          break;
      };
      // 数据指纹的操作回填到url上
      if (['compared', 'patternSize', 'isShowNear', 'group'].includes(operateType)) {
        this.$emit('backFillClusterRouteParams', 'clustering', {
          activeNav: this.active,
          requestData: this.requestData,
        });
      }
      if (isQuery) this.requestFinger();
    },
    handleLeaveCurrent() {
      // 不显示字段提取时跳转计算平台
      if (this.indexSetItem.scenario_id !== 'log' && !this.isHaveAnalyzed) {
        const jumpUrl = `${window.BKDATA_URL}`;
        window.open(jumpUrl, '_blank');
        return;
      };
      // 无清洗 去清洗
      if (!!this.configID) {
        this.$router.push({
          name: 'clean-edit',
          params: { collectorId: this.configID },
          query: {
            spaceUid: this.$store.state.spaceUid,
            backRoute: this.$route.name,
          },
        });
      }
    },
    /**
     * @desc: 同比自定义输入
     * @param { String } val
     */
    handleEnterCompared(val) {
      const matchVal = val.match(/^(\d+)h$/);
      if (!matchVal) {
        this.$bkMessage({
          theme: 'warning',
          message: this.$t('请按照提示输入'),
        });
        return;
      }
      this.fingerOperateData.isShowCustomize = true;
      const isRepeat = this.fingerOperateData.comparedList.some(el => el.id === Number(matchVal[1]));
      if (isRepeat) {
        this.requestData.year_on_year_hour = Number(matchVal[1]);
        return;
      }
      this.fingerOperateData.comparedList.push({
        id: Number(matchVal[1]),
        name: this.$t('{n} 小时前', { n: matchVal[1] }),
      });
      this.requestData.year_on_year_hour = Number(matchVal[1]);
    },
    /**
     * @desc: 数据指纹请求
     */
    requestFinger() {
      if (this.throttle || !this.signatureSwitch) return;

      this.throttle = true;
      this.tableLoading = true;
      this.$http.request('/logClustering/clusterSearch', {
        params: {
          index_set_id: this.$route.params.indexId,
        },
        data: {
          ...this.retrieveParams,
          ...this.requestData,
        },
      }, { cancelWhenRouteChange: false }) // 由于回填指纹的数据导致路由变化，故路由变化时不取消请求
        .then((res) => {
          this.fingerPage = 1;
          this.fingerList = [];
          this.allFingerList = res.data;
          const sliceFingerList = res.data.slice(0, this.fingerPageSize);
          this.fingerList.push(...sliceFingerList);
          this.showScrollTop = false;
        })
        .finally(() => {
          this.tableLoading = false;
          this.isClickFingerNav = true;
        });

      setTimeout(() => {
        this.throttle = false;
      }, 500);
    },
    /**
     * @desc: 数据指纹分页操作
     */
    async paginationOptions() {
      if (this.isPageOver || this.fingerList.length >= this.allFingerList.length) {
        return;
      }
      this.isPageOver = true;
      this.fingerPage += 1;
      const { fingerPage: page, fingerPageSize: pageSize } = this;
      const sliceFingerList = this.allFingerList.slice(pageSize * (page - 1), pageSize * page);
      setTimeout(() => {
        this.fingerList.push(...sliceFingerList);
        this.isPageOver = false;
      }, 300);
    },
    /**
     * @desc: 初始化分组select数组
     */
    filterGroupList() {
      const filterList = this.totalFields
        .filter(el => el.es_doc_values && !/^__dist/.test(el.field_name)) // 过滤__dist字段
        .map((item) => {
          const { field_name: id, field_alias: alias } = item;
          return { id, name: alias ? `${id}(${alias})` : id };
        });
      this.fingerOperateData.groupList = filterList;
      this.requestData.group_by.splice(0, this.requestData.group_by.length);
    },
    scrollToTop() {
      this.$easeScroll(0, 300, this.scrollEl);
    },
    handleScrollIsShow() {
      this.showScrollTop = this.scrollEl.scrollTop > 550;
    },
  },
};
</script>

<style lang="scss">
@import '@/scss/mixins/flex.scss';

.log-cluster-table-container {
  overflow: hidden;

  .cluster-nav {
    margin-bottom: 12px;
    color: #63656e;
    flex-wrap: nowrap;

    @include flex-justify(space-between);
  }

  .bk-button-group {
    flex-shrink: 0;
  }

  .bk-alert {
    margin-bottom: 16px;
  }
}

.no-text-table {
  .bk-table-empty-block {
    display: flex;
    justify-content: center;
    align-items: center;
    min-height: calc(100vh - 480px);
  }

  .empty-text {
    display: flex;
    flex-direction: column;
    justify-content: space-between;
    align-items: center;

    .bk-icon {
      font-size: 65px;
    }

    .empty-leave {
      color: #3a84ff;
      margin-top: 8px;
      cursor: pointer;
    }
  }
}

.fixed-scroll-top-btn {
  position: fixed;
  bottom: 24px;
  right: 14px;
  display: flex;
  justify-content: center;
  align-items: center;
  width: 36px;
  height: 36px;
  box-shadow: 0 1px 2px 0 rgba(0, 0, 0, .2);
  border: 1px solid #dde4eb;
  border-radius: 4px;
  color: #63656e;
  background: #f0f1f5;
  cursor: pointer;
  z-index: 2100;
  transition: all .2s;

  &:hover {
    color: #fff;
    background: #979ba5;
    transition: all .2s;
  }

  .bk-icon {
    font-size: 20px;
    font-weight: bold;
  }
}
</style>
