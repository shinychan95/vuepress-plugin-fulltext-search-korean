# vuepress-plugin-fulltext-search-korean

**[leo-buneev](https://github.com/leo-buneev)님의 프로젝트 [vuepress-plugin-fulltext-search](https://github.com/leo-buneev/vuepress-plugin-fulltext-search)를 바탕으로 한국어 정상 동작을 위해 추가 작업을 진행하였습니다.**

_(vuepress-plugin-fulltext-search [610de90](https://github.com/leo-buneev/vuepress-plugin-fulltext-search/commit/610de90389a84cc3cc9056b7b6a464959be4b5e0) 기준 추가 작업 진행)_

## 추가 작업한 부분

### 1. UI 단순하게 변경

![UI](https://user-images.githubusercontent.com/39409255/194307055-3c86af7f-abb0-4619-9142-e7aee09901d8.png)

### 2. 한국어 검색을 위한 처리

#### string normalization 제거

- 기존 `vuepress-plugin-fulltext-search`은 내부적으로 아래와 같이 동작하였다.
   - 색인을 생성할 때는 normalization 처리되지 않은 content 사용
   - 검색 질의는 normalization 처리되지 않은 문자열 사용
   - 하이라이팅 관련 로직에서는 질의 및 content를 normalization 처리된 문자열 사용

- 즉, 검색은 정상적이나 하이라이팅이 정상적이지 않아, 우선 normalization 처리를 모두 제거하는 방향으로 통일
   - _(추후 normailzation 제거로 인한 부가적인 문제 해결 예정)_

#### custom encoder 및 tokenizer 정의

- 필요에 따라 encoder 및 tokenizer를 정의하였다.
- encoder,
   - `#`, `space`, `tab`, `line split` 문자들을 space로 변경하고, 중첩 space를 단일 space로 변경
- tokenizer,
   - space 기준으로 분리하고
   - 분리된 단위를 flexsearch 내 forward 방식으로 분리하였다.
   
## Usage

First, install plugin.

```bash
npm i vuepress-plugin-fulltext-search -D
# or
yarn add vuepress-plugin-fulltext-search -D
```

Then, enable the plugin in your `docs/.vuepress/config.js`:

```js
// docs/.vuepress/config.js
module.exports = {
  // ...
  plugins: ['fulltext-search'],
}
```

And that is it! Just compile your app and see for yourself.

Webpack alias `@SearchBox` will be replaced with plugin's implementation, so it should work automatically with any
VuePress theme.

### Search parameters

The `query` URL search parameter can be provided to automatically populate and focus the search box. This is useful for
adding your VuePress website as a custom search engine in browsers. For example:

```none
https://your-website.com?query=hello+world
```

### Excluding pages from search

You can exclude pages from search suggestions by adding `search: false` to a page's frontmatter:

```none
---
search: false
---

<!-- page content -->
```

### Hooks

You can define several hooks to customize behaviour of various search features. For example:

```js
// /docs/.vuepress/searchHooks.js
export default {
  async processSuggestions(suggestions, queryString, queryTerms) {
    if (queryString) {
      // add a suggestion to start a search in an external service
      suggestions.push({
        path: 'https://sourcegraph.com/search?patternType=literal&q=',
        slug: queryString,
        parentPageTitle: 'Sourcegraph',
        title: 'Search code',
        contentStr: 'Search for "' + queryString + '" on Sourcegraph',
        external: true,
      })
    }
    return suggestions
  },

  async onGoToSuggestion(index, suggestion, queryString, queryTerms) {
    // e.g. create an analytics event

    // return true if you want to prevent default navigation
    return true
  },
}

// /docs/.vuepress/config.js
// Important: Because of the way Vuepress build works, you cannot use regular import/require,
// code must be provided as plaintext. Hence syntax below is required with fs.readFileSync
const fs = require('fs')
const { path } = require('@vuepress/shared-utils')

module.exports = {
  plugins: [
    [
      'fulltext-search',
      {
        // provide the contents of a JavaScript file
        hooks: fs.readFileSync(path.resolve(__dirname, './searchHooks.js')),
      },
    ],
  ],
}
```

Supported hooks are:

```ts
/**
 * Augment, adjust, or manipulate the suggestions shown to users.
 */
async function processSuggestions(suggestions: Suggestion[], queryString: string, queryTerms: string[]): Suggestion[]

/**
 * Callback function to call a suggestion.
 */
async function onGoToSuggestion(index: number, suggestion: Suggestion, queryString: string, queryTerms: string[]): Boolean?
```

### Additional configuration

You can set additional configuration options in the config file. For `tokenize`, `split`, and `encode`, see the
[flexsearch documentation][flexsearch-options] for more details.

- `tokenize`: The indexing mode (tokenizer). Choose one of the built-ins or pass a custom tokenizer function.
- `split`: The rule to split words when using non-custom tokenizer (built-ins e.g. "forward"). Use a string/char or use
  a regular expression (default: `/\W+/`).
- `encode`: The encoding type. Choose one of the built-ins or pass a custom encoding function.

The built-in `searchMaxSuggestions` will be used if set in the [theme config][search-max-suggestions].

For example:

```js
module.exports = {
  plugins: [
    [
      'fulltext-search',
      {
        tokenize: 'full',
        split: /\s+/,
        encode: 'icase',
      },
    ],
  ],
  themeConfig: {
    searchMaxSuggestions: 10,
  },
}
```

[flexsearch-options]: https://github.com/nextapps-de/flexsearch#initialize-index
[search-max-suggestions]: https://vuepress.vuejs.org/theme/default-theme-config.html#built-in-search
