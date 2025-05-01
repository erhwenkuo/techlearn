# uv Search API

SearXNG 的搜尋 API 支援 `GET` 和 `POST`。

此外有兩個端點 `/` 和 `/search` 可供查詢。

##### 參數

> | name      |  type     | data type               | description                                                           |
> |-----------|-----------|-------------------------|-----------------------------------------------------------------------|
> | `q`         |  required | string                  | 搜尋查詢。該字串被傳遞給外部搜尋服務。因此，SearXNG 支援每種搜尋服務的語法。例如，是 Google 的有效查詢。但是，如果只是將上述查詢傳遞給任何不根據此語法過濾結果的搜尋引擎，您可能無法獲得想要的結果。參閱 [Search syntax](https://docs.searxng.org/user/search-syntax.html#search-syntax)|
> | `categories`|  optional | string                  | 逗號分隔列表，指定搜尋類別。在 SearXNG 有下列的類別: `general`, `images`, `videos`, `news`, `map`, `music`, `it`, `science`, `files`, `social_media`。參閱 [Configured Engines](https://docs.searxng.org/user/configured_engines.html#configured-engines)|
> | `engines` |  optional | string                  | 逗號分隔的列表，指定搜尋引擎。參閱 [Configured Engines](https://docs.searxng.org/user/configured_engines.html#configured-engines)|
> | `language` |  optional | string                  | 逗號分隔的列表，指定搜尋語言類別。|
