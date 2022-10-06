<template>
  <div class="search-box">
    <input
      ref="input"
      aria-label="Search"
      :value="query"
      :class="{ focused: focused }"
      :placeholder="placeholder"
      autocomplete="off"
      spellcheck="false"
      @input="query = $event.target.value"
      @focus="focused = true"
      @blur="focused = false"
      @keyup.enter="go(focusIndex)"
      @keyup.up="onUp"
      @keyup.down="onDown"
    />
    <ul 
     v-if="showSuggestions" 
     class="suggestions" 
     :class="{ 'align-right': alignRight }" 
     @mouseleave="unfocus"
    >
      <li
        v-for="(s, i) in suggestions"
        :key="i"
        class="suggestion"
        :class="{ focused: i === focusIndex }"
        @mousedown="go(i)"
        @mouseenter="focus(i)"
      >
        <a :href="s.path + s.slug" @click.prevent>
          <div class="suggestion-content">
            <div v-if="s.headingStr" class="header">
              {{ s.headingDisplay.prefix }}<span class="highlight">{{ s.headingDisplay.highlightedContent }}</span>{{ s.headingDisplay.suffix }}
            </div>
            <div v-if="s.contentStr" class="content">
              {{ s.contentDisplay.prefix }}<span class="highlight">{{ s.contentDisplay.highlightedContent }}</span>{{ s.contentDisplay.suffix }}
            </div>
          </div>
        </a>
      </li>
    </ul>
  </div>
</template>

<script>
import flexsearchSvc from '../services/flexsearchSvc'

// see https://vuepress.vuejs.org/plugin/option-api.html#clientdynamicmodules
import hooks from '@dynamic/hooks'

/* global SEARCH_MAX_SUGGESTIONS, SEARCH_PATHS, SEARCH_HOTKEYS */
export default {
  name: 'SearchBox',
  data() {
    return {
      query: '',
      focused: false,
      focusIndex: 0,
      placeholder: undefined,
      suggestions: null,
    }
  },
  computed: {
    queryTerms() {
      if (!this.query) return []
      const result = this.query.split(/[^\p{L}\p{N}_]+/iu)
      return result
    },
    showSuggestions() {
      return this.focused && this.suggestions && this.suggestions.length
    },

    // make suggestions align right when there are not enough items
    alignRight() {
      const navCount = (this.$site.themeConfig.nav || []).length
      const repo = this.$site.repo ? 1 : 0
      return navCount + repo <= 2
    },
  },
  watch: {
    query() {
      this.getSuggestions()
    },
  },
  /* global OPTIONS */
  mounted() {
    const options = OPTIONS || {}
    flexsearchSvc.buildIndex(this.$site.pages, options)
    this.placeholder = this.$site.themeConfig.searchPlaceholder || ''
    document.addEventListener('keydown', this.onHotkey)

    // set query from URL
    const params = this.urlParams()
    if (params) {
      const query = params.get('query')
      if (query) {
        this.query = decodeURI(query)
        this.focused = true
      }
    }
  },
  beforeDestroy() {
    document.removeEventListener('keydown', this.onHotkey)
  },
  methods: {
    async getSuggestions() {
      if (!this.query || !this.queryTerms.length) {
        this.suggestions = []
        return
      }
      let suggestions = await flexsearchSvc.match(
        this.query,
        this.queryTerms,
        this.$site.themeConfig.searchMaxSuggestions || SEARCH_MAX_SUGGESTIONS,
      )
      if (hooks.processSuggestions) {
        // augment suggestions with user-provided function
        suggestions = await hooks.processSuggestions(suggestions, this.query, this.queryTerms)
      }
      this.suggestions = suggestions.map(s => ({
        ...s,
        headingDisplay: highlight(s.headingStr, s.headingHighlight),
        contentDisplay: highlight(s.contentStr, s.contentHighlight),
      }))
    },
    getPageLocalePath(page) {
      for (const localePath in this.$site.locales || {}) {
        if (localePath !== '/' && page.path.indexOf(localePath) === 0) {
          return localePath
        }
      }
      return '/'
    },
    isSearchable(page) {
      let searchPaths = SEARCH_PATHS
      // all paths searchables
      if (searchPaths === null) {
        return true
      }
      searchPaths = Array.isArray(searchPaths) ? searchPaths : new Array(searchPaths)
      return (
        searchPaths.filter(path => {
          return page.path.match(path)
        }).length > 0
      )
    },
    onHotkey(event) {
      if (event.srcElement === document.body && SEARCH_HOTKEYS.includes(event.key)) {
        this.$refs.input.focus()
        event.preventDefault()
      }
    },
    onUp() {
      if (this.showSuggestions) {
        if (this.focusIndex > 0) {
          this.focusIndex--
        } else {
          this.focusIndex = this.suggestions.length - 1
        }
      }
    },
    onDown() {
      if (this.showSuggestions) {
        if (this.focusIndex < this.suggestions.length - 1) {
          this.focusIndex++
        } else {
          this.focusIndex = 0
        }
      }
    },
    go(i) {
      if (!this.showSuggestions) {
        return
      }
      if (hooks.onGoToSuggestion) {
        const result = hooks.onGoToSuggestion(i, this.suggestions[i], this.query, this.queryTerms)
        if (result === true) return
      }
      if (this.suggestions[i].external) {
        window.open(this.suggestions[i].path + this.suggestions[i].slug, '_blank')
      } else {
        this.$router.push(this.suggestions[i].path + this.suggestions[i].slug)
        this.query = ''
        this.focusIndex = 0
        this.focused = false

        // reset query param
        const params = this.urlParams()
        if (params) {
          params.delete('query')
          const paramsString = params.toString()
          const newState = window.location.pathname + (paramsString ? `?${paramsString}` : '')
          history.pushState(null, '', newState)
        }
      }
    },
    focus(i) {
      this.focusIndex = i
    },
    unfocus() {
      this.focusIndex = -1
    },
    urlParams() {
      if (!window.location.search) {
        return null
      }
      return new URLSearchParams(window.location.search)
    },
  },
}

