<template>
  <v-row
    class="mt-15"
    justify="center"
    align="center"
  >
    <v-col cols="12" sm="8" md="6">
      <v-card>
        <v-card-text>
          <div
            class="list"
            ref="list"
          >
            <VTextField
              placeholder="Search"
              prepend-inner-icon="mdi-magnify"
              filled
              @input="searchDebounce"
            />

            <div
              v-for="user in list"
              :key="getUserUniqueIdentifier(user)"
              :class="{
                'list__item d-flex': true,
                'list__item--selected': isUserSelected(user)
              }"
            >
              <v-img
                :src="user.avatar"
                :lazy-src="user.avatar.replace('300x300', '20x20')"
                class="list__item-avatar"
              />

              <div>
                <v-list-item :key="user.title">
                  <v-list-item-content>
                    <v-list-item-title class="text-h5">
                      {{ user.name }}
                    </v-list-item-title>

                    <v-list-item-subtitle class="font-weight-bold">
                      {{ user.title }}
                    </v-list-item-subtitle>

                    <v-list-item-subtitle>
                      {{ user.address }}, {{ user.city }}
                    </v-list-item-subtitle>
                  </v-list-item-content>
                  <div class="list__item-action grey--text">
                    {{ user.email }}
                  </div>
                </v-list-item>

                <div class="list__item-actions">
                  <VBtn
                    color="teal"
                    text
                    @click="selectUser(user)"
                  >
                    {{ isUserSelected(user) ? 'Skip selection' : 'Mark as suitable' }}
                  </VBtn>
                </div>
              </div>
            </div>

            <ClientOnly>
              <!-- The v-if condition is needed to not display when <InfiniteLoading> is not required. -->
              <InfiniteLoading
                v-if="list.length >= PER_PAGE"
                :identifier="infiniteId"
                spinner="bubbles"
                @infinite="infiniteHandler"
              />
            </ClientOnly>

            <template v-if="!list.length && searchStr">
              Search "{{ searchStr }}" returned no results.
            </template>
          </div>
        </v-card-text>
      </v-card>
    </v-col>
  </v-row>
</template>

<script>
import InfiniteLoading from 'vue-infinite-loading'
import debounce from 'lodash/debounce'
import Mark from 'mark.js'

const PER_PAGE = 4
const CONTENT_NAME = 'users'

export default {
  components: {
    InfiniteLoading
  },
  data () {
    return {
      PER_PAGE,
      list: [],
      infiniteId: 0,
      searchStr: '',
      // I expected that the selected users should be saved after the search.
      selectedUsers: {},
      markInstance: null
    }
  },
  async fetch () {
    this.list = await this.$content(CONTENT_NAME)
      .search(this.searchStr)
      .limit(PER_PAGE)
      .fetch()
  },
  mounted () {
    this.markInstance = new Mark(this.$refs.list)
  },
  methods: {
    getUserUniqueIdentifier (user) {
      // Email was repeated for some users, so I had to do this.
      return user.email + user.name
    },
    isUserSelected (user) {
      return this.selectedUsers[this.getUserUniqueIdentifier(user)]
    },
    selectUser (user) {
      this.$set(this.selectedUsers, this.getUserUniqueIdentifier(user), true)
    },
    async infiniteHandler (state) {
      const moreItems = await this.$content(CONTENT_NAME)
        .search(this.searchStr)
        .skip(this.list.length)
        .limit(PER_PAGE)
        .fetch()

      this.list.push(
        ...moreItems
      )

      if (moreItems.length) {
        // There are no more items.
        state.loaded()
      } else {
        // Completed fetching of additional items.
        state.complete()
      }

      if (this.searchStr) {
        await this.$nextTick()
        // Applies highlighting to loaded elements.
        this.markControl(this.searchStr)
      }
    },
    searchDebounce: debounce(async function search (str) {
      this.searchStr = str

      this.list = await this.$content(CONTENT_NAME)
        .search(this.searchStr)
        .limit(PER_PAGE)
        .fetch()

      // For correct reinitialization https://peachscript.github.io/vue-infinite-loading/guide/use-with-filter-or-tabs.html
      this.infiniteId++

      await this.$nextTick()
      this.markControl(this.searchStr)
    }, 500),
    // Search highlight control.
    markControl (str) {
      this.markInstance.unmark()
      this.markInstance.mark(str)
    }
  }
}
</script>

<style lang="sass">
  .list
    height: 660px
    overflow-y: auto
    padding-right: 16px

    &__item
      margin-bottom: 20px
      background: #fafafa
      border-radius: 5px
      border: 1px solid transparent

      &-actions
        border-top: 1px solid #e4e4e4
        padding: 10px 16px

      &-avatar
        width: 140px
        max-width: 140px
        background: grey
        border-top-left-radius: 5px
        border-bottom-left-radius: 5px

      &-action
        margin-top: 10px
        align-self: flex-start

      &--selected
        border-color: blue
</style>
