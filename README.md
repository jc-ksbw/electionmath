# electionmath

A phone-first, iframe-ready results page for **local, single-seat** Monterey
County contests and ballot measures: how each race stands, and — for any race
still undecided — how many of the outstanding ballots the trailing candidate
would still need.

- **Live:** <https://jc-ksbw.github.io/electionmath/>
- **Embed demo:** <https://jc-ksbw.github.io/electionmath/embed.html>

What is in here:

| file | what it is |
|---|---|
| `index.html` | the built page, and what GitHub Pages serves at the site root. One self-contained file — no dependencies, no network calls, no build step to view it. |
| `build/template.html` | the page's markup, CSS and JavaScript, with a `/*__DATA__*/` placeholder where results get injected. Edit the design here, never in `index.html`. |
| `build/build.py` | scrapes a saved copy of the county results page and rewrites `index.html`. |
| `build/source/` | the source the current `index.html` was built from — a saved county page, or a reduced JSON of an XML report. |

Anything under `build/` is tooling; only `index.html` and `embed.html` are
meant to be served.

---

## Updating the numbers on election night

The county has no JSON feed — results are rendered straight into the HTML of
[the Registrar's results page][src]. So:

1. Open that page in a browser.
2. **File → Save Page As…** → save over `build/source/results.html`
   ("Web Page, HTML Only" is enough and much smaller).
3. Run the build, passing the number of ballots the Registrar says are still to
   be processed:

```bash
python3 build/build.py --left 38728
```

It prints the report number it found, the outstanding-ballot figure, and how
many contests it kept and dropped, then rewrites `index.html`. Commit and push;
GitHub Pages redeploys within a minute or so.

Leave `--left` off and the page says the outstanding count has not been
published, showing each margin but no catch-up maths. To build from a file
somewhere else, or to a different output path:

```bash
python3 build/build.py ~/Downloads/results.html --left 38728 -o /tmp/index.html
```

`build.py` needs only the Python standard library — no pip install, no Node.

### Two source formats

The county publishes each report two ways, and `build.py` takes either:

- **The results page** (`.html`) — saved from the browser, scraped directly.
- **An SSRS XML export** (`home/showdocument?id=...`), whose shape is nothing
  like the page. Rather than carry a second scraper, reduce that export once to
  the compact JSON in `build/source/report08.json` — per contest the official
  name, seats, precincts, ballots cast and registration for the jurisdiction,
  the vote-by-mail / polling-place split, and each candidate's votes — and pass
  the `.json` path instead. `build.py` picks the loader from the extension.

The XML carries no publish timestamp, so a build from JSON omits the "posted"
line in the page's dateline rather than invent one.

A useful check on a hand-reduced JSON: in every contest the vote-by-mail and
polling-place ballots must sum exactly to that jurisdiction's ballots cast. If
all of them reconcile, the transcription is almost certainly clean.

**If the build stops with an error**, the county has changed its page markup —
it already renamed a CSS class and added inline styles once mid-election. The
script refuses to publish a page it has parsed badly rather than emit an empty
or wrong one; the three regexes to check are at the top of `scrape()` in `build/build.py`.

[src]: https://www.countyofmonterey.gov/government/departments-a-h/elections/election-results

---

## Embedding

`index.html` is designed to sit in an iframe on a story page. It has no fixed
heights, no sticky positioning and no internal scroll area, so it can be sized
to its content and the reader never gets a scrollbar inside a scrollbar.

Paste this where the embed should appear:

```html
<iframe id="ksbw-monterey-election"
        src="https://jc-ksbw.github.io/electionmath/"
        title="Monterey County local election results"
        scrolling="no" loading="lazy"
        style="width:100%;border:0;height:1400px"></iframe>
<script>
(function(){
  var frame = document.getElementById('ksbw-monterey-election');
  window.addEventListener('message', function(e){
    if (!e.data || e.data.type !== 'ksbw:embed:height') return;
    if (e.source !== frame.contentWindow) return;
    frame.style.height = e.data.height + 'px';
  });
  frame.addEventListener('load', function(){
    frame.contentWindow.postMessage('ksbw:embed:ping', '*');
  });
})();
</script>
```

The page posts `{type:'ksbw:embed:height', slug:'monterey-election', height:N}`
to its parent whenever its height changes, and re-posts on request when it
receives `'ksbw:embed:ping'`. If the CMS strips the script, drop it and give the
iframe a fixed height plus `scrolling="yes"` — the page still works.

`embed.html` is a working demo of the snippet above, served at
<https://jc-ksbw.github.io/electionmath/embed.html>.

### URL options

Append these to the iframe `src` to pre-set the page — handy when the same
tracker is embedded in several stories:

| parameter | effect |
|---|---|
| `?compact=1` | hides the headline and standfirst (use when the story already carries them) |
| `?group=cities` | opens on one filter: `cities`, `schools`, `measures`, `special`, `federal`, `live`, `all` |
| `?q=salinas` | pre-fills the search box |
| `?sort=az` | `close` (default), `az`, `size` |
| `?left=20000` | overrides the outstanding-ballot count baked in at build time — useful if the Registrar updates the figure between builds |

Example: `https://jc-ksbw.github.io/electionmath/?compact=1&group=cities&q=seaside`

---

## Editorial rules the page applies

**Local races only.** Statewide offices, statewide judicial retention and
statewide propositions are dropped. Everything else is kept: county, city,
school, community college and special-district contests, local measures, and
the congressional and Assembly districts that sit inside the county. The rules
live in the `STATEWIDE` list at the top of `build/build.py` — delete a line there to
put a contest back.

**Single-seat contests only.** A contest that elects two or more people *and*
has more candidates than seats is dropped (`is_multiseat_contest` in
`build/build.py`) — 13 of them in Report 07, including the Seaside, Carmel and Sand
City councils and the five-seat King City and Greenfield school boards.

The reason is that the page's central number does not survive the move to
multi-seat. In a vote-for-N contest each voter casts up to N votes, so one
ballot can carry a vote for *both* the trailing candidate and the one they
have to pass. The share-of-remaining-ballots figure assumes a gain for one is
a loss for the other, which is false there, and its denominator is wrong as
well: candidates draw on up to N votes per ballot, not one. Single-seat
contests and yes/no measures are one vote per ballot, where the arithmetic
holds.

The page enforces this a second time, independently of the build: `analyze()`
in `build/template.html` refuses to compute the catch-up maths for any contest with
more than one seat and renders a plain explanation in place of the figures. So
loosening the build filter cannot put a wrong number back on the page.

**Uncontested races are called.** If a contest has no more candidates than
seats, every candidate is marked elected and no catch-up maths is shown. One
candidate for one seat and four candidates for four seats are both handled.
These are kept even when multi-seat, since they carry no arithmetic — only the
call.

**One scenario per contested race.** The page shows the maths for exactly one
candidate: second place, against the leader. Where three or four candidates
contest a single seat, the figure assumes the remaining ballots split between
those top two — a simplification the card states, not a unit error, because
each ballot still carries exactly one vote.

**Statuses** are assigned from that one calculation:

| status | meaning |
|---|---|
| Elected | unopposed |
| Multi-seat | a contest the maths does not apply to (never reached in a normal build) |
| Too close to call | the trailing candidate needs ≤ 60% of the ballots left |
| Leading | they need more than 60%, but under 100% |
| Safe lead | they would need more than every ballot left |

---

## How the maths works

The countywide outstanding-ballot total comes from the Registrar and is passed
in with `--left`. Everything else follows from figures each contest publishes:

```
ballots counted here      = registration × reported turnout
ballots still out here    = ballots still out countywide × (ballots counted here ÷ ballots counted countywide)
votes needed to pass      = (leader) − (second place) + 1
share of the ballots left = (ballots still out + votes needed) ÷ (2 × ballots still out)
```

That last line is the head-to-head share: to make up a gap of *g* out of *r*
remaining ballots, a candidate has to win those ballots by *g*, which means
taking *(r+g)/2* of them.

The countywide figure is known. The split between jurisdictions is not — it
assumes outstanding ballots are spread like the ballots already counted, which
is an assumption, not a fact, and the page says so.

Vote shares use the sum of every candidate's votes, not the county's "Total
Votes" field, which leaves write-in candidates out. The two are identical in
every contest without a write-in; where there is one, the county's own
percentages sum past 100%.

Other caveats stated on the page itself: results are unofficial until certified,
not every ballot carries a vote in every contest, and some measures need a 55%
or two-thirds majority that this page does not attempt to judge.

---

## Hosting

GitHub Pages serves this repo straight from `main` — there is no build step on
the server, because `index.html` is already a complete page. To turn it on:

**Settings → Pages → Build and deployment → Source: Deploy from a branch →
Branch: `main` / `(root)` → Save.**

The site appears at <https://jc-ksbw.github.io/electionmath/> within a minute
or two. After that, every push to `main` redeploys it.

`.nojekyll` is there to skip Jekyll processing, which this page does not need
and which would otherwise ignore any file or folder beginning with an
underscore.
