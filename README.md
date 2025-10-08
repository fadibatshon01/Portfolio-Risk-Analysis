<h1 id="market-analytics-dashboard-technical-case-study">Market Analytics Dashboard — Technical Case Study</h1>

<h2 id="problem-statement">1) Problem Statement &amp; Why It Matters</h2>
<p>
  Institutional-style portfolio analytics are usually locked behind paid terminals. I built a
  <strong>fully reproducible analytics stack in Google Sheets</strong> that performs
  <strong>return modeling, risk estimation, factor exposure, and portfolio attribution</strong>
  for a 10-asset universe plus an S&amp;P 500 benchmark—without macros or external databases.
  The workbook works end-to-end, scales with new tickers, and includes a mini-dashboard
  summarizing risk/return quality (Sharpe, Information Ratio, Alpha).
</p>

<p><strong>Use cases</strong></p>
<ul>
  <li>Rapid security screening and portfolio tuning over arbitrary date windows.</li>
  <li>Hands-on teaching artifact for variance/covariance, CAPM, and performance ratios.</li>
  <li>Lightweight alternative to Bloomberg/FactSet for small teams and students.</li>
</ul>

<hr />

<h2 id="system-architecture">2) System Architecture (Sheets-Native)</h2>

<p><strong>Inputs</strong></p>
<ul>
  <li><strong>Universe</strong>: <code>JNJ, NVDA, AAPL, XOM, NEE, JPM, XLP, VDC, TGT, COST</code> (extensible).</li>
  <li><strong>Benchmark</strong>: S&amp;P 500 index (close-only).</li>
  <li><strong>Controls</strong>: Quick Date Range, <code>Start Date (B3)</code>, <code>End Date (B6)</code>, optional manual end.</li>
  <li><strong>Market Assumptions</strong>: <code>Risk-Free Rate (AP23)</code>, <code>Market Risk Premium (AP24)</code>, <code>S&amp;P Annual Variance (AP22)</code>.</li>
</ul>

<p><strong>Data layer</strong></p>
<ul>
  <li>Prices are pulled with <code>GOOGLEFINANCE</code> into <strong>3-column blocks per asset</strong>: <em>Date | Close | %Δ</em>.
      Benchmark lives in <strong>A–C</strong>, assets in <strong>[D–F], [G–I], …, [AE–AG]</strong> beginning at row 10.</li>
  <li>Returns begin at <strong>row 11</strong> to avoid a spurious first observation.</li>
</ul>

<p><strong>Computation layer</strong></p>
<ul>
  <li>Daily returns are filtered to <code>B3–B6</code>, aligned to the benchmark calendar, then aggregated to annualized statistics
      (<strong>×252</strong> for variance/covariance; <strong>×√252</strong> for st.dev).</li>
  <li>All formulas use <strong>LET</strong>, <strong>XLOOKUP</strong>, <strong>FILTER</strong>, and <strong>array-safe guards</strong> to avoid <code>#DIV/0!</code>,
      mis-alignment, or truncation.</li>
</ul>

<p><strong>Presentation layer</strong></p>
<ul>
  <li>For each security: <strong>Variance, Volatility, Covariance w/ S&amp;P, Beta, CAPM Expected Return, Actual Return, Alpha</strong>.</li>
  <li>For the portfolio: <strong>weighted expected/actual</strong>, preliminary volatility, <strong>Sharpe</strong>, and <strong>Information Ratio</strong>
      (with clear note on tracking-error enhancement).</li>
</ul>

<hr />

<h2 id="modeling-math">3) Modeling &amp; Math (Ground Truth)</h2>

<h3 id="daily-return-model">3.1 Daily Return Model</h3>
<p>For any price series <code>P_t</code>:</p>
<p><code>r_t = P_t / P_{t-1} - 1</code></p>
<p>Implementation avoids artefacts:</p>
<pre><code>=IF(OR(B11="",B10=""),"",B11/B10-1)
</code></pre>

<h3 id="risk-estimation">3.2 Risk Estimation (Variance / Volatility)</h3>
<p>For a filtered set of daily returns <code>{r_t}</code>:</p>
<ul>
  <li><strong>Variance (annualized)</strong> <code>σ²_ann = VAR.P(r) × 252</code></li>
  <li><strong>Volatility (annualized)</strong> <code>σ_ann = STDEV.P(r) × √252</code></li>
