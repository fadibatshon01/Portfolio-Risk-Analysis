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
  <li><a href="#quickstart">Quick Start</a></li>
  <li><a href="#problem">Problem Statement &amp; Why It Matters</a></li>
  <li><a href="#architecture">System Architecture (Sheets-Native)</a></li>
  <li><a href="#modeling">Modeling &amp; Math</a></li>
  <li><a href="#implementation">Implementation Details</a></li>
  <li><a href="#portfolio">Portfolio Construction &amp; Portfolio-Level Metrics</a></li>
  <li><a href="#findings">Example Findings</a></li>
  <li><a href="#qa">Validation &amp; QA</a></li>
  <li><a href="#limits">Limitations &amp; Responsible Use</a></li>
  <li><a href="#roadmap">Extensions on the Roadmap</a></li>
  <li><a href="#troubleshooting">Troubleshooting</a></li>
  <li><a href="#appendix">Appendix — Key Formula Patterns</a></li>
</ol>

<hr />

<!-- Quick Start -->
<h2 id="quickstart">Quick Start</h2>

<ol>
  <li><strong>Create the control cells</strong>
    <ul>
      <li><kbd>B3</kbd> = Start Date (date)</li>
      <li><kbd>B6</kbd> = End Date (date)</li>
      <li><kbd>AP22</kbd> = S&amp;P Annual Variance (auto)</li>
      <li><kbd>AP23</kbd> = Risk-Free Rate (e.g., <code>0.04</code>)</li>
      <li><kbd>AP24</kbd> = Market Risk Premium (e.g., <code>0.05</code>)</li>
    </ul>
  </li>

  <li><strong>Benchmark block (A–C)</strong>
    <ul>
      <li><strong>A10</strong>: <code>=GOOGLEFINANCE("^GSPC","price",$B$3,$B$6)</code> (spills A=Date, B=Close)</li>
      <li><strong>C11</strong> (daily %Δ, copy down):<br/>
        <pre><code>=IF(OR(B11="",B10=""),"",B11/B10-1)</code></pre>
      </li>
      <li><strong>AP22</strong> (annual variance, safe):<br/>
        <pre><code>=LET(m, FILTER($C$11:$C, ($A$11:$A&gt;=$B$3)*($A$11:$A&lt;=$B$6)*ISNUMBER($C$11:$C)), IF(COUNT(m)&lt;2,"", VAR.P(m)*252))</code></pre>
      </li>
    </ul>
  </li>

  <li><strong>Stocks in 3-column blocks</strong> (each: Date|Close|%Δ starting at row 10):
    <ul>
      <li>Stock 1 → <strong>D:E:F</strong>, Stock 2 → <strong>G:H:I</strong>, …, Stock 10 → <strong>AE:AF:AG</strong></li>
      <li>Daily %Δ formula in the third column of each block (start at row 11):<br/>
        <pre><code>=IF(OR(E11="",E10=""),"",E11/E10-1)</code></pre>
        (When pasted in the next block, it becomes <code>H11/H10-1</code>, then <code>K11/K10-1</code>, etc.)
      </li>
    </ul>
  </li>

  <li><strong>Per-stock analytics (copy across to each block)</strong>
    <ul>
      <li><em>Covariance w/ S&amp;P (annualized), Stock 1 (D/E/F):</em>
        <pre><code>=IFERROR(
  COVARIANCE.P(
    FILTER( XLOOKUP(D$11:D, A$11:A, C$11:C, "", -1), (D$11:D&gt;=$B$3)*(D$11:D&lt;=$B$6) ),
    FILTER( F$11:F, (D$11:D&gt;=$B$3)*(D$11:D&lt;=$B$6) )
  )*252,
"")</code></pre>
      </li>
      <li><em>Variance (annualized), Stock 1 (D/E/F):</em>
        <pre><code>=LET(rf, FILTER(F$11:F,(D$11:D&gt;=$B$3)*(D$11:D&lt;=$B$6)*ISNUMBER(F$11:F)), IF(COUNT(rf)&lt;2,"", VAR.P(rf)*252))</code></pre>
      </li>
      <li><em>Volatility (annualized), Stock 1:</em>
        <pre><code>=LET(rf, FILTER(F$11:F,(D$11:D&gt;=$B$3)*(D$11:D&lt;=$B$6)*ISNUMBER(F$11:F)), IF(COUNT(rf)&lt;2,"", STDEV.P(rf)*SQRT(252)))</code></pre>
      </li>
      <li><em>Beta, Stock 1:</em>
        <pre><code>=LET(
  covA, IFERROR(
    COVARIANCE.P(
      FILTER( XLOOKUP(D$11:D, A$11:A, C$11:C, "", -1), (D$11:D&gt;=$B$3)*(D$11:D&lt;=$B$6) ),
      FILTER( F$11:F, (D$11:D&gt;=$B$3)*(D$11:D&lt;=$B$6) )
    )*252, ""),
  IF(OR(covA="", $AP$22&lt;=0),"", covA/$AP$22))</code></pre>
      </li>
      <li><em>Expected Return (CAPM), given Beta in e.g. F6:</em>
        <pre><code>=IF(OR(F6="", $AP$23="", $AP$24=""), "", $AP$23 + F6*$AP$24)</code></pre>
      </li>
      <li><em>Actual Return (B3–B6), Stock 1:</em>
        <pre><code>=IFERROR(
  XLOOKUP($B$6, D$10:D, E$10:E, "", -1) / XLOOKUP($B$3, D$10:D, E$10:E, "", 1) - 1,
"")</code></pre>
      </li>
      <li><em>Alpha = Actual − Expected:</em>
        <pre><code>=IF(OR(&lt;ActualCell&gt;="", &lt;ExpectedCell&gt;=""), "", &lt;ActualCell&gt; - &lt;ExpectedCell&gt;)</code></pre>
      </li>
    </ul>
  </li>
