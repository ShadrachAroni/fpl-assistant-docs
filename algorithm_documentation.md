# FPL Assistant Algorithm Documentation

This document provides a technical deep-dive into the intelligence layer of the FPL Assistant, detailing the predictive model, feature engineering pipeline, and optimization strategies.

## 1. Core Architecture: LightGBM
The FPL Assistant uses a **LightGBM (Light Gradient Boosting Machine)** architecture for expected points (xP) prediction. 

### Why LightGBM?
- **Speed & Efficiency**: Optimized for large datasets with high-dimensional features.
- **Handling Sparsity**: Effectively manages players with varying amounts of historical data.
- **Accuracy**: Consistently outperforms standard regression models in sports prediction tasks due to its leaf-wise tree growth strategy.

---

## 2. Model Training & Optimization

### Hybrid Incremental Learning
The model supports **incremental training**, allowing it to learn from new Gameweek data without retraining from scratch. Previous weights are stored in Supabase and used as an initialization state for new training runs.

### Cross-Validation Strategy: GroupKFold
To prevent data leakage, we use a **5-fold GroupKFold** cross-validation approach.
- **Target**: Predicted points for the next 1-5 Gameweeks.
- **Groups**: Grouped by `player_id`. This ensures that a player's historical data never appears in both the training and validation sets of the same fold, ensuring the model generalizes to "unseen" player patterns.

### Hyperparameter Tuning: Optuna
Once a month, the system triggers an **Optuna** optimization study to refine hyperparameters (learning rate, max depth, num leaves, etc.). It optimizes for minimizing Root Mean Squared Error (RMSE) on the validation set.

### Temporal Sample Weighting
Newer data is more relevant in FPL. We apply a **linear decay weighting** to samples:
- Recent GW data has a weight of **1.0**.
- Older data (up to 20 GWs ago) decays by **0.045 per GW**.
- This ensures the model adapts quickly to changes in player form or team tactics.

---

## 3. Feature Engineering (67 Canonical Features)

The "IQ" of the model comes from its 67 enriched features, categorized into five pillars:

### I. Form & Momentum
- **Rolling 4-GW Averages**: points, minutes, goals, assists, ICT index, xG, xA.
- **Lag Metrics**: Performance in the immediate previous GW (Lag 1) and 2 GWs ago (Lag 2).
- **Points Momentum**: The rate of change in points scoring over the last 3-5 appearances.

### II. Risk Assessment
- **Total Risk**: A composite score (0-1) reflecting availability.
- **Rotation Risk**: Calculated based on minutes volatility and team depth.
- **Fatigue Risk**: Based on midweek involvement (European competitions) and high minutes-per-game in short intervals.
- **Sentiment Analysis**: Integration of news feeds (Injury news, press conferences) to predict news severity.

### III. Ownership & Intelligence
- **Selected By %**: Real-time ownership from FPL API.
- **Template Multiplier**: Identifies if a player is a "must-have" or a "differential" based on High-EO (Effective Ownership) thresholds.
- **EO Prediction**: Predicts ownership for upcoming GWs based on transfer trends.

### IV. Fixture & Contextual Metrics
- **Fixture Adjusted Form**: Normalizes player form against the difficulty (FDR) of the opponents they faced.
- **Home/Away Delta**: Advantage factor based on specific team performance in home vs. away settings.
- **Manager Change Factor**: A decay factor applied when a team changes manager, introducing uncertainty.

### V. Advanced Technical Stats
- **Understat Integration**: xG, xA, and xGI (Expected Goal Involvement) per 90 minutes.
- **ICT Index**: Influence, Creativity, and Threat scores.
- **Price Trajectory**: Probabilities of price rises or falls based on Net Transfers.

---

## 4. Transfer Optimization & Simulation

The Transfer Planner uses a multi-objective utility function to recommend moves. It doesn't just look for points; it balances risk and value.

### Weighted Utility Function
The engine calculates a "Utility Score" for every possible transfer path:
- **Expected Points (35%)**: Direct output of the LightGBM model.
- **Safety/Risk (20%)**: Minimizing rotation and injury risk.
- **Market Value (15%)**: Optimization for team value growth.
- **Differential Edge (15%)**: Weighting towards low-ownership players in high-swing scenarios.
- **Fixture Horizon (15%)**: Upcoming 5-GW difficulty average.

### Monte Carlo Simulation (Optional)
For long-term planning, the engine runs simulations across 10,000 permutations of "Transfer -> Bench -> Captain" cycles to find the path with the highest terminal utility score over a 5-GW horizon.

---

## 5. Data Pipeline Integration
- **Primary Source**: FPL Official API.
- **Advanced Metrics**: Understat (Scraping/API).
- **Injury/Team News**: Custom scraper for Premier League News & RSS.
- **External API**: Sportmonks for detailed match event and xG data.
