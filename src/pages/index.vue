<template>
  <amplify-authenticator>
    <section>
      <div>
        <div>
          <ul>
            <li class="mt-1 text-center" v-for="p in posts" :key="p">
              <div class="card">
                <div class="card-content">
                  <p>title: {{ p.title }}</p>
                  <p>body: {{ p.content }}</p>
                  <p>owner: {{ p.owner }}</p>
                  <b-button type="is-primary" v-on:click="deletePost(p.id)"
                    >Primary</b-button
                  >
                </div>
              </div>
            </li>
          </ul>
        </div>
        <div>
          <b-field label="title">
            <b-input v-model="title"></b-input>
          </b-field>
          <b-field label="content">
            <b-input v-model="content"></b-input>
          </b-field>
          <b-button v-on:click="createPost()">upload</b-button>
        </div>
      </div>
      <amplify-sign-out></amplify-sign-out>
    </section>
  </amplify-authenticator>
</template>

<script>
import { API } from 'aws-amplify'
import { createPost, deletePost } from '~/graphql/mutations'
import { listPosts } from '~/graphql/queries'
import { GRAPHQL_AUTH_MODE } from '@aws-amplify/api-graphql/lib/types'

export default {
  name: 'HomePage',

  data() {
    return {
      title: '',
      content: '',
      posts: [],
    }
  },

  async created() {
    this.getPosts()
  },

  methods: {
    async createPost() {
      const { title, content } = this
      if (!title || !content) return
      const post = { title, content }
      await API.graphql({
        authMode: GRAPHQL_AUTH_MODE.AMAZON_COGNITO_USER_POOLS,
        query: createPost,
        variables: { input: post },
      })
      this.title = ''
      this.content = ''
      this.getPosts()
    },
    async getPosts() {
      const ps = (
        await API.graphql({
          query: listPosts,
        })
      ).data.listPosts.items
      this.posts = ps
    },
    async deletePost(postId) {
      try {
        const res = await API.graphql({
          authMode: GRAPHQL_AUTH_MODE.AMAZON_COGNITO_USER_POOLS,
          query: deletePost,
          variables: {
            input: {
              id: postId,
            },
          },
        })
        this.getPosts()
      } catch (e) {
        this.$buefy.snackbar.open({
          message: `${e.errors[0].errorType}`,
          type: 'is-danger',
          position: 'is-bottom',
          actionText: 'colse',
        })
        console.log(e)
      }
    },
  },
}
</script>