</ol>

<hr />

<!-- Problem -->
<h2 id="problem">1) Problem Statement &amp; Why It Matters</h2>
<p>
  High-quality analytics are paywalled. This project demonstrates that with careful formula design you can match
  institutional risk/return workflows: <strong>variance, volatility, covariance, beta, CAPM expected return, actual return, alpha</strong>,
  plus portfolio rollups—directly in Google Sheets, with reproducibility and no scripts.
</p>

<hr />

<!-- Architecture -->
<h2 id="architecture">2) System Architecture (Sheets-Native)</h2>

<p><strong>Inputs</strong></p>
<ul>
  <li><strong>Universe</strong>: <code>JNJ, NVDA, AAPL, XOM, NEE, JPM, XLP, VDC, TGT, COST</code> (extensible)</li>
  <li><strong>Benchmark</strong>: S&amp;P 500 (close-only)</li>
  <li><strong>Controls</strong>: <kbd>B3</kbd> Start, <kbd>B6</kbd> End, optional Manual End</li>
  <li><strong>Assumptions</strong>: <kbd>AP23</kbd> Rf, <kbd>AP24</kbd> MRP, <kbd>AP22</kbd> Var(S&amp;P) annual</li>
</ul>

<p><strong>Data layer</strong></p>
<ul>
  <li>Three-column blocks: <em>Date | Close | %Δ</em>. Benchmark in <strong>A–C</strong>; assets in <strong>D–F, G–I, …, AE–AG</strong>.</li>
  <li>Returns start at row 11 to avoid the first missing observation.</li>
</ul>

<p><strong>Computation layer</strong></p>
<ul>
  <li>Filter returns to <kbd>B3–B6</kbd> and align to benchmark via <code>XLOOKUP(..., -1)</code>.</li>
  <li>Annualize: ×252 for variance / covariance, ×√252 for st.dev.</li>
</ul>

<p><strong>Presentation layer</strong></p>
<ul>
  <li>Per-name cards with the seven core metrics; portfolio section with rollups.</li>
</ul>

<hr />

<!-- Modeling -->
<h2 id="modeling">3) Modeling &amp; Math (Ground Truth)</h2>

