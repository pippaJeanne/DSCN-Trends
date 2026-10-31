---
title: DSCN Scope Atlas
---

# DSCN Scope Atlas

<p class="lede">An article-level map of <em>Digital Studies / Le champ numérique</em>: what it studies, how it works, what it works on, what it contributes, and where its evidence is situated.</p>

<div class="note"><strong>How to read this atlas.</strong> Articles may carry more than one tag in a rubric, so tag totals are not mutually exclusive. Every bar can be checked against its linked article list. “Unclear” and “context not specified” remain visible rather than being treated as substantive findings.</div>

```js
import * as Plot from "npm:@observablehq/plot";
import * as Inputs from "npm:@observablehq/inputs";
import {html} from "npm:htl";
const articles = await FileAttachment("data/articles.csv").csv({typed: true});
const rubrics = {
  subjects: "subject_tags",
  methodology: "method_tags",
  material: "material_tags",
  contribution: "contribution_tags",
  context: "context_tags"
};
const splitTags = (value) => (value || "").split(";").map(d => d.trim()).filter(Boolean);
const countBy = (values, key) => Array.from(values.reduce((counts, value) => {
  const k = key(value); counts.set(k, (counts.get(k) || 0) + 1); return counts;
}, new Map()), ([key, count]) => ({key, count}));
const languageOf = (d) => /[àâçéèêëîïôûùüÿœ]|\b(le|la|les|des|une|dans|pour|avec|sur|et)\b/i.test(`${d.title}`) ? "French" : "English";
const enriched = articles.map(d => ({...d, year: +d.year, language: languageOf(d), special: d.special_issue === "Yes"}));
const period = (year) => year <= 2005 ? "1996–2005" : year <= 2015 ? "2006–2015" : year <= 2020 ? "2016–2020" : "2021–2026";
```

## Corpus at a glance

```js
const overview = [
  ["articles", enriched.length],
  ["years covered", `${Math.min(...enriched.map(d => d.year))}–${Math.max(...enriched.map(d => d.year))}`],
  ["special-issue articles", enriched.filter(d => d.special).length],
  ["tag assignments", Object.values(rubrics).reduce((n, col) => n + enriched.reduce((a, d) => a + splitTags(d[col]).length, 0), 0)]
];
display(html`<div class="card">${overview.map(([label, value]) => html`<span style="display:inline-block;min-width:10rem;margin:.25rem 1rem .25rem 0"><strong>${value}</strong><br><small>${label}</small></span>`)}</div>`)
```

## Five-rubric overview: clusters over time

Each heatmap is a distinct overview graph for one rubric. Rows are tag clusters; columns are publication periods; colour records the number of articles carrying that tag per 100 articles in the period. Tags are ordered by overall frequency and the twelve most recurrent clusters are shown, so the pattern remains readable before drilling into article-level evidence.

```js
const periodOrder = ["1996–2005", "2006–2015", "2016–2020", "2021–2026"];
const periodSizes = new Map(periodOrder.map(name => [name, enriched.filter(d => period(d.year) === name).length]));
const rubricOverview = Object.fromEntries(Object.entries(rubrics).map(([name, column]) => {
  const assignments = enriched.flatMap(d => splitTags(d[column]).map(tag => ({tag, period: period(d.year)})));
  const totals = new Map(countBy(assignments, d => d.tag).map(d => [d.key, d.count]));
  const tags = Array.from(totals, ([tag, count]) => ({tag, count})).sort((a, b) => b.count - a.count || a.tag.localeCompare(b.tag)).slice(0, 12).map(d => d.tag);
  const cells = tags.flatMap(tag => periodOrder.map(window => ({tag, period: window, count: assignments.filter(d => d.tag === tag && d.period === window).length, rate: 100 * assignments.filter(d => d.tag === tag && d.period === window).length / periodSizes.get(window)})));
  return [name, {tags, cells}];
}));
const rubricTitles = {subjects: "Subjects", methodology: "Methodology", material: "Material", contribution: "Contribution", context: "Context"};
```

