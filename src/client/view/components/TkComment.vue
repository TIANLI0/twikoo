<template>
  <div class="tk-comment" :id="comment.id" :class="{ 'tk-master': comment.master }" ref="tk-comment">
    <tk-avatar :config="config"
        :nick="comment.nick"
        :avatar="comment.avatar"
        :mail-md5="comment.mailMd5"
        :link="convertedLink" />
    <div class="tk-main">
      <div class="tk-row">
        <div class="tk-meta">
          <strong class="tk-nick" v-if="!convertedLink">{{ comment.nick }}</strong>
          <a class="tk-nick tk-nick-link" v-if="convertedLink" :href="convertedLink" target="_blank" rel="noopener noreferrer">
            <strong>{{ comment.nick }}</strong>
          </a>
          <span class="tk-tag tk-tag-green" v-if="comment.master">{{ config.MASTER_TAG || t('COMMENT_MASTER_TAG') }}</span>
          <span class="tk-tag tk-tag-red" v-if="comment.top">{{ t('COMMENT_TOP_TAG') }}</span>
          <span class="tk-tag tk-tag-yellow" v-if="comment.isSpam">{{ t('COMMENT_REVIEWING_TAG') }}</span>
          <small class="tk-time">
            <time :datetime="jsonTimestamp" :title="localeTime">{{ displayCreated }}</time>
          </small>
          <small class="tk-actions" v-if="isLogin">
            <a href="#" v-if="comment.isSpam" @click="handleSpam(false, $event)">{{ t('ADMIN_COMMENT_SHOW') }}</a>
            <a href="#" v-if="!comment.isSpam" @click="handleSpam(true, $event)">{{ t('ADMIN_COMMENT_HIDE') }}</a>
            <a href="#" v-if="!comment.rid && comment.top" @click="handleTop(false, $event)">{{ t('ADMIN_COMMENT_UNTOP') }}</a>
            <a href="#" v-if="!comment.rid && !comment.top" @click="handleTop(true, $event)">{{ t('ADMIN_COMMENT_TOP') }}</a>
          </small>
        </div>
        <tk-action :liked="liked"
            :like-count="like"
            :replies-count="comment.replies.length"
            @like="onLike"
            @reply="onReply" />
      </div>
      <div class="tk-content" :class="{ 'tk-content-expand': isContentExpanded || !showContentExpand }" ref="tk-content">
        <span v-if="comment.pid">{{ t('COMMENT_REPLIED') }} <a class="tk-ruser" :href="`#${comment.pid}`">@{{ comment.ruser }}</a> :</span>
        <span v-html="comment.comment" ref="comment" @click="popupLightbox"></span>
      </div>
      <div class="tk-expand-wrap" v-if="showContentExpand">
        <div class="tk-expand" @click="onContentExpand">{{ t('COMMENT_EXPAND') }}</div>
      </div>
      <div class="tk-collapse-wrap" v-if="showContentCollapse">
        <div class="tk-expand _collapse" @click="onContentCollapse">{{ t('COMMENT_COLLAPSE') }}</div>
      </div>
      <div class="tk-extras" v-if="comment.ipRegion || comment.os || comment.browser">
        <div class="tk-extra" v-if="comment.ipRegion">
          <span class="tk-icon __comment" v-html="iconLocation"></span>
          <span class="tk-extra-text">&nbsp;{{ comment.ipRegion }}</span>
        </div>
        <div class="tk-extra" v-if="comment.os">
          <span class="tk-icon __comment" v-html="iconOs"></span>
          <span class="tk-extra-text">&nbsp;{{ comment.os }}</span>
        </div>
        <div class="tk-extra" v-if="comment.browser">
          <span class="tk-icon __comment" v-html="iconBrowser"></span>
          <span class="tk-extra-text">&nbsp;{{ comment.browser }}</span>
        </div>
      </div>
      <!-- 回复框 -->
      <tk-submit v-if="replying && !pid"
          :reply-id="replyId ? replyId : comment.id"
          :pid="comment.id"
          :config="config"
          @load="onLoad"
          @cancel="onCancel" />
      <!-- 回复列表 -->
      <div class="tk-replies" :class="{ 'tk-replies-expand': isExpanded || !showExpand || replying }" ref="tk-replies">
        <tk-comment v-for="reply in comment.replies"
            :key="reply.id"
            :comment="reply"
            :replyId="comment.id"
            :replying="replying && pid === reply.id"
            :config="config"
            @expand="onExpand"
            @load="onLoad"
            @reply="onReplyReply" />
      </div>
      <div class="tk-expand-wrap" v-if="showExpand && !replying">
        <div class="tk-expand" @click="onExpand">{{ t('COMMENT_EXPAND') }}</div>
      </div>
      <div class="tk-collapse-wrap" v-if="showCollapse && !replying">
        <div class="tk-expand _collapse" @click="onCollapse">{{ t('COMMENT_COLLAPSE') }}</div>
      </div>
    </div>
  </div>
