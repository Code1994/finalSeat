<template>
  <div class="topBox">
    <div class="wrapper">
      <div class="canvas" ref="canvas">
        <canvas class="seats" @click="choose" ref="seats"></canvas>
      </div>
      <div class="row">
        <ul class="rowbar" ref="rowBar">
          <li v-for="(item,index) in data" :key="index">
            <span v-if="item.length" v-text="item[0].rowNum"></span>
          </li>
        </ul>
      </div>
    </div>
    <!--推荐选座组件 1人 2人 3人 4人 最多同时4人-->
    <div class="recommend" v-show="!changeView">
      <p class="title">推荐选座</p>
      <div class="lis">
        <span @click="getSeat(1)">1人座位</span>
        <span @click="getSeat(2)">2人座位</span>
        <span @click="getSeat(3)">3人座位</span>
        <span @click="getSeat(4)">4人座位</span>
      </div>
    </div>
    <!-- 座位号 -->
    <div class="seatNum" v-show="changeView">
      <span>这里是座位号</span>
        <div>
          <span v-for="(item,index) in xy" :key="index" @click="destroy(index)">
            {{ item[0] }}排{{ item[1] }}座
          </span>
        </div>
    </div>
    <!-- 未选座提示 -->
    <div class="greyComplete" v-show="!changeView">
      <p>单击座位区域选座,最多选4个座位</p>
      <div>
        <span>选好了</span>
      </div>
    </div>
    <!-- 选座后提示 -->
    <div class="blueComplete" v-show="changeView">
      <p>这里将来显示价格</p>
      <div>
        <span @click="chooseOk">选好了</span>
      </div>
    </div>
  </div>
</template>

