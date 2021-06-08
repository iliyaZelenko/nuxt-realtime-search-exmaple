<template>
  <VRow
    class="mt-15"
    justify="center"
    align="center"
  >
    <VCol
      cols="12"
      sm="8"
      md="6"
    >
      <VCard>
        <VCardText>
          <div class="list">
            <div class="list__search-field">
              <VTextField
                v-model="searchStr"
                :loading="loadingSearch"
                placeholder="Search"
                prepend-inner-icon="mdi-magnify"
                autofocus
                filled
                @input="searchDebounce"
              />
            </div>

            <div ref="markInitContainer">
              <div
                v-for="user in list"
                :key="getUserUniqueIdentifier(user)"
                :class="{
                  'list__item d-flex': true,
                  'list__item--selected': isUserSelected(user)
                }"
              >
                <VImg
                  :src="user.avatar"
                  :lazy-src="user.avatar.replace('300x300', '20x20')"
                  class="list__item-avatar"
                />

                <div>
                  <VListItem :key="user.title">
                    <VListItemContent>
                      <VListItemTitle class="text-h5">
                        {{ user.name }}
                      </VListItemTitle>

                      <VListItemSubtitle class="font-weight-bold">
                        {{ user.title }}
                      </VListItemSubtitle>

                      <VListItemSubtitle>
                        {{ user.address }}, {{ user.city }}
                      </VListItemSubtitle>
                    </VListItemContent>
                    <div class="list__item-action grey--text">
                      {{ user.email }}
                    </div>
                  </VListItem>

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
        </VCardText>
      </VCard>
    </VCol>
  </VRow>
</template>

<script>
import InfiniteLoading from 'vue-infinite-loading'
import debounce from 'lodash/debounce'
import Mark from 'mark.js'

const PER_PAGE = 4
const CONTENT_NAME = 'users'
const SEARCH_DEBOUNCE_WAIT = 500

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
      markInstance: null,
      loadingSearch: false
    }
  },
  async fetch () {
    // To pick up information from the URL's param.
    this.searchStr = this.$route.params.searchStr || ''
    this.list = await this.$content(CONTENT_NAME)
      .search(this.searchStr)
      .limit(PER_PAGE)
      .fetch()
  },
  mounted () {
    this.markInstance = new Mark(this.$refs.markInitContainer)

    if (this.searchStr) {
      this.markControl()
    }
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
      this.$set(
        this.selectedUsers,
        this.getUserUniqueIdentifier(user),
        !this.isUserSelected(user)
      )
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
        this.markControl()
      }
    },
    searchDebounce: debounce(async function search () {
      this.loadingSearch = true
      this.list = await this.$content(CONTENT_NAME)
        .search(this.searchStr)
        .limit(PER_PAGE)
        .fetch()

      // For correct reinitialization https://peachscript.github.io/vue-infinite-loading/guide/use-with-filter-or-tabs.html
      this.infiniteId++

      await this.$nextTick()
      this.markControl()

      // Hack to change URL without reloading page (miss Vue's state). There is no normal way to do this in Vue. If you try $router.replace, then the page state will be reset.
      history.pushState(
        {},
        null,
        this.$router.resolve({
          name: 'search',
          params: {
            searchStr: this.searchStr
          }
        }).href
      )
      this.loadingSearch = false
    }, SEARCH_DEBOUNCE_WAIT),
    // Search highlight control.
    markControl () {
      this.markInstance.unmark()
      this.markInstance.mark(this.searchStr)
    }
  }
}
</script>

<style lang="sass">
.list
  height: 660px
  overflow-y: auto
  padding-right: 16px

  &__search-field
    background: white
    position: sticky
    top: 0
    z-index: 999
    max-height: 66px

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
