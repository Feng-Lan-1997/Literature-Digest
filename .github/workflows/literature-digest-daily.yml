import { mkdir, readFile, writeFile } from "node:fs/promises";
import path from "node:path";

const DEFAULTS = {
  lookbackDays: 90,
  maxItems: 3,
  stateDir: ".literature-digest-state",
  pushTime: "08:30",
  timeZone: "America/New_York",
  excludePushed: true,
  autoJournalMetrics: true,
  searchRouteName: "dental_implant_specific",
  requiredKeywordGroups:
    "surgery|surgical|operative|operation|procedure|intervention|intraoperative|perioperative|operating room|surgeon;agent|agents|agentic|AI agent|LLM agent|autonomous agent|multi-agent|multiagent|large language model|LLM",
  priorityJournals: "npj Digital Medicine,Nature"
};

const DEFAULT_SEARCH_ROUTES = {
  ai_method_first: ["arXiv", "Semantic Scholar", "Google Scholar", "IEEE Xplore", "ACM DL"],
  medical_validation_first: ["PubMed", "Embase", "Web of Science", "Scopus", "Cochrane Library"],
  dental_implant_specific: ["PubMed", "Google Scholar", "Scopus", "Web of Science", "ScienceDirect", "SpringerLink"],
  engineering_screw_planning: ["IEEE Xplore", "PubMed", "ScienceDirect", "SpringerLink", "Web of Science"]
};

async function main() {
  const config = readConfig();
  const state = await loadState(config.stateDir);
  const scheduleDecision = shouldRunNow(config, state);

  if (!scheduleDecision.shouldRun) {
    console.log(scheduleDecision.reason);
    await writeGitHubOutput({ did_push: "false", reason: scheduleDecision.reason });
    return;
  }

  console.log(`Searching literature for: ${config.searchQuery}`);
  const result = await fetchWebRecommendations(config, state.pushedHistory || []);
  const batch = {
    generatedAt: new Date().toISOString(),
    query: config.researchQuery,
    source: result.sourceLabel,
    items: result.items,
    sourceDetails: result.sourceDetails
  };

  await sendWeComDigest(config, batch);
  state.lastRunLocalDate = scheduleDecision.localDate;
  state.lastRunAt = new Date().toISOString();
  state.lastBatch = batch;
  state.pushedHistory = mergePushedHistory(state.pushedHistory || [], batch.items);
  await saveState(config.stateDir, state);
  await writeGitHubOutput({ did_push: "true", item_count: String(batch.items.length) });

  console.log(`Pushed ${batch.items.length} literature item(s) to WeCom.`);
}

function readConfig() {
  const researchQuery = cleanText(process.env.RESEARCH_QUERY);
  const weComWebhookUrl = cleanText(process.env.WECOM_WEBHOOK_URL);
  const requiredKeywordGroups = parseRequiredKeywordGroups(
    process.env.REQUIRED_KEYWORD_GROUPS || DEFAULTS.requiredKeywordGroups
  );
  const priorityJournalTerms = parsePriorityJournalTerms(process.env.PRIORITY_JOURNALS || DEFAULTS.priorityJournals);
  const searchRoutes = parseSearchRoutes(process.env.SEARCH_ROUTES);
  const searchRouteName = cleanText(process.env.SEARCH_ROUTE_NAME) || DEFAULTS.searchRouteName;
  const searchSources = resolveSearchSources(searchRoutes, searchRouteName);

  if (!researchQuery) {
    throw new Error("Missing RESEARCH_QUERY.");
  }
  if (!weComWebhookUrl) {
    throw new Error("Missing WECOM_WEBHOOK_URL.");
  }
  if (!isWeComWebhookUrl(weComWebhookUrl)) {
    throw new Error("WECOM_WEBHOOK_URL must be a qyapi.weixin.qq.com webhook URL.");
  }

  return {
    researchQuery,
    searchQuery: buildSearchQuery(researchQuery, requiredKeywordGroups),
    weComWebhookUrl,
    lookbackDays: numberFromEnv("LOOKBACK_DAYS", DEFAULTS.lookbackDays),
    maxItems: numberFromEnv("MAX_ITEMS", DEFAULTS.maxItems),
    stateDir: cleanText(process.env.STATE_DIR) || DEFAULTS.stateDir,
    pushTime: cleanText(process.env.PUSH_TIME) || DEFAULTS.pushTime,
    timeZone: cleanText(process.env.TIME_ZONE) || DEFAULTS.timeZone,
    excludePushed: process.env.EXCLUDE_PUSHED !== "false",
    autoJournalMetrics: process.env.AUTO_JOURNAL_METRICS !== "false",
    impactFactorTable: String(process.env.IMPACT_FACTOR_TABLE || "").trim(),
    forcePush: process.env.FORCE_PUSH === "true",
    openAlexApiKey: cleanText(process.env.OPENALEX_API_KEY),
    semanticScholarApiKey: cleanText(process.env.SEMANTIC_SCHOLAR_API_KEY),
    searchRoutes,
    searchRouteName,
    searchSources,
    requiredKeywordGroups,
    strictRequiredKeywords: process.env.STRICT_REQUIRED_KEYWORDS !== "false",
    priorityJournalTerms
  };
}

async function loadState(stateDir) {
  try {
    const statePath = path.join(stateDir, "state.json");
    return JSON.parse(await readFile(statePath, "utf8"));
  } catch {
    return {
      lastRunLocalDate: "",
      pushedHistory: []
    };
  }
}

async function saveState(stateDir, state) {
  await mkdir(stateDir, { recursive: true });
  await writeFile(path.join(stateDir, "state.json"), `${JSON.stringify(state, null, 2)}\n`, "utf8");
}