<script>
import datas from '@/assets/js/data4.js'
import Bscroll from 'better-scroll'
// import AlloyFinger from 'alloyfinger'
export default {
  data () {
    return {
      // 图片
      selectImg: 'http://img.vcdianying.com/nwx/images/kexuan_icon.png',
      selectedImg: 'http://img.vcdianying.com/nwx/images/yishou_icon.png',
      selledImg: 'http://img.vcdianying.com/nwx/images/green/yixuan_icon.png',
      coupleImg: 'http://img.vcdianying.com/nwx/images/qinglv_icon.png',
      // 规整数据(包含过道,权重)
      data: null,
      // 没有已售座位的数据
      noyishouData: null,
      // 权重数据---将noyishouData按照权重进行排序
      weightSeats: null,
      // 不包含过道的数据
      regArr: null,
      // 记录行数---一共有多少行
      rows: null,
      // 记录列数---一共有多少列
      cols: null,
      // canvas标签元素的宽度
      // canvasWidth: null,
      // canvas标签元素的高度
      // canvasHeight: null,
      // 单个座位的宽度以及高度
      seatWidth: 14,
      seatHeight: 14,
      // 存储已选座位的code
      chooseCodes: [],
      // 遍历权重数据的索引
      initIndex: 0,
      // 存储已选座位所在的行数---主要是为了在chooseOk中使用
      chooseRows: [],
      // 普通座是否留空布尔值
      bool: false,
      // 情侣座是否留空布尔值
      coupleBool: false,
      // FIXME:画布宽度
      canvasWidth: null,
      // 画布高度
      canvasHeight: null,
      // 2d上下文---因为有一个ctx.scale(),不能在循环中使用ctx.scale(),因为在循环中可能会按照放大之后为基准值再次放大
      ctx: null,
      // 推荐选座视图与座位数视图的交替
      changeView: false,
      // 座位数
      xy: []
    }
  },
  // created处理数据
  created () {
    // 在进行handler之后,this.data中存储的会是所有规整的数据
    this.handler()
    // fixme:forEach没有返回值,暂时考虑使用深拷贝复制没有已售座位的数据
    // todo:深拷贝得到所有数据,再将已售的数据进行剔除---接下来就可以进行直接遍历找到最大权重的数据
    this.noyishouData = this.deepCopy(this.data)
    this.noyishouData.forEach((item1, index1) => {
      // 找到数据中的属性status不等于'available'的数据,剔除它---另外这里也要删除过道数据
      item1.forEach((item2, index2) => {
        if (item2.status !== 'available' || item2.code === '0001') {
          item1.splice(index2, 1)
        }
      })
    })
    // 清除掉已售数据后,将剩余的座位按照权重排序,存储在weightSeats中
    this.weightSeats = this.sortWeight(this.noyishouData)
  },
  mounted () {
    // 设定画布大小
    this.defineCanvas()
    this.save()
    // 渲染canvas
    this.makeCanvas()
    // this.$refs.seats.style.width = this.canvasWidth + 'px'
    // this.$refs.seats.style.width = this.canvasWidth + 'px'

    // 缩放的最小值需要重新设置---如果座位高度*总座位行数大于外层的容器高度,min就设置为0.5,否则为1
    // let min
    // if (this.seatHeight * this.rows > 300) {
    //   min = 0.5
    // } else {
    //   min = 1
    // }
    this.$nextTick(() => {
      this.scroll = new Bscroll(this.$refs.canvas, {
        startX: 0,
        click: true,
        scrollX: true,
        scrollY: true,
        // 必须设置成3,为了让行数栏跟随座位图移动
        probeType: 3,
        zoom: {
          start: 1,
          min: 1,
          max: 2
        }
      })
      let scale = 1
      this.scroll.on('scroll', (pos) => {
        // alert(1111)
        const y = pos.y
        this.$refs.rowBar.style.transform = `translateY(${y}px) scale(${scale})`
        this.$refs.rowBar.style.transformOrigin = 'center top'
      })
      // let zoom
      // let al = new AlloyFinger(this.$refs.canvas, {
      //   pinch (e) {
      //     zoom = e.zoom
      //   }
      // })
      this.scroll.on('zoomStart', () => {
        // this.$refs.rowBar.style.display = 'none'
        // this.$refs.rowBar.style.transform = "scale(5)"
      })
      this.scroll.on('zoomEnd', () => {
        // const str1 = 'scale(2)'
        // const str2 = 'scale(1)'
        // 获取seats的缩放因子,利用字符串
        const str = this.$refs.seats.style.transform
        const reg = /scale(\(\d+\.{0,1}(\d)*\))/i
        // 获取形如scale()的字符串
        const found = str.match(reg)[1]
        // 将scale()中的()进行替换,得到缩放因子
        const num = Number(found.replace(/\(|\)/g, ''))
        // 由于zoom事件最后会触发scroll事件,所以不在zoom直接设置rowbar放大或缩小,而是让scroll事件去处理
        scale = num
        // this.$refs.rowBar.style.display = 'block'
      })
    })
  },
  methods: {
    // 1.处理数据---排序,设置权重,创建过道,深拷贝,权重排序
    handler () {
      //  处理 datas  把属性data中的cinemaSeatpicDataList(对象)的属性data3---0000000000000001对应的数组中的对象  ---data4 15010
      let arr = JSON.parse(datas).data.cinemaSeatpicDataList['15010']
      //  得到数组arr,应该遍历,根据y的最大值来创建数组容器个数,再遍历根据x相同时,扔进同一个数组 item.yCoord
      let ylist = []
      let xlist = []
      arr.forEach((item) => {
        ylist.push(Number(item.yCoord))
        xlist.push(Number(item.xCoord))
      })
      //  得到y的最大值 x的最大值
      let ymax = Math.max(...ylist)
      let xmax = Math.max(...xlist)
      this.rows = ymax
      this.cols = xmax
      //  定义总容器数组,根据ymax创建数组,并判断x
      let resultArr = []
      for (let i = 0; i < ymax; i++) {
        resultArr[i] = []
        //  遍历arr , 其中的 y 如果等于 i,直接把当前项push  arr是总的一维数组
        for (let j = 0; j < arr.length; j++) {
          // 为规则2做标记:已售9 普通可选0 情侣可选2  ---普通已选是1 情侣已选是3 过道是4
          if (arr[j].status !== 'available') {
            arr[j].flag = '9'
          } else if (arr[j].loveSeat) {
            arr[j].flag = '2'
          } else {
            arr[j].flag = '0'
          }
          if (Number(arr[j].yCoord) === (i + 1)) {
            resultArr[i].push(arr[j])
          }
        }
      }
      // 由于x顺序不规则,所以要将x重新按照从小到大的顺序进行排列---这一步完成之后就是规整的数据了
      resultArr.forEach((item) => {
        for (let n = 0; n < item.length; n++) {
          this.bubbleSort(item)
        }
      })
      // console.log('resultArr')
      // console.log(resultArr)
      // 增加x权重 ---设置rows是为了测试了下data4
      resultArr.forEach((item, index) => {
        this.setXWeight(item)
        if (item.length === 0) {
          index += '空位'
        }
        // this.rows.push(index)
      })
      // 增加y权重
      this.setYWeight(resultArr)
      // 计算权重和,并设置给每个对象的新属性totalWeight
      resultArr.forEach((item1, index1) => {
        item1.forEach((item2, index2) => {
          item2.totalWeight = item2.xnum + item2.ynum
        })
      })
      console.log('resultArr')
      console.log(resultArr)
      this.regArr = this.deepCopy(resultArr)
      // console.log('regArr')
      // console.log(this.regArr)
      // 注意这里,上面与下面,我是先设置的权重,然后再去插入过道信息 ,所以不用再考虑过道会有权重的问题
      // 排完顺序之后,判断每一项xCoord与其索引之间的差值是否大于1,如果大于1 ,插入一个对象 i是行 j是列  ----设置纵向过道走廊
      for (let i = 0; i < resultArr.length; i++) {
        for (let j = 0; j < resultArr[i].length; j++) {
          if (Number(resultArr[i][j].xCoord) - j !== 1) {
            // console.log(j)
            // 为了行数栏的显示,要传入rowNum
            let passObj = this.createPassWay(i, j, resultArr[i][j].rowNum)
            resultArr[i].splice(j, 0, passObj)
          }
        }
      }
      // 设置横向过道---如果有横向过道,因为我按照最大行数来创建数组,又判断索引与数据中的y值的是否相等.所以横向过道存在时就是一个空数组
      // FIXME:发现即使创建了横向过道,末尾的纵向过道也是不能满足的,而且还会影响推荐选座,所以直接判断384行的this.data[y][x],不存在就是undefined,直接return.
      // 得到最后要使用的数据
      this.data = resultArr
    },
    // 排序
    bubbleSort (arr) {
      let sign
      for (let i = 0; i < arr.length - 1; i++) {
        sign = false
        for (let j = 0; j < arr.length - i - 1; j++) {
          if (Number(arr[j].xCoord) > (Number(arr[j + 1].xCoord))) {
            let oldVal = arr[j]
            arr[j] = arr[j + 1]
            arr[j + 1] = oldVal
            sign = true
          }
        }
        if (!sign) break
      }
    },
    // 设置x权重函数
    setXWeight (arr) {
      for (let i = 0; i < arr.length; i++) {
        if (i <= (arr.length / 2)) {
          arr[i].xnum = i
        } else {
          arr[i].xnum = arr.length - 1 - i
        }
      }
      return arr
    },
    // 设置y权重函数
    setYWeight (arr) {
      for (let i = 0; i < arr.length; i++) {
        for (let j = 0; j < arr[i].length; j++) {
          if (i <= (arr.length / 2)) {
            arr[i][j].ynum = i
          } else {
            arr[i][j].ynum = arr.length - 1 - i
          }
        }
      }
      return arr
    },
    // 创建过道
    createPassWay (i, j, rowNum) {
      let passWay = {}
      passWay.status = 'passway'
      passWay.xCoord = j + 1
      passWay.yCoord = i + 1
      passWay.rowNum = rowNum
      passWay.code = '0001'
      passWay.flag = '4'
      return passWay
    },
    // 深拷贝
    deepCopy (obj) {
      let objClone = Array.isArray(obj) ? [] : {}
      if (obj && typeof obj === 'object') {
        for (let key in obj) {
          if (obj.hasOwnProperty(key)) {
            // 判断obj子元素是否为对象，如果是，递归复制
            if (obj[key] && typeof obj[key] === 'object') {
              objClone[key] = this.deepCopy(obj[key])
            } else {
              // 如果不是，简单复制
              objClone[key] = obj[key]
            }
          }
        }
      }
      return objClone
    },
    // 权重排序--arr指的是二维数组
    sortWeight (arr) {
      let resultArr = []
      for (let i = arr.length - 1; i >= 0; i--) {
        for (let j = arr[i].length - 1; j >= 0; j--) {
          resultArr.push(arr[i][j])
        }
      }
      resultArr.sort(function (a, b) {
        return b.totalWeight - a.totalWeight
      })
      return resultArr
    },
    // 2.确定画布大小---startX绘座起始点横坐标 startY绘座起始点纵坐标 w座位宽度 h座位高度
    defineCanvas (w, h) {
      // canvas宽度===屏幕宽度->座位宽度   ---左右固定值各10
      // FIXME: 画布宽度应该由座位宽度决定, 座位宽度在375屏幕上是28*28 ---数据的最大x值乘以宽度
      // this.canvasWidth = document.body.clientWidth
      // this.canvasWidth = this.cols * 28 + 20
      // this.seatWidth = (this.canvasWidth-20)/this.cols
      // canvas 高度 ---座位高度固定值28
      this.canvasWidth = this.cols * this.seatWidth
      this.canvasHeight = this.rows * this.seatHeight
      // 这里不要使用style样式来设置width和height---原始的width和height用来设置绘图尺寸,css的width和height用来设置显示尺寸
      this.$refs['seats'].width = this.canvasWidth * 4
      this.$refs['seats'].style.width = this.canvasWidth + 'px'
      this.$refs['seats'].height = this.canvasHeight * 4
      this.$refs['seats'].style.height = this.canvasHeight + 'px'
    },
    // 渲染canvas
    makeCanvas () {
      let data = this.data
      // 通过data数据来创建
      for (let i = 0; i < data.length; i++) {
        for (let j = 0; j < data[i].length; j++) {
          if (data[i][j].code === '0001') {
            continue
          }
          if (data[i][j].status === 'damage') {
            // 已售
            this.loadImg(this.selledImg, i, j)
            continue
          }
          if (data[i][j].status === 'selected') {
            this.loadImg(this.selectedImg, i, j)
            continue
          }
          if (data[i][j].loveSeat) {
            // 情侣
            this.loadImg(this.coupleImg, i, j)
            continue
          }
          // 可选
          this.loadImg(this.selectImg, i, j)
        }
      }
      // 绘制线
      this.setLine()
    },
    // 设置一层来过滤ctx,为了保证永远是同一个ctx
    save () {
      let seats = document.querySelector('.seats')
      let ctx = seats.getContext('2d')
      ctx.scale(4, 4)
      this.ctx = ctx
    },
    // canvas 加载图片---i是行数 j是列数 scale是缩放因子,默认是1
    loadImg (src, i, j) {
      const w = this.seatWidth
      const h = this.seatHeight
      // 清除之前的图片
      this.ctx.clearRect(j * w, i * h, w, h)
      let img = new Image()
      img.src = src
      img.onload = () => {
        this.ctx.drawImage(img, j * w, i * h, w, h)
      }
    },
    setLine () {
      this.ctx.setLineDash([2])
      this.ctx.lineWidth = 2
      this.ctx.strokeStyle = 'rgba(0, 0, 0, 0.6)'
      this.ctx.beginPath()
      // moveTo() x横坐标 y纵坐标  ---x画布的一半  y 从0 到 画布高度,
      this.ctx.moveTo(this.canvasWidth / 2, 12)
      this.ctx.lineTo(this.canvasWidth / 2, this.canvasHeight)
      this.ctx.stroke()
    },
    // 3.用户自选点击
    choose (e) {
      // 点击座位,以点击点为中心,放大canvas---Math.floor(e.offsetX-10),Math.floor(e.offsetY-10)
      // this.zoom(1.5,Math.floor(e.offsetX-10),Math.floor(e.offsetY-10),e)

      // pageX pageY相对浏览器内窗口,也就是当前页面   screenX screenY相对的是整个电脑屏幕窗口   ---x,y正好是索引,
      // FIXME:不应该再使用e.pageX与e.pageY,应该利用canvas本身的宽度
      let x = Math.floor((e.offsetX) / this.seatWidth)
      let y = Math.floor((e.offsetY) / this.seatHeight)
      console.log(e.offsetX)
      if (x < 0 || x >= this.cols || y < 0 || y >= this.rows || !this.data[y][x]) {
        return
      }

      // 遍历data,根据y和x判断status,'passway' 'damage' 'select' 'available'
      if (this.data[y][x].status === 'passway' || this.data[y][x].status === 'damage') {
        return
      }
      // 放大---点击时将组件放大
      // this.$refs['seats'].style.cssText = 'transform:translateZ('+0+') scale('+ 1.5 +','+ 1.5 +');' + 'transform-origin:' + e.offsetX+'px '+ e.offsetY+'px;'
      // this.$refs['seats'].style.transition = 'all 2s ease 0s'
      // 给座位号组件传递座位号---这里要传递过去rowNum以及columnNum,这俩不包含过道
      let seatNum = []
      seatNum.push(this.data[y][x].rowNum, this.data[y][x].columnNum)
      // 修改座位号的视图
      this.changeView = true
      // 如果点击的是已选的,在chooseCodes中删除code,将status改为available,再次重新渲染canvas---将当前y从chooseRows中删除
      if (this.data[y][x].status === 'selected') {
        this.data[y][x].status = 'available'
        this.chooseCodes.splice(this.chooseCodes.findIndex(item => item === this.data[y][x].code), 1)
        this.chooseRows.splice(this.chooseRows.findIndex(item => item === y), 1)
        // this.data[y][x].loveSeat?(this.data[y][x].flag = '2'):(this.data[y][x].flag = '0')
        if (this.data[y][x].loveSeat) {
          this.data[y][x].flag = '2'
          this.loadImg(this.coupleImg, y, x)
        } else {
          this.data[y][x].flag = '0'
          this.loadImg(this.selectImg, y, x)
        }
        this.reduceSeatNum({ seatNum })
        if (this.chooseCodes.length === 0) {
          this.changeView = false
        }
      } else {
        if (this.chooseCodes.length >= 4) {
          return console.log('最多选择4个座位')
        }

        // 如果点击的是可选的,改变status,添加code---将y添加进chooseRows中
        this.loadImg(this.selectedImg, y, x)
        this.data[y][x].status = 'selected'
        this.chooseCodes.push(this.data[y][x].code)
        this.chooseRows.push(y)
        this.data[y][x].loveSeat ? (this.data[y][x].flag = '3') : (this.data[y][x].flag = '1')
        // this.makeCanvas()
        this.getSeatNum({ seatNum })
      }
      console.log(this.chooseCodes)
    },

    // zoom (scale,x,y,e) {
    //   获取canvas画布和行数栏---两者等比缩放
    //   let canvas = this.$refs['seats']
    //   let rowBar = this.$refs['rowBar']
    //   利用transform 中的scale
    //   canvas.style.transform = 'scale('+ scale +')'
    //   canvas.style.transformOrigin = x + 'px ' + y + 'px'
    //   canvas.style.transition = 'all ease 0.2s'

    //   rowBar.style.transform = 'scale('+ scale +')'
    //   rowBar.style.transformOrigin = 5 + 'px ' + y + 'px'
    //   rowBar.style.transition = 'all ease 0.2s'
    // },

    // 4.推荐选座点击
    judge (weightSeats, initIndex, num) {
      // FIXME:算法的停止条件
      if (initIndex >= weightSeats.length) {
        this.changeView = false
        console.log('无可用座位')
      }
      let bestSeat = weightSeats[initIndex]
      // 确定最优座位的坐标---这里使用data来处理,根据code确定索引位置,第几行,该行的状态拼接为字符串,以num为限制,以当前最优座位为判断点判断周遭
      let data = this.data
      let bestX = null
      let bestY = null
      for (let i = 0; i < data.length; i++) {
        for (let j = 0; j < data[i].length; j++) {
          if (data[i][j].code === bestSeat.code) {
            bestY = i
            bestX = j
            break
          }
        }
      }
      for (let i = 0; i < num; i++) {
        this.chooseRows.push(bestY)
      }
      // 4-1最优座位是普通座
      if (weightSeats[initIndex].flag === '0') {
        // 将最优普通座位的flag设为'1'
        this.data[bestY][bestX].flag = '1'
        // 将flag拼接---原先想使用regArr,后在data中设置了过道的flag为4,改用data
        let str = ''
        for (let i = 0; i < this.data[bestY].length; i++) {
          str += this.data[bestY][i].flag
        }
        console.log(str)
        // 分情况判断
        switch (num) {
          case 1:
            this.addStatus(bestY, bestX)
            break
          case 2:
            if ((/01/).test(str)) {
            /* 记录最优座位和其左边座位  ---将code保存在chooseCodes中,画图bestX-1,bestX
            FIXME:
            this.data[bestY][bestX].status = 'selected'
            this.data[bestY][bestX-1].status = 'selected'
            this.loadImg(this.selectedImg, bestY, bestX)
            this.loadImg(this.selectedImg, bestY, bestX-1)
            this.chooseCodes.push(this.data[bestY][bestX-1].code,this.data[bestY][bestX].code)
            创建一个二维数组,传递给seatnum.vue
            let seatNum = []
            let arr1 = []
            let arr2 = []
            arr1.push(bestY,bestX-1)
            arr2.push(bestY,bestX)
            seatNum.push(arr1,arr2)
            break */
              this.addStatus(bestY, bestX - 1, bestX)
              break
            }
            if ((/10/).test(str)) {
            // 记录最优座位和其左边座位
              this.addStatus(bestY, bestX, bestX + 1)
              break
            }
            // 如果到这里,说明当前最高权重座位不符合情况,去寻找第二权重高的座位
            this.initIndex++
            this.judge(this.weightSeats, this.initIndex, this.num)
            break
          case 3:
            if ((/010/).test(str)) {
              this.addStatus(bestY, bestX - 1, bestX, bestX + 1)
              break
            }
            if ((/001|221/).test(str)) {
              this.addStatus(bestY, bestX - 2, bestX - 1, bestX)
              break
            }
            if ((/100|122/).test(str)) {
              this.addStatus(bestY, bestX, bestX + 1, bestX + 2)
              break
            }
            this.initIndex++
            this.judge(this.weightSeats, this.initIndex, this.num)
            break

          case 4:
            if ((/0010|2210/).test(str)) {
              this.addStatus(bestY, bestX - 2, bestX - 1, bestX, bestX + 1)
              break
            }
            if ((/0100|0122/).test(str)) {
              this.addStatus(bestY, bestX - 1, bestX, bestX + 1, bestX + 2)
              break
            }
            if ((/1000|1022|1220/).test(str)) {
              this.addStatus(bestY, bestX, bestX + 1, bestX + 2, bestX + 3)
              break
            }
            if ((/0001|2201|0221/).test(str)) {
              this.addStatus(bestY, bestX - 3, bestX - 2, bestX - 1, bestX)
              break
            }
            this.initIndex++
            this.judge(this.weightSeats, this.initIndex, this.num)
            break
        }
      }
      // 4-2最优座位是情侣座
      if (weightSeats[initIndex].flag === '2') {
        this.data[bestY][bestX].flag = '3'
        let str = ''
        for (let i = 0; i < this.data[bestY].length; i++) {
          str += this.data[bestY][i].flag
        }
        switch (num) {
          case 1:
          // 情侣座位不允许只选一个,所以直接判断下一个权重的座位
            this.data[bestY][bestX].flag = '2'
            this.initIndex++
            console.log(111111111)
            this.judge(this.weightSeats, this.initIndex, this.num)
            break
          case 2:
          // FIXME:如果只有俩情侣座呢
            if ((/23/).test(str)) {
              if ((/2232/).test(str)) {
                this.addStatus(bestY, bestX, bestX + 1)
                break
              } else if ((/2322/).test(str)) {
                this.addStatus(bestY, bestX - 1, bestX)
                break
              }
            }
            if ((/32/).test(str)) {
              this.addStatus(bestY, bestX, bestX + 1)
              break
            }
            this.initIndex++
            this.judge(this.weightSeats, this.initIndex, this.num)
            break
          case 3:
            if ((/320/).test(str)) {
              this.addStatus(bestY, bestX, bestX + 1, bestX + 2)
              break
            }
            if ((/032|230/).test(str)) {
              this.addStatus(bestY, bestX - 1, bestX, bestX + 1)
              break
            }
            if ((/023/).test(str)) {
              this.addStatus(bestY, bestX - 2, bestX - 1, bestX)
              break
            }
            this.initIndex++
            this.judge(this.weightSeats, this.initIndex, this.num)
            break

          case 4:
            if ((/0320|2322|2300/).test(str)) {
              this.addStatus(bestY, bestX - 1, bestX, bestX + 1, bestX + 2)
              break
            }
            if ((/0032|2232|0230/).test(str)) {
              this.addStatus(bestY, bestX - 2, bestX - 1, bestX, bestX + 1)
              break
            }
            if ((/3200|3222/).test(str)) {
              this.addStatus(bestY, bestX, bestX + 1, bestX + 2, bestX + 3)
              break
            }
            if ((/0023|2223/).test(str)) {
              this.addStatus(bestY, bestX - 3, bestX - 2, bestX - 1, bestX)
              break
            }
            this.initIndex++
            this.judge(this.weightSeats, this.initIndex, this.num)
            break
        }
      }
    },
    // 逻辑公共代码封装
    addStatus (y, ...x) {
      let seatNum = []
      x.forEach((item) => {
        this.data[y][item].status = 'selected'
        this.loadImg(this.selectedImg, y, item)
        this.chooseCodes.push(this.data[y][item].code)
        this.data[y][item].loveSeat ? (this.data[y][item].flag = '3') : (this.data[y][item].flag = '1')
        let arr = []
        arr.push(this.data[y][item].rowNum, this.data[y][item].columnNum)
        seatNum.push(arr)
      })
      this.getSeatNum({ seatNum })
    },
    // 5.选好了---判断座位是否留空,将chooseCodes
    chooseOk () {
      // 每次点击之前都要把bool和coupleBool手动赋值为false
      this.bool = false
      this.coupleBool = false
      console.log('------------')
      console.log(this.chooseRows)
      if (this.chooseRows.length === 0) {
        return
      }
      let rows = [...new Set(this.chooseRows)]
      // 判断中间是否留空
      for (let i = 0; i < rows.length; i++) {
        let str = ''
        this.data[rows[i]].forEach((item) => {
          str += item.flag
        })
        console.log(str)
        if ((/10[1,2,3,4,9]|[1,2,3,4,9]01/).test(str)) {
          // 为了防止多次触发,设置一个bool值
          this.bool = true
        }
        // 判断情侣座是否单选
        if (((/3/).test(str) && !(/33/).test(str)) || (/32[0,1,3,9]|[0,1,3,9]23/).test(str)) {
          this.coupleBool = true
        }
        // 验证布尔值,先判断情侣再普通留空
        if (this.coupleBool) {
          return console.log('请不要单选情侣座')
        }
        if (this.bool) {
          return console.log('座位之间不能留空一个座位,不要在过道旁单留一个座位')
        }
        // TODO:到达这里说明用户选座符合规范,获取this.chooseCodes
        console.log(this.chooseCodes)
      }
    },
    // 6.推荐选座
    getSeat (num) {
      this.changeView = true
      this.num = num
      this.judge(this.weightSeats, this.initIndex, this.num)
    },
    // 7.点击画布获取以及减少座位号
    getSeatNum (obj) {
      if (typeof (obj.seatNum[0]) !== 'object') {
        this.xy.push(obj.seatNum)
      } else {
        console.log('二维数组')
        console.log(obj.seatNum)
        this.xy = obj.seatNum
      }
    },
    reduceSeatNum (obj) {
      this.xy.forEach((item, i) => {
        if (item[0] === obj.seatNum[0] && item[1] === obj.seatNum[1]) {
          this.xy.splice(i, 1)
        }
      })
    },
    // 8.点击座位号影响画布
    destroy (index) {
      // 1.清除时,从this.xy中拿出对应的rowNum与columnNum,
      let arr = this.xy[index]
      // arr[0] => rowNum   arr[1] => columnNum  遍历data,找出对应索引,调用this.loadImg,改变status   ---另外还要在chooseCodes取出对应的code,将flag对应改变
      for (let i = 0; i < this.data.length; i++) {
        for (let j = 0; j < this.data[i].length; j++) {
          if (this.data[i][j].rowNum === arr[0] && this.data[i][j].columnNum === arr[1]) {
            // 这里要判断下是否是loveseat
            // this.loadImg('http://img.vcdianying.com/nwx/images/kexuan_icon.png',y,x)
            this.data[i][j].loveSeat ? this.loadImg(this.coupleImg, i, j) : this.loadImg(this.selectImg, i, j)
            this.data[i][j].status = 'available'
            this.chooseCodes.splice(this.chooseCodes.findIndex(item => item === this.data[i][j].code), 1)
            this.chooseRows.splice(this.chooseRows.findIndex(item => item === i), 1)
            this.data[i][j].loveSeat ? (this.data[i][j].flag = '2') : (this.data[i][j].flag = '0')
          }
        }
      }
      // 初始化initIndex
      this.initIndex = 0
      // 2.将arr从this.xy中删除
      this.xy.splice(index, 1)
      // 3.如果长度为0,通知app.vue
      if (this.xy.length === 0) {
        this.changeView = false
      }
    }
  }
}
</script>

