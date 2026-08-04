---
name: Search Daring content and cite it correctly
description: Find anything on daring.com by keyword in one call, resolve each hit back to its full record, and produce a properly attributed citation with schema.org metadata or an oEmbed card.
api: openapi/daring-foods-search-api-openapi.yml
operations:
  - searchSiteContent
  - getRecipe
  - getProduct
  - getFoodserviceProduct
  - getPage
  - getSeoHead
  - getOEmbed
generated: '2026-08-04'
method: generated
source: openapi/daring-foods-search-api-openapi.yml, openapi/daring-foods-seo-api-openapi.yml, openapi/daring-foods-oembed-api-openapi.yml
---

# Search Daring content and cite it correctly

One call finds anything on daring.com across every searchable collection. Use this instead of
paging 208 recipes when the user's question is keyword-shaped.

Base URL: `https://daring.com/wp-json`. No authentication.

## Step 1 — search

`searchSiteContent` → `GET /wp/v2/search?search={query}&per_page=20&_fields=id,title,url,type,subtype`

Returns lightweight records:

```json
[{"id":1647,"title":"Crispy Buffalo Plant Chicken Quesadillas",
  "url":"https://daring.com/recipes/crispy-buffalo-plant-chicken-quesadillas/",
  "type":"post","subtype":"recipes"}]
```

`title` is plain text here (not entity-escaped, unlike the collection endpoints) and `url` is the
public permalink. Those two fields alone are often the whole answer.

Narrow with `subtype` when the intent is clear: `subtype=recipes`, `subtype=products`,
`subtype=foodservice-products`, `subtype=page`, `subtype=post`. Repeat the parameter for multiple.

## Step 2 — resolve a hit to its full record

`subtype` is the type discriminator that tells you which collection endpoint to call:

| subtype | operation | path |
|---|---|---|
| `recipes` | `getRecipe` | `GET /wp/v2/recipes/{id}` |
| `products` | `getProduct` | `GET /wp/v2/products/{id}` |
| `foodservice-products` | `getFoodserviceProduct` | `GET /wp/v2/foodservice-products/{id}` |
| `page` | `getPage` | `GET /wp/v2/pages/{id}` |

Only `page` returns full rendered body content. For everything else the resolve step adds dates,
categories and a featured image id — not the substance.

Skip this step entirely when the search result already answers the question. It usually does.

## Step 3 — cite it

Two ways, both single unauthenticated calls against any daring.com URL.

**schema.org JSON-LD** — `getSeoHead` →
`GET /yoast/v1/get_head?url={percent-encoded-url}`

Returns `{json, html}`. `json.schema` carries the full schema.org `@graph` (WebPage, Article,
Organization, ImageObject, BreadcrumbList) plus canonical URL, title, description and Open Graph
properties. This is the most structured description of a Daring recipe or product available from
this host — the collections themselves carry no semantic typing. The same payload is inlined on
every collection record as `yoast_head_json`, so if you already fetched the record you do not need
this call.

**oEmbed card** — `getOEmbed` →
`GET /oembed/1.0/embed?url={percent-encoded-url}`

Returns oEmbed 1.0: `title`, `author_name`, `thumbnail_url`, `provider_name` and an iframe `html`
payload. `&format=xml` returns the same as XML. Use this when you want a titled, illustrated card.

Percent-encode the `url` parameter. Both operations return `404` for a URL that is not a public
daring.com page.

## Attribution rules

- Always link the `url` / `link`. Content and imagery are Daring's, under
  <https://daring.com/terms-and-conditions/>.
- Attribute to **Daring** (the brand) or **Daring Foods** (the company). Since 2025 it is owned by
  the Australian plant-based meat manufacturer v2food and still trades under its own brand.
- The `author_name` on a recipe is a Daring staff CMS account, not a public byline. Do not present
  it as a chef credit.

## Do not

- Do not answer nutrition, ingredient, allergen, recipe-step or price questions from this API. None
  of it is exposed — the `acf` member is empty on every record. Point at the page.
- Do not treat `/wp/v2/posts` as a company blog. It holds one item, the WordPress install default,
  and 41 comments that are predominantly automated spam. Ignore both.
- Do not present a `404` on a single-item read as proof the content never existed; unpublished and
  nonexistent are indistinguishable anonymously.

## Pacing

`Cache-Control: max-age=600`, robots.txt `Crawl-delay: 10`, no RateLimit headers, Cloudflare in
front. Cache aggressively and space requests.
