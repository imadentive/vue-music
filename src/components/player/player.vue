<template>
	<div class="player" v-show="playlist.length>0">
		<transition name="normal"
					@enter="enter"
					@after-enter="afterEnter"
					@leave="leave"
					@after-leave="afterLeave"
		>
			<div class="normal-polayer" v-show="fullScreen">
				<div class="background">
					<img width="100%" height="100%" :src="currentSong.image">
				</div>
				<div class="top">
					<div class="back" @click="back">
						<i class="icon-back"></i>
					</div>
					<h1 class="title" v-html="currentSong.name"></h1>
					<h2 class="subtitle" v-html="currentSong.singer"></h2>
				</div>
				<div class="middle">
					<div class="middle-l">
						<div class="cd-wrapper" ref="cdWrapper">
							<div class="cd" :class="playAnimation">
								<img class="image" :src="currentSong.image">
							</div>
						</div>
					</div>
					<!--currentLyric.lines默认是null，所以加多一个currentLyric，不然会报错-->
					<scroll class="middle-r" ref="lyricList" :loadData="currentLyric && currentLyric.lines">
						<div class="lyric-wrapper">
							<div v-if="currentLyric">
								<p class="text" ref="lyricLine"
								   :class="{'current': currentLineNum === index}"
								   v-for="(line,index) in currentLyric.lines">
									{{line.txt}}</p>
							</div>
						</div>
					</scroll>
				</div>
				<div class="bottom">
					<div class="progress-wrapper">
						<span class="time time-l">{{format(currentTime)}}</span>
						<div class="progress-bar-wrapper">
							<progress-bar :percent="percent" @percentChange="onPercentChange"></progress-bar>
						</div>
						<span class="time time-r">{{format(currentSong.duration)}}</span>
					</div>
					<div class="operators">
						<div class="icon i-left" @click="changeMode">
							<i :class="iconMode"></i>
						</div>
						<div class="icon i-left" :class="disableCls">
							<i class="icon-prev" @click="prev"></i>
						</div>
						<div class="icon i-center" :class="disableCls">
							<i :class="playIcon" @click="togglePlaying"></i>
						</div>
						<div class="icon i-right" :class="disableCls">
							<i class="icon-next" @click="next"></i>
						</div>
						<div class="icon i-right">
							<i class="icon icon-not-favorite"></i>
						</div>
					</div>
				</div>
			</div>
		</transition>
		<transition name="mini">
			<div class="mini-player" v-show="!fullScreen" @click="open">
				<div class="icon">
					<img width="100%" height="100%" :src="currentSong.image" :class="playAnimation">
				</div>
				<div class="text">
					<h2 class="name" v-html="currentSong.name"></h2>
					<p class="desc" v-html="currentSong.singer"></p>
				</div>
				<div class="control">
					<progress-circle :radius="radius" :percent="percent">
						<i :class="miniIcon" class="icon-mini" @click.stop="togglePlaying"></i>
					</progress-circle>
				</div>
				<div class="control">
					<i class="icon-playlist"></i>
				</div>
			</div>
		</transition>
		<!--
		audio说明：
		1.歌曲可以播放的时候，派发一个oncanplay的事件，出错的时候派发onerror事件
		2.ontimeupdate事件：播放时间改变时(当前的播放位置发送改变时触发)
		3.onended事件：当歌曲播放完之后，派发的件事
		-->
		<audio :src="currentSong.url" ref="audio" @canplay="ready" @error="error" @timeupdate="updateTime"
			   @ended="end"></audio>
	</div>
</template>

