<!-- Header -->
<div align="center">

  <h1>Market Analytics Dashboard — Technical Case Study</h1>
  <p><em>Sheets-native portfolio analytics: returns, risk, factor exposure, and attribution</em></p>

  <p>
    <img alt="Google Sheets" src="https://img.shields.io/badge/Google%20Sheets-Model-green?logo=googlesheets">
    <img alt="No Macros" src="https://img.shields.io/badge/No%20Macros-Yes-blue">
    <img alt="Benchmark" src="https://img.shields.io/badge/Benchmark-S%26P%20500-black">
    <img alt="Status" src="https://img.shields.io/badge/Status-Production%20Ready-brightgreen">
  </p>

  <sub>
    10-asset universe + S&amp;P 500 • Fully reproducible • Formula-driven (LET / FILTER / XLOOKUP)
  </sub>
</div>

<hr />

<!-- TOC -->
<h2 id="toc">Table of Contents</h2>
<ol>
  <li><a href="#problem">Problem Statement &amp; Why It Matters</a></li>
  <li><a href="#architecture">System Architecture (Sheets-Native)</a></li>
  <li><a href="#modeling">Modeling &amp; Math</a></li>
  <li><a href="#implementation">Implementation Details</a></li>
  <li><a href="#findings">Example Findings</a></li>
  <li><a href="#qa">Validation &amp; QA</a></li>
  <li><a href="#limits">Limitations &amp; Responsible Use</a></li>
  <li><a href="#roadmap">Extensions on the Roadmap</a></li>
  <li><a href="#appendix">Appendix — Key Formula Patterns</a></li>
</ol>

<hr />

<!-- 1. Problem -->
<h2 id="problem">1) Problem Statement &amp; Why It Matters</h2>
<p>
  Institutional-style portfolio analytics are typically locked behind paid terminals.
  This project delivers a <strong>fully reproducible analytics stack in Google Sheets</strong> that performs
  <strong>return modeling, risk estimation, factor exposure, and portfolio attribution</strong>
  for a 10-asset universe plus an S&amp;P 500 benchmark—<em>without</em> macros or external databases.
  It scales with new tickers and ships with a compact dashboard summarizing Sharpe, Information Ratio, and Alpha.
</p>

<p><strong>Use cases</strong></p>
<ul>
  <li>Rapid security screening and portfolio tuning over arbitrary date windows.</li>
  <li>Teaching and interviewing: variance/covariance, CAPM, performance ratios.</li>
  <li>Lightweight alternative to Bloomberg/FactSet for small teams and students.</li>
</ul>

<hr />

<!-- 2. Architecture -->
<h2 id="architecture">2) System Architecture (Sheets-Native)</h2>

<p><strong>Inputs</strong></p>
<ul>
  <li><strong>Universe</strong>: <code>JNJ, NVDA, AAPL, XOM, NEE, JPM, XLP, VDC, TGT, COST</code> (extensible)</li>
  <li><strong>Benchmark</strong>: S&amp;P 500 (close-only)</li>
  <li><strong>Controls</strong>: Quick Date Range, <kbd>B3</kbd> Start Date, <kbd>B6</kbd> End Date</li>
  <li><strong>Assumptions</strong>: <kbd>AP23</kbd> Risk-Free Rate, <kbd>AP24</kbd> Market Risk Premium, <kbd>AP22</kbd> S&amp;P Annual Variance</li>
</ul>

<p><strong>Data layer</strong></p>
<ul>
  <li>Prices via <code>GOOGLEFINANCE</code> in three-column blocks per asset: <em>Date</em> | <em>Close</em> | <em>%Δ</em>.</li>
  <li>Benchmark in <strong>A–C</strong>; assets in <strong>[D–F], [G–I], …, [AE–AG]</strong> (start at row 10).</li>
  <li>Returns begin at row 11 to avoid a spurious first observation.</li>
</ul>

<p><strong>Computation layer</strong></p>
<ul>
  <li>Daily returns are filtered to <kbd>B3–B6</kbd>, aligned to the benchmark calendar, and annualized (×252 for variance/covariance; ×√252 for st.dev).</li>
  <li>Formulas use <code>LET</code>, <code>FILTER</code>, <code>XLOOKUP</code>, array guards to dodge <code>#DIV/0!</code> and mis-alignment.</li>
