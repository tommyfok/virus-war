<template>
  <div id="container">
    病毒图：
    <div id="virusmap"></div>
    第{{day}}天
  </div>
</template>

<script>
import * as PIXI from 'pixi.js'
import { clearInterval } from 'timers';
const 移动范围 = 2
const 感染率 = 0.1
const 样本数 = 20000
const 初始感染数 = 10
const 分块尺寸 = 5
const 潜伏期 = 14
const 自愈概率 = 0.5
const 治疗康复概率 = 0.8
const 治疗周期 = 20
const 床位数 = 样本数 / 1000

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
      day: 1,
      startTime: 0,
      configs: {
        totalCount: 样本数
      }
    }
  },
  mounted() {
    this.init()
  },
  methods: {
    createPoint(color=0xFFFFFF) {
      let x = gaussianRand() * pixi.renderer.width, y = gaussianRand() * pixi.renderer.height
      let g = new PIXI.Graphics()
      g.beginFill(color)
      g.drawCircle(0, 0, 1)
      g.endFill()
      pixi.stage.addChild(g)
      g.x = x
      g.y = y
      g.__xindex = Math.floor(g.x / 分块尺寸)
      g.__yindex = Math.floor(g.y / 分块尺寸)
      return g
    },
    init() {
      if (pixi) {
        pixi.destroy()
      }
      points = []
      this.startTime = Date.now()
      pixi = new PIXI.Application()
      document.getElementById('virusmap').innerHTML = ''
      document.getElementById('virusmap').appendChild(pixi.view)
      for (let i = 0; i < this.configs.totalCount; i++) {
        let p = this.createPoint()
        points.push(p)
      }
      this.infectPoints(初始感染数)
      pixi.ticker.add(() => {
        distMap = {}
        points.forEach(this.goAround)
      })
      this.startDayTimer()
    },
    startDayTimer() {
      if (this.dayTimer) {
        clearInterval(this.dayTimer)
      }
      this.dayTimer = setInterval(() => {
        this.day++
        this.updateInfect()
        this.updateQueZhen()
      }, 1000)
    },
    updateInfect() {
      let maxXIndex = Math.floor(pixi.renderer.width / 分块尺寸)
      let maxYIndex = Math.floor(pixi.renderer.height / 分块尺寸)
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
          let infectRate = badCount / _points.length
          healthyPoints.forEach(hp => {
            let isInfect = Math.random() < 感染率
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
      p.x += gaussianRand() * 移动范围 - 移动范围 / 2
      p.y += gaussianRand() * 移动范围 - 移动范围 / 2
      p.__xindex = Math.floor(p.x / 分块尺寸)
      p.__yindex = Math.floor(p.y / 分块尺寸)
    },
    // getNearByPoints(p) {
    //   let result = []
    //   let distSquare = square(传染距离)
    //   points.forEach(_p => {
    //     if (dist(p, _p) < distSquare) {
    //       result.push(_p)
    //     }
    //   })
    //   return result
    // },
    infect(p) {
      if (p.infectStatus > 0) {
        return
      }
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
      let newp = this.createPoint(0xff0000)
      newp.infectStatus = 2
      newp.x = p.x
      newp.y = p.y
      let pIndex = points.indexOf(p)
      points[pIndex] = newp
      p.destroy()
      pixi.stage.addChild(newp)
    }
  }
}
</script>

<style>

</style>
