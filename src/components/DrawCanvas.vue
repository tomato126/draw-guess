<template>
    <div style="width: 100%; height: 100%">
        <div v-if="showGameBefore">轮到你进行画画，这轮的词语是{{guessWord}}</div>
        <div v-if="showCanvas">本轮词语：{{guessWord}}
            <Canvas id="myCanvas" ref="myCanvas" width="700" height="380"></Canvas>
        </div>
    </div>
</template>

<script>
import generateWord from '../utils/guessWords'
import generateDrawer from '../utils/whoIsNextDrawer'
import { sleep } from '../utils/sleep'

export default {
    data() {
        return {
            showGameBefore: false,
            showCanvas: false,
            showAnswer: false,
            ws: this.$store.state.ws,
            gameBeginTimer: null,
            gameTimer: null,
            gameOverTimer: null,
            draw: null
        }
    },
    computed: {
        guessWord () {
            return this.$store.state.guessWord
        },
        userList () {
            return this.$store.state.userList
        },
        drawer () {
            return this.$store.state.drawer
        },
        gameStatus () {
            return this.$store.state.gameStatus
        },
        drawColor () {
            return this.$store.state.color
        },
        drawTool () {
            return this.$store.state.tool
        }
    },
    watch: {
        status (val) {
            if (val == 'begin') {
                console.log(1111)
            }
        },
        gameStatus (val) {
            if (val == 'begin') {
                this.init()
            }
        },
        drawColor (val) {
            if (this.draw) {
                this.draw.changeColor(val)
				this.draw.changeFillColor(val)
            }
        },
        drawTool (val) {
            // tool: brush 笔  eraser 橡皮檫 square 矩形 border 矩形框 circle 圆 radius 圆框 line 直线 arc 圆弧
            if (this.draw) {
				switch (val) {
					case 'brush':
						this.draw.draw()
						break
					case 'line':
						this.draw.drawLine()
						break
					case 'circle':
						this.draw.drawRound()
						break
					case 'radius':
						this.draw.drawRadius()
						break
					case 'square':
						this.draw.drawRect()
						break
					case 'border':
						this.draw.drawBorder()
						break
					default:
						break
				}
            }
        }
    },
    created () {
        this.init()
    },
    methods: {
        init () {
            if (this.$store.state.gameStatus == 'begin') {
                console.log('new game begin')
                let that = this

				async function play () {
					that.showGameBefore = true
					await sleep(10)

					that.showGameBefore = false
					that.showCanvas = true
					that.$nextTick(() => {
						that.draw = new canvasDraw("myCanvas", that.$store.state.ws);
						that.draw.draw()
                    })

					await sleep(60)
					that.ws.send(JSON.stringify({type: 'gameOver'}))
					that.showCanvas = false
					that.showAnswer = true

					await sleep(15)
					that.showAnswer = false
					//轮流当画家  // 随机一个选词
					let newDrawer = generateDrawer(that.drawer, that.userList)
					let word = generateWord()
					//发送信令告知新一轮游戏开始

					that.ws.send(JSON.stringify({type: 'gameBegin', drawer: newDrawer, guessWord: word, position: 'NewGame'}))
					console.log(newDrawer, word)
				}

				play()
            }
        }
    }
}