function shouldRunNow(config, state) {
  const local = localDateTimeParts(config.timeZone);
  if (config.forcePush) {
    return { shouldRun: true, localDate: local.date, reason: "Forced run." };
  }
  if (state.lastRunLocalDate === local.date) {
    return {
      shouldRun: false,
      localDate: local.date,
      reason: `Already pushed for ${local.date}.`
    };
  }
  const nowMinutes = local.hour * 60 + local.minute;
  const targetMinutes = timeToMinutes(config.pushTime);
  if (nowMinutes < targetMinutes) {
    return {
      shouldRun: false,
      localDate: local.date,
      reason: `Not time yet: ${local.time} < ${config.pushTime} (${config.timeZone}).`
    };
  }
  return { shouldRun: true, localDate: local.date, reason: "Scheduled run." };
}

async function fetchWebRecommendations(config, pushedHistory) {
  const tokens = rankingTokens(config);
  const pushedKeys = new Set(config.excludePushed ? pushedHistory.map((entry) => entry.key).filter(Boolean) : []);
  const manualImpactFactors = parseImpactFactorTable(config.impactFactorTable);
  const sourceResults = await Promise.all(config.searchSources.map((sourceName) => fetchRouteSource(sourceName, config, tokens)));

  const allItems = sourceResults.flatMap((result) => result.items);
  const merged = mergeRecommendations(allItems);
  const enriched = await enrichJournalMetrics(merged, manualImpactFactors, config);
  const candidates = enriched
    .map((item) => addRankingSignals(item, config))
    .filter((item) => !pushedKeys.has(recommendationKey(item)))
    .filter((item) => !config.strictRequiredKeywords || matchesRequiredKeywordGroups(item, config.requiredKeywordGroups));
  const items = candidates
    .sort(sortRecommendations)
    .slice(0, config.maxItems);

  const successfulSources = sourceResults.filter((result) => result.items.length).map((result) => result.label);
  const failedSources = sourceResults.filter((result) => result.error).map((result) => ({
    source: result.label,
    error: result.error
  }));
  const skippedSources = sourceResults.filter((result) => result.skipped).map((result) => ({
    source: result.label,
    reason: result.reason
  }));

  return {
    items,
    sourceLabel: successfulSources.length ? `全网学术搜索：${successfulSources.join(", ")}` : "全网学术搜索",
    sourceDetails: {
      searchRouteName: config.searchRouteName,
      requestedSources: config.searchSources,
      successfulSources,
      skippedSources,
      failedSources
    },
    sourceLabel: successfulSources.length ? `Academic web search: ${successfulSources.join(", ")}` : "Academic web search"
  };
}

async function fetchRouteSource(sourceName, config, tokens) {
  const label = cleanText(sourceName);
  const normalized = normalizeRouteSourceName(label);
  switch (normalized) {
    case "arxiv":
      return fetchSource("arXiv", () => fetchArxivRecommendations(config, tokens));
    case "semantic scholar":
      return fetchSource("Semantic Scholar", () => fetchSemanticScholarRecommendations(config, tokens));
    case "pubmed":
      return fetchSource("PubMed", () => fetchPubMedRecommendations(config, tokens));
    case "crossref":
      return fetchSource("Crossref", () => fetchCrossrefRecommendations(config, tokens));
    case "openalex":
      return fetchSource("OpenAlex", () => fetchOpenAlexRecommendations(config, tokens));
    case "europe pmc":
    case "europepmc":
      return fetchSource("Europe PMC", () => fetchEuropePmcRecommendations(config, tokens));
    default:
      return skipSource(label, "No public GitHub Actions friendly API is configured for this database.");
  }
}

async function fetchSource(label, task) {
  try {
    const items = await task();
    return { label, items };
  } catch (error) {
    console.warn(`${label} failed: ${error.message || error}`);
    return { label, items: [], error: error.message || String(error) };
  }
}

function skipSource(label, reason) {
  console.warn(`${label} skipped: ${reason}`);
  return { label, items: [], skipped: true, reason };
}

async function fetchCrossrefRecommendations(config, tokens) {
  const fromDate = formatDate(daysAgo(config.lookbackDays));
  const params = new URLSearchParams({
    "query.bibliographic": config.searchQuery,
    filter: `from-pub-date:${fromDate},type:journal-article`,
    sort: "published",
    order: "desc",
    rows: "80"
  });
  const data = await fetchJson(`https://api.crossref.org/works?${params.toString()}`);
  const rawItems = data?.message && Array.isArray(data.message.items) ? data.message.items : [];
  return rawItems.map((item) => normalizeCrossrefItem(item, config.researchQuery, tokens)).filter((item) => item.title);
}

function normalizeCrossrefItem(item, query, tokens) {
  const title = cleanText(first(item.title));
  const journal = cleanText(first(item["container-title"]));
  const doi = cleanDoi(item.DOI);
  const publicationDate = readCrossrefDate(item);
  const fullTextLink = cleanText(item.URL) || (doi ? `https://doi.org/${doi}` : "");
  const abstract = stripHtml(item.abstract || "");
  const citationCount = typeof item["is-referenced-by-count"] === "number" ? String(item["is-referenced-by-count"]) : "";
  const highlight = makeHighlight({ abstract, citationCount, source: "Crossref" }, query);

  return {
    recommendationId: doi || fullTextLink || title,
    title,
    journal,
    doi,
    publicationDate,
    citationCount,
    fullTextLink,
    abstract,
    highlight,
    sourceUrl: fullTextLink,
    source: "Crossref",
    relevanceScore: relevanceScore({ title, journal, abstract, publicationDate }, tokens)
  };
}

async function fetchOpenAlexRecommendations(config, tokens) {
  const fromDate = formatDate(daysAgo(config.lookbackDays));
  const params = new URLSearchParams({
    search: config.searchQuery,
    filter: `from_publication_date:${fromDate},type:article`,
    sort: "publication_date:desc",
    "per-page": "80"
  });
  addOpenAlexKey(params, config);
  const data = await fetchJson(`https://api.openalex.org/works?${params.toString()}`);
  const rawItems = Array.isArray(data.results) ? data.results : [];
  return rawItems.map((item) => normalizeOpenAlexItem(item, config.researchQuery, tokens)).filter((item) => item.title);
}

