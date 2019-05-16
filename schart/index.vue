<template>
  <canvas></canvas>
</template>
<script>
export default {
  name: 'schart',
  props: {
    options: {
      type: Object,
      default: () => {
        return {}
      }
    },
    data: { // 存放图表数据
      type: Array,
      default: () => {
        return []
      }
    },
    type: {
      type: String,
      default: () => {
        return 'bar'
      }
    },
    canvasId: {
      type: String,
      default: () => {
        return 'wrapper'
      }
    },
    showValue: {// 是否在图表中显示数值
      type: Boolean,
      default: () => {
        return true
      }
    },
    autoWidth: {// 宽高是否自适应
      type: Boolean,
      default: () => {
        return true
      }
    },
    yEqual: {// y轴分成5等分
      type: Number,
      default: () => {
        return 5
      }
    },
    bgColor: {// 默认背景颜色
      type: String,
      default: () => {
        return '#ffffff'
      }
    },
    fillColor: {// 默认填充颜色
      type: String,
      default: () => {
        return '#1E9FFF'
      }
    },
    axisColor: {// 坐标轴颜色
      type: String,
      default: () => {
        return '#1E9FFF'
      }
    },
    contentColor: {// 内容横线颜色
      type: String,
      default: () => {
        return '#eeeeee'
      }
    },
    titleColor: {// 图表标题颜色
      type: String,
      default: () => {
        return '#000000'
      }
    },
    title: {// 图表标题
      type: String,
      default: () => {
        return ''
      }
    },
    titlePosition: {// 图表标题位置: top / bottom
      type: String,
      default: () => {
        return 'top'
      }
    },
    legendColor: {// 图例文字颜色
      type: String,
      default: () => {
        return '#000000'
      }
    }

  },
  data () {
    return {
      canvas: null, // canvas ele
      ctx: null,
      dpi: 1,
      dataLength: 0, // 图表数据的长度
      width: 0, // canvas 宽度
      height: 0, // canvas 高度
      topPadding: 0,
      leftPadding: 0,
      rightPadding: 0,
      bottomPadding: 0,
      yLength: 0, // y轴坐标点之间的真实长度
      xLength: 0, // x轴坐标点之间的真实长度
      yFictitious: 0, // y轴坐标点之间显示的间距
      yRatio: 0, // y轴坐标真实长度和坐标间距的比
      radius: 0, // 饼图半径和环形图外圆半径
      innerRadius: 0, // 环形图内圆半径
      colorList: ['#1E9FFF', '#13CE66', '#F7BA2A', '#FF4949', '#72f6ff', '#199475', '#e08031', '#726dd1'], // 饼图颜色列表
      legendTop: 0, // 图例距离顶部高度
      totalValue: 0, // 获取饼图数据总和
      startNum: 0 // 竖坐标起点
    }
  },
  created () {
    this.init(this.options)
  },
  methods: {
    init () {
      this.canvas = document.getElementById(this.canvasId)
      this.ctx = this.canvas.getContext('2d')
      this.dpi = window.devicePixelRatio || 1
      this.dataLength = this.data.length // 图表数据的长度
      this.width = this.canvas.width * this.dpi // canvas 宽度
      this.height = this.canvas.height * this.dpi // canvas 高度
      this.topPadding = 50 * this.dpi
      this.leftPadding = 50 * this.dpi
      this.rightPadding = 0 * this.dpi
      this.bottomPadding = 50 * this.dpi
      this.radius = 100 * this.dpi // 饼图半径和环形图外圆半径
      this.innerRadius = 70 * this.dpi // 环形图内圆半径
      this.legendTop = 40 * this.dpi // 图例距离顶部高度
      this.totalValue = this.getTotalValue() // 获取饼图数据总和
      this.initCanvas(this.options)
    },
    initCanvas (options) {
      if (this.dataLength === 0) {
        return false
      }
      if (options) {
        let dpiList = ['topPadding', 'leftPadding', 'rightPadding', 'bottomPadding', 'radius', 'innerRadius', 'legendTop']
        for (let key in options) {
          if (key === 'colorList' && Array.isArray(options[key])) {
            this[key] = options[key].concat(this[key])
          } else if (dpiList.indexOf(key) > -1) {
            this[key] = options[key] * this.dpi
          } else {
            this[key] = options[key]
          }
        }
      }
      // 如果设置了自动宽高的话，则就宽高设为父元素的宽高
      if (options.autoWidth) {
        this.width = this.canvas.width = this.canvas.parentNode.offsetWidth * this.dpi
        this.height = this.canvas.height = this.canvas.parentNode.offsetHeight * this.dpi
        this.canvas.setAttribute('style', 'width:' + this.canvas.parentNode.offsetWidth + 'px;height:' + this.canvas.parentNode.offsetHeight + 'px;')
      } else {
        this.canvas.setAttribute('style', 'width:' + this.canvas.width + 'px;height:' + this.canvas.height + 'px;')
        this.canvas.width *= this.dpi
        this.canvas.height *= this.dpi
      }
      if (this.type === 'bar' || this.type === 'line') {
        this.yLength = Math.floor((this.height - this.topPadding - this.bottomPadding - 10) / this.yEqual)
        this.xLength = Math.floor((this.width - this.leftPadding - this.rightPadding - 10) / this.dataLength)
        this.yFictitious = this.getYFictitious(this.data)
        this.yRatio = this.yLength / this.yFictitious
        this.drawBarUpdate()
      } else {
        this.drawPieUpdate()
      }
    },
    /**
     * 绘制完整的柱状图或折线图
     */
    drawBarUpdate: function () {
      this.ctx.fillStyle = this.bgColor
      this.ctx.fillRect(0, 0, this.width * 2, this.height * 0.5)
      this.drawAxis()
      this.drawPoint()
      this.drawTitle()
      this.drawBarChart()
    },
    /**
     * 绘制完整的饼状图或环形图
     */
    drawPieUpdate: function () {
      this.ctx.fillStyle = this.bgColor
      this.ctx.fillRect(0, 0, this.width * 2, this.height * 0.5)
      this.drawLegend()
      this.drawTitle()
      this.drawPieChart()
    },
    /**
     * 把数据绘制出柱状或折线
     */
    drawBarChart: function () {
      this.ctx.fillStyle = this.fillColor
      this.ctx.strokeStyle = '#9F3535'
      for (let i = 0; i < this.dataLength; i++) {
        this.data[i].left = this.leftPadding + this.xLength * (i + 0.25)
        this.data[i].top = this.height - this.bottomPadding - (+this.data[i].value + Math.abs(this.startNum)) * this.yRatio
        this.data[i].right = this.leftPadding + this.xLength * (i + 0.75)
        this.data[i].bottom = this.height - this.bottomPadding
        // 绘制折线
        if (this.type === 'line') {
          this.ctx.beginPath()
          this.ctx.arc(this.data[i].left + this.xLength / 4, this.data[i].top, 2, 0, 2 * Math.PI, true)
          this.ctx.fill()
          if (i !== 0) {
            this.ctx.moveTo(this.data[i].left + this.xLength / 4, this.data[i].top)
            this.ctx.lineTo(this.data[i - 1].left + this.xLength / 4, this.data[i - 1].top)
          }
          this.ctx.stroke()
        } else if (this.type === 'bar') {
          // 绘制柱状
          this.ctx.fillRect(
            this.data[i].left,
            this.data[i].top,
            this.data[i].right - this.data[i].left,
            this.data[i].bottom - this.data[i].top
          )
        }
        if (this.showValue) {
          this.ctx.font = 12 * this.dpi + 'px Arial'
          this.ctx.textAlign = 'center'
          this.ctx.fillText(
            this.formatNum(this.data[i].value),
            this.data[i].left + this.xLength / 4,
            this.data[i].top - 5
          )
        }
      }
    },
    formatNum (num) {
      let str = String(num)
      let newStr = ''
      let count = 0
      // 当数字是整数
      if (str.indexOf('.') === -1) {
        for (let i = str.length - 1; i >= 0; i--) {
          if (count % 3 === 0 && count !== 0) {
            newStr = str.charAt(i) + ',' + newStr
          } else {
            newStr = str.charAt(i) + newStr
          }
          count++
        }
        str = newStr// 自动补小数点后两位
        return str
      } else {
        // 当数字带有小数
        for (let i = str.indexOf('.') - 1; i >= 0; i--) {
          if (count % 3 === 0 && count !== 0) {
            newStr = str.charAt(i) + ',' + newStr
          } else {
            newStr = str.charAt(i) + newStr // 逐个字符相接起来
          }
          count++
        }
        str = newStr + (str + '00').substr((str + '00').indexOf('.'), 3)
        return str
      }
    },
    /**
     * 把数据绘制出饼状或环形
     */
    drawPieChart: function () {
      let x = this.width / 2
      let y = this.height / 2
      // let x1 = 0
      // let y1 = 0
      for (let i = 0; i < this.dataLength; i++) {
        this.ctx.beginPath()
        this.ctx.fillStyle = this.colorList[i]
        this.ctx.moveTo(x, y)
        this.data[i].start = i === 0 ? -Math.PI / 2 : this.data[i - 1].end
        this.data[i].end = this.data[i].start + this.data[i].num / this.totalValue * 2 * Math.PI
        // 绘制扇形
        this.ctx.arc(x, y, this.radius, this.data[i].start, this.data[i].end)
        this.ctx.closePath()
        this.ctx.fill()
        this.data[i].middle = (this.data[i].start + this.data[i].end) / 2
        // x1 = Math.ceil(Math.abs(this.radius * Math.cos(this.data[i].middle)))
        // y1 = Math.floor(Math.abs(this.radius * Math.sin(this.data[i].middle)))
        this.ctx.strokeStyle = this.colorList[i]
        // 绘制各个扇形边上的数据
        // if(this.showValue){
        //     if (this.data[i].middle <= 0) {
        //         this.ctx.textAlign = 'left';
        //         // this.ctx.fontSize = '18px';
        //         this.ctx.moveTo(x + x1, y - y1);
        //         this.ctx.lineTo(x + x1 + 10, y - y1 - 10);
        //         this.ctx.moveTo(x + x1 + 10, y - y1 - 10);
        //         this.ctx.lineTo(x + x1 + this.radius / 2, y - y1 - 10);
        //         this.ctx.stroke();
        //         this.ctx.fillText(this.data[i].name, x + x1 + 5 + this.radius / 2, y - y1 - 5);
        //     } else if (this.data[i].middle > 0 && this.data[i].middle <= Math.PI / 2) {
        //         this.ctx.textAlign = 'left';
        //         this.ctx.moveTo(x + x1, y + y1);
        //         this.ctx.lineTo(x + x1 + 10, y + y1 + 10);
        //         this.ctx.moveTo(x + x1 + 10, y + y1 + 10);
        //         this.ctx.lineTo(x + x1 + this.radius / 2, y + y1 + 10);
        //         this.ctx.stroke();
        //         this.ctx.fillText(this.data[i].name, x + x1 + 5 + this.radius / 2, y + y1 + 15);
        //     } else if (this.data[i].middle > Math.PI / 2 && this.data[i].middle < Math.PI) {
        //         this.ctx.textAlign = 'right';
        //         this.ctx.moveTo(x - x1, y + y1);
        //         this.ctx.lineTo(x - x1 - 10, y + y1 + 10);
        //         this.ctx.moveTo(x - x1 - 10, y + y1 + 10);
        //         this.ctx.lineTo(x - x1 - this.radius / 2, y + y1 + 10);
        //         this.ctx.stroke();
        //         this.ctx.fillText(this.data[i].name, x - x1 - 5 - this.radius / 2, y + y1 + 15);
        //     } else {
        //         this.ctx.textAlign = 'right';
        //         this.ctx.moveTo(x - x1, y - y1);
        //         this.ctx.lineTo(x - x1 - 10, y - y1 - 10);
        //         this.ctx.moveTo(x - x1 - 10, y - y1 - 10);
        //         this.ctx.lineTo(x - x1 - this.radius / 2, y - y1 - 10);
        //         this.ctx.stroke();
        //         this.ctx.fillText(this.data[i].name, x - x1 - 5 - this.radius / 2, y - y1 - 5);
        //     }
        // }
      }
      // 如果类型是环形图，绘制一个内圆
      if (this.type === 'ring') {
        this.ctx.beginPath()
        this.ctx.fillStyle = this.bgColor
        this.ctx.arc(x, y, this.innerRadius, 0, Math.PI * 2)
        this.ctx.fill()
        // this.ctx.stroke();
        this.ctx.textAlign = 'center'
      }
    },
    /**
     * 绘制坐标轴
     */
    drawAxis: function () {
      this.ctx.beginPath()
      this.ctx.strokeStyle = this.axisColor
      // y轴线, +0.5是为了解决canvas画1像素会显示成2像素的问题
      this.ctx.moveTo(this.leftPadding + 20.5, this.height - this.bottomPadding + 0.5)
      this.ctx.lineTo(this.leftPadding + 20.5, this.topPadding + 0.5)
      // x轴线
      this.ctx.moveTo(this.leftPadding + 20.5, this.height - this.bottomPadding + 0.5)
      this.ctx.lineTo(this.width - this.rightPadding - 0.5, this.height - this.bottomPadding + 0.5)
      this.ctx.stroke()
    },
    /**
     * 绘制坐标轴上的点和值
     */
    drawPoint: function () {
      // x轴坐标点
      this.ctx.beginPath()
      this.ctx.font = 12 * this.dpi + 'px Microsoft YaHei'
      this.ctx.textAlign = 'center'
      this.ctx.fillStyle = this.axisColor
      for (let i = 0; i < this.dataLength; i++) {
        let name = this.data[i].title
        let xlen = this.xLength * (i + 1)
        this.ctx.moveTo(this.leftPadding + xlen + 0.5, this.height - this.bottomPadding + 0.5)
        this.ctx.lineTo(this.leftPadding + xlen + 0.5, this.height - this.bottomPadding + 5.5)
        this.ctx.fillText(name, this.leftPadding + xlen - this.xLength / 2, this.height - this.bottomPadding + 15 * this.dpi)
      }
      this.ctx.stroke()
      // y轴坐标点
      this.ctx.beginPath()
      this.ctx.font = 12 * this.dpi + 'px Microsoft YaHei'
      this.ctx.textAlign = 'right'
      this.ctx.fillStyle = this.axisColor
      this.ctx.moveTo(this.leftPadding + 0.5, this.height - this.bottomPadding + 0.5)
      this.ctx.lineTo(this.leftPadding - 4.5, this.height - this.bottomPadding + 0.5)
      this.ctx.fillText(this.startNum, this.leftPadding + 10, this.height - this.bottomPadding + 5)
      for (let j = 0; j < this.yEqual; j++) {
        let y = (+this.startNum + this.yFictitious * (j + 1)).toFixed(0)
        // 竖坐标大于一百万时 使用科学记数法
        if (+y > Math.pow(10, 6)) {
          y = (y / Math.pow(10, 6)).toFixed(0) + 'x10⁶'
        }
        let ylen = this.yLength * (j + 1)
        this.ctx.beginPath()
        this.ctx.strokeStyle = this.axisColor
        this.ctx.moveTo(this.leftPadding + 20.5, this.height - this.bottomPadding - ylen + 0.5)
        this.ctx.lineTo(this.leftPadding + 16.5, this.height - this.bottomPadding - ylen + 0.5)
        this.ctx.stroke()
        this.ctx.fillText(y, this.leftPadding + 10, this.height - this.bottomPadding - ylen + 5)
        this.ctx.beginPath()
        this.ctx.strokeStyle = this.contentColor
        this.ctx.moveTo(this.leftPadding + 20.5, this.height - this.bottomPadding - ylen + 0.5)
        this.ctx.lineTo(this.width - this.rightPadding - 0.5, this.height - this.bottomPadding - ylen + 0.5)
        this.ctx.stroke()
      }
    },
    /**
     * 绘制图表标题
     */
    drawTitle: function () {
      if (this.title) {
        this.ctx.beginPath()
        this.ctx.textAlign = 'center'
        this.ctx.fillStyle = this.titleColor
        this.ctx.font = 16 * this.dpi + 'px Microsoft YaHei'
        if (this.titlePosition === 'bottom' && this.bottomPadding >= 40) {
          this.ctx.fillText(this.title, this.width / 2, this.height - 5)
        } else {
          this.ctx.fillText(this.title, this.width / 2, this.topPadding / 2 + 5)
        }
      }
    },
    /**
     * 绘制饼状图或环形图的图例
     */
    drawLegend: function () {
      for (let i = 0; i < this.dataLength; i++) {
        this.ctx.fillStyle = this.colorList[i]
        // this.ctx.fillRect(10, this.legendTop + 15 * i * this.dpi, 20, 11);
        this.ctx.fillStyle = this.legendColor
        this.ctx.font = 14 * this.dpi + 'px Microsoft YaHei'
        this.ctx.textAlign = 'left'
        // this.ctx.fillText(this.data[i].name, 35, this.legendTop + 10 + 15 * i * this.dpi);
      }
    },
    /**
     * y轴坐标点之间显示的间距
     * @param data 生成图表的数据
     * @return y轴坐标间距
     */
    getYFictitious: function (data) {
      let arr = data.slice(0)
      arr.sort(function (a, b) {
        return -(a.value - b.value)
      })
      let len
      if (arr[arr.length - 1].value < 0) {
        len = Math.ceil((arr[0].value - arr[arr.length - 1].value) / this.yEqual)
      } else {
        len = Math.ceil(arr[0].value / this.yEqual)
      }
      let pow = len.toString().length - 1
      pow = pow > 2 ? 2 : pow
      // 计算开始数值
      if (arr[arr.length - 1].value < 0) {
        this.startNum = Math.floor(arr[arr.length - 1].value)
      } else {
        this.startNum = 0
      }
      return Math.ceil(len / Math.pow(10, pow)) * Math.pow(10, pow)
    },
    /**
     * 获取饼状或环形图的数据总和
     * @return {Number} total
     */
    getTotalValue: function () {
      let total = 0
      for (let i = 0; i < this.dataLength; i++) {
        total += this.data[i].num
      }
      return total
    }
  }
}
</script>
