# sg-haze

A single-page site showing Singapore's latest PM2.5 readings by region, built so that link previews (WhatsApp, Teams, Slack, Telegram) show the numbers too.

```
PM2.5 n80🔴 s130🔴 e119🔴 w158🔴 c139🔴 ug/m3
```

Values are the latest 1-hr PM2.5 concentration (µg/m³) for north, south, east, west and central. 🔴 marks readings of 56 and above (NEA band II or worse).

Data sources, both published by NEA:

- **Link preview:** `Chart1HRPM25` from haze.gov.sg (`https://www.haze.gov.sg/api/airquality/jsondata/<timestamp>`), last item of each region's `Data[]`. The trailing number is only a cache-buster.
- **Page table:** data.gov.sg's [PSI real-time API](https://data.gov.sg/datasets/d_fe37906a0182569d891506e815e819b7/view) (`https://api-open.data.gov.sg/v2/real-time/api/psi`).

## How it works

Link-preview crawlers don't run JavaScript, so the preview text has to be in the HTML they download.

- **`index.html`** is the whole site. In a browser, its script fetches the API directly and renders a table of PM2.5 (with NEA band) and 24-hr PSI (with band) per region, refreshing every 5 minutes.
- **`.github/workflows/update.yml`** runs every 30 minutes, on every push to `main`, and on manual runs. It fetches haze.gov.sg, writes the summary line into the `<title>`, `og:*`, `twitter:*` and `description` tags of `index.html`, and deploys the result to GitHub Pages. Nothing is committed back to the repo.

data.gov.sg doesn't always publish `pm25_one_hourly`; when it's missing, the page table uses `pm25_twenty_four_hourly`, so it can differ from the preview.

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
      | join(" ")) ug/m3"')
TITLE=$summary perl -ne 'print if s/(<meta property="og:title" content=")[^"]*/$1$ENV{TITLE}/' index.html
```