</template>

<script>
import { timeago, convertLink, call, renderLinks, renderMath, renderCode, t } from '../../utils'
import TkAction from './TkAction.vue'
import TkAvatar from './TkAvatar.vue'
import TkSubmit from './TkSubmit.vue'
import iconWindows from '@fortawesome/fontawesome-free/svgs/brands/windows.svg'
import iconApple from '@fortawesome/fontawesome-free/svgs/brands/apple.svg'
import iconAndroid from '@fortawesome/fontawesome-free/svgs/brands/android.svg'
import iconLinux from '@fortawesome/fontawesome-free/svgs/brands/linux.svg'
import iconUbuntu from '@fortawesome/fontawesome-free/svgs/brands/ubuntu.svg'
import iconChrome from '@fortawesome/fontawesome-free/svgs/brands/chrome.svg'
import iconFirefox from '@fortawesome/fontawesome-free/svgs/brands/firefox-browser.svg'
import iconSafari from '@fortawesome/fontawesome-free/svgs/brands/safari.svg'
import iconIe from '@fortawesome/fontawesome-free/svgs/brands/internet-explorer.svg'
import iconEdge from '@fortawesome/fontawesome-free/svgs/brands/edge.svg'
import iconOther from '@fortawesome/fontawesome-free/svgs/regular/window-maximize.svg'
import iconLocation from '@fortawesome/fontawesome-free/svgs/solid/location-arrow.svg'

