<?php
/**
 * Superstore Dashboard in one PHP file
 * - Reads a local CSV exported from Tableau's Sample - Superstore (US)
 * - Builds 3 dashboards: Sales, Profit, Quantity
 * - Shows: KPIs, Monthly trend, Category x Metric, Sub-Category x Metric,
 *          Segment x Metric, Ship Mode x Metric, State x Metric (Top N)
 * - Filter: Year
 *
 * HOW TO USE
 * 1) Put this file (dashboard.php) and your CSV (superstore.csv) in the same folder.
 * 2) Ensure the CSV has headers including: "Order Date","Ship Mode","Segment","Country","City","State","Postal Code","Region","Category","Sub-Category","Sales","Quantity","Discount","Profit"
 * 3) Run a local PHP server:   php -S localhost:8080
 * 4) Open http://localhost:8080/dashboard.php
 * 5) Use the Year filter and the Sales/Profit/Quantity tabs.
 */
$CSV_FILE = __DIR__ . '/superstore.csv';
$TOP_N_STATES = 15; // limit for state bars
$DATE_FORMATS = ['n/j/Y', 'n/j/y', 'Y-m-d', 'd-m-Y', 'm/d/Y']; // try a few common formats

if (!file_exists($CSV_FILE)) {
  http_response_code(500);
  echo "<h2 style='font-family:system-ui'>CSV not found</h2><p>Expected: <code>superstore.csv</code> next to this file.</p>";
  exit;
}

function parse_date_flex($s, $formats) {
  foreach ($formats as $fmt) {
    $dt = DateTime::createFromFormat($fmt, $s);
    if ($dt) return $dt;
  }
  // try strtotime fallback
  $ts = strtotime($s);
  return $ts ? (new DateTime())->setTimestamp($ts) : null;
}

$rows = [];
if (($h = fopen($CSV_FILE, 'r')) !== false) {
  $header = fgetcsv($h);
  if (!$header) { die('Empty CSV'); }
  // map headers (case-insensitive)
  $map = [];
  foreach ($header as $i => $col) { $map[strtolower(trim($col))] = $i; }
  $need = ['order date','ship mode','segment','state','region','category','sub-category','sales','quantity','profit'];
  foreach ($need as $n) {
    if (!isset($map[$n])) { die("Missing column: $n"); }
  }
  while (($r = fgetcsv($h)) !== false) {
    $dt = parse_date_flex($r[$map['order date']], $DATE_FORMATS);
    if (!$dt) continue; // skip invalid
    $sales = (float)$r[$map['sales']];
    $profit = (float)$r[$map['profit']];
    $qty = (float)$r[$map['quantity']];
    $rows[] = [
      'date' => $dt,
      'year' => (int)$dt->format('Y'),
      'month_key' => $dt->format('Y-m'),
      'ship_mode' => trim($r[$map['ship mode']]),
      'segment' => trim($r[$map['segment']]),
      'state' => trim($r[$map['state']]),
      'region' => trim($r[$map['region']]),
      'category' => trim($r[$map['category']]),
      'sub_category' => trim($r[$map['sub-category']]),
      'sales' => $sales,
      'profit' => $profit,
      'quantity' => $qty,
    ];
  }
  fclose($h);
}

if (empty($rows)) { die('No data rows parsed.'); }

// Collect available years
$years = array_values(array_unique(array_map(fn($r) => $r['year'], $rows)));
sort($years);
$selectedYear = isset($_GET['year']) && $_GET['year'] !== 'all' ? (int)$_GET['year'] : 'all';

// Filter rows by year if selected
$filtered = $rows;
if ($selectedYear !== 'all') {
  $filtered = array_values(array_filter($rows, fn($r) => $r['year'] === $selectedYear));
}

function kpi_sum($rows, $field) {
  $sum = 0.0;
  foreach ($rows as $r) $sum += $r[$field];
  return $sum;
}

function group_sum($rows, $keyField, $valueField) {
  $out = [];
  foreach ($rows as $r) {
    $k = $r[$keyField] === '' ? 'Unknown' : $r[$keyField];
    if (!isset($out[$k])) $out[$k] = 0.0;
    $out[$k] += $r[$valueField];
  }
  arsort($out);
  return $out;
}

