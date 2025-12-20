<template>
	<view class="player-wrapper" :id="videoWrapperId" :parentId="id" :randomNum="randomNum"
		:change:randomNum="domVideoPlayer.randomNumChange" :viewportProps="viewportProps"
		:change:viewportProps="domVideoPlayer.viewportChange" :videoSrc="videoSrc"
		:change:videoSrc="domVideoPlayer.initVideoPlayer" :command="eventCommand"
		:change:command="domVideoPlayer.triggerCommand" :func="renderFunc" :change:func="domVideoPlayer.triggerFunc"
		@click="videoWrapperClick" />
</template>

<script>
	export default {
		name: 'DomVideoPlayerFixed',
		props: {
			src: {
				type: String,
				default: ''
			},
			autoplay: {
				type: Boolean,
				default: false
			},
			loop: {
				type: Boolean,
				default: false
			},
			controls: {
				type: Boolean,
				default: false
			},
			controlsList: {
				type: String,
				default: ''
			},
			objectFit: {
				type: String,
				default: 'contain'
			},
			muted: {
				type: Boolean,
				default: false
			},
			playbackRate: {
				type: Number,
				default: 1
			},
			isLoading: {
				type: Boolean,
				default: false
			},
			poster: {
				type: String,
				default: ''
			},
			preload: {
				type: String,
				default: 'auto'
			},
			// https://developer.mozilla.org/zh-CN/docs/Web/HTML/Element/track
			trackList: {
				type: Array,
				default: () => []
			},
			isLog: {
				type: Boolean,
				default: true
			},
			id: {
				type: String,
				default: ''
			},
			// 新增：横屏校验参数
			videoSnapshotParams: {
				type: String,
				default: '?x-oss-process=video/snapshot,t_0,f_jpg,w_0,h_0,m_fast,ar_auto'
			},
			// 新增：是否显示横屏全屏按钮
			showLandscapeFullscreenButton: {
				type: Boolean,
				default: true
			},
			// 新增：强制横屏全屏
			forceLandscapeFullscreen: {
				type: Boolean,
				default: true
			}
		},
		data() {
			return {
				randomNum: Math.floor(Math.random() * 100000000),
				videoSrc: '',
				// 父组件向子组件传递的事件指令（video的原生事件）
				eventCommand: null,
				// 父组件传递过来的，对 renderjs 层的函数执行（对视频控制的自定义事件）
				renderFunc: {
					name: null,
					params: null
				},
				// 提供给父组件进行获取的视频属性
				currentTime: 0,
				duration: 0,
				playing: false,
				// 新增：视频方向信息
				videoOrientation: null,
				// 新增：是否显示全屏按钮
				showFullscreenButton: false,
				// 新增：是否全屏状态
				isFullscreen: false
			}
		},
		mounted() {
			// #ifdef H5
			this.isLog && console.warn('uniapp-video-player-fixed: 温馨提示，H5端建议使用原生HTML video标签效果更佳')
			// #endif
		},
		watch: {
			// 监听视频资源地址更新
			src: {
				handler(val) {
					// console.info("视频地址发生变化1");
					if (!val) return
					setTimeout(() => {
						this.videoSrc = val
						// 检查视频方向
						this.checkVideoOrientation()
					}, 0)
				},
				immediate: true
			}
		},
		computed: {
			videoWrapperId() {
				return `video-wrapper-${this.randomNum}`
			},
			// 聚合视图层的所有数据变化，传给renderjs的渲染层
			viewportProps() {
				return {
					autoplay: this.autoplay,
					muted: this.muted,
					controls: this.controls,
					controlsList: this.controlsList,
					loop: this.loop,
					objectFit: this.objectFit,
					poster: this.poster,
					isLoading: this.isLoading,
					playbackRate: this.playbackRate,
					trackList: this.trackList,
					// 新增：传递视频方向和按钮显示状态
					videoOrientation: this.videoOrientation,
					showFullscreenButton: this.showFullscreenButton,
					showLandscapeFullscreenButton: this.showLandscapeFullscreenButton,
					forceLandscapeFullscreen: this.forceLandscapeFullscreen,
					isFullscreen: this.isFullscreen
				}
			}
		},
		methods: {
			// 新增：隐藏状态栏
			hideStatusBar() {
				// #ifdef APP-PLUS
				if (plus && plus.navigator) {
					plus.navigator.setFullscreen(true)
					// 强制隐藏状态栏
					plus.navigator.setStatusBarBackground('#000000')
					plus.navigator.setStatusBarStyle('dark')
				}
				// #endif
			},

			// 新增：显示状态栏
			showStatusBar() {
				// #ifdef APP-PLUS
				if (plus && plus.navigator) {
					plus.navigator.setFullscreen(false)
					// 恢复状态栏
					plus.navigator.setStatusBarBackground('#ffffff')
					plus.navigator.setStatusBarStyle('light')
				}
				// #endif
			},
			videoWrapperClick(e) {
				this.$emit('video-click', e)
			},
			// 传递事件指令给父组件
			eventEmit({
				event,
				data
			}) {
				if (event === 'fullscreenchange') {
					this.isFullscreen = data
				}
				this.$emit(event, data)
			},
			// 修改view视图层的data数据
			setViewData({
				key,
				value
			}) {
				key && this.$set(this, key, value)
			},
			// 重置事件指令
			resetEventCommand() {
				this.eventCommand = null
			},
			// 播放指令
			play() {
				this.eventCommand = 'play'
			},
			// 暂停指令
			pause() {
				this.eventCommand = 'pause'
			},
			// 重置自定义函数指令
			resetFunc() {
				this.renderFunc = {
					name: null,
					params: null
				}
			},
			// 自定义函数 - 移除视频
			remove(params) {
				this.renderFunc = {
					name: 'removeHandler',
					params
				}
			},
			// 自定义函数 - 全屏播放
			fullScreen(params) {
				this.renderFunc = {
					name: 'fullScreenHandler',
					params
				}
			},
			// 自定义函数 - 跳转到指定时间点
			toSeek(sec, isDelay = false) {
				this.renderFunc = {
					name: 'toSeekHandler',
					params: {
						sec,
						isDelay
					}
				}
			},
			// 新增：检查视频方向
			async checkVideoOrientation() {

				if (!this.src || !this.showLandscapeFullscreenButton) return
				try {
					const posterUrl = `${this.src}${this.videoSnapshotParams}`
					const imageInfo = await uni.getImageInfo({
						src: posterUrl
					})
					// console.info("视频地址发生变化2",posterUrl);
					// 通常宽高比 > 0.9 为横屏
					const isLandscape = imageInfo.width / imageInfo.height > 0.9
					this.videoOrientation = {
						isLandscape,
						width: imageInfo.width,
						height: imageInfo.height
					}
					// console.info("视频地址发生变化3",this.videoOrientation);
					// 如果是横屏且需要显示按钮，则显示全屏按钮（不管是否播放中）
					this.showFullscreenButton = isLandscape && this.showLandscapeFullscreenButton

					// 发送方向检测完成事件
					this.$emit('orientation-detected', {
						isLandscape,
						width: imageInfo.width,
						height: imageInfo.height
					})

					return isLandscape
				} catch (error) {
					console.error('获取视频截图信息失败:', error)
					this.videoOrientation = {
						isLandscape: false
					}
					this.showFullscreenButton = false
					return false
				}
			},
			// 新增：手动触发全屏
			triggerFullscreen() {
				this.renderFunc = {
					name: 'triggerFullscreenWithButton',
					params: null
				}
			}
		}
	}