const iconHarmony = {
  template: '<svg t="1734782334717" class="icon" viewBox="0 0 1024 1024" version="1.1" xmlns="http://www.w3.org/2000/svg" p-id="7529" width="200" height="200"><path d="M156.586667 261.973333S77.653333 337.493333 73.386667 417.28v14.933333c3.413333 64.426667 52.053333 102.4 52.053333 102.4 78.08 76.373333 267.093333 172.373333 311.466667 194.133334 0 0 2.56 1.28 4.266666-0.426667l0.853334-1.706667v-1.706666C320.853333 460.8 156.586667 261.973333 156.586667 261.973333zM411.733333 793.6c-0.853333-3.413333-4.266667-3.413333-4.266666-3.413333l-314.88 11.093333c34.133333 61.013333 91.733333 107.946667 151.893333 93.866667 40.96-10.666667 134.826667-75.946667 165.546667-98.133334 2.56-2.133333 1.706667-3.84 1.706666-3.84z m3.413334-33.28C276.906667 666.88 8.96 523.946667 8.96 523.946667c-6.4 19.626667-8.533333 38.4-8.96 55.466666v2.986667c0 45.653333 17.066667 77.653333 17.066667 77.653333 34.133333 72.106667 99.84 93.866667 99.84 93.866667 29.866667 12.8 59.733333 13.226667 59.733333 13.226667 5.12 0.853333 187.733333 0 236.373333 0 2.133333 0 3.413333-2.133333 3.413334-2.133334v-2.56c0-1.28-1.28-2.133333-1.28-2.133333zM386.56 136.106667a145.92 145.92 0 0 0-109.653333 134.4v17.493333c1.28 25.6 6.826667 44.8 6.826666 44.8 28.16 123.733333 164.693333 326.4 194.133334 369.066667 2.133333 2.133333 4.266667 1.28 4.266666 1.28a4.266667 4.266667 0 0 0 2.56-4.266667c45.226667-452.266667-47.36-572.586667-47.36-572.586667-13.653333 0.853333-50.773333 9.813333-50.773333 9.813334z m354.133333 96.853333s-20.906667-76.8-104.106666-97.28c0 0-24.32-5.973333-49.92-9.386667 0 0-93.013333 119.893333-47.786667 573.013334 0.426667 2.986667 2.56 3.413333 2.56 3.413333 2.986667 1.28 4.266667-1.28 4.266667-1.28 30.72-43.946667 166.4-245.76 194.133333-368.64 0 0 15.36-59.733333 0.853333-99.84z m-124.586666 557.653333s-2.986667 0-3.84 2.133334c0 0-0.426667 2.986667 1.28 4.266666 29.866667 21.76 121.6 85.333333 165.546666 98.133334 0 0 6.826667 2.133333 18.346667 2.56h5.973333c29.44-0.853333 81.066667-15.786667 128-96.426667l-315.733333-10.666667z m334.08-358.826666c5.973333-87.893333-82.773333-169.386667-82.773334-169.813334 0 0-164.266667 198.826667-284.586666 460.8 0 0-1.28 3.413333 0.853333 5.546667l1.706667 0.426667h2.56c45.226667-22.613333 232.96-118.186667 310.613333-193.706667 0 0 49.066667-39.68 51.626667-103.253333z m64.853333 91.306666s-267.946667 143.786667-406.186667 236.8c0 0-2.133333 1.706667-1.28 4.693334 0 0 1.28 2.56 2.986667 2.56 49.493333 0 237.226667 0 241.92-0.853334 0 0 24.32-0.853333 54.186667-12.373333 0 0 66.56-21.333333 101.12-96.853333 0 0 31.146667-61.866667 7.253333-133.973334z" p-id="7530"></path></svg>'
}

const osList = {
  win: iconWindows,
  mac: iconApple,
  ipad: iconApple,
  iphone: iconApple,
  ios: iconApple,
  android: iconAndroid,
  ubuntu: iconUbuntu,
  linux: iconLinux,
  harmony: iconHarmony
}

const browserList = {
  edge: iconEdge,
  chrome: iconChrome,
  firefox: iconFirefox,
  safari: iconSafari,
  explorer: iconIe,
  ie: iconIe
}