```js
for (const name of Object.keys(rubrics)) {
  const {tags, cells} = rubricOverview[name];
  display(html`<div class="card"><strong>${rubricTitles[name]}</strong><br><small>Top twelve tag clusters, normalised to the number of articles published in each period.</small></div>`);
  display(Plot.plot({
    marginLeft: 190, height: Math.max(260, tags.length * 25 + 105),
    x: {domain: periodOrder, label: "Publication period"}, y: {domain: tags, label: null},
    color: {type: "linear", range: ["#e4f3ef", "#0f766e"], legend: true, label: "Articles per 100"},
    marks: [Plot.cell(cells, {x: "period", y: "tag", fill: "rate", inset: 1, tip: true}), Plot.text(cells, {x: "period", y: "tag", text: d => d.count || "", fill: d => d.rate > 35 ? "white" : "#153b4b"})]
  }));
}
```

## Explore tag clusters and article evidence

After comparing the five overview charts, choose a rubric and tag to inspect its full distribution and the complete cluster of linked articles.

```js
const rubric = view(Inputs.radio(Object.keys(rubrics), {label: "Rubric", format: name => name[0].toUpperCase() + name.slice(1), value: "subjects"}));
```

```js
const selectedRubric = rubrics[rubric];
const tagRows = enriched.flatMap(d => splitTags(d[selectedRubric]).map(tag => ({tag, title: d.title, url: d.url, year: d.year, language: d.language, special: d.special})));
const tagCounts = countBy(tagRows, d => d.tag).map(d => ({tag: d.key, count: d.count})).sort((a,b) => b.count-a.count || a.tag.localeCompare(b.tag));
const tag = view(Inputs.select(tagCounts.map(d => d.tag), {label: "Tag cluster", value: tagCounts[0]?.tag}));
```

```js
display(Plot.plot({
  marginLeft: 185, height: Math.max(340, tagCounts.length * 24),
  x: {label: "Articles (articles may appear in several clusters)", grid: true},
  y: {label: null},
  marks: [Plot.barX(tagCounts, {x: "count", y: "tag", sort: {y: "-x"}, fill: "#0f766e"}), Plot.text(tagCounts, {x: "count", y: "tag", text: "count", dx: 8, textAnchor: "start"})]
}));
```

```js
const cluster = tagRows.filter(d => d.tag === tag).sort((a,b) => b.year-a.year || a.title.localeCompare(b.title));
display(html`<div class="card"><strong>${cluster.length} articles in “${tag}”</strong><table><thead><tr><th>Year</th><th>Article</th><th>Language</th><th>Special issue</th></tr></thead><tbody>${cluster.map(d => html`<tr><td>${d.year}</td><td><a href=${d.url} target="_blank" rel="noopener">${d.title}</a></td><td>${d.language}</td><td>${d.special ? "Yes" : "No"}</td></tr>`)}</tbody></table></div>`);
```

## Trends through time and language

This view counts articles by year and tracks language as a transparent title/abstract-based classification. It is intended as a descriptive aid, especially where bilingual abstracts are present.

```js
const years = countBy(enriched, d => `${d.year}|${d.language}`).map(d => { const [year, language] = d.key.split("|"); return {year: +year, language, count: d.count}; });
display(Plot.plot({height: 360, x: {label: "Publication year", tickFormat: d => d}, y: {label: "Articles", grid: true}, color: {legend: true}, marks: [Plot.rectY(years, {x: "year", y: "count", fill: "language", interval: 1, tip: true}), Plot.ruleY([0])]}));
```

```js
const trendRubric = view(Inputs.select(Object.keys(rubrics), {label: "Trend rubric", value: "subjects"}));
const trendTags = enriched.flatMap(d => splitTags(d[rubrics[trendRubric]]).map(tag => ({year: d.year, tag, period: period(d.year)})));
const topTrendTags = countBy(trendTags, d => d.tag).sort((a, b) => b.count - a.count).slice(0, 8).map(d => d.key);
display(Plot.plot({height: 420, x: {label: "Year", tickFormat: d => d}, y: {label: "Articles carrying tag", grid: true}, color: {legend: true}, marks: [Plot.lineY(trendTags.filter(d => topTrendTags.includes(d.tag)), {x: "year", y: 1, z: "tag", stroke: "tag", curve: "step", tip: true}), Plot.dot(trendTags.filter(d => topTrendTags.includes(d.tag)), {x: "year", y: 1, stroke: "tag"})]}));
```