<h3>3.1 Daily Return Model</h3>
<p>For any price series <code>P_t</code>: <code>r_t = P_t / P_{t-1} - 1</code>. In Sheets, guard for blanks to avoid spurious values.</p>

<h3>3.2 Variance &amp; Volatility</h3>
<ul>
  <li><strong>Variance (daily)</strong>: <code>VAR.P(r)</code>. <strong>Annualize</strong>: <code>×252</code>.</li>
  <li><strong>Volatility (daily)</strong>: <code>STDEV.P(r)</code>. <strong>Annualize</strong>: <code>×√252</code>.</li>
</ul>

<h3>3.3 Covariance &amp; Alignment</h3>
<p>
  We pair each stock’s daily return with the market return from the same or nearest previous trading day to handle holidays &amp; gaps:
</p>
<pre><code>=IFERROR(
  COVARIANCE.P(
    FILTER( XLOOKUP(D$11:D, A$11:A, C$11:C, "", -1), (D$11:D&gt;=$B$3)*(D$11:D&lt;=$B$6) ),
    FILTER( F$11:F, (D$11:D&gt;=$B$3)*(D$11:D&lt;=$B$6) )
  )*252,
"")</code></pre>

<h3>3.4 Beta</h3>
<p><code>β = Cov(stock, market) / Var(market)</code>, with both terms annualized. Denominator is <kbd>AP22</kbd>.</p>

<h3>3.5 CAPM Expected Return</h3>
<p><code>E[r] = R_f + β × MRP</code>. Inputs in <kbd>AP23</kbd> and <kbd>AP24</kbd>.</p>

<h3>3.6 Actual Return</h3>
<p>Use last close on/before <kbd>B6</kbd> divided by first close on/after <kbd>B3</kbd>, minus 1.</p>

<h3>3.7 Alpha</h3>
<p><code>α = Actual − Expected</code>. Positive alpha indicates outperformance beyond risk exposure.</p>

<hr />

<!-- Implementation -->
<h2 id="implementation">4) Implementation Details</h2>
<ul>
  <li><strong>No circular references</strong>. Inputs are decoupled from outputs.</li>
  <li><strong>Null-safe returns</strong>. Return cells are blank if either price is blank (no −100% artefacts).</li>
  <li><strong>Alignment</strong>. Market returns are aligned via <code>XLOOKUP(...,-1)</code> to prevent <code>FILTER</code> size mismatches.</li>
  <li><strong>Dynamic windows</strong>. Every metric is filtered to <kbd>B3–B6</kbd>. No 252-row truncation.</li>
  <li><strong>Performance</strong>. <code>LET</code> scoping reduces recomputation; only <code>TODAY</code> and <code>GOOGLEFINANCE</code> are volatile.</li>
</ul>

<hr />

<!-- Portfolio -->
<h2 id="portfolio">5) Portfolio Construction &amp; Portfolio-Level Metrics</h2>

<p><strong>Weights</strong></p>
<ul>
  <li>Create a 10-cell weight row (e.g., above each %Δ column: <strong>F8, I8, L8, O8, R8, U8, X8, AA8, AD8, AG8</strong>). Ensure weights sum to 1.00.</li>
</ul>

<p><strong>Daily portfolio returns (aligned to benchmark dates)</strong></p>
<details>
  <summary>Construct a matrix of aligned returns, then weight them</summary>
  <pre><code>=LET(
  md, $A$11:$A,   /* market dates, row 11 down */
  r1, XLOOKUP(md, D$11:D, F$11:F, "", -1),
  r2, XLOOKUP(md, G$11:G, I$11:I, "", -1),
  r3, XLOOKUP(md, J$11:J, L$11:L, "", -1),
  r4, XLOOKUP(md, M$11:M, O$11:O, "", -1),
  r5, XLOOKUP(md, P$11:P, R$11:R, "", -1),
  r6, XLOOKUP(md, S$11:S, U$11:U, "", -1),
  r7, XLOOKUP(md, V$11:V, X$11:X, "", -1),
  r8, XLOOKUP(md, Y$11:Y, AA$11:AA, "", -1),
  r9, XLOOKUP(md, AB$11:AB, AD$11:AD, "", -1),
  r10,XLOOKUP(md, AE$11:AE, AG$11:AG, "", -1),
  R, HSTACK(r1,r2,r3,r4,r5,r6,r7,r8,r9,r10),
  w, HSTACK(F8,I8,L8,O8,R8,U8,X8,AA8,AD8,AG8),
  pr, MMULT(R, TRANSPOSE(w)),
  pr)</code></pre>
