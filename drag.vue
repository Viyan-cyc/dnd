<template>
  <div class="mobileDrag" :id="dragId">
    <div class="real" ref="real" :style="realStyle" @touchstart="touchStart" @touchmove="touchMove" @touchend="touchEnd">
      <slot></slot>
    </div>
    <div class="duplicate" ref="duplicate" :style="duplicateStyle" v-html="duplicateHTML"></div>
  </div>
</template>

<script>
import {overlap, getGUID} from './util';

export default {
  name: 'drag',

  props: {
    target: {type: String, default: null},
    // packet: {type: Object, default: {}},
    zIndex: {type: Number, default: 100},
    duplicate: {type: Boolean, default: false},
    constantly: {type: Boolean, default: false}
  },

  data() {
    return {
      dragId: getGUID(),
      realStyle: {zIndex: this.zIndex},
      duplicateStyle: {zIndex: this.zIndex},
      duplicateHTML: '',
      dropDown: false,
      timerId: null,
      longClick: false
    };
  },

  watch: {
    duplicateHTML(val) {
      if (val) {
        this.$emit('dragstart');
      }
    }
  },

  methods: {
    setStyle(style = {}) {
      style = Object.assign({}, {zIndex: this.zIndex}, style);
      if (this.duplicate) {
        this.createDuplicate();
        this.duplicateStyle = style;
      } else {
        this.realStyle = style;
      }
    },

    createDuplicate() {
      this.duplicateHTML = this.$refs.real.innerHTML;
    },

    getDropComponents() {
      let dropList = [];
      let filter = (components) => {
        if (!components.length) {
          return;
        }
        for (let i = 0; i < components.length; i++) {
          let child = components[i];
          // 自定义属性
          let options = child.$options;
          // 组件名为drop，且通过props传入的name跟target比较
          if (options._componentTag === 'drop') {
            if (!this.target) {

            } else {
              if (options.propsData && options.propsData.name === this.target) {
                dropList.push(child);
              }
            }
          }
          if (child.$children.length) {
            filter(child.$children);
          }
        }
      };
      // 从根元素开始检测
      filter(this.$root.$children);
      return dropList;
    },

    detectObjective(event) {
      let dropList = [];
      let dom1 = this.duplicate ? this.$refs.duplicate : this.$refs.real;
      let dropComponents = this.getDropComponents();
      dropComponents.forEach(item => {
        // 判断dragDom跟dropComponent是否有重合
        if (overlap(dom1, item.$el)) {
          dropList.push(item);
          // enter初始为false,当有碰撞时修改drop父组件enter为true,标识dragEnter
          if (!item.$parent.$parent.enter) {
            item.$parent.$parent.enter = true;
          }
          // 碰撞情况下，只要还没drop,就是dragOver状态
          if (!this.dropDown) {
            item.dragover(event);
          }
        } else {
          // 只要没有碰撞，那就触发dragLeave事件
          item.dragleave();
        }
        // 还没有drop时,先调用away，表示dragDom到达该组件但又离开了
        // item.away();
        if (!event.targetTouches) {
          item.$parent.$parent.enter = false;
        }
      });
      let drop = dropList.length ? dropList[0] : null;
      if (drop) {
        // 选择第一个drop组件，触发arrive
        // drop.arrive(this);
        // 检测到dropDown为true时，触发dropDown事件
        if (this.dropDown) {
          drop.dropdown(event);
        }
      }
      return drop;
    },

    touchStart(event) {
      event.preventDefault();
      this.duplicateHTML = '';
      this.timerId = setTimeout(() => {
        this.longClick = true;
      }, 400);
    },

    touchMove(event) {
      event.preventDefault();
      clearTimeout(this.timerId);
      this.timerId = null;
      if (this.longClick) {
        return;
      }
      if (event.targetTouches.length === 1) {
        let touch = event.targetTouches[0];
        let target = event.target;
        // if (this.duplicate) {
        //   this.createDuplicate();
        // }
        this.setStyle({
          position: 'fixed',
          display: 'block',
          opacity: '.6',
          top: `${touch.pageY - window.scrollY - target.offsetHeight / 2}px`,
          left: `${touch.pageX - window.scrollX - target.offsetWidth / 2}px`,
          width: `${target.offsetWidth}px`,
          height: `${target.offsetHeight}px`
        });
        if (this.constantly) {
          this.detectObjective(event);
        }
      }
    },

    touchEnd(event) {
      clearTimeout(this.timerId);
      this.longClick = false;
      this.setStyle();
      this.dropDown = true;
      // 监测duplicateHTML的值，当其从无到有时，触发dragStart
      // touchEnd时置为空，避免下次拖动同一个元素时，不会触发dragStart
      this.duplicateHTML = '';
      // 返回touchEvent，当前drag组件，如已经到达目标组件返回目标组件
      // 触发dragEnd之前，会调用detectObjective，此时dropDown为true，所以先触发drop事件
      this.detectObjective(event);
      this.$emit('dragend', event, this);

      // dragEnd触发后，把dropDown还原为初始false值，不然下次拖动同一元素时，会立马触发dropDown事件
      this.dropDown = false;
    }
  }
};
</script>

<style lang="scss">
  .drag {
    .duplicate {
      display: none;
    }
  }
</style>