<script type="text/ecmascript-6">
	import {mapGetters, mapMutations} from 'vuex'
	import animations from 'create-keyframe-animation'
	import {prefixStyle} from 'common/js/dom'
	import ProgressBar from 'base/progress-bar/progress-bar'
	import ProgressCircle from 'base/progress-circle/progress-circle'
	import {playMode} from 'common/js/config'
	import {shuffle} from 'common/js/util'
	import Lyric from 'lyric-parser'
	import Scroll from 'base/scroll/scroll'

	const transform = prefixStyle('transform')

	export default {
		data() {
			return {
				songReady: false,
				currentTime: 0,
				radius: 32,
				currentLyric: null, //歌词
				currentLineNum: 0 //当前歌词所在的行数
			}
		},
		methods: {
			back() { //收起播放器，并展开mini播放器
				this.setFullScreen(false)
			},
			open() { //展开播放器
				this.setFullScreen(true)
			},
			enter(el, done) {
				const {x, y, scale} = this._getPosAndScale()

				let animation = {
					0: {
						transform: `translate3d(${x}px,${y}px,0) scale(${scale})`
					},
					60: {
						transform: `translate3d(0,0,0) scale(1.1)`
					},
					100: {
						transform: `translate3d(0,0,0) scale(1)`
					}
				}
				//设置animations
				animations.registerAnimation({
					name: 'move', //名称
					animation, //动画
					presets: { //预设
						duration: 400, //动画间隔
						easing: 'linear'//缓动
					}
				})
				//运行animations
				animations.runAnimation(this.$refs.cdWrapper, 'move', done)
			},
			afterEnter() {
				animations.unregisterAnimation('move') //销毁move
				this.$refs.cdWrapper.style.animation = '' //置空
			},
			leave(el, done) {
				this.$refs.cdWrapper.style.transition = 'all 0.4s'
				const {x, y, scale} = this._getPosAndScale()
				this.$refs.cdWrapper.style[transform] = `translate3d(${x}px,${y}px,0) scale(${scale})`
				//监听transitionend事件完了之后，回调done，只有执行done之后才会执行afterLeave()
				this.$refs.cdWrapper.addEventListener('transitionend', done)
			},
			afterLeave() { //置空对DOM的操作
				this.$refs.cdWrapper.style.transition = ''
				this.$refs.cdWrapper.style[transform] = ''
			},
			changeMode() {
				const mode = (this.mode + 1) % 3 //3取余
				this.setPlayMode(mode)
				let list = null
				if (mode === playMode.random) {
					list = shuffle(this.sequenceList)
				} else {
					list = this.sequenceList
				}
				this.resetCurrentIndex(list) //解决：切换播放模式时，当前正在播放的歌曲不在发生改变
				this.setPlayList(list)
			},
			resetCurrentIndex(list) {
				let index = list.findIndex((item) => {
					return item.id === this.currentSong.id
				})
				this.setCurrentIndex(index)
			},
			togglePlaying() { //播放、暂停
				this.setPlayingState(!this.playing)
			},
			prev() { //上一首
				if (!this.songReady) {
					return
				}
				let index = this.currentIndex - 1
				if (index === -1) { //当歌曲是第一首歌的时候
					index = this.playlist.length - 1
				}
				this.setCurrentIndex(index)
				if (!this.playing) { //暂停时切换，切换完成之后自动播放
					this.togglePlaying()
				}
				this.songReady = false
			},
			end() { //当歌曲播放完之后
				if (this.mode === playMode.loop) { //单曲循环状态时
					this.loop()
				} else { //随机、顺序播放状态时
					this.next()
				}
			},
			loop() {
				this.$refs.audio.currentTime = 0
				this.$refs.audio.play()
			},
			next() { //下一首
				if (!this.songReady) {
					return
				}
				let index = this.currentIndex + 1
				if (index === this.playlist.length) { //当歌曲是最后首歌的时候
					index = 0
				}
				this.setCurrentIndex(index)
				if (!this.playing) {
					this.togglePlaying()
				}
				this.songReady = false
			},
			ready() {
				this.songReady = true //可以播放的时候设置为true的状态
			},
			error() {
				this.songReady = true
			},
			updateTime(e) {
				this.currentTime = e.target.currentTime
			},
			format(interval) {
				interval = interval | 0 //向下取整,相当于Math.floor
				const minute = interval / 60 | 0 //分钟，| 0为向下取整
				const second = this._pad(interval % 60)
				return `${minute}:${second}`
			},
			onPercentChange(percent) {
				const currentTime = this.currentSong.duration * percent
				/*
				 * currentTime属性设置或返回播放的当前位置（以秒计）。
				 * 当设置该属性时，播放会跳跃到指定的位置。
				 * */
				this.$refs.audio.currentTime = currentTime
				if (!this.playing) { //拖放完成之后，如果是暂停状态，则触发播放
					this.togglePlaying()
				}
			},
			getLyric() {
				this.currentSong.getLyric().then((lyric) => {
					this.currentLyric = new Lyric(lyric, this.handleLyric) //this.handleLyric回调函数
					if (this.playing) {
						this.currentLyric.play()
					}
				})
			},
			handleLyric({lineNum, txt}) { //当每行歌词发生改变的时候，回调
				this.currentLineNum = lineNum
				if (lineNum > 5) {
					let lineEl = this.$refs.lyricLine[lineNum - 5]
					this.$refs.lyricList.scrollToElement(lineEl, 1000)
				} else {
					this.$refs.lyricList.scrollTo(0, 0, 1000) //滚动到顶部
				}
			},
			_pad(num, n = 2) { //补0
				let len = num.toString().length
				while (len < n) {
					num = '0' + num
					len++
				}
				return num
			},
			_getPosAndScale() {
				const targetWidth = 40 //小CD的宽度
				const paddingLeft = 40 //小CD的中心点到左边线的宽度
				const paddingBottom = 30 //小CD的中心点到下边线的高度
				const paddingTop = 80 //大CD： top值的高度
				const width = window.innerWidth * 0.8 //大CD的宽度
				const scale = targetWidth / width //缩小比例（CD从大到小的缩放比例）
				const x = -(window.innerWidth / 2 - paddingLeft) //小CD的X轴坐标
				const y = window.innerHeight - paddingTop - width / 2 - paddingBottom //小CD的Y轴坐标
				return {
					x,
					y,
					scale
				}
			},
			...mapMutations({
				setFullScreen: 'SET_FULL_SCREEN',
				setPlayingState: 'SET_PLAYING_STATE',
				setCurrentIndex: 'SET_CURRENT_INDEX',
				setPlayMode: 'SET_PLAY_MODE',
				setPlayList: 'SET_PLAYLIST'
			})
		},
		watch: {
			currentSong(newSong, oldSong) {
				if (newSong.id === oldSong.id) {
					//解决：当前歌曲暂停时，切换播放模式的时候歌自动播放BUG
					return
				}
				setTimeout(() => { //确保DOM的渲染之后
					this.$refs.audio.play()
					this.getLyric()
				}, 20)
			},
			playing(newval) {
				const audio = this.$refs.audio
				this.$nextTick(() => { //确保DOM的渲染之后
					newval ? audio.play() : audio.pause()
				})
			}
		},
		computed: { //计算
			playIcon() {
				return this.playing ? 'icon-pause' : 'icon-play'
			},
			playAnimation() {
				return this.playing ? 'play' : 'play pause'
			},
			iconMode() {
				return this.mode === playMode.sequence ? 'icon-sequence' : this.mode === playMode.loop ? 'icon-loop' : 'icon-random'
			},
			miniIcon() {
				return this.playing ? 'icon-pause-mini' : 'icon-play-mini'
			},
			disableCls() { // 图标禁用状态
				return this.songReady ? '' : 'disable'
			},
			percent() {
				return this.currentTime / this.currentSong.duration
			},
			...mapGetters([
				'fullScreen', //控制播放器的显示和隐藏
				'playlist', //控制播放器的渲染
				'currentSong',
				'playing', //播放的状态（正在播放、暂停中）
				'currentIndex', //当前播放索引（当前播放的首歌下标）
				'mode',	//播放模式
				'sequenceList' //歌曲列表（随机、顺序、单曲）
			])
		},
		components: {
			ProgressBar,
			ProgressCircle,
			Scroll
		}
	}