export default {
  name: 'tk-comment', // 允许组件模板递归地调用自身
  components: {
    TkAction,
    TkAvatar,
    TkSubmit
  },
  data () {
    return {
      pid: '',
      like: 0,
      liked: false,
      likeLoading: false,
      isExpanded: false,
      hasExpand: false,
      isContentExpanded: false,
      hasContentExpand: false,
      isLogin: false
    }
  },
  props: {
    comment: Object,
    replyId: String,
    replying: Boolean,
    config: Object
  },
  computed: {
    displayCreated () {
      return timeago(this.comment.created)
    },
    jsonTimestamp () {
      return new Date(this.comment.created).toJSON()
    },
    localeTime () {
      return new Date(this.comment.created).toLocaleString()
    },
    iconOs () {
      return this.getIconBy(this.comment.os, osList)
    },
    iconBrowser () {
      return this.getIconBy(this.comment.browser, browserList)
    },
    iconLocation: () => iconLocation,
    showExpand () {
      return this.hasExpand && !this.isExpanded
    },
    showCollapse () {
      return this.hasExpand && this.isExpanded
    },
    showContentExpand () {
      return this.hasContentExpand && !this.isContentExpanded
    },
    showContentCollapse () {
      return this.hasContentExpand && this.isContentExpanded
    },
    convertedLink () {
      return convertLink(this.comment.link)
    }
  },
  methods: {
    t,
    getIconBy (name, list) {
      const lowerCaseName = name.toLowerCase()
      for (const key in list) {
        if (lowerCaseName.indexOf(key) !== -1) return list[key]
      }
      return iconOther
    },
    showExpandIfNeed () {
      if (this.comment.replies && this.comment.replies.length > 0 && this.$refs['tk-replies']) {
        // 200 是回复区域最大高度
        // 36 是展开按钮高度
        this.hasExpand = this.$refs['tk-replies'].scrollHeight > 200 + 36
      }
    },
    showContentExpandIfNeed () {
      // 如果已经折叠就不再判断 主要是为了防止图片在onload之前就已经折叠而导致图片在onload之后取消折叠
      this.hasContentExpand = this.hasContentExpand || this.$refs['tk-content'].scrollHeight > 500
    },
    showContentExpandIfNeedAfterImagesLoaded () {
      this.$refs['tk-content'].querySelectorAll('img').forEach((imgEl) => {
        imgEl.onload = this.showContentExpandIfNeed
      })
    },
    scrollToComment () {
      if (window.location.hash.indexOf(this.comment.id) !== -1) {
        this.$refs['tk-comment'].scrollIntoView({
          behavior: 'smooth'
        })
        this.$emit('expand')
      }
    },
    async onLike () {
      if (this.likeLoading) return // 防止连续点击
      this.likeLoading = true
      await call(this.$tcb, 'COMMENT_LIKE', { id: this.comment.id })
      if (this.liked) {
        this.like--
      } else {
        this.like++
      }
      this.liked = !this.liked
      this.likeLoading = false
    },
    onReply (id) {
      this.pid = id
      this.$emit('reply', this.comment.id)
    },
    onReplyReply (id) {
      // 楼中楼回复
      this.pid = id
      if (id) {
        // action 回复按钮 触发
        this.$emit('reply', this.comment.id)
      } else {
        // submit 取消按钮 触发
        this.$emit('reply', '')
      }
    },
    onCancel () {
      this.pid = ''
      this.$emit('reply', '')
    },
    onLoad () {
      if (this.comment.replies.length > 0) {
        this.$refs['tk-replies'].lastElementChild.scrollIntoView({
          behavior: 'smooth',
          block: 'center'
        })
      }
      this.pid = ''
      this.$emit('reply', '')
      this.$emit('load')
      this.onExpand()
    },
    onExpand () {
      this.isExpanded = true
    },
    onCollapse () {
      this.isExpanded = false
    },
    onContentExpand () {
      this.isContentExpanded = true
    },
    onContentCollapse () {
      this.isContentExpanded = false
    },
    async checkAuth () {
      // 检查用户身份
      if (this.$tcb) {
        const currentUser = await this.$tcb.auth.getCurrenUser()
        this.isLogin = currentUser.loginType === 'CUSTOM'
      } else {
        this.isLogin = this.$twikoo.serverConfig && this.$twikoo.serverConfig.IS_ADMIN
      }
    },
    handleSpam (isSpam, $event) {
      $event.preventDefault()
      this.setComment({ isSpam })
    },
    handleTop (top, $event) {
      $event.preventDefault()
      this.setComment({ top })
    },
    popupLightbox (event) {
      if (this.$twikoo.serverConfig.LIGHTBOX !== 'true') return
      const { target } = event
      if (target.tagName === 'IMG' && !target.classList.contains('tk-owo-emotion')) {
        const lightbox = document.createElement('div')
        lightbox.className = 'tk-lightbox'
        const lightboxImg = document.createElement('img')
        lightboxImg.className = 'tk-lightbox-image'
        lightboxImg.src = target.src
        lightbox.appendChild(lightboxImg)
        lightbox.addEventListener('click', () => {
          document.body.removeChild(lightbox)
        })
        document.body.appendChild(lightbox)
      }
    },
    async setComment (set) {
      this.loading = true
      await call(this.$tcb, 'COMMENT_SET_FOR_ADMIN', {
        id: this.comment.id,
        set
      })
      this.loading = false
      this.$emit('load')
    }
  },
  mounted () {
    this.$nextTick(this.showContentExpandIfNeed)
    this.$nextTick(this.showContentExpandIfNeedAfterImagesLoaded)
    this.$nextTick(this.showExpandIfNeed)
    this.$nextTick(this.scrollToComment)
    this.$nextTick(() => {
      renderLinks(this.$refs.comment)
      renderMath(this.$refs.comment, this.$twikoo.katex)
    })
    this.checkAuth()
  },
  watch: {
    'comment.like': {
      handler: function (like) {
        this.like = this.comment.like
        this.liked = this.comment.liked
      },
      immediate: true
    },
    'config.HIGHLIGHT': {
      handler: function (highlight) {
        if (highlight === 'true') {
          this.$nextTick(() => {
            renderCode(this.$refs.comment, this.config.HIGHLIGHT_THEME, this.config.HIGHLIGHT_PLUGIN)
          })
        }
      },
      immediate: true
    }
  }
}
</script>

