---
name: Harvest the Daring product catalog
description: Pull the full Daring retail and foodservice plant-chicken catalog with permalinks and packaging imagery, and read the company's own ingredient and mission statements from the marketing pages.
api: openapi/daring-foods-products-api-openapi.yml
operations:
  - listPostTypes
  - listProducts
  - getProduct
  - listFoodserviceProducts
  - getFoodserviceProduct
  - listPages
  - getPage
  - getMediaItem
generated: '2026-08-04'
method: generated
source: openapi/daring-foods-products-api-openapi.yml, openapi/daring-foods-foodservice-api-openapi.yml, openapi/daring-foods-pages-api-openapi.yml, openapi/daring-foods-discovery-api-openapi.yml
---

# Harvest the Daring product catalog

Daring ships two separate product lines, on two separate WordPress post types. Treat them as
distinct collections — a foodservice item is not a variant of a retail item.

Base URL: `https://daring.com/wp-json`. No authentication.

## Step 0 — confirm the collections still exist

`listPostTypes` → `GET /wp/v2/types`

`products`, `foodservice-products` and `recipes` are **custom** post types registered by this site's
theme or plugins, not WordPress core. Their routes can move or disappear with any theme or plugin
change, silently. Read `rest_base` and `rest_namespace` off this call rather than assuming the path.

## Step 1 — the retail line

`listProducts` → `GET /wp/v2/products?per_page=100&_fields=id,slug,title,link,featured_media`

14 products at capture: Original Shredded and Diced Plant Chicken; six Plant Chicken Bowls (Buffalo
Mac & Cheese, Queso Burrito, Fly By Jing Chili Crisp, Harvest, Spicy Fajita, Penne Primavera,
Teriyaki); Buffalo and Original Plant Chicken Wings; Cajun, Teriyaki and Original Pieces.

Titles come back HTML-entity-escaped in `title.rendered` — `Buffalo Mac &#038; Cheese Plant Chicken
Bowl`. Decode entities before displaying.

`getProduct` → `GET /wp/v2/products/{id}` for a single item.

## Step 2 — the foodservice line

`listFoodserviceProducts` → `GET /wp/v2/foodservice-products?per_page=100&_fields=id,slug,title,link,featured_media`

7 items at capture, including the Unbreaded Gluten-Free Diced Plant Chicken. Same record shape as
retail. `getFoodserviceProduct` for a single item.

## Step 3 — packaging imagery

`getMediaItem` → `GET /wp/v2/media/{featured_media}`

Returns `source_url` for the original plus a `media_details.sizes` map with every generated variant
and its own URL, width and height. Pick the smallest variant that meets your need. Images are served
from `daring.com/wp-content/uploads/` under the site's Terms & Conditions — link rather than
redistribute.

Alternatively append `_embed` to the product call to inline the media record and skip this step.

## Step 4 — the company's own claims

`listPages` → `GET /wp/v2/pages?per_page=100&_fields=id,slug,title,link`
`getPage` → `GET /wp/v2/pages/{id}?_fields=id,slug,title,content`

Pages are the **only** collection on this host that returns full rendered `content`. Useful ids at
capture: 199 Ingredients, 14 Our Mission, 12 How To Cook, 447 FAQ, 16 Terms & Conditions, 15
Foodservice (with Products/Mission/How To Cook/Support/Store Locator hanging off it as children).

When you need to state what is in a Daring product or what the company claims about sourcing, quote
these pages. They are the provider's own words.

## What this catalog does NOT contain

There is **no SKU, GTIN, UPC, price, pack size, nutrition panel, ingredient list, allergen
declaration or retail availability field** on any product record. All of it is in Advanced Custom
Fields, which this host does not expose over REST — `acf` is present and empty on every record.

If asked for nutrition or ingredients for a specific product, fetch the Ingredients page for the
company's general statement and give the product's `link` for the panel. Never reconstruct a
nutrition panel from memory or inference.

There is also no store or retailer entity — the locator at `https://daring.com/locator/` is a
third-party embed outside this API.

## Errors and pacing

`400 rest_invalid_param` on `per_page` over 100; `404 rest_post_invalid_id` on an unknown id; `401
rest_cannot_create` on any write attempt (writes are credential-gated and out of scope). Cache for
at least 10 minutes; robots.txt asks for `Crawl-delay: 10`. The catalog changes rarely — the newest
product at capture was dated 2025-11-03.