</script>

<style scoped lang="stylus" rel="stylesheet/stylus">
	@import "~common/stylus/variable"
	@import "~common/stylus/mixin"

	.player
		.normal-polayer
			position: fixed
			left: 0
			right: 0
			top: 0
			bottom: 0
			z-index: 150
			background: $color-background
			.background
				position: absolute
				top: 0
				left: 0
				width: 100%
				height: 100%
				z-index: -1
				opacity: 0.6
				filter: blur(20px)
			.top
				position: relative
				margin-bottom: 20px
				.back
					position: absolute
					top: 0
					left: 6px
					z-index: 50
					.icon-back
						display: block
						padding: 9px
						font-size: $font-size-large-x
						color: $color-theme
						transform: rotate(-90deg)
				.title
					width: 70%
					margin: 0 auto
					line-height: 40px
					text-align: center
					no-wrap()
					font-size: $font-size-large
					color: $color-text
				.subtitle
					line-height: 20px
					text-align: center
					font-size: $font-size-medium
					color: $color-text
			.middle
				position: fixed
				width: 100%
				top: 80px
				bottom: 170px
				white-space: nowrap
				font-size: 0
				.middle-l
					display: inline-block
					vertical-align: top
					position: relative
					width: 100%
					height: 0
					padding-top: 80%
					.cd-wrapper
						position: absolute
						left: 10%
						top: 0
						width: 80%
						height: 100%
						.cd
							width: 100%
							height: 100%
							box-sizing: border-box
							border: 10px solid rgba(255, 255, 255, 0.1);
							border-radius: 50%
							&.play
								animation: rotate 20s linear infinite
							&.pause
								animation-play-state: paused
							.image
								position: absolute
								left: 0
								top: 0
								width: 100%
								height: 100%
								border-radius: 50%
				.middle-r
					display: inline-block
					vertical-align: top
					width: 100%
					height: 100%
					overflow: hidden
					.lyric-wrapper
						width: 80%
						margin: 0 auto
						overflow: hidden
						text-align: center
						.text
							line-height: 32px
							color: $color-text-l
							font-size: $font-size-medium
							&.current
								color: $color-text
			.bottom
				position: absolute
				bottom: 50px
				width: 100%
				.progress-wrapper
					display: flex
					align-items: center
					width: 80%
					margin: 0 auto
					padding: 10px 0
					.time
						flex: 0 0 30px
						width: 30px
						ine-height: 30px
						color: $color-text
						font-size: $font-size-small
						&.time-l
							text-align: left
						&.time-r
							text-align: right
					.progress-bar-wrapper
						flex: 1
				.operators
					display: flex
					align-items: center
					.icon
						flex: 1
						color: $color-theme
						&.disable
							color: $color-theme-d
						i
							font-size: 30px
					.i-left
						text-align: right
					.i-center
						padding: 0 20px
						text-align: center
						i
							font-size: 40px
					.i-right
						text-align: left
					.icon-favorite
						color: $color-sub-theme
			&.normal-enter-active, &.normal-leave-active
				transition: all 0.4s
				.top, .bottom
					transition: all 0.4s cubic-bezier(0.86, 0.18, 0.82, 1.32)
			&.normal-enter, &.normal-leave-to
				opacity: 0
				.top
					transform: translate3d(0, -100px, 0)
				.bottom
					transform: translate3d(0, 100px, 0)
		.mini-player
			display: flex
			align-items: center
			position: fixed
			left: 0
			bottom: 0
			z-index: 180
			width: 100%
			height: 60px
			background: $color-highlight-background
			&.mini-enter-active, &.mini-leave-active
				transition: all 0.4s
			&.mini-enter, &.mini-leave-to
				opacity: 0
			.icon
				flex: 0 0 40px
				width: 40px
				padding: 0 10px 0 20px
				img
					border-radius: 50%
					&.play
						animation: rotate 10s linear infinite
					&.pause
						animation-play-state: paused
			.text
				display: flex
				flex-direction: column
				justify-content: center
				flex: 1
				line-height: 20px
				overflow: hidden
				.name
					margin-bottom: 2px
					no-wrap()
					font-size: $font-size-medium
					color: $color-text
				.desc
					no-wrap()
					font-size: $font-size-small
					color: $color-text-d
			.control
				flex: 0 0 30px
				width: 30px
				padding: 0 10px
				.icon-play-mini, .icon-pause-mini, .icon-playlist
					font-size: 30px
					color: $color-theme-d
				.icon-mini
					font-size: 32px
					position: absolute
					left: 0
					top: 0
	@keyframes rotate
		0%
			transform: rotate(0)
		100%
			transform: rotate(360deg)
</style>