function highlight(str, strHighlight) {
  if (!str) return {}
  if (!strHighlight) return { prefix: str }
  const [start, length] = strHighlight
  const end = start + length

  const prefix = str.slice(0, start)
  const highlightedContent = str.slice(start, end)
  const suffix = str.slice(end)
  return { prefix, highlightedContent, suffix }

}
</script>

<style lang="stylus">
.search-box
  display inline-block
  position relative
  margin-right 1rem
  input
    cursor text
    width 10rem
    height: 2rem
    color lighten($textColor, 25%)
    display inline-block
    border 1px solid darken($borderColor, 10%)
    border-radius .4rem
    font-size 0.9rem
    line-height 2rem
    padding 0 0.5rem 0 2rem
    outline none
    transition all .2s ease
    background #fff url(../assets/search.svg) 0.6rem 0.5rem no-repeat
    background-size 1rem
    &:focus
      cursor auto
      width: 15rem;
      border-color $accentColor
  .suggestions
    background #fff
    min-width 500px
    max-width 700px
    position absolute
    top 2 rem
    border 1px solid darken($borderColor, 10%)
    border-radius 0.6rem
    padding 0.4rem
    list-style-type none
    &.align-right
      right 0
  .suggestion
    padding 0.4rem 0.6rem
    cursor pointer
    a
      display block
      white-space normal
      color lighten($textColor, 15%)
      .suggestion-content
        .highlight
          color $accentColor
          font-weight bold
          font-style normal
        border-collapse collapse
        display table
        
        font-weight 400
        border-right none
        display table-cell
        padding 3px
        .header
          font-weight 600
        .content
          font-size 0.9em
    &.focused
      background-color lighten($accentColor, 93%)
@media (max-width: $MQNarrow)
  .search-box
    input
      cursor pointer
      width 0
      border-color transparent
      position relative
      &:focus
        cursor text
        left 0
        width 10rem
// Match IE11
@media all and (-ms-high-contrast: none)
  .search-box input
    height 2rem
@media (max-width: $MQNarrow) and (min-width: $MQMobile)
  .search-box
    .suggestions
      left 0
@media (max-width: $MQMobile)
  .search-box
    margin-right 0
    input
      left 1rem
    .suggestions
      right 0
@media (max-width: $MQMobileNarrow)
  .search-box
    .suggestions
      width calc(100vw - 4rem)
    input:focus
      width 8rem
</style>