function normalizeOpenAlexItem(item, query, tokens) {
  const title = cleanText(item.display_name);
  const doi = cleanDoi(item.doi || "");
  const abstract = reconstructOpenAlexAbstract(item.abstract_inverted_index);
  const source = item.primary_location?.source || null;
  const journal = cleanText(source ? source.display_name : "");
  const fullTextLink = firstNonEmpty([
    item.open_access?.oa_url,
    item.primary_location?.landing_page_url,
    item.id,
    doi ? `https://doi.org/${doi}` : ""
  ]);
  const citationCount = typeof item.cited_by_count === "number" ? String(item.cited_by_count) : "";
  const publicationDate = cleanText(item.publication_date);
  const highlight = makeHighlight({ abstract, citationCount, source: "OpenAlex" }, query);

  return {
    recommendationId: doi || fullTextLink || title,
    title,
    journal,
    doi,
    publicationDate,
    citationCount,
    fullTextLink,
    abstract,
    highlight,
    sourceUrl: fullTextLink,
    source: "OpenAlex",
    openAlexSourceId: source ? cleanText(source.id) : "",
    issnL: source ? cleanText(source.issn_l) : "",
    relevanceScore: relevanceScore({ title, journal, abstract, publicationDate }, tokens)
  };
}

async function fetchSemanticScholarRecommendations(config, tokens) {
  const params = new URLSearchParams({
    query: config.searchQuery,
    limit: "80",
    fields: "paperId,title,abstract,venue,year,publicationDate,citationCount,url,openAccessPdf,externalIds,publicationVenue"
  });
  const headers = { Accept: "application/json" };
  if (config.semanticScholarApiKey) headers["x-api-key"] = config.semanticScholarApiKey;
  const data = await fetchJson(`https://api.semanticscholar.org/graph/v1/paper/search?${params.toString()}`, headers);
  const rawItems = Array.isArray(data.data) ? data.data : [];
  const earliest = daysAgo(config.lookbackDays);
  return rawItems
    .map((item) => normalizeSemanticScholarItem(item, config.researchQuery, tokens))
    .filter((item) => item.title)
    .filter((item) => isWithinLookback(item.publicationDate, earliest));
}

function normalizeSemanticScholarItem(item, query, tokens) {
  const title = cleanText(item.title);
  const abstract = cleanText(item.abstract);
  const doi = cleanDoi(item.externalIds?.DOI || "");
  const journal = cleanText(item.publicationVenue?.name || item.venue || "");
  const publicationDate = normalizeLooseDate(item.publicationDate || item.year || "");
  const citationCount = typeof item.citationCount === "number" ? String(item.citationCount) : "";
  const fullTextLink = firstNonEmpty([
    item.openAccessPdf?.url,
    item.url,
    doi ? `https://doi.org/${doi}` : ""
  ]);
  const highlight = makeHighlight({ abstract, citationCount, source: "Semantic Scholar" }, query);

  return {
    recommendationId: doi || item.paperId || fullTextLink || title,
    title,
    journal,
    doi,
    publicationDate,
    citationCount,
    fullTextLink,
    abstract,
    highlight,
    sourceUrl: fullTextLink,
    source: "Semantic Scholar",
    relevanceScore: relevanceScore({ title, journal, abstract, publicationDate }, tokens)
  };
}

async function fetchPubMedRecommendations(config, tokens) {
  const fromDate = formatPubMedDate(daysAgo(config.lookbackDays));
  const toDate = formatPubMedDate(new Date());
  const term = `${config.searchQuery} AND ("${fromDate}"[Date - Publication] : "${toDate}"[Date - Publication])`;
  const searchParams = new URLSearchParams({
    db: "pubmed",
    term,
    retmode: "json",
    sort: "pub date",
    retmax: "80"
  });
  const searchData = await fetchJson(`https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi?${searchParams.toString()}`);
  const ids = Array.isArray(searchData?.esearchresult?.idlist) ? searchData.esearchresult.idlist : [];
  if (!ids.length) return [];

  const summaryParams = new URLSearchParams({
    db: "pubmed",
    id: ids.join(","),
    retmode: "json"
  });
  const summaryData = await fetchJson(`https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esummary.fcgi?${summaryParams.toString()}`);
  const result = summaryData?.result || {};
  return ids.map((id) => normalizePubMedItem(result[id], config.researchQuery, tokens)).filter((item) => item.title);
}

function normalizePubMedItem(item, query, tokens) {
  if (!item) return {};
  const title = cleanText(stripHtml(item.title || ""));
  const journal = cleanText(item.fulljournalname || item.source || "");
  const doiRecord = Array.isArray(item.articleids) ? item.articleids.find((entry) => entry?.idtype === "doi") : null;
  const doi = cleanDoi(doiRecord ? doiRecord.value : "");
  const publicationDate = normalizeLooseDate(item.pubdate || item.epubdate || item.sortpubdate || "");
  const fullTextLink = `https://pubmed.ncbi.nlm.nih.gov/${item.uid}/`;
  const highlight = makeHighlight({ abstract: "", citationCount: "", source: "PubMed" }, query);

  return {
    recommendationId: doi || fullTextLink || title,
    title,
    journal,
    doi,
    publicationDate,
    citationCount: "",
    fullTextLink,
    abstract: "",
    highlight,
    sourceUrl: fullTextLink,
    source: "PubMed",
    relevanceScore: relevanceScore({ title, journal, abstract: "", publicationDate }, tokens)
  };
}