</details>

<p><strong>Portfolio annualized volatility</strong></p>
<pre><code>=LET(pr, &lt;reference_to_portfolio_return_series_above&gt;, 
     rf, FILTER(pr, ISNUMBER(pr)),
     IF(COUNT(rf)&lt;2,"", STDEV.P(rf)*SQRT(252)))</code></pre>

<p><strong>Portfolio actual return over window</strong></p>
<pre><code>=LET(pr, &lt;portfolio_return_series&gt;, rf, FILTER(pr, (md&gt;=$B$3)*(md&lt;=$B$6)*ISNUMBER(pr)),
     IF(COUNT(rf)&lt;2,"", PRODUCT(1+rf) - 1))</code></pre>
<p><em>Note:</em> This compounds daily returns; if you want a simple (end/start − 1) portfolio result, compute end/start at the position level and weight by ending market value or use an index series.</p>

<p><strong>Sharpe Ratio</strong></p>
<pre><code>=( &lt;PortfolioActual&gt; - $AP$23 ) / &lt;PortfolioVolAnnual&gt;</code></pre>

<p><strong>Information Ratio (proper)</strong></p>
<details>
  <summary>Use tracking error (st.dev of daily excess returns × √252)</summary>
  <pre><code>=LET(
  md, $A$11:$A,
  rm, $C$11:$C,                               /* market return series */
  pr, &lt;portfolio_return_series&gt;,
  ex, FILTER(pr - rm, ISNUMBER(pr)*ISNUMBER(rm)),
  IF(COUNT(ex)&lt;2,"", (&lt;PortfolioActual&gt; - &lt;BenchmarkActual&gt;) / (STDEV.P(ex)*SQRT(252)))
)</code></pre>
</details>

<hr />

<!-- Findings -->
<h2 id="findings">6) Example Findings (from the showcased window)</h2>
<ul>
  <li><strong>NVDA, AAPL</strong>: high beta &amp; realized returns; strong positive alpha (risk-on leadership).</li>
  <li><strong>JNJ</strong>: defensive beta; moderate vol; positive alpha (quality ballast).</li>
  <li><strong>XOM</strong>: negative alpha during tech-led rally periods.</li>
  <li><strong>XLP, VDC</strong>: stabilizers; alpha can lag in risk-on periods.</li>
  <li><strong>Portfolio Sharpe ~0.4</strong>: could be improved by concentrating alpha, trimming redundant exposures, or lowering cross-name covariance.</li>
</ul>

<hr />

<!-- QA -->
<h2 id="qa">7) Validation &amp; QA</h2>
<ul>
  <li><strong>Alpha cross-check</strong>: <code>Actual − (Rf + β × MRP)</code> matches the card calculation.</li>
  <li><strong>S&amp;P 1Y Return</strong>: provide both calendar-year and 252-trading-day variants; they reconcile for liquid periods.</li>
  <li><strong>Edge windows</strong>: short ranges (&lt;5 obs) intentionally return blanks to avoid spurious stats.</li>
  <li><strong>Unit tests</strong>: swapping MRP, Rf, or the window updates Expected/Alpha consistently.</li>
</ul>

<hr />

<!-- Limits -->
<h2 id="limits">8) Limitations &amp; Responsible Use</h2>
<ul>
  <li>Close-only (no dividends) → for dividend-heavy names/ETFs, prefer total-return proxies.</li>
  <li>Betas are window-specific; correlations are non-stationary.</li>
  <li>IR shown with proper tracking error; avoid simplified proxies for decision-grade work.</li>
