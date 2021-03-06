<template>
  <transition-group tag="div" name="lazy-component" class="main-container">
    <div v-if="visible" key="component">
      <slot :loading="loading"></slot>
    </div>
    <Skeleton v-else></Skeleton>
  </transition-group>
</template>
<script>
import Skeleton from './Skeleton.vue'
export default {
  name: 'VueLazyView',
  props: {
    show: {
      type: Boolean,
      default: false,
    },
    timeout: {
      type: Number,
    },
    threshold: {
      type: String,
      default: '0px',
    },
    direction: {
      type: String,
      default: 'vertical',
    },
    maxWaitingTime: {
      type: Number,
      default: 50,
    },
  },
  watch: {
    show: {
      handler(n) {
        if (n === true) {
          this.init()
        }
      },
      immediate: true,
    },
  },
  data() {
    return {
      visible: false,
      timer: null,
      io: null,
      loading: false,
    }
  },
  components: {
    Skeleton,
  },
  created() {
    // 如果指定timeout则无论可见与否都是在timeout之后初始化
    if (this.timeout) {
      this.timer = setTimeout(() => {
        this.init()
      }, this.timeout)
    }
  },
  mounted() {
    if (!this.timeout && this.show) {
      // 根据滚动方向来构造视口外边距，用于提前加载
      let rootMargin
      switch (this.direction) {
        case 'vertical':
          rootMargin = `${this.threshold} 0px`
          break
        case 'horizontal':
          rootMargin = `0px ${this.threshold}`
          break
      }
      try {
        // 观察视口与组件容器的交叉情况
        this.io = new window.IntersectionObserver(this.intersectionHandler, {
          rootMargin,
          threshold: [0, Number.MIN_VALUE, 0.01],
        })
        this.io.observe(this.$el)
      } catch (e) {
        this.init()
      }
    }
  },
  beforeDestroy() {
    // 在组件销毁前取消观察
    if (this.io) {
      this.io.unobserve(this.$el)
    }
  },
  methods: {
    // 交叉情况变化处理函数
    intersectionHandler(entries) {
      if (
        // 正在交叉
        entries[0].isIntersecting ||
        // 交叉率大于0
        entries[0].intersectionRatio
      ) {
        this.init()
        this.io.unobserve(this.$el)
      }
    },
    // 处理组件和骨架组件的切换
    init() {
      // 此时可以准备加载懒加载组件的资源
      this.loading = true
      this.$emit('initDone', true)
      // 由于函数会在主线程中执行，加载懒加载组件非常耗时，容易卡顿
      // 所以在requestAnimationFrame回调中延后执行
      this.requestAnimationFrame(() => {
        this.visible = true
      })
    },
    requestAnimationFrame(callback) {
      // 防止等待太久没有执行回调
      // 设置最大等待时间
      setTimeout(() => {
        if (this.visible) {
          return
        }
        callback()
      }, this.maxWaitingTime)
      // 兼容不支持requestAnimationFrame 的浏览器
      return (
        window.requestAnimationFrame ||
        ((callback) => setTimeout(callback, 1000 / 60))
      )(callback)
    },
  },
}
</script>
<style lang="scss" scoped>
.lazy-component-enter {
  opacity: 0;
}
.lazy-component-enter-to {
  opacity: 1;
}
.lazy-component-enter-active {
  transition: opacity 0.25s;
}
.lazy-component-leave {
  opacity: 1;
}
.lazy-component-leave-to {
  opacity: 0;
}
.lazy-component-leave-active {
  transition: opacity 0.5s;
}
</style>