async function fetchEuropePmcRecommendations(config, tokens) {
  const fromDate = formatDate(daysAgo(config.lookbackDays));
  const toDate = formatDate(new Date());
  const params = new URLSearchParams({
    query: `(${config.searchQuery}) FIRST_PDATE:[${fromDate} TO ${toDate}]`,
    format: "json",
    resulttype: "core",
    pageSize: "80",
    sort_date: "y"
  });
  const data = await fetchJson(`https://www.ebi.ac.uk/europepmc/webservices/rest/search?${params.toString()}`);
  const rawItems = data?.resultList && Array.isArray(data.resultList.result) ? data.resultList.result : [];
  return rawItems.map((item) => normalizeEuropePmcItem(item, config.researchQuery, tokens)).filter((item) => item.title);
}

function normalizeEuropePmcItem(item, query, tokens) {
  const title = cleanText(stripHtml(item.title || ""));
  const journal = cleanText(item.journalTitle || item.bookOrReportDetails || "");
  const doi = cleanDoi(item.doi || "");
  const publicationDate = normalizeLooseDate(item.firstPublicationDate || item.pubYear || "");
  const abstract = stripHtml(item.abstractText || "");
  const citationCount = item.citedByCount ? String(item.citedByCount) : "";
  const fullTextLink = firstNonEmpty([
    findEuropePmcFullTextUrl(item),
    item.pmcid ? `https://pmc.ncbi.nlm.nih.gov/articles/${item.pmcid}/` : "",
    item.pmid ? `https://pubmed.ncbi.nlm.nih.gov/${item.pmid}/` : "",
    doi ? `https://doi.org/${doi}` : ""
  ]);
  const highlight = makeHighlight({ abstract, citationCount, source: "Europe PMC" }, query);

  return {
    recommendationId: doi || item.pmid || fullTextLink || title,
    title,
    journal,
    doi,
    publicationDate,
    citationCount,
    fullTextLink,
    abstract,
    highlight,
    sourceUrl: fullTextLink,
    source: "Europe PMC",
    relevanceScore: relevanceScore({ title, journal, abstract, publicationDate }, tokens)
  };
}

async function fetchArxivRecommendations(config, tokens) {
  const fromDate = formatArxivDate(daysAgo(config.lookbackDays));
  const toDate = formatArxivDate(new Date());
  const searchQuery = `${buildArxivTextQuery(config.searchQuery)} AND submittedDate:[${fromDate}0000 TO ${toDate}2359]`;
  const params = new URLSearchParams({
    search_query: searchQuery,
    start: "0",
    max_results: "80",
    sortBy: "submittedDate",
    sortOrder: "descending"
  });
  const response = await fetch(`https://export.arxiv.org/api/query?${params.toString()}`, {
    headers: { Accept: "application/atom+xml" }
  });
  if (!response.ok) throw new Error(`arXiv request failed: ${response.status}`);
  const xml = await response.text();
  return parseArxivEntries(xml).map((item) => normalizeArxivItem(item, config.researchQuery, tokens)).filter((item) => item.title);
}

function normalizeArxivItem(item, query, tokens) {
  const title = cleanText(item.title);
  const abstract = cleanText(item.summary);
  const publicationDate = normalizeLooseDate(item.published);
  const fullTextLink = firstNonEmpty([item.pdfUrl, item.url]);
  const highlight = makeHighlight({ abstract, citationCount: "", source: "arXiv" }, query);

  return {
    recommendationId: item.url || fullTextLink || title,
    title,
    journal: "arXiv",
    doi: "",
    publicationDate,
    citationCount: "",
    fullTextLink,
    abstract,
    highlight,
    sourceUrl: item.url || fullTextLink,
    source: "arXiv",
    relevanceScore: relevanceScore({ title, journal: "arXiv", abstract, publicationDate }, tokens)
  };
}

async function enrichJournalMetrics(items, manualImpactFactors, config) {
  const metricCache = new Map();
  const enriched = [];

  for (const item of items) {
    const manualImpactFactor = findImpactFactor(item.journal, manualImpactFactors);
    if (manualImpactFactor !== null) {
      enriched.push({ ...item, impactFactor: manualImpactFactor, impactFactorSource: "手动影响因子表" });
      continue;
    }
    if (!config.autoJournalMetrics) {
      enriched.push({ ...item, impactFactor: null, impactFactorSource: "" });
      continue;
    }
    const cacheKey = item.openAlexSourceId || item.issnL || normalizeJournalName(item.journal);
    if (!cacheKey) {
      enriched.push({ ...item, impactFactor: null, impactFactorSource: "" });
      continue;
    }
    if (!metricCache.has(cacheKey)) {
      metricCache.set(cacheKey, fetchOpenAlexSourceMetric(item, config).catch(() => null));
    }
    const metric = await metricCache.get(cacheKey);
    enriched.push({
      ...item,
      impactFactor: metric ? metric.value : null,
      impactFactorSource: metric ? metric.label : ""
    });
  }
  return enriched;
}

async function fetchOpenAlexSourceMetric(item, config) {
  const sourceUrl = buildOpenAlexSourceMetricUrl(item, config);
  if (!sourceUrl) return null;
  const source = await fetchJson(sourceUrl);
  const resolved = Array.isArray(source.results) ? source.results[0] : source;
  const stats = resolved?.summary_stats || {};
  const twoYearMeanCitedness = firstFiniteNumber([
    stats["2yr_mean_citedness"],
    stats.two_year_mean_citedness,
    stats.twoYearMeanCitedness
  ]);
  const hIndex = firstFiniteNumber([stats.h_index, resolved?.h_index]);
  const i10Index = firstFiniteNumber([stats.i10_index, resolved?.i10_index]);

  if (twoYearMeanCitedness !== null) return { value: twoYearMeanCitedness, label: "OpenAlex 2-year mean citedness" };
  if (hIndex !== null) return { value: hIndex, label: "OpenAlex h-index" };
  if (i10Index !== null) return { value: i10Index, label: "OpenAlex i10-index" };
  return null;
}