</ul>

<hr />

<!-- Roadmap -->
<h2 id="roadmap">9) Extensions on the Roadmap</h2>
<ol>
  <li>Full 10×10 covariance matrix (annualized) and portfolio <code>σ_p = √(wᵀ Σ w)</code> from daily series.</li>
  <li>Drawdown analytics: MaxDD, time-to-recover, underwater charts.</li>
  <li>Sortino, Calmar, Omega ratios; downside-only dispersion.</li>
  <li>Log-return mode, CAGR reporting, roll-window beta/vol charts.</li>
  <li>Dividend-adjusted prices, multi-asset benchmarks, currency handling.</li>
</ol>

<hr />

<!-- Troubleshooting -->
<h2 id="troubleshooting">10) Troubleshooting</h2>

<details>
  <summary><strong>“Array result was not expanded because it would overwrite…”</strong></summary>
  Clear the spill range below/next to the formula (Delete, not Backspace). Arrays need empty target ranges.
</details>

<details>
  <summary><strong>“FILTER has mismatched range sizes”</strong></summary>
  Ensure both arrays filtered by the same mask have identical shapes. In this model, align dates first with <code>XLOOKUP(...,-1)</code>, then apply a single mask to both arrays.
</details>

<details>
  <summary><strong>“An array value could not be found”</strong></summary>
  Occurs when exact date matches fail. Use <code>XLOOKUP(date, dates, values, "", -1)</code> to return the previous trading day when an exact match doesn’t exist.
</details>

<details>
  <summary><strong>“DIVIDE parameter 2 cannot be zero”</strong></summary>
  Guard denominators: check <code>AP22 &gt; 0</code> for Beta, and ensure start price &gt; 0 in Actual Return. Use <code>IF()</code> gates or <code>IFERROR()</code>.
</details>

<details>
  <summary><strong>GOOGLEFINANCE quirks</strong></summary>
  Data can be delayed or missing on holidays &amp; far history. The exact-or-previous lookup pattern prevents gaps from breaking covariance.
</details>

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
  <summary><strong>Variance / Volatility (annualized)</strong></summary>
  <pre><code>=LET(rf, FILTER(F$11:F,(D$11:D&gt;=$B$3)*(D$11:D&lt;=$B$6)*ISNUMBER(F$11:F)), IF(COUNT(rf)&lt;2,"", VAR.P(rf)*252))
=LET(rf, FILTER(F$11:F,(D$11:D&gt;=$B$3)*(D$11:D&lt;=$B$6)*ISNUMBER(F$11:F)), IF(COUNT(rf)&lt;2,"", STDEV.P(rf)*SQRT(252)))</code></pre>
</details>

<details>
  <summary><strong>Beta</strong></summary>
  <pre><code>=LET(
  covA, IFERROR(
    COVARIANCE.P(
      FILTER( XLOOKUP(D$11:D, A$11:A, C$11:C, "", -1), (D$11:D&gt;=$B$3)*(D$11:D&lt;=$B$6) ),
      FILTER( F$11:F, (D$11:D&gt;=$B$3)*(D$11:D&lt;=$B$6) )
    )*252, ""),
  IF(OR(covA="", $AP$22&lt;=0),"", covA/$AP$22))</code></pre>
</details>

<details>
  <summary><strong>Actual return (window; Stock 1 D/E/F)</strong></summary>
  <pre><code>=IFERROR(
  XLOOKUP($B$6, D$10:D, E$10:E, "", -1) / XLOOKUP($B$3, D$10:D, E$10:E, "", 1) - 1,
""
)</code></pre>
</details>

<details>
  <summary><strong>Expected (CAPM) &amp; Alpha</strong></summary>
  <pre><code>=$AP$23 + &lt;BetaCell&gt; * $AP$24
=&lt;ActualCell&gt; - &lt;ExpectedCell&gt;</code></pre>
</details>

<hr />

<p align="center"><em>Statistical rigor • Defensive spreadsheet engineering • Product thinking</em></p>
