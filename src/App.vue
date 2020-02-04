<template>
  <div id="container">
    <div id="avatar" v-if="avatar">
      <img :src="avatar">
    </div>
    <div id="virusmap"></div>
    <div id="dayinfo">第{{day}}天</div>
    <ul id="info">
      <li style="color:white">未感染({{totalCount-infectCount}})</li>
      <li style="color:yellow">已感染({{infectCount}})</li>
      <li style="color:red">已确诊({{quezhenCount}})</li>
      <li style="color:white">空余床位数：{{freeBedNum}}</li>
      <li style="color:white">死亡人数：{{deadCount}}</li>
      <li style="color:green">康复人数：{{healCount}}</li>
    </ul>
    <div id="form" :class="{expand:showForm}">
      <a class="btn btn-open" v-if="!showForm" @click="toggleForm">+ 调整参数</a>
      <a class="btn btn-close" v-if="showForm" @click="toggleForm">+</a>
      <div class="form-content" v-if="showForm">
        <div>人员活动范围：<input v-model="人员活动范围" type="number"></div>
        <div>每天活动人数比例：<input v-model="每天活动人数比例" type="number"></div>
        <div>最大床位数：<input v-model="maxFreeBedNum" type="number"></div>
        <div>感染范围：<input v-model="infectSize" type="number"></div>
        <div>初始感染人数：<input v-model="initInfectCount" type="number"></div>
        <div>同区域感染率：<input v-model="infectRate" type="number"></div>
        <div>样本数：<input v-model="totalCount" type="number"></div>
        <div>自愈概率：<input v-model="selfHealRate" type="number"></div>
        <div>住院康复概率：<input v-model="hospitalHealRate" type="number"></div>
        <button class="confirm" @click="init" v-if="!inited">开始模拟</button>
        <button class="confirm" @click="toggleForm" v-if="inited&&paused">继续</button>
        <button class="confirm" @click="init" v-if="inited&&paused">重新开始模拟</button>
      </div>
    </div>
  </div>
</template>

<script>
import * as PIXI from 'pixi.js'
const Luban = require('./sdk')
let lb
if (location.hostname === 'cdn.nodesrv.com') {
  lb = new LubanClient('10fcd40767474054ad3c25ae1a4d9c86', 'wx46f954bd23744e04')
}
const 样本数 = 10000
const 人员活动范围 = 5
const 每天活动人数比例 = 0.3
const 分块尺寸 = 5
const 同区域感染率 = 0.1 / 分块尺寸
const 初始感染数 = 样本数 / 2000
const 潜伏期 = 14
const 自愈概率 = 0.1
const 治疗康复概率 = 0.9
const 最大治疗天数 = 25
const 床位数 = 样本数 / 500

let pixi = null
let points = []
let distMap = {}

function gaussianRand() {
  let rand = 0
  for (let i = 0; i < 6; i++) {
    rand += Math.random()
  }
  return rand / 6
}

function square(num) {
  return Math.pow(num, 2)
}

function kaiSquare(num) {
  return Math.pow(num, .5)
}

function dist(p1, p2) {
  let p1x = Math.round(p1.x)
  let p1y = Math.round(p1.y)
  let p2x = Math.round(p2.x)
  let p2y = Math.round(p2.y)
  let key1 = `${p1x}${p1y}${p2x}${p2y}`
  let key2 = `${p2x}${p2y}${p1x}${p1y}`
  let result = distMap[key1] || distMap[key2]
  if (!result) {
    result = square(p1x - p2x) + square(p1y - p2y)
  }
  distMap[key1] = result
  distMap[key2] = result
  return result
}

function getRandItem(arr) {
  return arr[Math.floor(Math.random() * arr.length)]
}

function getRandItems(arr, count=5) {
  let results = []
  if (arr.length <= count) {
    return results.concat(arr)
  } else {
    while (results.length < count) {
      let item = getRandItem(arr)
      if (!results.includes(item)) {
        results.push(item)
      }
    }
    return results
  }
}

