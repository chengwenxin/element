<template>
  <div class="el-transfer-panel">
    <p class="el-transfer-panel__header">
      <el-checkbox
        v-model="allChecked"
        @change="handleAllCheckedChange"
        :indeterminate="isIndeterminate"
      >
        {{ title }}
        <span>{{ checkedSummary }}</span>
      </el-checkbox>
    </p>

    <div :class="['el-transfer-panel__body', hasFooter ? 'is-with-footer' : '']">
      <el-input
        class="el-transfer-panel__filter"
        v-model="query"
        size="small"
        :placeholder="placeholder"
        @mouseenter.native="inputHover = true"
        @mouseleave.native="inputHover = false"
        v-if="filterable"
      >
        <i slot="prefix" :class="['el-input__icon', 'el-icon-' + inputIcon]" @click="clearQuery"></i>
      </el-input>
      <el-checkbox-group
        v-infinite-scroll="loadMore"
        :infinite-scroll-immediate-check="false"
        v-model="checked"
        v-show="!hasNoMatch && data.length > 0"
        :class="{ 'is-filterable': filterable }"
        class="el-transfer-panel__list"
      >
        <template v-for="i in chunkNumber">
          <template v-if="i <= currentChunk">
            <el-checkbox
              class="el-transfer-panel__item"
              :label="item[keyProp]"
              :disabled="item[disabledProp]"
              :key="item[keyProp]"
              v-for="item in filteredData.slice(chunkSize*(i-1),chunkSize*i)"
            >
              <option-content :option="item"></option-content>
            </el-checkbox>
          </template>
        </template>
      </el-checkbox-group>
      <p class="el-transfer-panel__empty" v-show="hasNoMatch">{{ t('el.transfer.noMatch') }}</p>
      <p
        class="el-transfer-panel__empty"
        v-show="data.length === 0 && !hasNoMatch"
      >{{ t('el.transfer.noData') }}</p>
    </div>
    <p class="el-transfer-panel__footer" v-if="hasFooter">
      <slot></slot>
    </p>
  </div>
</template>

