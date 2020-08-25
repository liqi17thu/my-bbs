<template>
  <div>
    <div v-if="loading">Loading...</div>
    <div v-else>
      <div class="header">
        <div class="title">{{ title }}</div>
        <el-switch v-model="authorOnly" active-text="只看楼主"></el-switch>
        <el-switch v-model="mark" active-text="收藏"></el-switch>
      </div>
      <div>
        <Post @reflesh="loadData" v-for="i in showNum" :key="posts[i-1].id" :post="posts[i-1]"></Post>
        <div style="margin-top:20px">
          <div v-if="noMore">没有更多了 囧</div>
          <div v-else>
            <el-button type="primary" @click="loadMore">加载更多</el-button>
          </div>
        </div>
      </div>
      <div class="footer">
        <div class="row">
          <span class="hint">回复楼主</span>
        </div>
        <Editor ref="editor"></Editor>
        <el-button type="success" icon="el-icon-upload" class="submit" @click="submit">发表</el-button>
      </div>
    </div>
  </div>
</template>

<script>
import Post from '@/components/Post.vue'
import Editor from '@/components/Editor.vue'

export default {
  data () {
    return {
      authorOnly: false,
      mark: false,
      loading: false,
      title: '',
      posts: [],
      showNum: 1
    }
  },
  components: {
    Editor,
    Post
  },
  computed: {
    noMore () {
      return this.showNum >= this.posts.length
    }
  },
  methods: {
    submit () {
      return 0
    },
    loadMore () {
      this.showNum = parseInt(this.showNum) + 3
      if (this.showNum > this.posts.length) {
        this.showNum = this.posts.length
      }
    },
    loadData () {
      this.posts = []
      this.$http({
        url: `/api/v1/post/${this.$route.params.id}`,
        method: 'get',
        headers: {
          Authorization: this.$store.getters.getToken
        }
      })
        .then(response => {
          // console.log(response)
          this.title = response.data.title

          const postRoots = {}
          const users = {}

          // check bookmark
          if (this.$store.getters.getBookmark != null) {
            this.mark = response.data.id in this.$store.getters.getBookmark
          }

          // add to history
          this.$store.commit('addHistory', {
            id: response.data.id,
            title: response.data.title,
            nickname: response.data.nickname,
            userId: response.data.userId,
            content: response.data.content,
            created: response.data.created,
            updated: response.data.updated,
            visitTime: new Date().toGMTString()
          })

          this.posts.push({
            id: 0,
            title: response.data.title,
            postId: response.data.id,
            userId: response.data.userId,
            nickname: response.data.nickname,
            content: response.data.content,
            created: response.data.created,
            updated: response.data.updated,
            reply: []
          })

          let currentPostNum = 0
          postRoots[0] = currentPostNum++
          users[0] = {
            userId: response.data.userId,
            nickname: response.data.nickname
          }

          for (const rep of response.data.reply) {
            users[rep.id] = {
              userId: rep.userId,
              nickname: rep.nickname
            }
            const post = {
              id: rep.id,
              replyId: rep.replyId,
              postId: rep.postId,
              userId: rep.userId,
              nickname: rep.nickname,
              content: rep.content,
              created: rep.created,
              updated: rep.updated,
              reply: [],
              repliedUser: '',
              repliedUserId: 0
            }
            if (rep.replyId === 0) {
              this.posts.push(post)
              postRoots[rep.id] = currentPostNum++
            } else {
              if (rep.replyId !== this.posts[postRoots[rep.replyId]].id) {
                post.repliedUser = users[rep.replyId].nickname
                post.repliedUserId = users[rep.replyId].userId
              }
              postRoots[rep.id] = postRoots[rep.replyId]
              this.posts[postRoots[rep.id]].reply.push(post)
            }
          }

          if (this.authorOnly) {
            const authorsPosts = []
            for (const post of this.posts) {
              if (post.userId === response.data.userId) {
                authorsPosts.push(post)
              }
            }
            this.posts = authorsPosts
          }

          let totalPostNum = 0
          let i = 0
          while (i < this.posts.length) {
            totalPostNum += 1 + this.posts[i].reply.length
            i += 1
            if (totalPostNum <= 15) {
              this.showNum = i
            }
          }

          this.loading = false
        })
        .catch(error => {
          console.log(error)
        })
    }
  },
  beforeMount () {
    this.loading = true
    this.loadData()
  },
  watch: {
    authorOnly: function (val) {
      this.loadData()
    },
    mark: function (val) {
      if (val) {
        const post = JSON.parse(JSON.stringify(this.posts[0]))
        delete post.reply
        post.id = post.postId
        this.$store.commit('addBookMark', post)
      } else {
        this.$store.commit('removeBookMark', this.posts[0].postId)
      }
    }
  }
}
</script>

<style scoped>
.header {
  margin: 20px;
}

.footer {
  margin: 30px;
}

.title {
  font-size: 32px;
}

.el-switch {
  margin: 10px;
}

.row {
  position: relative;
  display: block;
  line-height: 30px;
  font-size: 14px;
  color: #bfbfbf;
}

.hint {
  font-size: 16px;
  color: #979797;
}

.title-input {
  width: 500px;
}

.submit {
  margin-top: 45px;
}
</style>