export default {
  name: 'app',
  data: () => {
    return {
      showForm: true,
      day: 1,
      dayTimer: null,
      deadCount: 0,
      healCount: 0,
      infectCount: 0,
      quezhenCount: 0,
      人员活动范围,
      每天活动人数比例,
      freeBedNum: 床位数,
      maxFreeBedNum: 床位数,
      infectSize: 分块尺寸,
      initInfectCount: 初始感染数,
      totalCount: 样本数,
      selfHealRate: 自愈概率,
      hospitalHealRate: 治疗康复概率,
      infectRate: 同区域感染率,
      inited: false,
      paused: false,
      nickname: '',
      avatar: ''
    }
  },
  mounted() {
    if (location.hostname === 'cdn.nodesrv.com') {
      lb.login().then(res => {
        this.showForm = true
        this.nickname = res.nickname
        this.avatar = res.headimgurl.replace(/132$/, '0')
      }).catch(e => {
        this.showForm = true
      })
    } else {
      this.showForm = true
    }
  },
  methods: {
    createPoint(color=0xFFFFFF) {
      let x = gaussianRand() * pixi.renderer.width, y = gaussianRand() * pixi.renderer.height
      let g = new PIXI.Graphics()
      g.beginFill(color)
      g.drawCircle(0, 0, .5)
      g.endFill()
      pixi.stage.addChild(g)
      g.x = x
      g.y = y
      g.__xindex = Math.floor(g.x / this.infectSize)
      g.__yindex = Math.floor(g.y / this.infectSize)
      return g
    },
    pause() {
      this.paused = true
    },
    goon() {
      this.paused = false
    },
    init() {
      if (pixi) {
        pixi.destroy()
      }
      // 初始化数据
      points = []
      this.inited = true
      this.paused = false
      this.day = 1
      this.freeBedNum = this.maxFreeBedNum || 床位数
      this.deadCount = 0
      this.healCount = 0
      this.infectCount = 0
      this.quezhenCount = 0
      this.人员活动范围 = this.人员活动范围 || 人员活动范围
      this.selfHealRate = this.selfHealRate || 自愈概率
      this.hospitalHealRate = this.hospitalHealRate || 治疗康复概率
      this.每天活动人数比例 = this.每天活动人数比例 || 每天活动人数比例
      this.infectSize = this.infectSize || 分块尺寸
      this.initInfectCount = this.initInfectCount || 初始感染数
      this.infectRate = this.infectRate || 同区域感染率
      // 创建舞台实例
      pixi = new PIXI.Application({
        resizeTo: document.getElementById('virusmap')
      })
      document.getElementById('virusmap').innerHTML = ''
      document.getElementById('virusmap').appendChild(pixi.view)
      for (let i = 0; i < this.totalCount; i++) {
        let p = this.createPoint()
        points.push(p)
      }
      this.infectPoints(this.initInfectCount)
      pixi.ticker.add(this.tickHandler)
      this.startDayTimer()
      this.showForm = false
    },
    tickHandler() {
      if (!this.paused) {
        distMap = {}
        points.forEach(this.goAround)
      }
    },
    startDayTimer() {
      if (this.dayTimer) {
        clearInterval(this.dayTimer)
      }
      this.dayTimer = setInterval(() => {
        if (!this.paused) {
          this.day++
          this.updateHealed()
          this.updateDead()
          this.updateQueZhen()
          this.updateInfect()
          if (this.infectCount === this.quezhenCount && this.infectCount > 0) {
            alert(`恭喜${this.nickname||'您'}在第${this.day}天控制住了疫情！`)
            this.pause()
          }
        }
      }, 1000)
    },
    heal(p) {
      if (p.infectStatus !== 2) {
        return
      }
      if (p.__inHospital === true) {
        this.freeBedNum++
      }
      let newp = this.createPoint(0x00ff00)
      newp.infectStatus = 3
      newp.x = p.x
      newp.y = p.y
      newp.__xindex = p.__xindex
      newp.__yindex = p.__yindex
      let pIndex = points.indexOf(p)
      points[pIndex] = newp
      p.destroy()
      pixi.stage.addChild(newp)
      this.healCount++
    },
    dead(p) {
      if (p.infectStatus !== 2) {
        return
      }
      this.deadCount++
      if (p.__inHospital === true) {
        this.freeBedNum++
      }
      let newp = this.createPoint(0x333333)
      newp.infectStatus = 4
      newp.x = p.x
      newp.y = p.y
      newp.__xindex = p.__xindex
      newp.__yindex = p.__yindex
      let pIndex = points.indexOf(p)
      points[pIndex] = newp
      p.destroy()
      pixi.stage.addChild(newp)
    },
    updateHealed() {
      points.forEach(p => {
        if (p.__healDay === this.day) {
          this.heal(p)
        }
      })
    },
    updateDead() {
      points.forEach(p => {
        if (p.__deadDay === this.day) {
          this.dead(p)
        }
      })
    },
    updateInfect() {
      let maxXIndex = Math.floor(pixi.renderer.width / this.infectSize)
      let maxYIndex = Math.floor(pixi.renderer.height / this.infectSize)
      let blocks = {}
      points.forEach(p => {
        let key = `${p.__xindex},${p.__yindex}`
        blocks[key] = blocks[key] || []
        blocks[key].push(p)
      })
      for (let k in blocks) {
        let _points = blocks[k]
        let healthyPoints = _points.filter(item => !item.infectStatus)
        let infectedPoints = _points.filter(item => item.infectStatus === 1)
        let quezhenPoints = _points.filter(item => item.infectStatus === 2)
        let badCount = infectedPoints.length + quezhenPoints.length
        if (badCount > 0) {
          healthyPoints.forEach(hp => {
            let isInfect = Math.random() < this.infectRate
            if (isInfect) {
              this.infect(hp)
            }
          })
        }
      }
    },
    updateQueZhen() {
      points.forEach(p => {
        if (p.__queZhenDay === this.day) {
          this.queZhen(p)
        }
      })
    },
    infectPoints(count) {
      getRandItems(points, count).forEach(this.infect)
    },
    goAround(p) {
      if ((p.infectStatus === 2 && p.__inHospital) || p.infectStatus === 4) {
        // dead or in hospital
        return
      }
      if (Math.random() < this.每天活动人数比例) {
        p.x += gaussianRand() * this.人员活动范围 - this.人员活动范围 / 2
        p.y += gaussianRand() * this.人员活动范围 - this.人员活动范围 / 2
        p.__xindex = Math.floor(p.x / 分块尺寸)
        p.__yindex = Math.floor(p.y / 分块尺寸)
      }
    },
    infect(p) {
      if (p.infectStatus > 0) {
        return
      }
      this.infectCount++
      let newp = this.createPoint(0xf3ff5d)
      newp.infectStatus = 1
      newp.x = p.x
      newp.y = p.y
      newp.__xindex = p.__xindex
      newp.__yindex = p.__yindex
      newp.__infectDay = this.day
      newp.__queZhenDay = this.day + Math.ceil(潜伏期 * gaussianRand())
      let pIndex = points.indexOf(p)
      points[pIndex] = newp
      p.destroy()
      pixi.stage.addChild(newp)
    },
    queZhen(p) {
      if (p.infectStatus > 1) {
        return
      }
      this.quezhenCount++
      let newp = this.createPoint(0xff0000)
      newp.infectStatus = 2
      newp.x = p.x
      newp.y = p.y
      // 确诊了，看看是否有床位？
      if (this.freeBedNum > 0) {
        // 有床位
        this.freeBedNum--
        let willGood = Math.random() < this.hospitalHealRate
        let outDay = this.day + Math.ceil(最大治疗天数 * gaussianRand())
        newp.__inHospital = true
        if (willGood) {
          newp.__healDay = outDay
        } else {
          newp.__deadDay = outDay
        }
      } else {
        // 没有床位，只能自求多福
        let willGood = Math.random() < this.selfHealRate
        let outDay = this.day + Math.ceil(最大治疗天数 * gaussianRand())
        if (willGood) {
          newp.__healDay = outDay
        } else {
          newp.__deadDay = outDay
        }
      }
      let pIndex = points.indexOf(p)
      points[pIndex] = newp
      p.destroy()
      pixi.stage.addChild(newp)
    },
    toggleForm() {
      this.showForm = !this.showForm
      if (this.showForm === true) {
        if (!this.paused) {
          this.pause()
        }
      } else {
        if (this.paused) {
          this.goon()
        }
      }
    }
  }
}
</script>