</ul>

<p><strong>Presentation layer</strong></p>
<ul>
  <li>Per-name cards: Variance, Volatility, Covariance w/ S&amp;P, Beta, CAPM Expected Return, Actual Return, Alpha.</li>
  <li>Portfolio rollups: weighted expected/actual, preliminary volatility, Sharpe, Information Ratio.</li>
</ul>

<hr />

<!-- 3. Modeling -->
<h2 id="modeling">3) Modeling &amp; Math (Ground Truth)</h2>

<h3>3.1 Daily Return Model</h3>
<p>For any price series <code>P_t</code>: <code>r_t = P_t / P_{t-1} - 1</code></p>
<pre><code>=IF(OR(B11="",B10=""),"",B11/B10-1)
</code></pre>

<h3>3.2 Risk Estimation (Variance / Volatility)</h3>
<ul>
  <li><strong>Variance (annualized):</strong> <code>σ²_ann = VAR.P(r) × 252</code></li>
  <li><strong>Volatility (annualized):</strong> <code>σ_ann = STDEV.P(r) × √252</code></li>
</ul>

<h3>3.3 Covariance &amp; Alignment</h3>
<p>Align market and security calendars by exact-or-previous benchmark date to handle holidays/stale rows:</p>
<pre><code>=IFERROR(
  COVARIANCE.P(
    FILTER( XLOOKUP(D$11:D, A$11:A, C$11:C, "", -1), (D$11:D&gt;=$B$3)*(D$11:D&lt;=$B$6) ),
    FILTER( F$11:F, (D$11:D&gt;=$B$3)*(D$11:D&lt;=$B$6) )
  )*252,
""
)
</code></pre>

<h3>3.4 Beta (Market Sensitivity)</h3>
<p><code>β = Cov(r_stock, r_mkt) / Var(r_mkt)</code> (both annualized). Denominator from <kbd>AP22</kbd>.</p>
<pre><code>=IFERROR( ( &lt;annualized_covar&gt; ) / $AP$22, "")
</code></pre>

<h3>3.5 CAPM Expected Return</h3>
<p><code>E[r] = R_f + β × (E[r_m] − R_f)</code></p>
<pre><code>=$AP$23 + &lt;BetaCell&gt; * $AP$24
</code></pre>

<h3>3.6 Actual (Realized) Return Over the Window</h3>
<p>Last close on/before End ÷ first close on/after Start − 1:</p>
<pre><code>=IFERROR(
  XLOOKUP($B$6, D$10:D, E$10:E, "", -1) / XLOOKUP($B$3, D$10:D, E$10:E, "", 1) - 1,
""
)
</code></pre>

<h3>3.7 Alpha</h3>
<p><code>α = r_actual − (R_f + β × MRP)</code></p>

<h3>3.8 Portfolio Aggregation</h3>
<ul>
  <li><strong>Weighted expected/actual:</strong> <code>r_p = Σ w_i r_i</code></li>
  <li><strong>Portfolio volatility (extension):</strong> <code>σ_p = √(wᵀ Σ w)</code> (Σ = annualized covariance matrix)</li>
  <li><strong>Sharpe:</strong> <code>(r_p − R_f) / σ_p</code></li>
  <li><strong>Information Ratio (proper):</strong> <code>(r_p − r_m) / TE</code>, where <code>TE = STDEV(r_{p,t} − r_{m,t}) × √252</code></li>
</ul>

<hr />

<!-- 4. Implementation -->
<h2 id="implementation">4) Implementation Details</h2>
<ul>
  <li><strong>No circular references</strong>; inputs (dates, Rf, MRP) are decoupled from outputs.</li>
  <li><strong>Null-safe returns</strong> avoid −100% artefacts; stats ignore blanks.</li>
  <li><strong>Calendar alignment</strong> via <code>XLOOKUP(...,-1)</code> keeps covariance arrays sized correctly.</li>
  <li><strong>Dynamic ranges</strong>: everything filters to <kbd>B3–B6</kbd> rather than a hard 252-row cap.</li>
  <li><strong>Performance</strong>: <code>LET</code> scoping reduces recomputation; only <code>TODAY</code> and <code>GOOGLEFINANCE</code> are volatile.</li>
  <li><strong>Reusability</strong>: add a ticker (next 3-col block), copy formulas horizontally, done.</li>
