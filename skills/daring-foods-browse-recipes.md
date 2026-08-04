---
name: Browse Daring recipes by cooking method
description: Find Daring plant-chicken recipes filtered by how they are cooked (air fry, sauté, oven baked, microwave, pan fry, grill pan, dutch oven, braise), and resolve each hit to a permalink and image.
api: openapi/daring-foods-recipes-api-openapi.yml
operations:
  - listCategories
  - listRecipes
  - getRecipe
  - getMediaItem
generated: '2026-08-04'
method: generated
source: openapi/daring-foods-recipes-api-openapi.yml, openapi/daring-foods-taxonomy-api-openapi.yml, openapi/daring-foods-media-api-openapi.yml
---

# Browse Daring recipes by cooking method

Daring's recipe library is 208 recipes, and the only machine-readable classification on it is the
WordPress `category` taxonomy — which on this site holds **cooking methods**, not subjects. That
makes "show me Daring recipes I can make in an air fryer" a two-call question.

Base URL: `https://daring.com/wp-json`. No authentication, no key, no signup.

## Step 1 — get the cooking-method vocabulary

`listCategories` → `GET /wp/v2/categories?per_page=100&_fields=id,slug,name,count`

Returns the 9 terms with their recipe counts. As captured on 2026-08-04:

| id | slug | name | recipes |
|----|------|------|---------|
| 3 | saute | Sauté | 22 |
| 4 | oven-baked | Oven Baked | 13 |
| 15 | air-fry | Air Fry | 7 |
| 17 | microwave | Microwave | 7 |
| 5 | grill-pan | Grill Pan | 5 |
| 6 | pan-fry | Pan Fry | 5 |
| 7 | dutch-oven | Dutch Oven | 1 |
| 16 | braise | Braise | 1 |
| 1 | uncategorized | Uncategorized | 4 |

Always re-read the ids rather than hard-coding them — these are site-configuration values and can
change. Match the user's phrasing to a `slug`, not to a `name`.

## Step 2 — list recipes for that method

`listRecipes` → `GET /wp/v2/recipes?categories={id}&per_page=100&_fields=id,slug,title,link,featured_media`

- `per_page` is hard-capped at 100. Asking for more returns HTTP 400 `rest_invalid_param`.
- Read `X-WP-Total` for the true count and follow the `Link` header's `rel="next"` to page.
- Always pass `_fields`. Without it every record drags a multi-kilobyte `yoast_head` HTML blob.

To list the whole library instead, drop `categories`. Note the term counts sum to 65 against 208
recipes — **roughly two thirds of the library carries no cooking-method term at all**, so never
present a filtered list as if it were exhaustive.

## Step 3 — read one recipe

`getRecipe` → `GET /wp/v2/recipes/{id}?_embed`

`_embed` inlines the featured image and assigned terms in the same round trip, which is preferable
to a follow-up `getMediaItem` call per recipe. Use `getMediaItem` →
`GET /wp/v2/media/{featured_media}` only when you already hold an id from a `_fields`-trimmed list.

## The critical limitation — state it, do not work around it

**The API does not return recipe ingredients, steps, or cook times.** Those live in Advanced Custom
Fields, and this host does not expose ACF over REST: the `acf` member is present on every record and
is always empty. What you get is the recipe's title, slug, permalink, dates, cooking-method terms and
featured image.

If the user wants the actual recipe, give them the `link` and say plainly that the full method is on
the page, not in the API. Do not invent ingredients or steps, and do not infer them from the title.

There is also **no field linking a recipe to the Daring product it uses**. If asked "which product
does this recipe need", answer from the recipe title if it is explicit, and otherwise say the API
does not carry that association.

## Errors

- `400 rest_invalid_param` — a parameter failed validation; read `data.params` (almost always
  `per_page` over 100).
- `404 rest_post_invalid_id` — no published recipe with that id. Unpublished and nonexistent are
  indistinguishable.

## Rate and cache discipline

Responses carry `Cache-Control: max-age=600` behind a WP Engine/Cloudflare edge, and robots.txt asks
for `Crawl-delay: 10`. Cache for at least 10 minutes and space requests by a second or more. No
RateLimit headers are returned and Cloudflare may throttle without warning. Content changes on the
order of weeks — the newest recipe at capture was 2026-06-23.