<style lang="less">
html, body {
  padding: 0;
  margin: 0;
  height: 100%;
}
#container {
  height: 100%;
  background-color: #000;
}
#dayinfo {
  position: fixed;
  background: rgba(0,0,0,.5);
  color:#fff;
  top: 10px;
  left: 0;
  right: 0;
  text-align: center;
  border-radius: 3px;
  font-size: 20px;
  line-height: 40px;
}
#virusmap {
  width: 100%;
  height: 50%;

  canvas {
    width: 100%;
    height: 100%;
    transform: translateY(50%);
  }
}
#info {
  position: fixed;
  bottom: 60px;
  left: 10px;
  font-size: 12px;
  line-height: 18px;
  background: rgba(0,0,0,.5);
  padding: 10px 10px 10px 25px;
  margin: 0;
  border-radius: 5px;
}
#form {
  position: fixed;
  right: 0;
  bottom: 0;
  color: #fff;
  padding: 10px 20px;
  width: 100%;
  box-sizing: border-box;
  font-size: 12px;
  z-index: 100;
  line-height: 40px;
  transition: all .3s ease-out;

  &.expand {
    background: rgba(85, 115, 252, 0.836);
  }

  .btn {
    font-size: 16px;
    line-height: 40px;

    &.btn-close {
      display: block;
      position: absolute;
      right: 10px;
      top: -50px;
      width: 40px;
      height: 40px;
      background: rgba(85, 115, 252, 0.836);
      color: #fff;
      line-height: 40px;
      text-align: center;
      border-radius: 20px;
      transform: rotate(45deg);
      font-size: 30px;
    }
  }

  .confirm {
    background: #fff;
    display: inline-block;
    margin-right: 10px;
  }
}
#avatar {
  position: fixed;
  right: 10px;
  bottom: 10px;
  width: 40px;
  height: 40px;
  border-radius: 20px;
  overflow: hidden;
  z-index: 10;

  img {
    width: 100%;
    height: 100%;
    display: block;
  }
}
</style>