</ul>

<h3 id="covariance-alignment">3.3 Covariance &amp; Alignment</h3>
<p>
  We align market and security calendars by <strong>exact-or-previous</strong> benchmark date to handle
  holidays/stale last rows:
</p>
<pre><code>=IFERROR(
  COVARIANCE.P(
    FILTER( XLOOKUP(D$11:D, A$11:A, C$11:C, "", -1), (D$11:D&gt;=$B$3)*(D$11:D&lt;=$B$6) ),
    FILTER( F$11:F, (D$11:D&gt;=$B$3)*(D$11:D&lt;=$B$6) )
  )*252,
""
)
</code></pre>

<h3 id="beta">3.4 Beta (Market Sensitivity)</h3>
<p><code>β = Cov(r_stock, r_mkt) / Var(r_mkt)</code></p>
<p>Both numerator and denominator are <strong>annualized</strong>; denominator comes from <code>AP22</code>:</p>
<pre><code>=IFERROR( ( &lt;annualized_covar&gt; ) / $AP$22, "")
</code></pre>

<h3 id="capm">3.5 CAPM Expected Return</h3>
<p><code>E[r] = R_f + β × (E[r_m] − R_f)</code></p>
<p>Implementation: <code>=$AP$23 + Beta * $AP$24</code></p>

<h3 id="actual-return">3.6 Actual (Realized) Return Over Window</h3>
<p>Uses last close <strong>on/before</strong> end and first close <strong>on/after</strong> start:</p>
<pre><code>=IFERROR(
  XLOOKUP($B$6, D$10:D, E$10:E, "", -1) / XLOOKUP($B$3, D$10:D, E$10:E, "", 1) - 1,
""
)
</code></pre>

<h3 id="alpha">3.7 Alpha (Security)</h3>
<p><code>α = r_actual − (R_f + β × MRP)</code></p>

<h3 id="portfolio-aggregation">3.8 Portfolio Aggregation</h3>
<ul>
  <li><strong>Weighted expected/actual</strong>: <code>r_p = Σ w_i r_i</code></li>
  <li><strong>Portfolio volatility (extension)</strong>: <code>σ_p = √(wᵀ Σ w)</code> where <code>Σ</code> is the annualized covariance matrix of all names.</li>
  <li><strong>Sharpe</strong>: <code>(r_p − R_f) / σ_p</code></li>
  <li><strong>Information Ratio (proper)</strong>: <code>(r_p − r_m) / TE</code> with tracking error <code>TE = STDEV(r_{p,t} − r_{m,t}) × √252</code>.</li>
</ul>

<hr />

<h2 id="implementation-details">4) Implementation Details</h2>

<p><strong>Data quality &amp; resiliency</strong></p>
<ul>
  <li><strong>No circular references</strong>; inputs (dates, Rf, MRP) are decoupled from outputs.</li>
  <li><strong>Null-safe returns</strong> prevent −100% artefacts at series tails; stats ignore blanks.</li>
  <li><strong>Calendar alignment</strong> via <code>XLOOKUP(...,-1)</code> guarantees covariance won’t mis-size arrays.</li>
</ul>

<p><strong>Dynamic ranges (no hard caps)</strong></p>
<ul>
  <li>All stats operate on <code>B3–B6</code> filtered arrays, not on fixed 252-row windows.</li>
</ul>

<p><strong>Performance</strong></p>
<ul>
  <li>Column-wise arrays and <code>LET</code> eliminate repeated recalculation; only two volatile functions (<code>TODAY</code>, <code>GOOGLEFINANCE</code>).</li>
</ul>

<p><strong>Explainability</strong></p>
<ul>
  <li>Every card displays both model-based expectations (CAPM) and realized outcomes, plus first-order factor (beta) and dispersion (st.dev). Stakeholders can tell <em>what happened</em> and <em>why</em>.</li>
</ul>

<p><strong>Reusability</strong></p>
<ul>
  <li>New tickers drop into the next 3-column block; dashboard formulas copy horizontally without edits.</li>
