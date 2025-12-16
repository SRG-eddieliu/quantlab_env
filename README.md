Quantlib End to End process

1. Data Pipeline (Done): Data Ingestion by calling API and Perform basic data cleaning
2. Factor Library (Done): Create factors based on data from pipeline and perform factor analysis: Rank IC/IR, decile performance, LS portfolio, FF coefficient
   - Factor Aggregation (Thematic composite):
#,Thematic Composite Name,Core Theme,Included Raw Factors (51 total)
1,θMomentum_RiskAdj​,Trend/Low-Vol,"momentum_12m, residual_momentum_12m, volatility_60d, downside_vol_60d, industry_momentum"
2,θPure_Value​,Structural Cheapness,"earnings_yield, book_to_price, ev_to_ebitda_inv, cashflow_yield, free_cashflow_yield"
3,θPure_Quality​,Profitability/Safety,"profitability_roe, roa, gross_profitability, piotroski_fscore, accruals"
4,θShortTerm_Reversal​,Short-term Price Action,"mean_reversion_5d, max_daily_return_1m (sign-flipped), high52w_proximity"
5,θGrowth_Acceleration​,Fundamental Growth,"sales_growth, sales_growth_accel, rd_intensity, dividend_growth"
6,θInfo_Forensic_Drift​,Accounting/Analyst Info,"sue, earnings_surprise, analyst_revision_eps_30d, benford_chi2_d1, benford_chi2_d2"
7,θStructural_Liquidity​,Illiquidity Premium,"dollar_volume_20d, amihud_illiq_252d, amihud_illiq_log_20d, turnover"
8,θSystematic_Risk​,Systemic Risk/Investment,"beta_252d, residual_vol_252d, downside_beta_252d, leverage, asset_growth, investment_to_assets, net_issuance (sign-flipped)"
9,θSize_MarketCap​,Small Cap Premium,"log_market_cap, inverse_market_cap, enterprise_value, total_assets"
10,θInvestor_Sentiment​,Crowding and Behavioral Bias,"short_interest_ratio, put_call_ratio, retail_trading_volume, press_sentiment_score, search_interest_anomaly"
11,θCapital_Efficiency​,Investment and Reinvestment,"capex_to_assets, change_in_fixed_assets, investment_rate_qtr, debt_to_equity_change"
  - Weights:
    - Equal weight
    - Inverse volatility
    - IC/IR weight
  - IR = mean(IC)/std(IC)
3. Alpha Research (In Progress): Create alpha from factors and perform alpha analysis: Sharpe Ratio, Alpha Decay, Correlation
  - Core Systematic Alpha
Component (θ or F),Rationale
θPure_Value​,Essential structural factor for long-term premium.
θPure_Quality​,"Acts as a filter for Value stocks, preventing 'value traps'."
θMomentum_RiskAdj​,Diversifier to Value; provides exposure to trend-following.
θSystematic_Risk​,"Sign-Flipped. We want to short high-beta/high-leverage stocks, aligning with the Low-Risk Anomaly."
FSize_log_mktcap​ (Raw),Raw size factor included to explicitly control size exposure during optimization (Small-cap stocks often drive other factors).

  - Event-Driven & Information Alpha
Component (θ or F),Rationale
θInfo_Forensic_Drift​,Core driver: post-earnings drift and forensic accounting signals.
θShortTerm_Reversal​,Acts as a hedge against trend/momentum and captures microstructure effects.
θStructural_Liquidity​,"Sign-Flipped. This strategy may trade less liquid stocks, but we must be aware of the cost associated with high illiquidity."
θPure_Quality​,Used as a filter: Information events in high-quality stocks are often more reliable than those in junky stocks.

  - Cyclical & Growth Momentum Alpha
Component (θ or F),Rationale
θGrowth_Acceleration​,"Core driver: captures accelerating sales, R&D intensity, and fundamental investment growth."
θPath_and_Acceleration​,"Captures the quality of the trend (Hurst/Efficiency Ratio), ensuring the momentum is stable and less noisy."
θMomentum_RiskAdj​,Provides the traditional price momentum support for the growth narrative.
Fnet_buyback_yield​ (Raw),Directly includes the raw capital structure signal to boost the long-term quality of the growth group.

  - Global ML-Optimized Alpha
Component (θ or F),Rationale
ALL 8 Thematic Composites (θ),"Provides the model with robust, low-noise inputs covering all major economic themes."
Selected Raw Factors (F),Highly useful raw factors like amihud_illiq_252d and size\_log\_mktcap are often added as direct inputs to the ML model for explicit risk management.
Model:,"XGBoost or Random Forest with strong regularization (L1/L2, Dropout/Pruning)."
Output:,Single predicted AML​ score.

  - Weights:
    - MLR: may overfit; purify by using ret_excess ~ beta * factors where ret_excess = ret - sum(beta * FF_factors); then alpha = beta*factor
    - ML: may overfit, need dropout (NN)/regularization, takes all factors
     
