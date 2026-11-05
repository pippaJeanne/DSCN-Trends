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
const outlierAnalysis = await FileAttachment("data/outlier-analysis/outlier_articles_analysis.csv").csv({typed: true});
const outlierNeighbours = await FileAttachment("data/outlier-analysis/outlier_nearest_neighbours.csv").csv({typed: true});
const outlierMetadata = await FileAttachment("data/outlier-analysis/analysis_metadata.json").json();
const rubrics = {
  subjects: "subject_tags",
  methodology: "method_tags",
  material: "material_tags",
  contribution: "contribution_tags",
  context: "context_tags"
};
const trendRubrics = {...rubrics, articleType: "article_type"};
const splitTags = (value) => (value || "").split(";").map(d => d.trim()).filter(Boolean);
const valuesFor = (article, field) => field === "article_type" ? [article[field]].filter(Boolean) : splitTags(article[field]);
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
const trendRubric = view(Inputs.select(Object.keys(trendRubrics), {
  label: "Trend rubric",
  format: name => name === "articleType" ? "Article type" : rubricTitles[name],
  value: "subjects"
}));
```

```js
const trendField = trendRubrics[trendRubric];
const trendTags = enriched.flatMap(d => valuesFor(d, trendField).map(tag => ({year: d.year, tag, period: period(d.year)})));
const topTrendTags = countBy(trendTags, d => d.tag).sort((a, b) => b.count - a.count || a.key.localeCompare(b.key)).slice(0, 8).map(d => d.key);
const trendSeries = countBy(trendTags.filter(d => topTrendTags.includes(d.tag)), d => `${d.year}|${d.tag}`).map(d => {
  const [year, tag] = d.key.split("|");
  return {year: +year, tag, count: d.count};
});
display(Plot.plot({height: 420, x: {label: "Year", tickFormat: d => d}, y: {label: trendRubric === "articleType" ? "Articles" : "Articles carrying tag", grid: true}, color: {legend: true}, marks: [Plot.lineY(trendSeries, {x: "year", y: "count", z: "tag", stroke: "tag", curve: "step", tip: true}), Plot.dot(trendSeries, {x: "year", y: "count", fill: "tag", r: 3, tip: true})]}));
```

## Special issues: within-issue and period comparison

Special issues are first treated as independent clusters. The comparison table contrasts each special issue’s tag frequency with the tag frequency for its publication period across the whole corpus. A positive percentage-point difference signals a tag unusually concentrated in that issue; it does not establish a general editorial trend.

```js
const issues = Array.from(new Set(enriched.filter(d => d.special).map(d => d.issue))).sort();
const selectedIssue = view(Inputs.select(issues, {label: "Special issue / collection", value: issues[0]}));
```

```js
const specialIssueRubric = view(Inputs.select(Object.keys(trendRubrics), {
  label: "Comparison rubric",
  format: name => name === "articleType" ? "Article type" : rubricTitles[name],
  value: "subjects"
}));
```

```js
const specialIssueField = trendRubrics[specialIssueRubric];
const issueArticles = enriched.filter(d => d.issue === selectedIssue);
const issuePeriod = period(issueArticles[0]?.year);
const peerArticles = enriched.filter(d => period(d.year) === issuePeriod && d.issue !== selectedIssue);
const issueTagRows = issueArticles.flatMap(d => valuesFor(d, specialIssueField).map(tag => ({tag})));
const peerTagRows = peerArticles.flatMap(d => valuesFor(d, specialIssueField).map(tag => ({tag})));
const counts = (rows) => new Map(countBy(rows, d => d.tag).map(d => [d.key, d.count]));
const iCounts = counts(issueTagRows), pCounts = counts(peerTagRows);
const issueComparison = Array.from(new Set([...iCounts.keys(), ...pCounts.keys()]), tag => ({tag, issue: iCounts.get(tag) || 0, issueRate: (iCounts.get(tag) || 0) / issueArticles.length, periodRate: peerArticles.length ? (pCounts.get(tag) || 0) / peerArticles.length : 0})).sort((a,b) => (b.issueRate-b.periodRate) || b.issue-a.issue);
```

```js
display(html`<div class="card"><strong>${issueArticles.length} articles · publication period ${issuePeriod}</strong><table><thead><tr><th>Tag (${rubricTitles[specialIssueRubric]})</th><th>Issue</th><th>Period rate</th><th>Difference</th></tr></thead><tbody>${issueComparison.map(d => html`<tr><td>${d.tag}</td><td>${d.issue}</td><td>${d.periodRate.toLocaleString(undefined,{style:"percent",maximumFractionDigits:0})}</td><td>${(d.issueRate-d.periodRate).toLocaleString(undefined,{style:"percent",signDisplay:"always",maximumFractionDigits:0})}</td></tr>`)}</tbody></table></div>`);
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