<style lang="less" scoped>
  .topBox {
    padding-top: 200px;
  }
  .wrapper {
    position: relative;
    width: 100%;
    height: 300px;
    background-color: rgb(238,238,238);
    overflow: hidden;
    display: flex;
    justify-content: center;
    align-items: center;
    .canvas {
      display: flex;
      justify-content: center;
      align-items: center;
      width: 100%;
      height: 100%;
    }
    .row {
      position: absolute;
      top: 50%;
      left: 0;
      // right: 0;
      // bottom: 0;
      // margin: auto;
      transform: translateY(-50%);
      .rowbar {
        margin: 0;
        padding: 5px 0;
        background-color: rgba(0,0,0,0.5);
        border-radius: 10px;
        li {
          height: 14px;
          line-height: 14px;
          text-align: center;
          color: #fff;
          font-family: sans-serif, Geneva, Verdana;
          font-size: 6px;
        }
      }
    }
  }
  .recommend {
    height: 80px;
    .title {
      font-size: 16px;
    }
    .lis {
      display: flex;
      justify-content: space-between;
      span {
        color: #007582;
        background-color: #eee;
        border-radius: 3px;
      }
    }
  }
  .seatNum {
    height: 80px;
    margin-top: 10px;
    span {
      background-color: #eee;
      color: #00bcd4;
      display: inline-block;
      margin: .5em;
      border-radius: 3px;
      padding: .2em;
    }
  }
  .greyComplete {
    display: flex;
    margin-top: 10px;
    >p {
      color: #717171;
      font-size: 14px;
      flex: 1;
      display: flex;
      align-items: center;
    }
    >div {
      width: 90px;
      height: 32px;
      line-height: 32px;
      margin-right: 20px;
      >span {
        width: 100%;
        height: 100%;
        background-color: #ccc;
        text-align: center;
        border-radius: 10px;
        display: block;
      }
    }
  }
  .blueComplete {
    display: flex;
    margin-top: 10px;
    p {
      color: #717171;
      font-size: 14px;
      flex: 1;
      display: flex;
      align-items: center;
    }
    div {
      width: 90px;
      height: 32px;
      line-height: 32px;
      margin-right: 20px;
      span {
        width: 100%;
        height: 100%;
        background-color: #00bcd4;
        text-align: center;
        border-radius: 10px;
        display: block;
      }
    }
  }
</style>