<style>
.tk-main {
  flex: 1;
  width: 0;
}
.tk-row {
  flex: 1;
  display: flex;
  flex-direction: row;
  justify-content: space-between;
}
.tk-nick-link {
  color: inherit;
  text-decoration: none;
}
.tk-replies .tk-nick-link {
  font-size: .9em;
}
.tk-nick-link:hover {
  color: #409eff;
}
.tk-actions {
  display: none;
  margin-left: 1em;
}
.tk-comment:hover .tk-actions {
  display: inline;
}
.tk-extras {
  color: #999999;
  font-size: 0.875em;
  display: flex;
  flex-wrap: wrap;
}
.tk-extra {
  margin-top: 0.5rem;
  margin-right: 0.75rem;
  display: flex;
  align-items: center;
}
.tk-icon.__comment {
  height: 1em;
  width: 1em;
  line-height: 1;
}
.tk-extra-text {
  line-height: 1;
}
.tk-tag {
  display: inline-block;
  padding: 0 0.5em;
  font-size: 0.75em;
  background-color: #f2f6fc;
}
.tk-tag-green {
  background-color: rgba(103,194,58,0.13);
  border: 1px solid rgba(103,194,58,0.50);
  border-radius: 2px;
  color: #67c23a;
}
.tk-tag-yellow {
  background-color: rgba(230,162,60,0.13);
  border: 1px solid rgba(230,162,60,0.50);
  border-radius: 2px;
  color: #e6a23c;
}
.tk-tag-blue {
  background-color: rgba(64,158,255,0.13);
  border: 1px solid rgba(64,158,255,0.50);
  border-radius: 2px;
  color: #409eff;
}
.tk-tag-red {
  background-color: rgba(245,108,108,0.13);
  border: 1px solid rgba(245,108,108,0.50);
  border-radius: 2px;
  color: #f56c6c;
}
.tk-comment {
  margin-top: 1rem;
  display: flex;
  flex-direction: row;
  word-break: break-all;
}
.tk-content {
  margin-top: 0.5rem;
  overflow: hidden;
  max-height: 500px;
  position: relative;
}
.tk-content-expand {
  max-height: none;
}
.tk-replies .tk-content {
  font-size: .9em;
}
.tk-comment .vemoji {
  max-height: 2em;
  vertical-align: middle;
}
.tk-replies {
  max-height: 200px;
  overflow: hidden;
  position: relative;
}
.tk-replies-expand {
  max-height: none;
  overflow: unset;
}
.tk-submit {
  margin-top: 1rem;
}
.tk-expand {
  font-size: 0.75em;
}
.tk-lightbox {
  display: block;
  position: fixed;
  background-color: rgba(0, 0, 0, 0.3);
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  z-index: 999;
}
.tk-lightbox-image {
  min-width: 100px;
  min-height: 30px;
  width: auto;
  height: auto;
  max-width: 95%;
  max-height: 95%;
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background: linear-gradient(90deg, #eeeeee 50%, #e3e3e3 0);
  background-size: 40px 100%;
}
</style>