## Outlier similarity analysis

This analysis is limited to the 26 recent boundary/outlier articles identified in the review list. It asks whether they nevertheless form local patterns: the subject view uses the complete controlled <code>subject_tags</code> assigned to each article, while the abstract view uses bilingual word-and-bigram TF-IDF. PCA and UMAP are alternative two-dimensional views of the same underlying representation; clusters and nearest neighbours are calculated from the original feature space, not from the displayed coordinates.

```js
const outlierMode = view(Inputs.select(new Map([
  ["Subjects / themes", "subject"],
  ["Abstracts (TF-IDF)", "abstract"]
]), {label: "Similarity evidence", value: "subject"}));
const outlierProjection = view(Inputs.select(new Map([
  ["UMAP", "umap"],
  ["PCA", "pca"]
]), {label: "Map", value: "umap"}));
// Explicit primitive values ensure a changed control always selects a new data field.
const outlierFieldPrefix = outlierMode === "abstract" ? "abstract" : "subject";
const outlierMetadataKey = outlierMode === "abstract" ? "abstract_tfidf" : "subjects";
const outlierNeighbourRepresentation = outlierMode === "abstract" ? "abstract_tfidf" : "subjects";
const outlierPoints = outlierAnalysis.map(d => ({
  ...d,
  x: +d[`${outlierFieldPrefix}_${outlierProjection}_x`],
  y: +d[`${outlierFieldPrefix}_${outlierProjection}_y`],
  cluster: `Cluster ${d[`${outlierFieldPrefix}_cluster`]}`
}));
const outlierClusters = Array.from(new Set(outlierPoints.map(d => d.cluster))).sort((a, b) => +a.slice(8) - +b.slice(8));
const outlierClusterFilter = view(Inputs.select(["All clusters", ...outlierClusters], {label: "Show cluster", value: "All clusters"}));
const visibleOutlierPoints = outlierClusterFilter === "All clusters" ? outlierPoints : outlierPoints.filter(d => d.cluster === outlierClusterFilter);
const outlierDimensionSettings = outlierMetadata.dimensionality_reduction[outlierMetadataKey];
const outlierPcaVariance = outlierDimensionSettings.pca.explained_variance_ratio.reduce((total, value) => total + value, 0);
display(html`<div class="card"><strong>${outlierPoints.length} explicitly listed outliers</strong><br><small>${outlierMode === "abstract" ? "Abstracts (TF-IDF)" : "Subjects / themes"}: ${outlierMetadata.representations[outlierMetadataKey].feature_count} features · ${outlierMetadata.clustering[outlierMetadataKey].selected_k} clusters selected by cosine silhouette score · PCA axes explain ${(outlierPcaVariance * 100).toFixed(1)}% together.</small></div>`);
```