function monthly_sum($rows, $valueField) {
  $out = [];
  foreach ($rows as $r) {
    $k = $r['month_key'];
    if (!isset($out[$k])) $out[$k] = 0.0;
    $out[$k] += $r[$valueField];
  }
  ksort($out); // by month ascending
  return $out;
}

$metrics = ['sales','profit','quantity'];
$data = [];
foreach ($metrics as $m) {
  $data[$m] = [
    'kpi' => kpi_sum($filtered, $m),
    'monthly' => monthly_sum($filtered, $m),
    'by_category' => group_sum($filtered, 'category', $m),
    'by_subcategory' => group_sum($filtered, 'sub_category', $m),
    'by_segment' => group_sum($filtered, 'segment', $m),
    'by_shipmode' => group_sum($filtered, 'ship_mode', $m),
    'by_state' => group_sum($filtered, 'state', $m),
  ];
}

// limit states to top N
foreach ($metrics as $m) {
  $by_state = $data[$m]['by_state'];
  $limited = array_slice($by_state, 0, $TOP_N_STATES, true);
  $data[$m]['by_state'] = $limited;
}

// Safe JSON for JS
function j($x){ return json_encode($x, JSON_UNESCAPED_UNICODE|JSON_UNESCAPED_SLASHES); }

?>
<!doctype html>
<html>
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Superstore Dashboard — PHP</title>
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <script src="https://cdn.tailwindcss.com"></script>
  <style>
    .card { @apply rounded-2xl bg-white shadow p-4; }
    .tab { @apply px-4 py-2 rounded-full cursor-pointer; }
    .tab-active { @apply bg-gray-900 text-white; }
    .kpi { @apply text-2xl font-semibold; }
  </style>
</head>
<body class="bg-gray-100 text-gray-900">
  <div class="max-w-7xl mx-auto p-4 space-y-4">
    <header class="flex flex-col md:flex-row md:items-end md:justify-between gap-3">
      <div>
        <h1 class="text-3xl font-bold">Superstore Analytics</h1>
        <p class="text-sm text-gray-600">Dashboards for Sales, Profit, and Quantity — with Category / Sub-Category / Segment / Ship Mode / State breakdowns.</p>
      </div>
      <form method="get" class="flex items-center gap-2">
        <label class="text-sm">Year</label>
        <select name="year" class="border rounded-lg px-3 py-2 bg-white">
          <option value="all" <?= $selectedYear==='all'?'selected':'' ?>>All</option>
          <?php foreach ($years as $y): ?>
            <option value="<?= htmlspecialchars($y) ?>" <?= ($selectedYear===$y)?'selected':'' ?>><?= htmlspecialchars($y) ?></option>
          <?php endforeach; ?>
        </select>
        <button class="px-4 py-2 rounded-lg bg-gray-900 text-white">Apply</button>
      </form>
    </header>

    <nav class="flex gap-2">
      <button class="tab tab-active" data-metric="sales">Sales</button>
      <button class="tab" data-metric="profit">Profit</button>
      <button class="tab" data-metric="quantity">Quantity</button>
    </nav>

    <!-- KPI Row -->
    <section class="grid grid-cols-1 sm:grid-cols-3 gap-4">
      <div class="card"><div class="text-gray-500 text-sm">Total Sales</div><div id="kpi-sales" class="kpi">—</div></div>
      <div class="card"><div class="text-gray-500 text-sm">Total Profit</div><div id="kpi-profit" class="kpi">—</div></div>
      <div class="card"><div class="text-gray-500 text-sm">Total Quantity</div><div id="kpi-quantity" class="kpi">—</div></div>
    </section>

    <!-- Charts -->
    <section class="grid grid-cols-1 md:grid-cols-2 gap-4">
      <div class="card"><h3 class="font-semibold mb-2">Monthly Trend</h3><canvas id="trend"></canvas></div>
      <div class="card"><h3 class="font-semibold mb-2">Category × <span class="mc"></span></h3><canvas id="byCategory"></canvas></div>
      <div class="card"><h3 class="font-semibold mb-2">Sub-Category × <span class="mc"></span></h3><canvas id="bySubCategory"></canvas></div>
      <div class="card"><h3 class="font-semibold mb-2">Segment × <span class="mc"></span></h3><canvas id="bySegment"></canvas></div>
      <div class="card"><h3 class="font-semibold mb-2">Ship Mode × <span class="mc"></span></h3><canvas id="byShipMode"></canvas></div>
      <div class="card"><h3 class="font-semibold mb-2">State × <span class="mc"></span> (Top <?= (int)$TOP_N_STATES ?>)</h3><canvas id="byState"></canvas></div>
    </section>

    <footer class="text-xs text-gray-500 py-4">Built with PHP + Chart.js. Data source: Sample - Superstore CSV.</footer>
  </div>

