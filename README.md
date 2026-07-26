# Horse Race Analyzer

A web rebuild of the 1979 Mattel Electronics Horse Race Analyzer ("Handicapping Computer"), the handheld designed by Dr. William L. Quirin. Upload a Daily Racing Form past-performance PDF (or photos), and the app scans the card, shows which races qualify under the original 1979 rules (thoroughbreds 3+, no maidens, 6 furlongs or more), extracts the race you pick using Claude's vision API, and rates each horse with a Quirin-inspired formula: early speed weighted heaviest, plus recent form, Beyer speed figures, class, consistency, freshness, and post position.

This is an analytical toy in the spirit of the original device, not betting advice. As the 1979 manual put it: some unknowns just can't be predicted.

## Running it on GitHub Pages

1. Create a new GitHub repository (public or private) and add `index.html` and this `README.md` to it.
2. In the repo, go to **Settings → Pages**, set Source to **Deploy from a branch**, choose your default branch and the `/ (root)` folder, and save.
3. After a minute, your app is live at `https://<your-username>.github.io/<repo-name>/`.

No build step, no dependencies to install — it's a single HTML file that loads React from a CDN.

## API key

The app calls the Anthropic API directly from your browser to read racing forms, so it needs your own API key:

1. Create a key at [console.anthropic.com](https://console.anthropic.com) (Settings → API Keys).
2. Paste it into the field on the app's start screen.

The key is stored only in your browser's localStorage on your own site and is sent only to `api.anthropic.com`. Never commit the key to the repository. Each card scan plus race extraction costs a few cents (a full DRF card PDF is a large input). Consider setting a monthly spend limit on your key in the console.

## Usage

1. Download a Classic PPs PDF from [shop.drf.com](https://shop.drf.com) (single cards ~$4.95, includes Beyer Speed Figures), or use the free daily race at [drf.com/race-of-the-day](https://www.drf.com/race-of-the-day).
2. Upload the PDF (or straight-on photos of the pages). The app lists every race; ones that don't meet the 1979 rules are greyed out with the reason.
3. Tap a qualifying race. Review the extracted numbers — vision extraction of dense PP tables is good but not perfect, so check the lengths-behind values especially — then hit **Analyze**.
4. Tap any horse in the results to see its component score breakdown. Keep the speed-figure toggle on **Beyer (modern)** for current DRF cards.

Manual entry mode is also available if you'd rather punch the numbers in yourself, just like 1979.

## Known limitations

- The formula is Quirin-inspired, reconstructed from his published research (*Winning at the Races*, 1979) — the exact ROM of the original device was never published.
- Like the original, it doesn't account for surface switches (dirt/turf), trainer/jockey stats, or workouts.
- Very large multi-track PDFs may exceed the API's document limits; upload one track's card at a time.
