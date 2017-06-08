<template>
  <div class="vue-zoom-container" v-el:v-zoom-container>
    <div class="vue-zoom-drag-container grab" :class="{ 'grabbing': dragState }" :style="dragStyle" v-el:v-zoom-drag-container>
      <div class="vue-zoom-outer" :style="outerStyle">
        <div class="vue-zoom-inner" v-el:v-zoom-inner :style="innerStyle">
          <slot></slot>
        </div>
      </div>
    </div>
    <div class="vue-zoom-tools-container">
      <a class="vue-zoom-tools-btn" v-for="(index, btn) in toolsBtns" @click.stop="getZoomHandler(btn.actionName)" :title="btn.title" :disabled="toolsBtnStates[state.state][btn.actionName]">
        <i :class="btn.iconClass"></i>
      </a>
    </div>
  </div>
</template>

<script>
import wheelevent from './assets/addWheelListener'
import $ from 'jquery'
import 'jquery-ui/ui/widgets/draggable'

import { pageSizeChange } from '../../../vuex/getters/page-getters'

export default {
  data () {
    return {
      scale: 1,
      suitScale: 1,
      targetTop: 0,
      targetLeft: 0,
      jQTarget: null,
      jQContainer: null,
      jQDragContainer: null,
      targetStyle: {
        width: 0,
        height: 0
      },
      containerStyle: {
        width: 0,
        height: 0,
        top: 0,
        left: 0,
        outerWidth: 0,
        outerHeight: 0
      },
      dragState: false,
      centerPosition: null,
      state: {
        scale: 1,
        state: 'def'
      },
      toolsBtnStates: {
        max: {
          zoomToMax: true,
          zoomIn: true,
          zoomOut: false,
          zoomToSuitable: false
        },
        min: {
          zoomToMax: false,
          zoomIn: false,
          zoomOut: true,
          zoomToSuitable: false
        },
        def: {
          zoomToMax: false,
          zoomIn: false,
          zoomOut: false,
          zoomToSuitable: false
        }
      },
      toolsBtns: [
        {
          'actionName': 'zoomToMax',
          'title': '最大化',
          'iconClass': 'icon-enlarge',
          'disabled': ''
        },
        {
          'actionName': 'zoomIn',
          'title': '放大',
          'iconClass': 'icon-plus',
          'disabled': ''
        },
        {
          'actionName': 'zoomOut',
          'title': '缩小',
          'iconClass': 'icon-minus',
          'disabled': ''
        },
        {
          'actionName': 'zoomToSuitable',
          'title': '适屏',
          'iconClass': 'icon-shrink',
          'disabled': ''
        }
      ]
    }
  },
  props: {
    step: {
      required: false,
      type: Number,
      default: 0.05
    },
    max: {
      required: false,
      type: Number,
      default: 2
    },
    min: {
      required: false,
      type: Number,
      default: 0.4
    },
    afterZoom: {
      required: false,
      type: Function,
      default () {
        return function () {}
      }
    },
    cancel: {
      required: false,
      type: String,
      default: ''
    }
  },
  computed: {
    dragStyle () {
      let _top = this.containerStyle.height / 2 + this.targetStyle.height / 2
      let _left = this.containerStyle.width / 2 + this.containerStyle.width / 2
      return {
        'padding': `${_top}px ${_left}px`,
        'margin-top': `${-_top}px`,
        'margin-left': `${-_left}px`
      }
    },
    outerStyle () {
      return {
        'width': `${this.targetStyle.width}px`,
        'height': `${this.targetStyle.height}px`
      }
    },
    innerStyle () {
      return {
        'top': `${this.targetTop}px`,
        'left': `${this.targetLeft}px`,
        'transform': `scale(${this.scale})`
      }
    }
  },
  ready () {
    this.jQTarget = $(this.$els.vZoomInner)
    this.jQContainer = $(this.$els.vZoomContainer)
    this.jQDragContainer = $(this.$els.vZoomDragContainer)
    this.initZoomer()
    this.initDraggable()
    this.initWheelListener()

    this.$nextTick(() => {
      this.zoomToSuitable(true)
    })
  },
  methods: {
    initZoomer () {
      this.setTargetStyle()
      this.setContainerStyle()
      this.setSuitScale()
    },
    setSuitScale () {
      var suitScale = Math.min(this.containerStyle.width / this.targetStyle.outerWidth, this.containerStyle.height / this.targetStyle.outerHeight)
      suitScale = suitScale < 1 ? suitScale : 1
      this.suitScale = this.getFixedScale(suitScale)
    },
    setTargetStyle () {
      this.targetStyle.width = this.$els.vZoomInner.clientWidth
      this.targetStyle.height = this.$els.vZoomInner.clientHeight
      this.targetStyle.outerWidth = this.jQTarget.outerWidth(false)
      this.targetStyle.outerHeight = this.jQTarget.outerHeight(false)
    },
    setContainerStyle () {
      this.containerStyle.width = this.$els.vZoomContainer.clientWidth
      this.containerStyle.height = this.$els.vZoomContainer.clientHeight
      this.containerStyle.top = this.jQContainer.offset().top
      this.containerStyle.left = this.jQContainer.offset().left
      this.containerStyle.outerWidth = this.jQContainer.outerWidth(false)
      this.containerStyle.outerHeight = this.jQContainer.outerHeight(false)
      this.centerPosition = this.getCenterPosition()
    },
    initWheelListener () {
      wheelevent(window, document)
      window.addWheelListener(this.$els.vZoomContainer, $.proxy(this.wheelZoom, this), false)
    },
    initDraggable () {
      var self = this
      this.jQDragContainer.draggable({
        scroll: false,
        cancel: self.cancel,
        start: function (event) {
          self.dragState = true
        },
        drag: $.proxy(function (event) {
        }, self),
        stop: function (event) {
          self.dragState = false
        }
      })
    },
    zoom (originPoint, newScale, noCallback) {
      if (newScale === this.scale) {
        if (this.scale === this.max) {
          return 'Max'
        } else if (this.scale === this.min) {
          return 'Min'
        } else {
          return 'NotChange'
        }
      } else {
        var dy = (newScale / this.scale - 1) * (this.getTargetPosition().top + this.targetStyle.height * this.scale / 2 - originPoint.top)
        var dx = (newScale / this.scale - 1) * (this.getTargetPosition().left + this.targetStyle.width * this.scale / 2 - originPoint.left)
        return this.zoomTarget(this.targetTop + dy, this.targetLeft + dx, newScale, noCallback)
      }
    },
    zoomTarget (top, left, newScale, noCallback) {
      this.scale = newScale
      this.targetTop = top
      this.targetLeft = left
      this.setState()
      if (!noCallback) {
        this.afterZoom(this.state)
      }
      return this.scale
    },
    wheelZoom (e) {
      let direct = 0
      e = e || window.event
      if (e.wheelDelta) {
        direct = e.wheelDelta > 0 ? 1 : -1
      } else if (e.detail) {
        direct = e.detail < 0 ? 1 : -1
      }
      this.zoom({
        top: e.pageY,
        left: e.pageX
      }, this.getNewScale(direct))
      if (e.preventDefault) {
        e.preventDefault()
        e.stopPropagation()
      } else {
        e.returnValue = false
        e.cancelBubble = true
      }
    },
    zoomIn (noCallback) {
      return this.zoom(this.centerPosition, this.getNewScale(1), noCallback)
    },
    zoomOut (noCallback) {
      return this.zoom(this.centerPosition, this.getNewScale(-1), noCallback)
    },
    zoomToMax (noCallback) {
      return this.zoom(this.centerPosition, this.max, noCallback)
    },
    zoomToSuitable (noCallback) {
      this.jQDragContainer.css({
        left: 0,
        top: 0
      })
      return this.zoomTarget(0, 0, this.suitScale, noCallback)
    },
    getNewScale (direct) {
      var _newScale = this.scale + this.step * direct
      return this.getFixedScale(_newScale)
    },
    getFixedScale (scale) {
      if (scale < this.min) {
        scale = this.min
      } else if (scale > this.max) {
        scale = this.max
      }
      scale = this.fixNumber(scale)
      return scale
    },
    fixNumber (num) {
      return num.toFixed(6) - 0
    },
    setState () {
      var _state = 'def'
      if (this.scale === this.max) {
        _state = 'max'
      } else if (this.scale === this.min) {
        _state = 'min'
      }
      this.state.scale = this.scale
      this.state.state = _state
    },
    getTargetPosition () {
      return this.jQTarget.offset()
    },
    getCenterPosition () {
      return {
        top: this.containerStyle.top + this.containerStyle.outerHeight / 2,
        left: this.containerStyle.left + this.containerStyle.outerWidth / 2
      }
    },
    getZoomHandler (name) {
      return this[name]()
    }
  },
  watch: {
  }
}
</script>
<style src="./vueZoomer.less" lang="less"></style>