```js
display(Plot.plot({
  height: 520, marginLeft: 50, marginBottom: 45,
  x: {label: `${outlierProjection.toUpperCase()} dimension 1`, grid: true},
  y: {label: `${outlierProjection.toUpperCase()} dimension 2`, grid: true},
  color: {legend: true, label: "Cluster"},
  marks: [
    Plot.ruleX([0], {stroke: "#cbd5e1"}),
    Plot.ruleY([0], {stroke: "#cbd5e1"}),
    Plot.dot(visibleOutlierPoints, {
      x: "x", y: "y", fill: "cluster", r: 7, stroke: "white", strokeWidth: 1.5, tip: true,
      title: d => `${d.title}\n${d.year} · ${d.cluster}\nReview tags: ${d.review_tags}`
    })
  ]
}));
```

```js
const outlierClusterCounts = countBy(outlierPoints, d => d.cluster)
  .map(d => ({cluster: d.key, articles: d.count}))
  .sort((a, b) => +a.cluster.slice(8) - +b.cluster.slice(8));
display(Plot.plot({
  height: 250, marginLeft: 100,
  x: {label: "Outlier articles", grid: true}, y: {label: null}, color: {legend: false},
  marks: [
    Plot.barX(outlierClusterCounts, {x: "articles", y: "cluster", fill: "cluster", tip: true}),
    Plot.text(outlierClusterCounts, {x: "articles", y: "cluster", text: "articles", dx: 8, textAnchor: "start"})
  ]
}));
```

```js
const visibleOutlierArticles = visibleOutlierPoints.sort((a, b) => +b.year - +a.year || a.title.localeCompare(b.title));
display(html`<div class="card"><strong>${outlierClusterFilter === "All clusters" ? "Articles shown in all clusters" : `Articles in ${outlierClusterFilter}`}</strong><table><thead><tr><th>Cluster</th><th>Year</th><th>Article</th><th>Review-trigger tags</th></tr></thead><tbody>${visibleOutlierArticles.map(d => html`<tr><td>${d.cluster}</td><td>${d.year}</td><td><a href=${d.url} target="_blank" rel="noopener">${d.title}</a></td><td>${d.review_tags}</td></tr>`)}</tbody></table></div>`);
```

## Inspect cosine neighbours

Select an article to see its closest other outliers in the current representation. The score is cosine similarity (1 = identical feature direction; 0 = no overlap). Read the linked articles before treating a proximity as a substantive relationship.

```js
const outlierArticleTitle = view(Inputs.select(outlierPoints.map(d => d.title).sort(), {
  label: "Outlier article", value: outlierPoints[0]?.title
}));
const selectedOutlierArticle = outlierPoints.find(d => d.title === outlierArticleTitle);
const selectedOutlierNeighbours = outlierNeighbours
  .filter(d => String(d.article_id) === String(selectedOutlierArticle?.id) && d.representation === outlierNeighbourRepresentation)
  .sort((a, b) => a.rank - b.rank);
display(html`<div class="card"><strong>${selectedOutlierArticle ? selectedOutlierArticle.title : "No article selected"}</strong><br><small>Review-trigger tags: ${selectedOutlierArticle?.review_tags || "—"}</small><table><thead><tr><th>Rank</th><th>Nearest outlier</th><th>Year</th><th>Cosine similarity</th></tr></thead><tbody>${selectedOutlierNeighbours.map(d => {
  const neighbour = outlierPoints.find(article => String(article.id) === String(d.neighbour_id));
  return html`<tr><td>${d.rank}</td><td>${neighbour?.url ? html`<a href=${neighbour.url} target="_blank" rel="noopener">${d.neighbour_title}</a>` : d.neighbour_title}</td><td>${d.neighbour_year}</td><td>${(+d.cosine_similarity).toFixed(3)}</td></tr>`;
})}</tbody></table></div>`);
```

<div class="note">Reproducibility: the exported source hashes, feature construction, PCA and UMAP settings, clustering candidates and silhouette scores, package versions, and execution settings are retained in <code>data/outlier-analysis/analysis_metadata.json</code>.</div>