class canvasDraw {
	constructor (id,ws) {
		this.ws = ws
		this.canvas = document.getElementById(id)
		this.ctx = this.canvas.getContext('2d')
		this.canvasRect = this.canvas.getBoundingClientRect()
		this.path = {
			beginX: 0,
			beginY: 0,
			endX: 0,
			endY: 0
		}
		this.isDraw = false
		this.points = []
	}
	changeColor (color) {
		this.ctx.strokeStyle = color
	}
	changeLineWidth (lineWidth) {
		this.ctx.lineWidth = lineWidth
	}
	changeFillColor (color) {
		this.ctx.fillStyle = color
	}
	drawLine () {
		let that = this
		this.canvas.onmousedown = (e) => {
			this.ctx.beginPath()
			this.path.beginX = e.pageX - this.canvasRect.left
			this.path.beginY = e.pageY - this.canvasRect.top
			this.ctx.moveTo(
				this.path.beginX,
				this.path.beginY
			)
		}
		this.canvas.onmouseup = (e) => {
			this.path.endX = e.pageX - this.canvasRect.left
			this.path.endY = e.pageY - this.canvasRect.top
			this.ctx.lineTo(
				this.path.endX,
				this.path.endY
			)
			this.ctx.stroke();
			that.ws.send(JSON.stringify({type: 'drawLine', beginX: this.path.beginX, beginY: this.path.beginY, endX: this.path.endX, endY: this.path.endY}))
		}
	}
	drawRect () {
		let that = this
		this.canvas.onmousedown = (e) => {
			this.ctx.beginPath()
			this.path.beginX = e.pageX - this.canvasRect.left
			this.path.beginY = e.pageY - this.canvasRect.top
		}
		this.canvas.onmouseup = (e) => {
			this.path.endX = e.pageX - this.canvasRect.left
			this.path.endY = e.pageY - this.canvasRect.top
			this.ctx.rect(this.path.beginX, this.path.beginY, this.path.endX - this.path.beginX, this.path.endY - this.path.beginY) 
			this.ctx.fill()
			this.ctx.stroke()
			that.ws.send(JSON.stringify({type: 'drawRect', beginX: this.path.beginX, beginY: this.path.beginY, rectWidth: this.path.endX - this.path.beginX, rectHeight: this.path.endY - this.path.beginY}))
		}
	}
	drawBorder () {
		let that = this
		this.canvas.onmousedown = (e) => {
			this.ctx.beginPath()
			this.path.beginX = e.pageX - this.canvasRect.left
			this.path.beginY = e.pageY - this.canvasRect.top
		}
		this.canvas.onmouseup = (e) => {
			this.path.endX = e.pageX - this.canvasRect.left
			this.path.endY = e.pageY - this.canvasRect.top
			this.ctx.rect(this.path.beginX, this.path.beginY, this.path.endX - this.path.beginX, this.path.endY - this.path.beginY) 
			this.ctx.stroke()
			that.ws.send(JSON.stringify({type: 'drawBorder', beginX: this.path.beginX, beginY: this.path.beginY, rectWidth: this.path.endX - this.path.beginX, rectHeight: this.path.endY - this.path.beginY}))
		}
	}
	drawRound () {
		let that = this
		this.canvas.onmousedown = (e) => {
			this.ctx.beginPath()
			this.path.beginX = e.pageX - this.canvasRect.left
			this.path.beginY = e.pageY - this.canvasRect.top
		}
		this.canvas.onmouseup = (e) => {
			this.path.endX = e.pageX - this.canvasRect.left
			this.path.endY = e.pageY - this.canvasRect.top
			let width = this.path.endX - this.path.beginX
			let height = this.path.endY - this.path.beginY
			this.ctx.arc(this.path.beginX + width / 2, this.path.beginY + height / 2, Math.sqrt(width * width + height * height) / 2, 0, Math.PI * 2)
			this.ctx.fill()
			this.ctx.stroke()
			that.ws.send(JSON.stringify({type: 'drawRound', beginX: this.path.beginX, beginY: this.path.beginY, roundWidth: width, roundHeight: height}))
		}
	}
	drawRadius() {
		let that = this
		this.canvas.onmousedown = (e) => {
			this.ctx.beginPath()
			this.path.beginX = e.pageX - this.canvasRect.left
			this.path.beginY = e.pageY - this.canvasRect.top
		}
		this.canvas.onmouseup = (e) => {
			this.path.endX = e.pageX - this.canvasRect.left
			this.path.endY = e.pageY - this.canvasRect.top
			let width = this.path.endX - this.path.beginX
			let height = this.path.endY - this.path.beginY
			this.ctx.arc(this.path.beginX + width / 2, this.path.beginY + height / 2, Math.sqrt(width * width + height * height) / 2, 0, Math.PI * 2)
			this.ctx.stroke()
			that.ws.send(JSON.stringify({type: 'drawRadius', beginX: this.path.beginX, beginY: this.path.beginY, roundWidth: width, roundHeight: height}))
		}
	}
	draw () {
		let that = this
		this.canvas.onmousedown = (e) => {
			that.ctx.beginPath()
			that.ctx.lineWidth = 1.5
			that.ctx.lineJoin = that.ctx.lineCap = 'round'
			that.ctx.shadowBlur = 1.5
			that.ctx.shadowColor = 'rgb(0, 0, 0)'
			that.path.beginX = e.pageX - that.canvasRect.left
			that.path.beginY = e.pageY - that.canvasRect.top
			// 收集点
			that.points.push({ x: that.path.beginX, y: that.path.beginY })
			that.ctx.moveTo(
				that.path.beginX,
				that.path.beginY
			)
			that.isDraw = true
		}
		this.canvas.onmousemove = (e) => {
			if (that.isDraw) {
				that.drawing(e)
			}
		}
		this.canvas.onmouseup = () => {
			that.isDraw = false
            that.ws.send(JSON.stringify({type: 'stop'}))
		}
	}
	drawing (e) {
		this.path.endX = e.pageX - this.canvasRect.left
		this.path.endY = e.pageY - this.canvasRect.top
		this.ws.send(JSON.stringify({type: 'draw', beginX: this.path.beginX, beginY: this.path.beginY, endX: this.path.endX, endY: this.path.endY}))

		this.points.push({ x: this.path.endX, y: this.path.endY })

		if (this.points.length > 3) {
			const lastTwoPoints = this.points.slice(-2)
			const controlPoint = lastTwoPoints[0]
			const endPoint = {
				x: (lastTwoPoints[0].x + lastTwoPoints[1].x) / 2,
				y: (lastTwoPoints[0].y + lastTwoPoints[1].y) / 2,
			}
			this.handleSawtooth({x: this.path.beginX, y: this.path.beginY }, controlPoint, endPoint)
			this.path.beginX = endPoint.x
			this.path.beginY = endPoint.y
		}
		this.ctx.stroke()
	}
	getMidPoint (p1, p2) {
		return {
			x: p1.x + (p2.x - p1.x) / 2,
			y: p1.y + (p2.y - p1.y) / 2
		}
	}
	handleSawtooth (beginPoint, controlPoint, endPoint) {
		this.ctx.beginPath();
		this.ctx.moveTo(beginPoint.x, beginPoint.y);
		this.ctx.quadraticCurveTo(controlPoint.x, controlPoint.y, endPoint.x, endPoint.y);
	}
}
</script>

<style lang="less" scoped>

</style>