<script>
import ElCheckboxGroup from 'element-ui/packages/checkbox-group';
import ElCheckbox from 'element-ui/packages/checkbox';
import ElInput from 'element-ui/packages/input';
import Locale from 'element-ui/src/mixins/locale';
import InfiniteScroll from 'element-ui/src/directives/infinite-scroll';
export default {
  mixins: [Locale],

  name: 'ElTransferPanel',

  componentName: 'ElTransferPanel',
  directives: {
    infiniteScroll: InfiniteScroll
  },

  components: {
    ElCheckboxGroup,
    ElCheckbox,
    ElInput,
    // 确定是否嵌套在其它组件中
    OptionContent: {
      props: {
        option: Object
      },
      /* $options可获取vm实例中自定义属性 */
      render(h) {
        const getParent = vm => {
          if (vm.$options.componentName === 'ElTransferPanel') {
            return vm;
          } else if (vm.$parent) {
            return getParent(vm.$parent);
          } else {
            return vm;
          }
        };
        // 骚操作  似懂非懂
        const panel = getParent(this);
        const transfer = panel.$parent || panel;
        return panel.renderContent ? (
          panel.renderContent(h, this.option)
        ) : transfer.$scopedSlots.default ? (
          transfer.$scopedSlots.default({ option: this.option })
        ) : (
          <span>
            {this.option[panel.labelProp] || this.option[panel.keyProp]}
          </span>
        );
      }
    }
  },

  props: {
    // 所有可选数据
    data: {
      type: Array,
      default() {
        return [];
      }
    },
    // 内容渲染函数
    renderContent: Function,
    // 常规属性
    placeholder: String,
    title: String,
    filterable: Boolean,
    // 是否checked
    format: Object,
    // 搜索函数
    filterMethod: Function,
    // 默认已选的值
    defaultChecked: Array,
    // 其他用户绑定的属性
    props: Object
  },

  data() {
    return {
      // 选中的数组
      checked: [],
      // 全选按钮
      allChecked: false,
      // 搜索框查询字段
      query: '',
      // 鼠标悬停事件
      inputHover: false,
      // 用户手动改变 （区别于首次渲染）
      checkChangeByUser: true,
      currentChunk: 1,
      chunkSize: 10
    };
  },

  watch: {
    // 选中
    checked(val, oldVal) {
      this.updateAllChecked();
      if (this.checkChangeByUser) {
        const movedKeys = val
          .concat(oldVal)
          .filter(v => val.indexOf(v) === -1 || oldVal.indexOf(v) === -1);
        this.$emit('checked-change', val, movedKeys);
      } else {
        // 第一次移动到右边
        this.$emit('checked-change', val);
        this.checkChangeByUser = true;
      }
    },

    data() {
      const checked = [];
      const filteredDataKeys = this.filteredData.map(
        item => item[this.keyProp]
      );
      this.checked.forEach(item => {
        if (filteredDataKeys.indexOf(item) > -1) {
          checked.push(item);
        }
      });
      // 首次
      this.checkChangeByUser = false;
      this.checked = checked;
    },

    checkableData() {
      this.updateAllChecked();
    },

    // 没有用到
    defaultChecked: {
      immediate: true,
      handler(val, oldVal) {
        if (
          oldVal &&
          val.length === oldVal.length &&
          val.every(item => oldVal.indexOf(item) > -1)
        ) {return;}
        const checked = [];
        const checkableDataKeys = this.checkableData.map(
          item => item[this.keyProp]
        );
        val.forEach(item => {
          if (checkableDataKeys.indexOf(item) > -1) {
            checked.push(item);
          }
        });
        this.checkChangeByUser = false;
        this.checked = checked;
      }
    }
  },

  computed: {
    chunkNumber() {
      let len = this.filteredData.length;
      let number = Math.ceil(len / this.chunkSize);
      if (this.currentChunk !== 1 && number < this.currentChunk) {
        this.currentChunk = number;
      }
      return number;
    },
    // 搜索过滤后的值
    filteredData() {
      return this.data.filter(item => {
        if (typeof this.filterMethod === 'function') {
          return this.filterMethod(this.query, item);
        } else {
          const label = item[this.labelProp] || item[this.keyProp].toString();
          return label.toLowerCase().indexOf(this.query.toLowerCase()) > -1;
        }
      });
    },
    // 遍历disabled属性
    checkableData() {
      return this.filteredData.filter(item => !item[this.disabledProp]);
    },
    // 更新选中与未选中数量
    checkedSummary() {
      const checkedLength = this.checked.length;
      const dataLength = this.data.length;
      const { noChecked, hasChecked } = this.format;
      if (noChecked && hasChecked) {
        return checkedLength > 0
          ? hasChecked
            .replace(/\${checked}/g, checkedLength)
            .replace(/\${total}/g, dataLength)
          : noChecked.replace(/\${total}/g, dataLength);
      } else {
        return `${checkedLength}/${dataLength}`;
      }
    },
    // 全选的不确定状态
    isIndeterminate() {
      const checkedLength = this.checked.length;
      return checkedLength > 0 && checkedLength < this.checkableData.length;
    },
    // 没有匹配
    hasNoMatch() {
      return this.query.length > 0 && this.filteredData.length === 0;
    },

    inputIcon() {
      return this.query.length > 0 && this.inputHover
        ? 'circle-close'
        : 'search';
    },

    labelProp() {
      return this.props.label || 'label';
    },

    keyProp() {
      return this.props.key || 'key';
    },

    disabledProp() {
      return this.props.disabled || 'disabled';
    },

    hasFooter() {
      return !!this.$slots.default;
    }
  },

  methods: {
    loadMore() {
      if (this.currentChunk < this.chunkNumber) {
        this.currentChunk++;
      }
      console.log(this.currentChunk);
    },
    // 更新全选的状态
    updateAllChecked() {
      const checkableDataKeys = this.checkableData.map(
        item => item[this.keyProp]
      );
      this.allChecked =
        checkableDataKeys.length > 0 &&
        checkableDataKeys.every(item => this.checked.indexOf(item) > -1);
    },
    // 全选与反选
    handleAllCheckedChange(value) {
      this.checked = value
        ? this.checkableData.map(item => item[this.keyProp])
        : [];
    },

    clearQuery() {
      if (this.inputIcon === 'circle-close') {
        this.query = '';
      }
    }
  }
};
</script>
