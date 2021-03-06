<template>
	<div class="music-list">
		<div class="back" @click="back">
			<i class="icon-back"></i>
		</div>
		<h1 class="title" v-html="title"></h1>
		<div class="bg-images" :style="bgStyle" ref="bgImage">
			<div class="play-warpper">
				<div class="play" v-show="songs.length>0" ref="playBtn" @click="random">
					<i class="icon-play"></i>
					<span class="text">随机播放全部</span>
				</div>
			</div>
			<div class="filter" ref="filter"></div>
		</div>
		<div class="bg-layer" ref="layer"></div>
		<scroll :loadData="songs" :probeType="probeType" :listenScroll="listenScroll" @scroll="scroll" class="list"
				ref="list">
			<div class="song-list-wrapper">
				<song-list :songs="songs" @select="selectItem"></song-list>
			</div>
			<div class="loading-wrapper" v-show="!songs.length">
				<loading></loading>
			</div>
		</scroll>
	</div>
</template>

<script type="text/ecmascript-6">
	import scroll from 'base/scroll/scroll'
	import Loading from 'base/loading/loading'
	import SongList from 'base/song-list/song-list'
	import {prefixStyle} from 'common/js/dom'
	import {mapActions} from 'vuex'

	const RESERVED_HEIGHT = 40
	const transform = prefixStyle('transform')
	const backdrop = prefixStyle('backdrop')

	export default {
		props: {
			songs: {
				type: Array,
				default: []
			},
			bgImage: {
				type: String,
				default: ''
			},
			title: {
				type: String,
				default: ''
			}
		},
		data() {
			return {
				scrollY: 0
			}
		},
		created() {
			this.probeType = 3
			this.listenScroll = true
		},
		mounted() {
			this.imageHeight = this.$refs.bgImage.clientHeight //图片的高度
			this.minTranslateY = this.imageHeight - RESERVED_HEIGHT //translateY最大滚动高度
			this.$refs.list.$el.style['top'] = `${this.imageHeight}px`
		},
		methods: {
			back() {
				this.$router.back()
			},
			scroll(pos) {
				this.scrollY = pos.y
			},
			selectItem(item, index) {
				//item是只单首歌曲，传过来不一定要使用
				this.selectPlay({
					list: this.songs, //因为是播放整个列表所以把事个songs传过去，而不是单首歌
					index
				})
			},
			random() {
				this.randomPlay({
					list: this.songs
				})
			},
			...mapActions([
				'selectPlay',
				'randomPlay'
			])
		},
		watch: {
			scrollY(newY) {
				/*
				 * Math.max()返回两个指定的数中带有较大的值的那个数
				 * translateY 滚动不能超过this.imageHeight（图像）的最大高度
				 * 向上滚动则this.minTranslateY、newY为负值
				 * */
				let translateY = Math.max(-this.minTranslateY, newY)
				let zIndex = 0
				let scale = 1
				let blur = 0
				this.$refs.layer.style[transform] = `translate3d(0,${translateY}px,0)`
				/*
				 * 滚动到顶部：z-index=10，padding-top=0，height=40(图像的高)
				 * 滚动非顶部（CSS默认时）：z-index=0，padding-top=70%，height=0
				 * */
				if (newY < -this.minTranslateY) {
					zIndex = 10
					this.$refs.bgImage.style.paddingTop = 0
					this.$refs.bgImage.style.height = `${RESERVED_HEIGHT}px`
					this.$refs.playBtn.style.display = 'none'
				} else {
					this.$refs.bgImage.style.paddingTop = '70%'
					this.$refs.bgImage.style.height = 0
					this.$refs.playBtn.style.display = ''
				}
				/*
				 * 向下拉的时候，原理：图片的高度添加了newY的高度
				 * Math.abs() 取绝对值
				 * scale 向下的比例
				 * */
				const percent = Math.abs(newY / -this.minTranslateY)
				if (newY > 0) {
					scale = 1 + percent
					zIndex = 10
				}
				this.$refs.bgImage.style[transform] = `scale(${scale})`
				/*
				 *向上滚动实现高斯模糊（只支持iOS）
				 * */
				if (newY < 0) {
					blur = Math.min(20, percent * 20)
				}
				this.$refs.filter.style[backdrop] = `blur(${blur}px)`
				//向上滚动与向下拉时，bgImage的z-index的状态
				this.$refs.bgImage.style.zIndex = zIndex
			}
		},
		computed: {
			bgStyle() {
				return `background-image:url(${this.bgImage})`
			}
		},
		components: {
			scroll,
			SongList,
			Loading
		}
	}

</script>

<style scoped lang="stylus" rel="stylesheet/stylus">
	@import "~common/stylus/variable"

	.music-list
		position: fixed
		z-index: 100
		top: 0
		left: 0
		bottom: 0
		right: 0
		background: $color-background
		.back
			position: absolute
			z-index: 50
			top: 0
			left: 6px
			.icon-back
				display: block
				padding: 10px
				font-size: $font-size-large-x
				color: $color-theme
		.title
			position: absolute
			top: 0
			left: 10%
			z-index: 40
			width: 80%
			no-wrap()
			text-align: center
			line-height: 40px
			font-size: $font-size-large
			color: $color-text
		.bg-images
			position: relative
			width: 100%
			height: 0
			padding-top: 70%
			transform-origin: top
			background-size: cover
			.play-warpper
				position: absolute
				bottom: 20px
				z-index: 50
				width: 100%
				.play
					box-sizing: border-box
					width: 135px
					padding: 7px 0
					margin: 0 auto
					text-align: center
					border: 1px solid $color-theme
					color: $color-theme
					border-radius: 100px
					font-size: 0
					.icon-play
						display: inline-block
						vertical-align: middle
						margin-right: 6px
						font-size: $font-size-medium-x
					.text
						display: inline-block
						vertical-align: middle
						font-size: $font-size-small
		.filter
			position: absolute
			top: 0
			left: 0
			width: 100%
			height: 100%
			background: rgba(7, 17, 27, 0.4)
		.bg-layer
			position: relative
			height: 100%
			background-color: $color-background
		.list
			position: fixed
			top: 0
			bottom: 0
			width: 100%
			background-color: $color-background
			.song-list-wrapper
				padding: 20px 30px
			.loading-wrapper
				position: absolute
				top: 50%
				transform: translateY(-50%)
				width: 100%
</style>