<script>
const RAW = <?= j($data) ?>;
const FORMATTERS = {
  sales: v => new Intl.NumberFormat(undefined, { style:'currency', currency:'USD', maximumFractionDigits:0 }).format(v),
  profit: v => new Intl.NumberFormat(undefined, { style:'currency', currency:'USD', maximumFractionDigits:0 }).format(v),
  quantity: v => new Intl.NumberFormat(undefined, { maximumFractionDigits:0 }).format(v),
};

const state = { metric: 'sales' };

// KPI fill
function setKPIs(){
  document.getElementById('kpi-sales').textContent = FORMATTERS.sales(RAW.sales.kpi);
  document.getElementById('kpi-profit').textContent = FORMATTERS.profit(RAW.profit.kpi);
  document.getElementById('kpi-quantity').textContent = FORMATTERS.quantity(RAW.quantity.kpi);
}

// Chart helpers
function dictToArrays(d){
  const labels = Object.keys(d);
  const values = labels.map(k => d[k]);
  return { labels, values };
}

let charts = {};
function makeBar(ctxId, labels, values, title){
  const ctx = document.getElementById(ctxId).getContext('2d');
  if (charts[ctxId]) { charts[ctxId].destroy(); }
  charts[ctxId] = new Chart(ctx, {
    type: 'bar',
    data: { labels, datasets: [{ label: title, data: values }] },
    options: {
      responsive: true,
      plugins: { legend: { display: false }, tooltip: { mode: 'index', intersect: false } },
      scales: { x: { ticks: { autoSkip: false, maxRotation: 45, minRotation: 0 } }, y: { beginAtZero: true } },
    }
  });
}

function makeLine(ctxId, labels, values, title){
  const ctx = document.getElementById(ctxId).getContext('2d');
  if (charts[ctxId]) { charts[ctxId].destroy(); }
  charts[ctxId] = new Chart(ctx, {
    type: 'line',
    data: { labels, datasets: [{ label: title, data: values, tension: 0.3, fill: false }] },
    options: {
      responsive: true,
      plugins: { legend: { display: false }, tooltip: { mode: 'index', intersect: false } },
      scales: { y: { beginAtZero: true } },
    }
  });
}

function titleCase(s){ return s.charAt(0).toUpperCase() + s.slice(1); }
function updateMetric(metric){
  state.metric = metric;
  document.querySelectorAll('.mc').forEach(el => el.textContent = titleCase(metric));
  const pack = RAW[metric];

  // trend
  const m = dictToArrays(pack.monthly);
  makeLine('trend', m.labels, m.values, 'Monthly ' + titleCase(metric));

  // by category
  let x = dictToArrays(pack.by_category);
  makeBar('byCategory', x.labels, x.values, 'Category × ' + titleCase(metric));

  // by subcategory
  x = dictToArrays(pack.by_subcategory);
  makeBar('bySubCategory', x.labels, x.values, 'Sub-Category × ' + titleCase(metric));

  // by segment
  x = dictToArrays(pack.by_segment);
  makeBar('bySegment', x.labels, x.values, 'Segment × ' + titleCase(metric));

  // by ship mode
  x = dictToArrays(pack.by_shipmode);
  makeBar('byShipMode', x.labels, x.values, 'Ship Mode × ' + titleCase(metric));

  // by state (already limited to Top N in PHP)
  x = dictToArrays(pack.by_state);
  makeBar('byState', x.labels, x.values, 'State × ' + titleCase(metric));
}

// Tabs
function initTabs(){
  const tabs = document.querySelectorAll('.tab');
  tabs.forEach(btn => btn.addEventListener('click', () => {
    tabs.forEach(b => b.classList.remove('tab-active'));
    btn.classList.add('tab-active');
    updateMetric(btn.dataset.metric);
  }));
}

setKPIs();
initTabs();
updateMetric(state.metric);
</script>
</body>
</html>