</ul>

<hr />

<h2 id="example-findings">5) Example Findings (from the showcased window)</h2>
<ul>
  <li><strong>Growth names (NVDA, AAPL)</strong>: High beta, high realized returns, strong positive alpha → risk-on leadership consistent with tech-led rallies.</li>
  <li><strong>Defensive (JNJ)</strong>: Low beta, moderate volatility; modest positive alpha → quality ballast.</li>
  <li><strong>Energy (XOM)</strong>: Mixed; negative alpha in a tech-dominant window.</li>
  <li><strong>Staples (XLP, VDC)</strong>: Risk dampeners; alpha may lag in risk-on tapes—useful for regime rotation.</li>
  <li><strong>Portfolio Sharpe ~0.4</strong> in the example → acceptable but improvable; targeting higher alpha concentration or lower covariance would raise it.</li>
</ul>

<hr />

<h2 id="validation-qa">6) Validation &amp; QA</h2>
<ul>
  <li><strong>Unit checks</strong>: Alpha recomputed manually from CAPM and actual return; matches card values.</li>
  <li><strong>S&amp;P 1Y Return</strong>: Both calendar-year and 252-trading-day variants cross-checked.</li>
  <li><strong>Edge windows</strong>: Very short ranges (&lt;5 obs) return blanks by design; avoids misleading stats.</li>
</ul>

<hr />

<h2 id="limitations">7) Limitations &amp; Responsible Use</h2>
<ul>
  <li>Close-only series (no dividend adjustments) → prefer <strong>total-return proxies</strong> for dividend-heavy names.</li>
  <li>In-sample correlations are non-stationary; treat Betas as window-specific.</li>
  <li>Information Ratio in the sheet is labeled with the correct definition and an optional simplified variant; prefer the <strong>tracking-error</strong> implementation for decisions.</li>
</ul>

<hr />

<h2 id="extensions">8) Extensions on the Roadmap</h2>
<ol>
  <li><strong>Full covariance matrix</strong> and true portfolio <code>σ_p</code>.</li>
  <li><strong>Tracking-error IR</strong> using excess return series.</li>
  <li><strong>Drawdown &amp; recovery time</strong> analytics; <strong>Sortino</strong> and <strong>Calmar</strong> ratios.</li>
  <li><strong>Log-return mode</strong> and <strong>CAGR</strong> reporting.</li>
  <li><strong>Dividend-adjusted prices</strong> (total-return series) for income names.</li>
</ol>

<hr />

<h2 id="appendix">Appendix — Key Formula Patterns</h2>

<p><strong>Daily return (safe)</strong></p>
<pre><code>=IF(OR(B11="",B10=""),"",B11/B10-1)
</code></pre>

<p><strong>Covariance with S&amp;P (annualized; Stock 1 D/E/F)</strong></p>
<pre><code>=IFERROR(
  COVARIANCE.P(
    FILTER( XLOOKUP(D$11:D, A$11:A, C$11:C, "", -1), (D$11:D&gt;=$B$3)*(D$11:D&lt;=$B$6) ),
    FILTER( F$11:F, (D$11:D&gt;=$B$3)*(D$11:D&lt;=$B$6) )
  )*252,
""
)
</code></pre>

<p><strong>Actual return (window; Stock 1 D/E/F)</strong></p>
<pre><code>=IFERROR(
  XLOOKUP($B$6, D$10:D, E$10:E, "", -1) / XLOOKUP($B$3, D$10:D, E$10:E, "", 1) - 1,
""
)
</code></pre>

<p><strong>CAPM expected</strong></p>
<pre><code>=$AP$23 + &lt;BetaCell&gt; * $AP$24
</code></pre>

<p><strong>Alpha</strong></p>
<pre><code>=&lt;ActualCell&gt; - &lt;ExpectedCell&gt;
</code></pre>

<hr />

<h3 id="final-note">Final Note</h3>
<p>
  This project demonstrates <strong>statistical rigor, defensive spreadsheet engineering, and product thinking</strong>.
  It’s designed for live use: change the date window or weights and watch the risk/return story update in real time.
</p>