5. Risk Modeling (To Do): multi-factor risk model
  - Centralized risk matrix

7. Portfolio Constructor (To Do): Multi Strategies; Output portfolio weights via optimizer
  - input alpha and risk matrices and set constrains:
    PM1 Core Ststematic Alpha - Capture structural, long-term premiums (Value, Quality, Momentum) with high capacity and low friction.
Constraint Category	Specific Constraint	Rationale
Factor Exposure	Strict Neutrality to Fama-French (FF) Factors	Must maintain near-zero Beta to MKT-RF, SMB, HML, and UMD. This ensures the returns are purely $\alpha$.
Turnover	Very Low Annual Limit	Limit turnover to < 100% annually. Core strategies are slow-moving; high turnover destroys $\alpha$ via transaction costs.
Position Sizing	Higher Single-Stock Limit	Higher capacity mandates mean the optimizer needs flexibility for larger positions (e.g., max 2.5% single-stock weight) to reduce the number of trades.
Liquidity	Minimum Dollar Volume Filter	Exclude the bottom quartile of the universe based on daily dollar volume, ensuring the strategy can be scaled.

    PM2 Event-Driven & Information Alpha - Exploit short-term, high-frequency information and post-event drifts. This strategy is driven by speed and high breadth ($N_{TS}$).
Constraint Category	Specific Constraint	Rationale
Turnover	Very High Daily/Weekly Limit	Allow aggressive rebalancing (e.g., > 500% annual turnover) to capture rapidly decaying informational $\alpha$.
Position Sizing	Lower Single-Stock Limit	Limit single stock to (e.g., max 0.5%). This forces high diversification to avoid single-event risk (idiosyncratic risk) and maximize breadth.
Liquidity	Tight Market Impact Limit	Constraint based on estimated transaction cost per trade. Prevents the optimizer from taking large positions in illiquid stocks, which would destroy the short-term $\alpha$.
Holding Period	Explicit Holding Period Constraint	Force the optimizer to target a portfolio with a short average holding period (e.g., 20 days) to quickly rotate capital.

    PM3 : Cyclical & Growth Momentum Alpha
Constraint Category	Specific Constraint	Rationale - Capture longer-term fundamental trends and acceleration, often tolerating cyclical market exposure.
Factor Exposure	Tolerated Directional Exposure	May allow non-neutral Beta exposure to Growth/Momentum factors within a controlled range (e.g., UMD Beta $\in [0.1, 0.5]$) to ride cyclical trends.
Sector/Industry	Active Weight Limits	Tighter limits on the deviation of sector weights from the benchmark (e.g., $\pm 5\%$). Since growth is often concentrated (Tech/Health), this manages concentration risk.
Turnover	Mid-Range Limit	(e.g., 200%-300% annual turnover). Balances the need to track fundamental acceleration with avoiding excessive transaction costs.

    PM4 : Global ML-Optimized Alpha - Maximize the non-linear prediction by forcing extreme diversification and purity, acting as a market-neutral diversifying sleeve.
Constraint Category	Specific Constraint	Rationale
Purity	Extreme Market Neutrality	Must maintain strict dollar neutrality (Long $\approx$ Short) and near-zero Beta to MKT-RF and all risk factors.
Diversification	High Effective Number of Stocks (ENS)	Constraint added to force the optimizer to allocate capital to a large number of names, maximizing the realized Breadth ($\sqrt{N}$).
Risk Budget	Tight Tracking Error Limit	The tightest limit on residual risk (e.g., 4% annual tracking error) to ensure its low volatility and high Sharpe/IR profile.

  - Consider calling open source pkg: MVO, Black Litterman
  - Future development: reinforcement learning

6. Back Testing (To Do):
  - Liquidity/Slippage cost:
    a) mid price exeution with spread penalty: mid-price +/- (1/2 * BidAsk spread)
    b) Percentage-based slippage: 0.05% of trade value
    c) market impact model (future dev): TC ~ OrderSize / MarketLiquidity * Volatility
  - Liquidity Cost: model by trading volume, Bid/Ask Spread

7. Strategy Integrator (To Do): Risk Budgeting to set allocations on different Strategies
  - Risk Parity, Equal weight, MVO
  - Factor exposure, short the factor flattening exposure for pure alpha
  - 
  - IR = mean(alpha)/std(alpha) : tracking error
  - fundamental law of active management (FLAM): IR_ptf ~ IC_ptf * sqrt(Breadth)
    - IC_ptf = corr(weight rank, fwd residual return rank)
    - IC_ptf ~ IR_factor