</script>

<!-- renderjs -->
<script module="domVideoPlayer" lang="renderjs">
	const PLAYER_ID = 'DOM_VIDEO_PLAYER'
	export default {
		data() {
			return {
				num: '',
				videoEl: null,
				loadingEl: null,
				// 延迟生效的函数
				delayFunc: null,
				renderProps: {},
				// 新增：全屏按钮元素
				fullscreenButtonEl: null,
				// 新增：是否禁用视频点击播放
				disableVideoClick: false,
				// 新增：横屏样式元素
				landscapeStyleEl: null,
				// 新增：是否已添加横屏样式
				landscapeStyleAdded: false
			}
		},
		computed: {
			playerId() {
				return `${PLAYER_ID}_${this.num}`
			},
			wrapperId() {
				return `video-wrapper-${this.num}`
			}
		},
		methods: {
			isApple() {
				const ua = navigator.userAgent.toLowerCase()
				return ua.indexOf('iphone') !== -1 || ua.indexOf('ipad') !== -1
			},
			async initVideoPlayer(src) {
				this.delayFunc = null
				await this.$nextTick()
				if (!src) return
				if (this.videoEl) {
					// 切换视频源
					if (!this.isApple() && this.loadingEl) {
						this.loadingEl.style.display = 'block'
					}
					this.videoEl.src = src
					// 更新全屏按钮状态
					this.updateFullscreenButton()
					return
				}

				const videoEl = document.createElement('video')
				this.videoEl = videoEl
				// 开始监听视频相关事件
				this.listenVideoEvent()

				const {
					autoplay,
					muted,
					controls,
					controlsList,
					loop,
					playbackRate,
					objectFit,
					poster,
					preload
				} = this.renderProps
				videoEl.src = src
				videoEl.autoplay = autoplay
				videoEl.controls = controls
				videoEl.loop = loop
				videoEl.muted = muted
				videoEl.playbackRate = playbackRate
				videoEl.id = this.playerId
				videoEl.setAttribute('preload', preload)
				videoEl.setAttribute('playsinline', true)
				videoEl.setAttribute('webkit-playsinline', true)
				let finalControlsList = 'nodownload nofullscreen noremoteplayback'
				if (controlsList) {
					finalControlsList += ' ' + controlsList
					console.info('finalControls', finalControlsList);
				}
				this.videoEl.setAttribute('controlslist', finalControlsList)
				// controlsList && videoEl.setAttribute('controlslist', controlsList)
				videoEl.setAttribute('disablePictureInPicture', true)
				videoEl.style.objectFit = objectFit
				poster && (videoEl.poster = poster)
				videoEl.style.width = '100%'
				videoEl.style.height = '100%'

				// 插入视频元素
				const playerWrapper = document.getElementById(this.wrapperId)
				playerWrapper.insertBefore(videoEl, playerWrapper.firstChild)

				// 创建字幕
				this.createTrack()

				// 插入loading 元素（遮挡安卓的默认加载过程中的黑色播放按钮）
				this.createLoading()

				// 创建全屏按钮（如果需要）
				this.createFullscreenButton()

				// 添加横屏全屏样式
				this.addLandscapeFullscreenStyle()
			},
			// 添加横屏全屏样式
			addLandscapeFullscreenStyle() {
				if (this.landscapeStyleAdded) return

				const style = document.createElement('style')
				this.landscapeStyleEl = style

				// 横屏全屏时的旋转和缩放样式
				const css = `
	  .video-landscape-fullscreen {
		transform: rotate(90deg) !important;
		transform-origin: center center !important;
		width: 100vh !important;
		height: 100vw !important;
		position: fixed !important;
		top: 50% !important;
		left: 50% !important;
		margin-top: -50vw !important;
		margin-left: -50vh !important;
		z-index: 999999 !important;
		background: #000 !important;
	  }
	  
	  .video-landscape-container {
		position: fixed !important;
		top: 0 !important;
		left: 0 !important;
		width: 100% !important;
		height: 100% !important;
		background: #000 !important;
		z-index: 999998 !important;
		overflow: hidden !important;
	  }
	  
	  .video-landscape-button {
		position: fixed !important;
		top: 35px !important;
		transform: rotate(90deg) !important;
		right: 20px !important;
		z-index: 1000000 !important;
		background: rgba(0, 0, 0, 0.7) !important;
		color: white !important;
		border: none !important;
		border-radius: 50% !important;
		padding: 0 !important;
		width: 48px !important;
		height: 48px !important;
		display: flex !important;
		align-items: center !important;
		justify-content: center !important;
		cursor: pointer !important;
		transition: background-color 0.3s ease !important;
	  }
	  
	  .video-landscape-button:hover {
		background: rgba(0, 0, 0, 0.9) !important;
	  }
	  
	  .video-landscape-button img {
		width: 24px !important;
		height: 24px !important;
		object-fit: contain !important;
		pointer-events: none !important;
	  }
	`

				style.textContent = css
				document.head.appendChild(style)
				this.landscapeStyleAdded = true
			},
			// 创建全屏按钮 - 修复事件冲突
			createFullscreenButton() {
				const {
					showLandscapeFullscreenButton,
					videoOrientation
				} = this.renderProps

				// 如果不需要显示按钮或已经存在按钮，则返回
				if (!showLandscapeFullscreenButton || this.fullscreenButtonEl) return

				// 检查视频方向
				const isLandscape = videoOrientation?.isLandscape || false
				if (!isLandscape) return

				// 移除已有的按钮
				if (this.fullscreenButtonEl) {
					this.fullscreenButtonEl.remove()
					this.fullscreenButtonEl = null
				}

				// 创建按钮 - 使用div而不是button，避免默认行为
				const button = document.createElement('div')
				this.fullscreenButtonEl = button
				button.innerHTML =
					'<img src="https://zeno-img.oss-cn-shenzhen.aliyuncs.com/image_17662122424244555d407154f528acd87281c33396fae.png" style="display:block;width:96px;height:30px;">'
				button.style.position = 'absolute'

				button.style.left = '50%' // 改为left: 50%
				button.style.transform = 'translateX(-50%)' // 使用transform向左移动自身宽度的一半
				button.style.zIndex = '20' // 提高z-index确保在视频上方
				button.style.backgroundColor = 'rgba(0, 0, 0, 0.7)'
				button.style.color = 'white'
				button.style.border = 'none'
				button.style.borderRadius = '20px'
				button.style.cursor = 'pointer'
				button.style.display = 'block' // 横屏模式下一直显示
				button.style.userSelect = 'none' // 防止文字被选中
				button.style.pointerEvents = 'auto' // 确保按钮可以点击

				// 确保图片容器正确显示
				button.style.display = 'flex'
				button.style.alignItems = 'center'
				button.style.justifyContent = 'center'
				button.style.padding = '5px' // 添加内边距让图片与背景有间距

				// 添加点击事件 - 完全阻止事件冒泡
				const handleButtonClick = (e) => {
					// 完全阻止事件传播
					e.stopPropagation()
					e.stopImmediatePropagation()
					e.preventDefault()
					console.info("testaaaaa")

					// 临时禁用视频点击，防止冲突
					this.disableVideoClick = true
					setTimeout(() => {
						this.disableVideoClick = false
					}, 100)

					// 触发横屏全屏
					this.triggerLandscapeFullscreen()
					return false
				}

				// 使用捕获阶段处理点击
				button.addEventListener('click', handleButtonClick, true)

				// 阻止touch事件
				button.addEventListener('touchstart', (e) => {
					e.stopPropagation()
					e.preventDefault()
				}, {
					passive: false
				})

				button.addEventListener('touchend', (e) => {
					e.stopPropagation()
					e.preventDefault()
					this.triggerLandscapeFullscreen()
				}, {
					passive: false
				})

				// 插入按钮
				const wrapper = document.getElementById(this.wrapperId)
				if (wrapper) {
					wrapper.style.position = 'relative'
					// console.info("this.videoEl.src",this.videoEl.src)
					if (!this.videoEl || !button) return
					const {
						videoOrientation
					} = this.renderProps
					// console.info("videoOrientation.height=",videoOrientation.height);
					// // 获取video元素的位置和尺寸
					// const finalHeight = window.innerHeight / 2  +  videoOrientation.height /2
					// console.info("finalHeight=",finalHeight);
					// const fromBottomDistance = window.innerHeight - finalHeight + 15;
					// console.info("fromBottomDistance=",fromBottomDistance);
					button.style.bottom = `calc(49px + env(safe-area-inset-bottom) + 128px)`
					button.style.transform = 'translateX(-50%)' // 向左移动自身宽度的一半实现居中		
					wrapper.append(button)
					// 确保视频元素有正确的z-index
					if (this.videoEl) {
						this.videoEl.style.zIndex = '1'
					}
				}
			},
			// 更新全屏按钮显示状态 - 修改：横屏模式下一直显示
			updateFullscreenButton() {
				if (!this.fullscreenButtonEl) return

				const {
					showLandscapeFullscreenButton,
					videoOrientation,
					isFullscreen
				} = this.renderProps
				const isLandscape = videoOrientation?.isLandscape || false

				// 显示条件：需要显示按钮、视频是横屏、且不在全屏状态
				if (showLandscapeFullscreenButton && isLandscape && !isFullscreen) {
					this.fullscreenButtonEl.style.display = 'block'
				} else {
					this.fullscreenButtonEl.style.display = 'none'
				}
			},
			// 触发横屏全屏
			triggerLandscapeFullscreen() {
				if (!this.videoEl) return

				const {
					forceLandscapeFullscreen
				} = this.renderProps
				console.info("testaaaaa2")
				if (forceLandscapeFullscreen) {
					// 使用自定义横屏全屏
					this.enterLandscapeFullscreen()
				} else {
					// 使用原生全屏
					this.triggerFullscreenWithButton()
				}
			},
			// 进入横屏全屏模式
			enterLandscapeFullscreen() {
				if (!this.videoEl) return

				// 创建全屏容器
				const container = document.createElement('div')
				container.className = 'video-landscape-container'
				container.id = `landscape-container-${this.num}`

				// 克隆视频元素
				const videoClone = this.videoEl.cloneNode(true)
				videoClone.className = 'video-landscape-fullscreen'
				videoClone.controls = true // 横屏全屏时显示控件

				// 添加退出按钮
				const exitButton = document.createElement('button')
				exitButton.className = 'video-landscape-button'
				exitButton.innerHTML =
					'<img src="https://zeno-img.oss-cn-shenzhen.aliyuncs.com/image_17662199265742063c6b1214e62c83fef7db2832faa03.png" alt="退出全屏" />'
				exitButton.onclick = () => this.exitLandscapeFullscreen()

				// 添加到容器
				container.appendChild(videoClone)
				container.appendChild(exitButton)

				// 添加到body
				document.body.appendChild(container)

				// 播放视频
				setTimeout(() => {
					videoClone.play().catch(e => console.log('Auto-play prevented:', e))
				}, 100)

				// 发送全屏事件
				this.$ownerInstance.callMethod('eventEmit', {
					event: 'fullscreenchange',
					data: true
				})

				// 隐藏原按钮
				if (this.fullscreenButtonEl) {
					this.fullscreenButtonEl.style.display = 'none'
				}
			},
			// 退出横屏全屏模式
			exitLandscapeFullscreen() {
				const container = document.getElementById(`landscape-container-${this.num}`)
				if (container) {
					// 暂停视频
					const video = container.querySelector('.video-landscape-fullscreen')
					if (video) {
						const currentTime = video.currentTime
						// 同步时间到原视频
						if (this.videoEl) {
							this.videoEl.currentTime = currentTime
						}
					}

					// 移除容器
					document.body.removeChild(container)

					// 发送退出全屏事件
					this.$ownerInstance.callMethod('eventEmit', {
						event: 'fullscreenchange',
						data: false
					})

					// 重新显示按钮
					this.updateFullscreenButton()
				}
			},
			// 通过按钮触发全屏 - 修复版
			triggerFullscreenWithButton() {
				if (!this.videoEl) return

				const handleError = (error) => {
					console.warn('Fullscreen error (handled):', error.message || error)
					this.$ownerInstance.callMethod('eventEmit', {
						event: 'fullscreenerror',
						data: error.message || 'Fullscreen failed'
					})
				}

				const attemptFullscreen = () => {
					try {
						// 检查浏览器兼容性
						if (this.isApple()) {
							if (this.videoEl.webkitEnterFullscreen) {
								this.videoEl.webkitEnterFullscreen()
							} else {
								handleError(new Error('iOS fullscreen not supported'))
							}
						} else {
							// 检查所有可能的全屏API
							if (this.videoEl.requestFullscreen) {
								const promise = this.videoEl.requestFullscreen()
								if (promise && typeof promise.catch === 'function') {
									promise.catch(handleError)
								}
							} else if (this.videoEl.mozRequestFullScreen) {
								this.videoEl.mozRequestFullScreen()
							} else if (this.videoEl.webkitRequestFullscreen) {
								this.videoEl.webkitRequestFullscreen()
							} else if (this.videoEl.msRequestFullscreen) {
								this.videoEl.msRequestFullscreen()
							} else {
								handleError(new Error('Fullscreen API not supported'))
							}
						}
					} catch (error) {
						handleError(error)
					}
				}

				// 立即执行
				attemptFullscreen()
			},
			// 创建视频字幕
			createTrack() {
				// 移除掉之前的字幕
				const tracks = this.videoEl.querySelectorAll('track')
				tracks.forEach((track) => {
					this.videoEl.removeChild(track)
				})

				// 创建新的字幕
				const {
					trackList
				} = this.renderProps
				if (!trackList.length) return
				trackList.forEach((item) => {
					const {
						src,
						kind,
						label,
						srclang,
						default: isDefault
					} = item
					const trackEl = document.createElement('track')
					trackEl.kind = kind
					trackEl.label = label
					trackEl.srclang = srclang
					trackEl.default = isDefault
					trackEl.src = src
					this.videoEl.appendChild(trackEl)
				})
			},
			// 创建 loading
			createLoading() {
				const {
					isLoading
				} = this.renderProps
				if (!this.isApple() && isLoading) {
					const loadingEl = document.createElement('div')
					this.loadingEl = loadingEl
					loadingEl.className = 'loading-wrapper'
					loadingEl.style.position = 'absolute'
					loadingEl.style.top = '0'
					loadingEl.style.left = '0'
					loadingEl.style.zIndex = '1'
					loadingEl.style.width = '100%'
					loadingEl.style.height = '100%'
					loadingEl.style.backgroundColor = 'black'
					document.getElementById(this.wrapperId).appendChild(loadingEl)

					// 创建 loading 动画
					const animationEl = document.createElement('div')
					animationEl.className = 'loading'
					animationEl.style.zIndex = '2'
					animationEl.style.position = 'absolute'
					animationEl.style.top = '50%'
					animationEl.style.left = '50%'
					animationEl.style.marginTop = '-15px'
					animationEl.style.marginLeft = '-15px'
					animationEl.style.width = '30px'
					animationEl.style.height = '30px'
					animationEl.style.border = '2px solid #FFF'
					animationEl.style.borderTopColor = 'rgba(255, 255, 255, 0.2)'
					animationEl.style.borderRightColor = 'rgba(255, 255, 255, 0.2)'
					animationEl.style.borderBottomColor = 'rgba(255, 255, 255, 0.2)'
					animationEl.style.borderRadius = '100%'
					animationEl.style.animation = 'circle infinite 0.75s linear'
					loadingEl.appendChild(animationEl)

					// 创建 loading 动画所需的 keyframes
					const style = document.createElement('style')
					const keyframes = `
          @keyframes circle {
            0% {
              transform: rotate(0);
            }
            100% {
              transform: rotate(360deg);
            }
          }
        `
					style.type = 'text/css'
					if (style.styleSheet) {
						style.styleSheet.cssText = keyframes
					} else {
						style.appendChild(document.createTextNode(keyframes))
					}
					document.head.appendChild(style)
				}
			},
			// 监听视频相关事件 - 修复点击冲突
			listenVideoEvent() {
				// 播放事件监听 - 修改：播放时不隐藏按钮
				const playHandler = () => {
					this.$ownerInstance.callMethod('eventEmit', {
						event: 'play'
					})
					this.$ownerInstance.callMethod('setViewData', {
						key: 'playing',
						value: true
					})

					if (this.loadingEl) {
						this.loadingEl.style.display = 'none'
					}

					// 播放时不隐藏全屏按钮（横屏模式下一直显示）
					// 只在updateFullscreenButton中根据全屏状态控制显示
				}
				this.videoEl.removeEventListener('play', playHandler)
				this.videoEl.addEventListener('play', playHandler)

				// 暂停事件监听
				const pauseHandler = () => {
					this.$ownerInstance.callMethod('eventEmit', {
						event: 'pause'
					})
					this.$ownerInstance.callMethod('setViewData', {
						key: 'playing',
						value: false
					})

					// 暂停时更新按钮显示状态
					this.updateFullscreenButton()
				}
				this.videoEl.removeEventListener('pause', pauseHandler)
				this.videoEl.addEventListener('pause', pauseHandler)

				// 结束事件监听
				const endedHandler = () => {
					this.$ownerInstance.callMethod('eventEmit', {
						event: 'ended'
					})
					this.$ownerInstance.callMethod('resetEventCommand')

					// 结束时更新按钮显示状态
					this.updateFullscreenButton()
				}
				this.videoEl.removeEventListener('ended', endedHandler)
				this.videoEl.addEventListener('ended', endedHandler)

				// 加载完成事件监听
				const canPlayHandler = () => {
					this.$ownerInstance.callMethod('eventEmit', {
						event: 'canplay'
					})
					this.execDelayFunc()
				}
				this.videoEl.removeEventListener('canplay', canPlayHandler)
				this.videoEl.addEventListener('canplay', canPlayHandler)

				// 加载失败事件监听
				const errorHandler = (e) => {
					if (this.loadingEl) {
						this.loadingEl.style.display = 'block'
					}
					this.$ownerInstance.callMethod('eventEmit', {
						event: 'error',
						data: e
					})
				}
				this.videoEl.removeEventListener('error', errorHandler)
				this.videoEl.addEventListener('error', errorHandler)

				// loadedmetadata 事件监听
				const loadedMetadataHandler = () => {
					this.$ownerInstance.callMethod('eventEmit', {
						event: 'loadedmetadata'
					})
					// 获取视频的长度
					const duration = this.videoEl.duration
					this.$ownerInstance.callMethod('eventEmit', {
						event: 'durationchange',
						data: duration
					})

					this.$ownerInstance.callMethod('setViewData', {
						key: 'duration',
						value: duration
					})

					// 加载首帧视频 模拟出封面图
					this.loadFirstFrame()
				}
				this.videoEl.removeEventListener('loadedmetadata', loadedMetadataHandler)
				this.videoEl.addEventListener('loadedmetadata', loadedMetadataHandler)

				// 播放进度监听
				const timeupdateHandler = (e) => {
					const currentTime = e.target.currentTime
					this.$ownerInstance.callMethod('eventEmit', {
						event: 'timeupdate',
						data: currentTime
					})

					this.$ownerInstance.callMethod('setViewData', {
						key: 'currentTime',
						value: currentTime
					})

				}
				this.videoEl.removeEventListener('timeupdate', timeupdateHandler)
				this.videoEl.addEventListener('timeupdate', timeupdateHandler)

				// 倍速播放监听
				const ratechangeHandler = (e) => {
					const playbackRate = e.target.playbackRate
					this.$ownerInstance.callMethod('eventEmit', {
						event: 'ratechange',
						data: playbackRate
					})
				}
				this.videoEl.removeEventListener('ratechange', ratechangeHandler)
				this.videoEl.addEventListener('ratechange', ratechangeHandler)

				// 添加视频点击事件监听 - 避免与按钮冲突
				const videoClickHandler = (e) => {
					// 如果点击的是全屏按钮区域或临时禁用了点击，则不触发播放/暂停
					if (this.disableVideoClick) {
						return
					}

					// 检查点击目标是否是按钮或按钮的子元素
					if (this.fullscreenButtonEl &&
						(this.fullscreenButtonEl === e.target ||
							this.fullscreenButtonEl.contains(e.target))) {
						return
					}

					// 正常处理视频点击（播放/暂停切换）
					if (this.videoEl.paused) {
						this.videoEl.play()
					} else {
						this.videoEl.pause()
					}
				}

				this.videoEl.removeEventListener('click', videoClickHandler)
				this.videoEl.addEventListener('click', videoClickHandler)

				// 全屏事件监听
				if (this.isApple()) {
					const webkitbeginfullscreenHandler = () => {
						const presentationMode = this.videoEl.webkitPresentationMode
						let isFullScreen = null
						if (presentationMode === 'fullscreen') {
							isFullScreen = true
						} else {
							isFullScreen = false
						}
						this.$ownerInstance.callMethod('eventEmit', {
							event: 'fullscreenchange',
							data: isFullScreen
						})

						// 更新按钮显示状态
						this.updateFullscreenButton()
					}
					this.videoEl.removeEventListener('webkitpresentationmodechanged', webkitbeginfullscreenHandler)
					this.videoEl.addEventListener('webkitpresentationmodechanged', webkitbeginfullscreenHandler)
				} else {
					const fullscreenchangeHandler = () => {
						let isFullScreen = null
						if (document.fullscreenElement) {
							isFullScreen = true
						} else {
							isFullScreen = false
						}
						this.$ownerInstance.callMethod('eventEmit', {
							event: 'fullscreenchange',
							data: isFullScreen
						})

						// 更新按钮显示状态
						this.updateFullscreenButton()
					}
					document.removeEventListener('fullscreenchange', fullscreenchangeHandler)
					document.addEventListener('fullscreenchange', fullscreenchangeHandler)
				}
			},
			// 加载首帧视频，模拟出封面图
			loadFirstFrame() {
				let {
					autoplay,
					poster,
					muted
				} = this.renderProps
				if (this.isApple()) {
					if (poster) return
					this.videoEl.play()
					if (!autoplay) {
						this.videoEl.pause()
					}
				} else {
					if (poster) return
					this.videoEl.muted = true
					setTimeout(() => {
						this.videoEl.play()
						this.videoEl.muted = muted
						if (!autoplay) {
							setTimeout(() => {
								this.videoEl.pause()
							}, 100)
						}
					}, 10)
				}
			},
			triggerCommand(eventType) {
				if (eventType) {
					this.$ownerInstance.callMethod('resetEventCommand')
					this.videoEl && this.videoEl[eventType]()
				}
			},
			triggerFunc(func) {
				const {
					name,
					params
				} = func || {}
				if (name) {
					this[name](params)
					this.$ownerInstance.callMethod('resetFunc')
				}
			},
			removeHandler() {
				if (this.videoEl) {
					this.videoEl.pause()
					this.videoEl.src = ''
					this.$ownerInstance.callMethod('setViewData', {
						key: 'videoSrc',
						value: ''
					})
					this.videoEl.load()
				}
			},
			fullScreenHandler() {
				const handleError = (error) => {
					console.warn('Fullscreen error (handled):', error.message || error)
					this.$ownerInstance.callMethod('eventEmit', {
						event: 'fullscreenerror',
						data: error.message || 'Fullscreen failed'
					})
				}

				try {
					if (this.isApple()) {
						this.videoEl.webkitEnterFullscreen()
					} else {
						const promise = this.videoEl.requestFullscreen()
						if (promise && typeof promise.catch === 'function') {
							promise.catch(handleError)
						}
					}
				} catch (error) {
					handleError(error)
				}
			},
			toSeekHandler({
				sec,
				isDelay
			}) {
				const func = () => {
					if (this.videoEl) {
						this.videoEl.currentTime = sec
					}
				}

				if (isDelay) {
					this.delayFunc = func
				} else {
					func()
				}
			},
			execDelayFunc() {
				this.delayFunc && this.delayFunc()
				this.delayFunc = null
			},
			viewportChange(props) {
				this.renderProps = props
				const {
					autoplay,
					muted,
					controls,
					loop,
					playbackRate
				} = props
				if (this.videoEl) {
					this.videoEl.autoplay = autoplay
					this.videoEl.controls = controls
					this.videoEl.loop = loop
					this.videoEl.muted = muted
					this.videoEl.playbackRate = playbackRate
					this.createTrack()
				}

				// 更新全屏按钮状态
				this.updateFullscreenButton()

				// 如果按钮不存在但需要显示，则创建按钮
				if (!this.fullscreenButtonEl && props.showLandscapeFullscreenButton) {
					this.createFullscreenButton()
				}
			},
			randomNumChange(val) {
				this.num = val
			}
		}
	}
</script>

<style scoped>
	.player-wrapper {
		overflow: hidden;
		height: 100%;
		padding: 0;
		position: relative;
	}
</style>