</ul>

<hr />

<!-- 5. Findings -->
<h2 id="findings">5) Example Findings</h2>
<ul>
  <li><strong>NVDA, AAPL</strong>: high beta &amp; realized returns; strong positive alpha (risk-on leadership).</li>
  <li><strong>JNJ</strong>: defensive beta; moderate vol; positive alpha (quality ballast).</li>
  <li><strong>XOM</strong>: negative alpha in a tech-dominant window.</li>
  <li><strong>XLP, VDC</strong>: stabilizers; alpha can lag in risk-on periods.</li>
  <li><strong>Portfolio Sharpe ~0.4</strong>: improvable by concentrating alpha or lowering covariance.</li>
</ul>

<hr />

<!-- 6. QA -->
<h2 id="qa">6) Validation &amp; QA</h2>
<ul>
  <li>Alpha checked by recomputing CAPM vs. actual; values match the cards.</li>
  <li>S&amp;P 1Y return cross-checked: calendar-year vs. 252-trading-day variant.</li>
  <li>Short windows (&lt;5 obs) return blanks by design to avoid misleading stats.</li>
</ul>

<hr />

<!-- 7. Limits -->
<h2 id="limits">7) Limitations &amp; Responsible Use</h2>
<ul>
  <li>Close-only series (no dividends). For income names, prefer total-return proxies.</li>
  <li>Betas are window-specific; correlations are non-stationary.</li>
  <li>Information Ratio: prefer the tracking-error implementation for decision-grade use.</li>
</ul>

<hr />

<!-- 8. Roadmap -->
<h2 id="roadmap">8) Extensions on the Roadmap</h2>
<ol>
  <li>Full covariance matrix and true portfolio <code>σ_p</code>.</li>
  <li>Tracking-error Information Ratio using excess returns.</li>
  <li>Drawdown, recovery time, Sortino, Calmar.</li>
  <li>Log-return mode and CAGR reporting.</li>
  <li>Dividend-adjusted (total-return) price sources.</li>
</ol>

<hr />

<!-- Appendix -->
<h2 id="appendix">Appendix — Key Formula Patterns</h2>

<details open>
  <summary><strong>Daily return (safe)</strong></summary>
  <pre><code>=IF(OR(B11="",B10=""),"",B11/B10-1)</code></pre>
</details>

<details>
  <summary><strong>Covariance with S&amp;P (annualized; Stock 1 D/E/F)</strong></summary>
  <pre><code>=IFERROR(
  COVARIANCE.P(
    FILTER( XLOOKUP(D$11:D, A$11:A, C$11:C, "", -1), (D$11:D&gt;=$B$3)*(D$11:D&lt;=$B$6) ),
    FILTER( F$11:F, (D$11:D&gt;=$B$3)*(D$11:D&lt;=$B$6) )
  )*252,
""
)</code></pre>
</details>

<details>
  <summary><strong>Actual return (window; Stock 1 D/E/F)</strong></summary>
  <pre><code>=IFERROR(
  XLOOKUP($B$6, D$10:D, E$10:E, "", -1) / XLOOKUP($B$3, D$10:D, E$10:E, "", 1) - 1,
""
)</code></pre>
</details>

<details>
  <summary><strong>CAPM expected</strong></summary>
  <pre><code>=$AP$23 + &lt;BetaCell&gt; * $AP$24</code></pre>
</details>

<details>
  <summary><strong>Alpha</strong></summary>
  <pre><code>=&lt;ActualCell&gt; - &lt;ExpectedCell&gt;</code></pre>
</details>

<hr />

<p align="center"><em>Statistical rigor • Defensive spreadsheet engineering • Product thinking</em></p>