function buildOpenAlexSourceMetricUrl(item, config) {
  let url = "";
  if (item.openAlexSourceId) {
    if (/^https:\/\/openalex\.org\//i.test(item.openAlexSourceId)) {
      url = item.openAlexSourceId.replace("https://openalex.org/", "https://api.openalex.org/sources/");
    } else {
      url = `https://api.openalex.org/sources/${encodeURIComponent(item.openAlexSourceId)}`;
    }
  } else if (item.issnL) {
    url = `https://api.openalex.org/sources/issn:${encodeURIComponent(item.issnL)}`;
  } else if (item.journal) {
    const params = new URLSearchParams({ search: item.journal, per_page: "1" });
    addOpenAlexKey(params, config);
    return `https://api.openalex.org/sources?${params.toString()}`;
  }
  if (!url) return "";
  if (!config.openAlexApiKey) return url;
  const separator = url.includes("?") ? "&" : "?";
  return `${url}${separator}api_key=${encodeURIComponent(config.openAlexApiKey)}`;
}

function mergeRecommendations(items) {
  const merged = new Map();
  for (const item of items) {
    const key = recommendationKey(item);
    if (!key) continue;
    if (!merged.has(key)) {
      merged.set(key, item);
    } else {
      merged.set(key, mergeRecommendation(merged.get(key), item));
    }
  }
  return [...merged.values()];
}

function mergeRecommendation(current, incoming) {
  const source = combineSourceNames(current.source, incoming.source);
  const publicationDate = newestDate(current.publicationDate, incoming.publicationDate);
  const citationCount = maxCount(current.citationCount, incoming.citationCount);
  const impactFactor = Number.isFinite(Number(current.impactFactor)) ? current.impactFactor : incoming.impactFactor;

  return {
    ...current,
    journal: current.journal || incoming.journal,
    doi: current.doi || incoming.doi,
    publicationDate,
    citationCount,
    fullTextLink: current.fullTextLink || incoming.fullTextLink,
    sourceUrl: current.sourceUrl || incoming.sourceUrl,
    abstract: longerText(current.abstract, incoming.abstract),
    highlight: longerText(current.highlight, incoming.highlight),
    source,
    openAlexSourceId: current.openAlexSourceId || incoming.openAlexSourceId || "",
    issnL: current.issnL || incoming.issnL || "",
    impactFactor,
    impactFactorSource: current.impactFactorSource || incoming.impactFactorSource || "",
    relevanceScore: Math.max(current.relevanceScore, incoming.relevanceScore) + 1.5
  };
}

function mergePushedHistory(history, items) {
  const now = new Date().toISOString();
  const byKey = new Map(history.filter((entry) => entry?.key).map((entry) => [entry.key, entry]));
  for (const item of items || []) {
    const key = recommendationKey(item);
    if (!key) continue;
    const existing = byKey.get(key);
    byKey.set(key, {
      key,
      title: item.title || existing?.title || "",
      doi: item.doi || existing?.doi || "",
      journal: item.journal || existing?.journal || "",
      source: item.source || existing?.source || "",
      firstPushedAt: existing ? existing.firstPushedAt : now,
      lastPushedAt: now
    });
  }
  return [...byKey.values()].sort((a, b) => (b.lastPushedAt || "").localeCompare(a.lastPushedAt || "")).slice(0, 1000);
}

function sortRecommendations(a, b) {
  const priorityDiff = journalPrioritySortValue(b) - journalPrioritySortValue(a);
  if (priorityDiff !== 0) return priorityDiff;
  const relevanceDiff = b.relevanceScore - a.relevanceScore;
  if (Math.abs(relevanceDiff) > 1.5) return relevanceDiff;
  const impactDiff = impactFactorSortValue(b) - impactFactorSortValue(a);
  if (impactDiff !== 0) return impactDiff;
  if (relevanceDiff !== 0) return relevanceDiff;
  return compareDateDesc(a.publicationDate, b.publicationDate);
}

async function sendWeComDigest(config, batch) {
  const content = buildWeComMarkdown(config, batch);
  const response = await fetch(config.weComWebhookUrl, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ msgtype: "markdown", markdown: { content } })
  });
  const data = await response.json().catch(() => null);
  if (!response.ok) throw new Error(`WeCom request failed: ${response.status}`);
  if (data?.errcode && data.errcode !== 0) throw new Error(data.errmsg || `WeCom error: ${data.errcode}`);
}

function buildWeComMarkdown(config, batch) {
  if (!batch.items.length) {
    return [
      "**今日文献推送**",
      `> 研究方向：${batch.query || config.researchQuery}`,
      "> 今天没有找到未推送过的新文献。"
    ].join("\n");
  }

  const lines = [
    "**今日 3 篇最新相关文献**",
    `> 研究方向：${batch.query || config.researchQuery}`,
    `> 搜索来源：${batch.source || "全网学术搜索"}`,
    ""
  ];
  for (const [index, item] of batch.items.entries()) {
    const title = escapeMarkdownLinkText(truncate(item.title || "Untitled paper", 120));
    const link = item.fullTextLink || item.sourceUrl || (item.doi ? `https://doi.org/${item.doi}` : "");
    const metric = Number.isFinite(Number(item.impactFactor))
      ? `${item.impactFactorSource || "期刊指标"}：${item.impactFactor}`
      : "";
    const meta = [
      item.journal,
      item.journalPriority ? "Priority journal" : "",
      item.publicationDate,
      item.source ? `来源：${item.source}` : "",
      metric,
      item.doi ? `DOI：${item.doi}` : "",
      item.citationCount ? `引用：${item.citationCount}` : ""
    ].filter(Boolean).join(" | ");
    lines.push(link ? `${index + 1}. [${title}](${link})` : `${index + 1}. ${title}`);
    if (meta) lines.push(`> ${meta}`);
    if (item.highlight) lines.push(`> 亮点：${truncate(item.highlight, 260)}`);
    lines.push("");
  }
  return truncate(lines.join("\n"), 3900);
}

