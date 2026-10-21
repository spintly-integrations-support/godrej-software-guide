# IOT9 Consumer App Guide

Change notes for the IoT9 B2C integration, following the structure of Binary Veda's own
documentation site page by page.

Published by GitHub Actions on every push to `main`:
https://spintly-integrations-support.github.io/godrej-software-guide/

## Working on it locally

    pip install -r requirements.txt
    mkdocs serve          # http://127.0.0.1:8000, reloads on save
    mkdocs build --strict # what CI runs

Pages are in `docs/`, navigation is in `mkdocs.yml`. Each flow step heading carries a
badge (`new` / `changed` / `unchanged`) and an explicit anchor, for example
`## 4. Customise the lock <span class="chg chg-changed">changed</span> { #4-customise-the-lock data-toc-label="4. Customise the lock" }`.

`docs/stylesheets/extra.css` and `docs/javascripts/diagrams.js` are adapted from Binary
Veda's site so that participants keep the same colours in the sequence diagrams.

When an open item is settled, update `docs/pending.md` and the page it points to.
