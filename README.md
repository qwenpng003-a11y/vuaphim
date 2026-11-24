
**# WEBPHIM - MVC Website for Watching Free Movies Online

Welcome to WEBPHIM, a PHP-based MVC website that lets you watch free movies online. This repository contains the source code for the WEBPHIM website, along with various components and features such as login management pages, admin pages, search page, navigation, sidebar, topbar, footer, header, movie content, playback bar, and movie rating.

## Requirements
- PHP 7.4 or later
- PHP `mysqli` extension enabled
- MySQL 5.7 or later
- Apache web server

## Installation
1. Clone this repository to your local machine.
2. Import the single complete database file `database/assfilm.sql` into MySQL.
3. Configure the database connection details in `libs/db.php`.
4. Enable `mysqli` in the `php.ini` used by your web server and restart the server.
5. Launch the website through Apache.

The complete dump includes all tables and columns used by the current features,
including advertisements, video quality and skip markers, SEO/site settings,
SEO articles, watch history, copyright reports, bank accounts, watch parties,
danmaku, VNPay payments, and memberships. `database/assfilm.sql` is the only SQL
file required for a fresh installation.

The movie listing filter uses the existing film, category, and nation data to
combine country, movie type, genre, and release year, with sorting by newest,
views, title, or duration. Age classifications and IMDb scores are not stored
in the current schema, so those filters are not shown.

This SQL file initializes a new database and includes sample site data. Do not
import it over an existing database. Back up existing data before planning a
database upgrade. For an existing installation that only needs bank-account
management, create or update `bank_accounts` with this statement instead of
re-importing the full dump:

```sql
CREATE TABLE IF NOT EXISTS `bank_accounts` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `bank_name` varchar(80) NOT NULL,
  `bank_code` varchar(12) NOT NULL DEFAULT '',
  `account_name` varchar(120) NOT NULL,
  `account_number` varchar(34) NOT NULL,
  `branch` varchar(120) NOT NULL DEFAULT '',
  `transfer_note` varchar(120) NOT NULL DEFAULT '',
  `active` tinyint(1) NOT NULL DEFAULT 1,
  `created_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `bank_accounts_active` (`active`, `id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

If `bank_accounts` already exists, add the VietQR code column before using the
updated admin page:

```sql
ALTER TABLE `bank_accounts`
  ADD COLUMN `bank_code` varchar(12) NOT NULL DEFAULT '' AFTER `bank_name`;
```

Edit each existing account in Admin and enter its VietQR bank code to enable
the QR image.

For an existing installation, create the SEO articles table with this statement
before opening **Bài viết SEO** in Admin:

```sql
CREATE TABLE IF NOT EXISTS `seo_articles` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `title` varchar(180) NOT NULL,
  `slug` varchar(180) NOT NULL,
  `meta_title` varchar(180) NOT NULL,
  `meta_description` varchar(320) NOT NULL,
  `content` mediumtext NOT NULL,
  `published` tinyint(1) NOT NULL DEFAULT 0,
  `created_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `seo_articles_slug` (`slug`),
  KEY `seo_articles_published_updated` (`published`, `updated_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

For an existing installation, create the admin calendar table before opening
**Lịch sự kiện & ghi chú**:

```sql
CREATE TABLE IF NOT EXISTS `admin_calendar_events` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `title` varchar(180) NOT NULL,
  `event_date` date NOT NULL,
  `description` text NOT NULL,
  `created_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `admin_calendar_events_date` (`event_date`, `id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### Movie tags tables (`tags` and `film_tag`)
`Admin -> Tag phim` (`admin/tags.php`) and public tag pages (`/?mod=tag&slug=...`)
automatically create the `tags` and `film_tag` tables if missing, and they are
also included in `database/assfilm.sql`. If you prefer to create them manually:

```sql
CREATE TABLE IF NOT EXISTS `tags` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(60) NOT NULL,
  `slug` varchar(80) NOT NULL,
  `created_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `tags_slug` (`slug`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `film_tag` (
  `film_id` int(11) NOT NULL,
  `tag_id` int(11) NOT NULL,
  PRIMARY KEY (`film_id`, `tag_id`),
  KEY `film_tag_tag_id` (`tag_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### VNPay configuration
Configure these environment variables on the PHP web server; never place the
hash secret in a public PHP or JavaScript file:

- `SITE_BASE_URL`: the canonical HTTPS origin and optional project path. The
  included `.htaccess` sets this to `https://vuaphim.io.vn`; update that
  directive if deploying the site on a different domain.
- `VNPAY_TMN_CODE`: the merchant terminal code issued by VNPay
- `VNPAY_HASH_SECRET`: the merchant signing secret issued by VNPay
- `VNPAY_URL`: optional payment URL; defaults to the VNPay sandbox

Register `SITE_BASE_URL/vnpayIpn` as the VNPay IPN endpoint in the merchant
portal. The browser return page does not mark donations as paid; the signed
server-side IPN does. Public VIP package purchases are disabled; the support
page accepts voluntary donations only.

### Importing licensed movie data
Admins can import movie metadata and episode URLs from **Admin → Nhập phim
JSON**. The importer accepts a JSON file with a top-level `movies` array (up to
100 movies and 100 episodes per movie, 2 MB total). It uses existing category,
movie type, and country IDs, skips movies whose title already exists, and
validates the whole file before saving it in one database transaction. See the
example and current IDs on the Admin import page. Only import metadata and
media URLs that you own or are authorized to use; the importer does not connect
to third-party movie catalogs.

Admins with authorization for the configured provider can also use **Admin →
Crawler phim** to import the provider's latest catalog. It processes at most
10 pages per run, pauses between detail requests, matches source categories and
countries by name, and uses the selected defaults when no match exists. Source
IDs and update timestamps are stored in `movie_import_sources` (created
automatically). Existing movies without a source mapping are not overwritten
when their title matches. The crawler imports only media URLs supported by the
site player (MP4, YouTube, Vimeo, or Dailymotion); HLS/M3U8 requires HLS player
support and is skipped. Use this only when you have permission to reuse the
catalog data and media URLs.
The same page can save a scheduled run (up to 3 pages); configure the displayed
cron command on the server to run `ophimSchedule.php`. The CLI job is locked
against overlapping runs and logs failures through PHP's error log. Because CLI
does not inherit web server environment settings, the cron command sets
`OPHIM_CRAWLER_API_BASE` explicitly if overriding the default.
`libs/ophimCrawler.php` defaults `OPHIM_CRAWLER_API_BASE` to
`https://danyphim.online` (and prioritizes the value saved in `site_settings`
from **Admin → Crawler phim**, avoiding `SetEnv` in `.htaccess` on servers
without `mod_env`). The crawler supports DanyPhim, KKPhim, and OPhim JSON endpoints at
`/danh-sach/phim-moi-cap-nhat?page=N` and `/phim/{slug}` with full HLS/M3U8 stream support.
Change the environment value in Apache and the CLI environment if the provider
changes its API. Keep TLS verification enabled; if PHP cannot locate a trusted
CA bundle, configure `curl.cainfo` in `php.ini` or set `OPHIM_CRAWLER_CA_BUNDLE`
to a trusted CA bundle path in both environments.

The player also offers browser-supported picture-in-picture, remote playback,
download of the provided licensed MP4, and five-second WebM clip export. Clip
creation and sharing depend on browser MediaRecorder/Web Share support.

Google AdSense is loaded for regular visitors across the site, allowing
AdSense Auto ads formats such as Anchor ads to be managed by Google. Enable
Auto ads and the Anchor format for this site in the AdSense account; loading
the script alone does not enable those formats. The script is not loaded for
admin accounts or premium members. Configured third-party ad-network scripts
are also loaded for regular visitors. Admin-managed image banners can be placed
at the top, middle, or bottom of the homepage, film detail, film list, and
movie-watch pages.

Admins can enable or disable advertising site-wide from Admin → Quảng cáo.
When disabled, AdSense, third-party network scripts, and video pre-roll ads
are not shown to regular visitors. Admin accounts and premium members remain
ad-free.
Admins can also enable or disable each configured network zone and edit its
numeric zone ID on the same page, or add additional zones from a supported
provider. Added zones and custom providers can be removed individually from
the network configuration table; built-in network entries are not deletable.
Built-in script hosts are fixed in the application; arbitrary
JavaScript is not accepted as inline code. Admins may also register a custom
provider with a name, an HTTPS script URL, and an optional zone delivered as
`data-zone` or a query parameter. Only add script URLs from providers you
trust because they execute in visitors' browsers.

The root `ads.txt` file authorizes the configured AdSense publisher account.
Deploy it so `https://your-domain/ads.txt` is publicly accessible; AdSense may
take time to recrawl the file and serve ads after account or site changes.

### Clean URLs
The included `.htaccess` removes `.php` from page URLs and routes extensionless
requests to their PHP files. Run the site on Apache with `mod_rewrite` enabled
and `AllowOverride FileInfo Options` enabled for the project directory. If you
use a different web server, configure equivalent rewrite rules there.
Movie detail pages use short Vietnamese-transliterated paths such as
`/phim/ten-phim-123`; the numeric ID keeps titles unique. Existing
`?mod=detail&film_id=123` links are permanently redirected to the matching
movie path.

### SEO
Public pages provide Vietnamese page titles, descriptions, and Open Graph
metadata. Admins can publish multiple SEO articles, each with a unique slug,
page title, meta title, and description; published pages are listed at
`?mod=seo` and can be opened at `?mod=seo&slug=your-article-slug`. Search and
video playback pages are marked `noindex`; provide the site's production
domain and a sitemap in the web-server deployment for domain-specific
canonical URLs and search-engine submission.
Movie titles and descriptions are generated from the stored language, quality,
year, and synopsis data. Movie structured data includes only catalog details
that are available, such as year, director, and cast; no ratings are invented.
Admin pages also send `X-Robots-Tag: noindex, nofollow, noarchive` because
configuration screens are not public search results. Validate structured data
on public pages such as the homepage, movie details, and published SEO
articles, not on Admin screens.
Filtered movie-list pages include titles and descriptions based on their
selected country, genre, movie type, and release year. Their canonical URLs
retain valid filters but omit sort order; non-default sort variants are marked
`noindex`. The sitemap includes movie types and countries/genres that have
movies, plus release years present in the catalog.
The sitemap is available at `/sitemap.xml` and robots instructions at
`/robots.txt`; submit the sitemap in Google Search Console. Add the HTML
verification token from Search Console under Admin → **Thông tin website & SEO**
to add another property verification token. The supplied verification token is
already emitted on public pages. Search Console verification and sitemap
submission help Google discover and report on pages but do not guarantee
rankings.
The SEO article editor supports bold and italic text, headings, HTTP/HTTPS
links, image uploads, and a live preview. Article content is rendered from a
small safe formatting syntax; raw HTML is not executed. Uploaded images are
stored in `images/seo-articles/`, so the web server must be able to create
files there.
The footer also links to public terms of use (`?mod=terms`) and a copyright
and content responsibility page (`?mod=copyright`) with instructions for
submitting a report from the relevant movie detail page.

## Features

### Login Management Pages
WEBPHIM includes login management pages that allow users to sign in and access various features of the website. Users can create accounts and log in using their email address and password.

The public movie site uses a cosmic dark theme by default, with deep-space
surfaces and cyan/purple accents. Visitors can switch to light mode with the
theme button in the header; their selection is remembered in the browser.

### Admin Pages
Admin pages are also included to allow website administrators to manage users, movies, and other website content.
The **Thêm phim** and **Quản lý phim** pages let admins create, search, edit,
and remove movies. Removing a movie also removes its episodes and related
watching data while preserving copyright reports.
The **Quản lý người dùng** page shows each account's VIP status. Edit a member
to grant or revoke VIP and set its expiration date; active VIP members skip
website advertisements, including video pre-roll ads.
The **Thêm tập phim** page accepts a local MP4 upload (up to 100 GB), a direct
HTTP/HTTPS MP4 URL, a local MP4 path under `images/video/`, or a YouTube link
or video ID. Uploaded videos are stored in `images/video/episodes/`; the web
server must allow PHP to write there. To accept a 100 GB upload, the host must
support large request bodies and have enough temporary and permanent storage.
Configure PHP `upload_max_filesize` to `100G` and `post_max_size` above `100G`
(for example `101G`), and raise the web-server/proxy request-body and timeout
limits as needed. Shared hosting commonly imposes a lower limit; a direct MP4
URL avoids uploading the file through PHP.
The **Quảng cáo** page lets admins create and edit banner images by
uploading a JPG, PNG, GIF, or WebP image (up to 5 MB), or using an HTTP/HTTPS
image URL. An optional HTTP/HTTPS destination link can be set for each banner.
Ads are hidden from premium members and admin accounts. Admins can change an
ad's image and active state, or delete it. Each banner can be assigned to the
top, middle, or bottom of the homepage, film detail, film list, or movie-watch
content. It can also be shown as a dismissible overlay on every page. For an
overlay, admins can choose one of five positions (four screen corners or
center), set width and offsets, and preview the placement live in the Admin
page.
Each overlay banner can also be individually enabled, positioned, resized, and
offset from the Admin banner list, with its own live preview. Multiple active
overlay banners may be assigned separate screen positions. Admins can choose
**Vị trí tự do** or drag a banner in its preview to set a custom position;
the saved horizontal and vertical coordinates are percentages of the screen.
Each banner can independently allow or hide its dismiss (×) button; the
default for existing banners is dismissible.
The create-ad form includes an image preview, placement selection, and
per-banner overlay controls, so the image, destination, and placement can be
checked before saving.
Only image banners managed by Admin are displayed. Third-party ad scripts and
Google Auto ads are not loaded, preventing pop-under redirects from ordinary
page interactions. A destination opens only when a visitor clicks a banner.
Admins can explicitly enable third-party scripts from **Quản lý quảng cáo**;
the setting defaults to off, and individual network zones remain configurable
there. Admins can also select which page types may load the scripts (default:
homepage and movie-watch pages). Enabling scripts may reintroduce pop-ups or
redirects from those networks; those behaviors cannot be reliably controlled
by the site, so disable the relevant provider or all scripts to prevent them.
Admins can reorder banners on the **Quảng cáo** page by dragging rows or using
the up/down buttons. Existing banners saved with legacy homepage or movie-watch
placements are retained but not displayed; edit them and choose a supported
page-and-position or overlay placement to show them.
The web server must allow PHP to create files in
`images/advertisements/`; configure PHP `upload_max_filesize` and
`post_max_size` to at least 5 MB.

Admins can add an episode using an uploaded MP4, a direct MP4 URL, a YouTube
link, or an HTTPS embed URL/iframe from YouTube, Vimeo (`player.vimeo.com`), or
Dailymotion. `phimmotchill.bar` page links can be used to request the public
page title, but are not playable sources because the site currently sends
`X-Frame-Options: SAMEORIGIN` and `Content-Security-Policy: frame-ancestors
'none'`. Iframe markup is reduced to its validated source URL; arbitrary
HTML and unapproved iframe hosts are not rendered. Public provider metadata can
be requested to fill a short episode name; for `phimmotchill.bar` the public
page title is used. Video files are not copied from third-party sites. Do not
proxy or bypass a provider's iframe restrictions; use a source that explicitly
allows embedding. The
`episode.content` column supports up to 2048 characters;
the Admin add-episode page expands the column on existing databases when a
long embed URL is first saved.

Admins can also manage MP4 pre-roll video ads at **Quảng cáo trước khi xem phim**.
Each ad has an HTTPS/HTTP MP4 URL and a skip delay (default 5 seconds). Ads
appear before native MP4 and YouTube playback and are hidden from premium
members and admin accounts. Pre-roll is limited to once every 12 hours per
browser. If the browser blocks autoplay, viewers can start the ad with a button;
the film remains paused until the ad is skipped or finishes. For an existing
database, create the table before using this page. The legacy
`start_at_seconds` column remains in the schema for compatibility and is no
longer used:

```sql
CREATE TABLE IF NOT EXISTS `video_advertisements` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `title` varchar(100) NOT NULL,
  `video_url` varchar(2048) NOT NULL,
  `start_at_seconds` int(11) NOT NULL DEFAULT 300,
  `skip_after_seconds` int(11) NOT NULL DEFAULT 5,
  `active` tinyint(1) NOT NULL DEFAULT 1,
  `created_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `video_advertisement_active` (`active`, `id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```
Admins can update site name, SEO description, Telegram contact, theme accent
colors, header tagline, footer copy, and which homepage content sections are
visible. They can also review content reports, publish SEO articles, and
inspect VNPay transactions in their respective admin pages. The site settings
page also accepts a JPG, PNG, or
WebP site image up to 5 MB; it is used as the header/footer logo and the
browser tab and mobile home-screen icon, as well as the Open Graph preview
image when sharing pages. Public movie pages adapt their navigation, filters,
movie cards, detail pages, and player layout to mobile screens. PHP `fileinfo`
must be enabled, and the web server must be able to create files in
`images/site/`.
The **Quản lý ngân hàng** page lets admins add, edit, hide, or remove bank
accounts displayed as VietQR codes on the VIP/support page. Enter the bank's
VietQR short code or BIN (for example, `VCB`, `MB`, or a six-digit BIN) as well
as the account details. QR images are provided by `img.vietqr.io`; bank code,
account number, account name, and transfer note are included in the image URL.
Supporters can choose a transfer amount of at least 10,000 VND to generate a QR
with the amount prefilled, or scan the QR without an amount and enter one in
their banking app. Bank transfers require manual reconciliation and do not
activate VIP automatically.

### Search Page
The search page allows users to search for movies based on various criteria, such as title, genre, and rating.

### Navigation, Sidebar, Topbar, Footer, Header
The website includes a navigation bar, sidebar, topbar, footer, and header to provide easy navigation and access to various features of the website.

### Movie Content
WEBPHIM lets users view movie content, including movie titles, descriptions, and ratings.

### Playback Bar
The playback bar allows users to play movies directly on the website.

### Movie Rating
Users can rate movies on a scale of 1 to 5 stars, and the average rating is displayed on the movie's page.

## Contributors
WEBPHIM is an open-source project and welcomes contributions from developers worldwide. If you are interested in contributing to this project, please contact the project maintainer for more information.

## License
This project is licensed under the MIT License. See the `LICENSE` file for details.