## Special issues: within-issue and period comparison

Special issues are first treated as independent clusters. The comparison table contrasts each special issue’s tag frequency with the tag frequency for its publication period across the whole corpus. A positive percentage-point difference signals a tag unusually concentrated in that issue; it does not establish a general editorial trend.

```js
const issues = Array.from(new Set(enriched.filter(d => d.special).map(d => d.issue))).sort();
const selectedIssue = view(Inputs.select(issues, {label: "Special issue / collection"}));
const issueArticles = enriched.filter(d => d.issue === selectedIssue);
const issuePeriod = period(issueArticles[0]?.year);
const peerArticles = enriched.filter(d => period(d.year) === issuePeriod);
const issueTagRows = issueArticles.flatMap(d => splitTags(d[selectedRubric]).map(tag => ({tag})));
const peerTagRows = peerArticles.flatMap(d => splitTags(d[selectedRubric]).map(tag => ({tag})));
const counts = (rows) => new Map(countBy(rows, d => d.tag).map(d => [d.key, d.count]));
const iCounts = counts(issueTagRows), pCounts = counts(peerTagRows);
const issueComparison = Array.from(new Set([...iCounts.keys(), ...pCounts.keys()]), tag => ({tag, issue: iCounts.get(tag) || 0, issueRate: (iCounts.get(tag) || 0) / issueArticles.length, periodRate: (pCounts.get(tag) || 0) / peerArticles.length})).sort((a,b) => (b.issueRate-b.periodRate) || b.issue-a.issue);
```

```js
display(html`<div class="card"><strong>${issueArticles.length} articles · publication period ${issuePeriod}</strong><table><thead><tr><th>Tag (${rubric})</th><th>Issue</th><th>Period rate</th><th>Difference</th></tr></thead><tbody>${issueComparison.map(d => html`<tr><td>${d.tag}</td><td>${d.issue}</td><td>${d.periodRate.toLocaleString(undefined,{style:"percent",maximumFractionDigits:0})}</td><td>${(d.issueRate-d.periodRate).toLocaleString(undefined,{style:"percent",signDisplay:"always",maximumFractionDigits:0})}</td></tr>`)}</tbody></table></div>`);
display(html`<div class="card"><strong>Article access — read the issue cluster</strong><ul>${issueArticles.sort((a,b) => a.title.localeCompare(b.title)).map(d => html`<li><a href=${d.url} target="_blank" rel="noopener">${d.title}</a> (${d.year})</li>`)}</ul></div>`);
```

## Outliers and boundary cases

Outliers are operationalised here as articles whose selected-rubric tag occurs only once in the corpus, plus records marked <em>unclear</em> or <em>context not specified</em>. These are not errors: they are the records most worth reading when testing the observed scope.

```js
const globalCounts = new Map(tagCounts.map(d => [d.tag, d.count]));
const outliers = enriched.filter(d => {
  const tags = splitTags(d[selectedRubric]);
  return tags.some(t => globalCounts.get(t) === 1 || /unclear|not specified/i.test(t));
});
display(html`<div class="card"><strong>${outliers.length} boundary/outlier articles for ${rubric}</strong><table><thead><tr><th>Year</th><th>Article</th><th>Tags triggering review</th></tr></thead><tbody>${outliers.sort((a,b) => b.year-a.year).map(d => html`<tr><td>${d.year}</td><td><a href=${d.url} target="_blank" rel="noopener">${d.title}</a></td><td>${splitTags(d[selectedRubric]).filter(t => globalCounts.get(t) === 1 || /unclear|not specified/i.test(t)).join("; ")}</td></tr>`)}</tbody></table></div>`);
```

<div class="note">The article links deliberately point to the DOI landing pages supplied by the corpus, so each charted claim can be checked against the published item.</div>