async function fetchJson(url, headers = {}) {
  const response = await fetch(url, { headers: { Accept: "application/json", ...headers } });
  if (!response.ok) throw new Error(`${url} returned ${response.status}`);
  return response.json();
}

function addOpenAlexKey(params, config) {
  if (config.openAlexApiKey) params.set("api_key", config.openAlexApiKey);
}

function makeHighlight(item, query) {
  const intro = item.abstract
    ? sentenceSummary(item.abstract)
    : `与“${query}”相关的最新论文，可重点查看其方法、数据集和结论是否与你的课题匹配。`;
  const parts = [intro];
  if (item.citationCount) parts.push(`${item.source || "来源"} 引用/参考计数：${item.citationCount}。`);
  return cleanText(parts.join(" "));
}

function rankingTokens(config) {
  const terms = [config.searchQuery, ...config.requiredKeywordGroups.flat()].join(" ");
  return [...new Set(keywordTokens(terms))];
}

function addRankingSignals(item, config) {
  return {
    ...item,
    journalPriority: journalPriorityScore(item.journal, config.priorityJournalTerms)
  };
}

function matchesRequiredKeywordGroups(item, groups) {
  if (!groups.length) return true;
  const text = itemTextForMatching(item);
  return groups.every((group) => group.some((term) => textMatchesTerm(text, term)));
}

function itemTextForMatching(item) {
  return [item.title, item.journal, item.abstract].map(cleanText).filter(Boolean).join(" ");
}

function journalPriorityScore(journal, priorityTerms) {
  const normalized = normalizeJournalName(journal);
  if (!normalized) return 0;
  let score = 0;

  if (normalized === "npj digital medicine") score = Math.max(score, 300);
  if (isNatureFamilyJournal(normalized)) score = Math.max(score, 250);

  for (const [index, term] of priorityTerms.entries()) {
    const priorityName = normalizeJournalName(term);
    if (!priorityName) continue;
    const baseScore = 220 - index;
    if (priorityName === "nature" && isNatureFamilyJournal(normalized)) {
      score = Math.max(score, baseScore);
    } else if (normalized === priorityName) {
      score = Math.max(score, baseScore);
    } else if (normalized.includes(priorityName) || priorityName.includes(normalized)) {
      score = Math.max(score, baseScore - 40);
    }
  }

  return score;
}

function isNatureFamilyJournal(normalizedJournal) {
  return normalizedJournal === "nature" || normalizedJournal.startsWith("nature ");
}

function relevanceScore(item, tokens) {
  const title = cleanText(item.title).toLowerCase();
  const journal = cleanText(item.journal).toLowerCase();
  const abstract = cleanText(item.abstract).toLowerCase();
  const text = `${title} ${journal} ${abstract}`;
  let score = 0;
  for (const token of tokens) {
    if (title.includes(token)) score += 4;
    if (abstract.includes(token)) score += 2;
    if (journal.includes(token)) score += 1;
    if (text.includes(token)) score += 1;
  }
  const date = new Date(item.publicationDate);
  if (!Number.isNaN(date.getTime())) {
    const ageDays = Math.max(0, (Date.now() - date.getTime()) / 86400000);
    score += Math.max(0, 3 - ageDays / 30);
  }
  return score;
}

function parseImpactFactorTable(value) {
  const table = new Map();
  for (const line of String(value || "").split(/\r?\n/)) {
    const cleaned = cleanText(line);
    if (!cleaned || cleaned.startsWith("#")) continue;
    const match = cleaned.match(/^(.+?)(?:=|,|\t)\s*([0-9]+(?:\.[0-9]+)?)\s*$/);
    if (match) table.set(normalizeJournalName(match[1]), Number(match[2]));
  }
  return table;
}

function findImpactFactor(journal, impactFactors) {
  const normalized = normalizeJournalName(journal);
  if (!normalized || !impactFactors.size) return null;
  if (impactFactors.has(normalized)) return impactFactors.get(normalized);
  for (const [name, value] of impactFactors.entries()) {
    if (normalized.includes(name) || name.includes(normalized)) return value;
  }
  return null;
}

function readCrossrefDate(item) {
  const dateHolder = item["published-print"] || item["published-online"] || item.published || item.issued;
  const dateParts = dateHolder && Array.isArray(dateHolder["date-parts"]) ? dateHolder["date-parts"][0] : null;
  if (!dateParts?.length) return "";
  const [year, month = 1, day = 1] = dateParts;
  return [year, String(month).padStart(2, "0"), String(day).padStart(2, "0")].join("-");
}

function reconstructOpenAlexAbstract(index) {
  if (!index || typeof index !== "object") return "";
  const words = [];
  for (const [word, positions] of Object.entries(index)) {
    if (!Array.isArray(positions)) continue;
    for (const position of positions) words[position] = word;
  }
  return cleanText(words.filter(Boolean).join(" "));
}

function findEuropePmcFullTextUrl(item) {
  const links = item?.fullTextUrlList && Array.isArray(item.fullTextUrlList.fullTextUrl)
    ? item.fullTextUrlList.fullTextUrl
    : [];
  const preferred =
    links.find((link) => /pdf/i.test(link.documentStyle || link.availability || "")) ||
    links.find((link) => /html|full/i.test(link.documentStyle || link.availability || "")) ||
    links[0];
  return preferred ? cleanText(preferred.url) : "";
}

function parseArxivEntries(xml) {
  return String(xml || "").split(/<entry>/i).slice(1).map((chunk) => {
    const entry = chunk.split(/<\/entry>/i)[0] || "";
    return {
      title: xmlText(entry, "title"),
      summary: xmlText(entry, "summary"),
      published: xmlText(entry, "published"),
      url: arxivLink(entry, "alternate"),
      pdfUrl: arxivLink(entry, "related")
    };
  });
}

