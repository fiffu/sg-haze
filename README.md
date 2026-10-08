# sg-haze

A single-page site showing Singapore's latest PM2.5 readings by region, built so that link previews (WhatsApp, Teams, Slack, Telegram) show the numbers too.

```
PM2.5 n80🔴 s130🔴 e119🔴 w158🔴 c139🔴 µg/m³
```

Values are the latest 1-hr PM2.5 concentration (µg/m³) for north, south, east, west and central. 🔴 marks readings of 56 and above (NEA band II or worse).

Data source: `https://www.haze.gov.sg/` (NEA). Both the page and the link preview read `Chart1HRPM25` from `https://www.haze.gov.sg/api/airquality/jsondata/<timestamp>`, taking the last item of each region's `Data[]`. The trailing number is only a cache-buster.

## How it works

Link-preview crawlers don't run JavaScript, so the preview text has to be in the HTML they download.

- **`index.html`** is the whole site. In a browser, its script fetches the API directly and renders a table of 1-hr PM2.5 and its NEA band per region, refreshing every 5 minutes.
- **`.github/workflows/update.yml`** runs every 30 minutes, on every push to `main`, and on manual runs. It fetches haze.gov.sg, writes the summary line into the `<title>`, `og:*`, `twitter:*` and `description` tags of `index.html`, and deploys the result to GitHub Pages. Nothing is committed back to the repo.

## Setup

1. Push this repo to GitHub with `main` as the default branch.
2. In **Settings → Pages → Build and deployment → Source**, choose **GitHub Actions**.
3. Run the **Update preview** workflow from the Actions tab, or push a commit.

Free GitHub Pages needs a public repo.

## Notes

- **Previews are cached per URL.** WhatsApp and others can hold a preview for days. To force fresh numbers, share the link with a changing query string, e.g. `https://<user>.github.io/sg-haze/?t=0908`.
- **Scheduled runs are approximate.** GitHub can start them 10–30 minutes late, which is fine for hourly data.
- **Schedules can switch off.** In a public repo, GitHub disables scheduled workflows after 60 days without repository activity. Deploys don't count, so push a commit occasionally or re-enable the workflow from the Actions tab.
- **Thresholds live in two places.** The 🔴 threshold and band logic are in both `index.html` and `update.yml`; change them together.

## Local testing

Open `index.html` in a browser to see the live table.

To preview the baked tags (needs `curl`, `jq` and `perl`):

```sh
summary=$(curl -sf "https://www.haze.gov.sg/api/airquality/jsondata/$(date +%s)" | jq -r '
  .Chart1HRPM25 as $pm
  | "PM2.5 \(["North", "South", "East", "West", "Central"]
      | map(($pm[.].Data[-1].value // null | if . then round else . end) as $v
            | "\(.[:1] | ascii_downcase)\($v // "?")\(if ($v // 0) >= 56 then "🔴" else "" end)")
      | join(" ")) µg/m³"')
TITLE=$summary perl -ne 'print if s/(<meta property="og:title" content=")[^"]*/$1$ENV{TITLE}/' index.html
```
