<template>
  <div class="drop" :name="name" :enter="enter" :leave="leave">
    <slot></slot>
  </div>
</template>

<script>
  import {getGUID} from './util';

  export default {
    name: 'drop',
    props: {
      name: {type: String, default: getGUID()},
      enter: {type: Boolean, default: false},  // 监测dragEnter
      leave: {type: Boolean, default: false}   // 监测dragLeave
    },
    watch: {
      // 在drag组件中，当有碰撞时，修改传入的enter为true，触发dragenter事件
      enter(val) {
        if (val) {
          this.dragenter();
        }
      }
    },
    methods: {
      arrive(element) {
        this.$emit('arrive', element.packet, this);
      },
      away() {
        this.$emit('away', this);
      },
      dragenter() {
        this.$emit('dragenter');
      },
      dragover(event) {
        if (this.enter) {
          // 确保dragOver事件在dragEnter发生后才触发
          // 不然会一有碰撞就触发
          this.$emit('dragover', event);
        }
      },
      dropdown(event) {
        // 从drag组件传入touch事件，以便获取拖动元素的坐标位置
        this.$emit('dropdown', event);
        // 把enter设为false，不然下次拖动同一个元素，会触发dragLeave
        this.$parent.$parent.enter = false;
      },
      dragleave() {
        // dragLeave触发后把enter状态改为fasle，而dragLeave只有在enter状态为真时触发
        // 避免在非碰撞状态下均触发dragLeave事件
        if (this.enter) {
          this.$emit('dragleave');
          this.$parent.$parent.enter = false;
        }
      }
    }
  };
</script>