function xmlText(chunk, tagName) {
  const match = chunk.match(new RegExp(`<${tagName}[^>]*>([\\s\\S]*?)<\\/${tagName}>`, "i"));
  return match ? decodeXml(match[1]) : "";
}

function arxivLink(chunk, rel) {
  const links = [...chunk.matchAll(/<link\b([^>]+)>/gi)];
  for (const link of links) {
    const attrs = link[1];
    if (!new RegExp(`rel=["']${rel}["']`, "i").test(attrs)) continue;
    const href = attrs.match(/href=["']([^"']+)["']/i);
    if (href) return decodeXml(href[1]);
  }
  return "";
}

function decodeXml(value) {
  return cleanText(String(value || "")
    .replace(/<!\[CDATA\[([\s\S]*?)\]\]>/g, "$1")
    .replace(/&#x([0-9a-f]+);/gi, (_match, hex) => String.fromCodePoint(parseInt(hex, 16)))
    .replace(/&#(\d+);/g, (_match, number) => String.fromCodePoint(parseInt(number, 10)))
    .replace(/&amp;/g, "&")
    .replace(/&lt;/g, "<")
    .replace(/&gt;/g, ">")
    .replace(/&quot;/g, '"')
    .replace(/&apos;/g, "'"));
}

function buildArxivTextQuery(query) {
  const terms = keywordTokens(query)
    .slice(0, 8)
    .map((token) => token.replace(/["():]/g, "").trim())
    .filter(Boolean)
    .map((token) => `all:${token}`);
  return terms.length ? terms.join(" AND ") : `all:${query}`;
}

function localDateTimeParts(timeZone) {
  const parts = new Intl.DateTimeFormat("en-CA", {
    timeZone,
    year: "numeric",
    month: "2-digit",
    day: "2-digit",
    hour: "2-digit",
    minute: "2-digit",
    hour12: false
  }).formatToParts(new Date());
  const value = Object.fromEntries(parts.map((part) => [part.type, part.value]));
  return {
    date: `${value.year}-${value.month}-${value.day}`,
    time: `${value.hour}:${value.minute}`,
    hour: Number(value.hour),
    minute: Number(value.minute)
  };
}

function timeToMinutes(value) {
  const [hour, minute] = String(value || "08:30").split(":").map(Number);
  return (Number.isFinite(hour) ? hour : 8) * 60 + (Number.isFinite(minute) ? minute : 30);
}

function recommendationKey(item) {
  if (item.doi) return `doi:${item.doi.toLowerCase()}`;
  const normalizedTitle = cleanText(item.title).toLowerCase().replace(/[^a-z0-9\u4e00-\u9fa5]+/g, " ");
  return normalizedTitle ? `title:${normalizedTitle}` : "";
}

function keywordTokens(query) {
  return cleanText(query).toLowerCase().split(/[\s,;，；、|/]+/).map((token) => token.trim()).filter((token) => token.length >= 2);
}

function parseSearchRoutes(value) {
  const raw = String(value || "").trim();
  if (!raw) return DEFAULT_SEARCH_ROUTES;
  let parsed;
  try {
    parsed = JSON.parse(raw);
  } catch (error) {
    throw new Error(`SEARCH_ROUTES must be valid JSON: ${error.message}`);
  }
  if (!parsed || typeof parsed !== "object" || Array.isArray(parsed)) {
    throw new Error("SEARCH_ROUTES must be a JSON object whose values are database-name arrays.");
  }
  return normalizeSearchRoutes({ ...DEFAULT_SEARCH_ROUTES, ...parsed });
}

function normalizeSearchRoutes(routes) {
  const normalized = {};
  for (const [routeName, sources] of Object.entries(routes || {})) {
    if (!Array.isArray(sources)) continue;
    normalized[cleanText(routeName)] = uniqueValues(sources.map(cleanText).filter(Boolean));
  }
  return normalized;
}

function resolveSearchSources(routes, routeName) {
  const normalizedName = cleanText(routeName);
  const sources = routes[normalizedName];
  if (!sources?.length) {
    throw new Error(`Unknown SEARCH_ROUTE_NAME "${normalizedName}". Available routes: ${Object.keys(routes).join(", ")}`);
  }
  return sources;
}

function normalizeRouteSourceName(value) {
  return cleanText(value)
    .toLowerCase()
    .replace(/&/g, "and")
    .replace(/[^a-z0-9]+/g, " ")
    .replace(/\s+/g, " ")
    .trim();
}

function parseRequiredKeywordGroups(value) {
  return String(value || "")
    .split(/[;\n]+/)
    .map((group) => group.split(/[|,]+/).map(cleanText).filter(Boolean))
    .filter((group) => group.length);
}

function parsePriorityJournalTerms(value) {
  return String(value || "")
    .split(/[,\n;]+/)
    .map(cleanText)
    .filter(Boolean);
}

function buildSearchQuery(query, requiredKeywordGroups) {
  const missingRequiredTerms = requiredKeywordGroups
    .filter((group) => !group.some((term) => textMatchesTerm(query, term)))
    .map((group) => group[0])
    .filter(Boolean);
  return cleanText([query, ...missingRequiredTerms].join(" "));
}

function textMatchesTerm(text, term) {
  const normalizedText = normalizeMatchText(text);
  const normalizedTerm = normalizeMatchText(term);
  if (!normalizedText || !normalizedTerm) return false;
  if (/^[a-z0-9]+$/i.test(normalizedTerm)) {
    return new RegExp(`\\b${escapeRegExp(normalizedTerm)}\\b`, "i").test(normalizedText);
  }
  return normalizedText.includes(normalizedTerm);
}

function normalizeMatchText(value) {
  return cleanText(value)
    .toLowerCase()
    .replace(/[‐‑‒–—]/g, "-")
    .replace(/[^a-z0-9\u4e00-\u9fa5]+/g, " ")
    .replace(/\s+/g, " ")
    .trim();
}

function escapeRegExp(value) {
  return String(value).replace(/[.*+?^${}()|[\]\\]/g, "\\$&");
}

function normalizeJournalName(value) {
  return cleanText(value)
    .toLowerCase()
    .replace(/&/g, "and")
    .replace(/[^a-z0-9\u4e00-\u9fa5]+/g, " ")
    .replace(/\b(the|journal|of)\b/g, " ")
    .replace(/\s+/g, " ")
    .trim();
}

function normalizeLooseDate(value) {
  const text = cleanText(value);
  if (!text) return "";
  const isoMatch = text.match(/\b(19|20)\d{2}-\d{2}-\d{2}\b/);
  if (isoMatch) return isoMatch[0];
  const yearMatch = text.match(/\b(19|20)\d{2}\b/);
  if (!yearMatch) return text;
  const monthNames = {
    jan: "01", feb: "02", mar: "03", apr: "04", may: "05", jun: "06",
    jul: "07", aug: "08", sep: "09", oct: "10", nov: "11", dec: "12"
  };
  const monthMatch = text.match(/\b(jan|feb|mar|apr|may|jun|jul|aug|sep|oct|nov|dec)[a-z]*\b/i);
  const dayMatch = text.match(/\b([0-2]?\d|3[01])\b/);
  const month = monthMatch ? monthNames[monthMatch[1].slice(0, 3).toLowerCase()] : "01";
  const day = dayMatch ? String(dayMatch[1]).padStart(2, "0") : "01";
  return `${yearMatch[0]}-${month}-${day}`;
}

function sentenceSummary(value) {
  const text = cleanText(value);
  if (!text) return "";
  const sentences = text.match(/[^.!?。！？]+[.!?。！？]?/g) || [text];
  return truncate(sentences.slice(0, 2).join(" "), 420);
}

function first(value) {
  return Array.isArray(value) ? value[0] || "" : value || "";
}

function firstNonEmpty(values) {
  return values.map(cleanText).find(Boolean) || "";
}

function uniqueValues(values) {
  return [...new Set(values.map(cleanText).filter(Boolean))];
}

function firstFiniteNumber(values) {
  for (const value of values) {
    const number = Number(value);
    if (Number.isFinite(number)) return number;
  }
  return null;
}

function isWithinLookback(value, earliestDate) {
  const time = Date.parse(value);
  if (!Number.isFinite(time)) return true;
  return time >= earliestDate.getTime();
}

function cleanDoi(value) {
  return cleanText(value).replace(/^https?:\/\/(dx\.)?doi\.org\//i, "");
}

function stripHtml(value) {
  return cleanText(String(value || "").replace(/<[^>]*>/g, " "));
}

function truncate(value, maxLength) {
  const text = cleanText(value);
  if (text.length <= maxLength) return text;
  return `${text.slice(0, maxLength - 1).trim()}...`;
}

function cleanText(value) {
  return String(value || "").replace(/\s+/g, " ").trim();
}

function daysAgo(days) {
  const date = new Date();
  date.setDate(date.getDate() - days);
  return date;
}

function formatDate(date) {
  return [date.getFullYear(), String(date.getMonth() + 1).padStart(2, "0"), String(date.getDate()).padStart(2, "0")].join("-");
}

function formatPubMedDate(date) {
  return [date.getFullYear(), String(date.getMonth() + 1).padStart(2, "0"), String(date.getDate()).padStart(2, "0")].join("/");
}

function formatArxivDate(date) {
  return [date.getFullYear(), String(date.getMonth() + 1).padStart(2, "0"), String(date.getDate()).padStart(2, "0")].join("");
}

function compareDateDesc(left, right) {
  return (Date.parse(right) || 0) - (Date.parse(left) || 0);
}

function impactFactorSortValue(item) {
  const value = Number(item.impactFactor);
  return Number.isFinite(value) ? value : Number.NEGATIVE_INFINITY;
}

function journalPrioritySortValue(item) {
  const value = Number(item.journalPriority);
  return Number.isFinite(value) ? value : 0;
}

function combineSourceNames(left, right) {
  const names = new Set(`${left || ""},${right || ""}`.split(",").map((name) => cleanText(name)).filter(Boolean));
  return [...names].join(", ");
}

function maxCount(left, right) {
  const leftNumber = Number(left);
  const rightNumber = Number(right);
  if (Number.isFinite(leftNumber) && Number.isFinite(rightNumber)) return String(Math.max(leftNumber, rightNumber));
  return cleanText(left || right);
}

function newestDate(left, right) {
  if (!left) return right || "";
  if (!right) return left || "";
  return Date.parse(right) > Date.parse(left) ? right : left;
}

function longerText(left, right) {
  return cleanText(right).length > cleanText(left).length ? cleanText(right) : cleanText(left);
}

function escapeMarkdownLinkText(value) {
  return cleanText(value).replace(/[\[\]]/g, "");
}

function isWeComWebhookUrl(url) {
  try {
    const parsed = new URL(url);
    return parsed.protocol === "https:" && parsed.hostname === "qyapi.weixin.qq.com" && parsed.pathname.includes("/webhook/send");
  } catch {
    return false;
  }
}

function numberFromEnv(name, fallback) {
  const value = Number(process.env[name]);
  return Number.isFinite(value) ? value : fallback;
}

async function writeGitHubOutput(values) {
  if (!process.env.GITHUB_OUTPUT) return;
  const lines = Object.entries(values).map(([key, value]) => `${key}=${String(value).replace(/\r?\n/g, " ")}`);
  await writeFile(process.env.GITHUB_OUTPUT, `${lines.join("\n")}\n`, { flag: "a" });
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
