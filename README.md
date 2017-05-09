# NFL Betting Market Prediction and Analysis

Sports betting is a multi-billion dollar per year industry.  Estimates vary, but the Nevada Gaming Commission reported over $3.2B in legal gambling in 2011, with 41% of that money being wagered on football alone. <sup id="a1">[__1__](#fn1)</sup>  According to NBA commissioner Adam Silver, if illegal gambling is counted the annual total for American gambling jumps to $400B. <sup id="a1">[__2__](#fn2)</sup>   The single biggest betting event in America is the Super Bowl, which now attracts over $100M in legal bets and potentially ten times that amount illegally. <sup id="a1">[__3__](#fn3)</sup>  With such sums of money flowing into the betting market, I was curious to see if I could isolate some -- _any_ -- notable inefficiency which would give a leg up on betting intelligently.  

The oddsmakers in Vegas use networks of supercomputers to set the odds, so expecting to beat them outright with a single machine learning model is a bad bet, no pun intended.  However, my hope was that systemic inefficiencies could be ferreted out as Vegas must also shift the odds based on __how the public bets__ in order to prevent a catastrophic loss of house money should an upset occur, granting the one-sided public bets all large payouts.  This opens the door for the informed bettor (us) to profit by the uninformed bettors (the public) making bad choices and forcing Vegas to alter the odds in an inefficient manner. In the interest of full disclosure, I have never bet a penny in my life (not even on slot machines -- sorry, casinos).  

<BR>
<BR>

## Table of Contents
1. [Dataset](#dataset)
    + [Advanced Metrics](#advanced-metrics)
    + [Acquisition and Error Correction](#acquisition-and-error-correction)
    + [Feature Engineering](#feature-engineering)
2. [NFL Betting Primer](#nfl-betting-primer)  
    + [Interpreting Odds and the Payout](#interpreting-odds-and-the-payout)
    + [The Spread](#the-spread)
    + [The Over/Under](#the-over-under)
    + [The Money Line](#the-money-line)  
3. [Wise Bets](#wise-bets)  
4. [Model Selection](#model-selection)   
    + [Data Selection](#data-selection)
    + [Classification vs. Regression](#regression-vs-classification)
    + [Model Selection and Training](#model-selection-and-training)
      + [Tree-based Feature Importance](#feature-separation-and-importance)
5. [Results](#results)
    + [Class Inspection](#class-inspection)
      + [PCA](#principal-component-analysis)
      + [t-SNE](#t---sne)
      + [Feature Overlaps](#feature-overlaps)
    + [Spread Results](#spread-results)
      + [Accuracy](#spread-accuracy)
      + [Feature Importance](#feature-importance)
      + [Analysis](#spread-analysis)
    + [Over/Under](#over---under)
      + [Accuracy](#over---under-accuracy)
      + [Feature Importance](#over---under-feature-importance)
      + [Analysis](#over---under-analysis)
    + [Money Line](#money-line-results)
    + [Predicting Bet Outcomes](#predicting-bet-outcomes)
      + **ROC** /**CMAT**
      + [Accuracy](#winning-team-accuracy)
      + [Feature Importance](#winning-team-feature-importance)
    + Weather
      + TEMP/wind/wc plots
    + Wise Bets Results
      + Spread
      + Over/Under
      + Money Line
        + 91.5% R2 with Spread when predicting proba...
        + Try Seaborn RegPlot and lmplot with some targets/wise/proba/etc
      + Biggest Bet Upsets?
    + Hypothetical Bettor Using This Model
      + Money Line
    + Clusters - Four Types
6. [Future Considerations](#future-considerations)  
    + Dynamic Web App
    + Player-specific Information
      + Injuries
      + Current-season Performance
    + Miles Traveled


<BR>
<BR>
<BR>



## Dataset
The point of this entire project was to use team-level data to identify trends in how Vegas oddsmakers set the odds for a given game.  However, as this model aims to predict single game results, it requires the stats and information for each of the two teams in a given game up to that point in the given season.  This meant I needed to procure game-by-game detailed information for every statistic, and not merely season-long summary information.  The types of statistics and information I wanted to model included all in-game statistics, weather, stadium information, and advanced analytics.

### Advanced Metrics
Sports analytics has grown from a small cottage industry in the mid-1980s to a robust field unto itself in 2017.  My aim was to leverage as many 'advanced' as possible metrics to improve my model's accuracy.  Some of these metrics are proprietary and available only through subscriptions to their respective stat-owning websites, such as the _Defense-adjusted Value Over Average_ (DVOA) metrics from [Football Outsiders](http://www.footballoutsiders.com/) or _Clutch-weighted Quarterback Rating_ (QBR) and Brian Burke's _Football Power Index_ (FPI) from [ESPN Insider](http://www.espn.com/insider/).  

Other metrics, such as _Pythagenport Win Expectancy_, a mildly revised descendent of baseball Sabremetrics godfather Bill James' famous _Pythagorean Win Expectancy_ metric, or _Adjusted Net Yards Per Attempt_ (ANY/A), most recently modified by Chase Stuart of [Football Perspective](http://www.footballperspective.com/tag/anya/), must be calculated.  Another excellent team-level advanced metric is _Expected Points Added_ (EPA), which originates from the seminal "Hidden Game of Football" published in the late 1980s by Bob Carroll and Pete Palmer, and has been updated by the folks at [Pro-Football Reference](http://www.sports-reference.com/blog/2012/03/features-expected-points/).   

Due to the proprietary nature of some of the advanced metrics, and the reliance upon more granular statistics for others, they are not available for the entirety of the dataset itself.  Vegas line information only goes back to the 1978 season, meaning 1978 is the earliest possible season for this model.  Time of possession information starts in 1983, 3rd and 4th Down success rates as well as DVOA begin in 1991, and EPA starts in 1994.  In the future as more old game logs and old game films are parsed and logged, it will be possible for these insightful advanced metrics to be extended further back into league history.  The selection of data from this parent dataset will be discussed below under [Data Selection](#data-selection).

### Acquisition and Error Correction
Unfortunately, no single source exists which has all these statistics.  In an effort to use as many of these stats as possible I decided to scrape the desired single-game statistics from [Pro Football Reference](http://www.pro-football-reference.com) (PFR) using BeautifulSoup and urllib.  PFR is known as the online encyclopedia for all things pro football, and has detailed information for nearly each game played in pro football history, including stadium type, time of game, weather, and Vegas betting information.  Regarding scraping of their site, PFR makes the following pro-scraping statement on their [data use](http://www.sports-reference.com/data_use.html) page:
>We will not fulfill any requests for data for custom downloads, unless you are prepared to pay a minimum of $1,000 for any such request.
>
>We realize this will be insurmountable for any student requests. However, I would point out that learning how to accumulate data is often a more valuable skill than actually analyzing the data, so we encourage you as a student or professional to learn how.

In total, I obtained 181, 285 separate tables.  Around 30,000 of these were used in this project.  In order to utilize them, I first had to do modify them for uniformity and then create a table for each team that summarized their progress through a given season, game by game, with around 300 added features that covered both single-game and running total statistics.  Once completed, the final database was formed by stitching the statistics for each home team and road team together for every game from 1950-2016 into a single entry. For games that have Vegas-related information, which starts in 1978, this totals around 12,500 games.

Surprisingly, most of PFR's data was well-maintained.  There were, however, two notable errors.  First, _time of possession_ data for all post-season games from 1991-1998 was missing.  I looked up each of these games and manually entered the correct data.  Second, 87 games had missing weather data (temperature, wind chill, wind speed (mph), and humidity), which forced me to manually look up the weather in the city the game was played in on the date of the game, and insert into the database one-by-one.  (_Surprise_, it's hot and dry in Arizona).  

A total of 1817 games were played in a closed-roof, climate-controlled dome, starting in 1968.  (For clarity, some stadia have retractable roofs, so only games played with the roof closed are logged as 'dome' games).  After reading online about typical conditions inside a domed stadium, I set dome-game temperatures to 67° F, no wind, and no humidity.  There were 1961 games inside of a domed stadium from 1978-2016, totaling around 21% of all games played.  Wind chill was also calculated for each game with a temperature below 50° F using the modern formula of 35.74 \+ (0.6215 \* Temp) - (35.75 \* Wind<sup>0.16</sup>) \+ (0.4275 \* Temp \* Wind<sup>0.16</sup>). <sup>[__4__](#fn4)</sup>

Last, as a long-time paying member at Football Outsiders, I was able to obtain all the DVOA data in their databases, which runs back to 1991 as of February 2017.

### Feature Engineering
I engineered features in seven distinct ways.
1. __Per-game averages__ for each team in every statistic.  This was necessary for two concrete reasons.
    1. Not every season in NFL history has had the same number of games.
    2. Within a season, teams regularly play an opponent that has not played the same number of games at they have at that point in the season.  Simply using season-long running-total values would skew things in favor of the team that had played more games for positive stats and in favor of the team that had played fewer games for negative stats.   By converting every statistic to a per-game value, we are comparing apples-to-apples.
2. __Deltas__ between the two teams in given statistics, resulting in a single statistic describing the relationship between the teams for the given game.   
    _Example:_ home team averages +103 more yards passing per game than the road team.
3. __Aggregation__ of similar statistics into a single metric.  
    _Example:_ find how many standard deviations a team is above or below league average in a given stat for a given week, and accumulate these for all relevant offensive or defensive stats into a single statistic, such as "Offensive Sigma".
4. __Binning__ of a given statistic into quintiles.
5. __Clustering__ of advanced metrics in an effort to identify "types" of game matchups.
6. __Dummying__ of categorical data, such as day of the week per game, type of stadium (dome or open), type of playing surface, and week of the season.
7. Calculating exact hours of rest between games, not merely days.  



<BR>
<BR>
<BR>



## NFL Betting Primer
Each bet has different odds depending on which side of the bet you take.  There are three primary types of wagers made on NFL games:
1. The spread
2. The over/under
3. The money line  

#### Interpreting Odds and the Payout
When a money line or spread is negative for a given team, this means that team is favored to win.  As such, the payout for that bet is less favorable than for the underdog.  All odds are given relative to a wager of $100.  An example is easiest to demonstrate.  If Team A is the favorite and has odds (money line) of -200, this means you must bet $200 to win (net) $100 (the $200 you originally bet plus $100 in winnings).  Since Team A is favored this makes sense -- you must risk more money in order to profit since they're expected to win.  Conversely, if Team B is an underdog and has odds of +300, you will win (net) $300 with a wager of only $100.  Again, Team B is not expected to win, so to entice bettors to take the bet, the reward must be greater than the risk.

#### The Spread
The spread, also called the "line", is a measure of how much better Vegas thinks Team A is than Team B.  Vegas sets the spread in the amount of points the favored team is expected to win by.  A negative spread indicates a team is favored, positive an underdog.  For example, a spread of -3.0 means the favored team is expected to win by a field goal (3 points).  You can bet on either team, the favorite or underdog.  In order to win a bet on the spread, your team must exceed the spread in your favor.  So, if you bet on the favorite at -3.0, they must win by _more_ than 3 points for your bet to win.  If they win by exactly 3 points, the result is called a "push", and all money is returned to bettors, none having been won nor lost.  

Even moderate sports fans are doubtless familiar with the notion of "home field advantage," and we see it borne out in the history of the Vegas NFL spread.  The peaks in the distribution represent the most common increments of scoring in football: 3 points, 7 points, 10 points, and 13 points.  Note the aversion to setting the line at 0 points, as this is equivalent to simply picking the winner outright.  Also note the significant majority of lines are set favoring the home team, offering real evidence of the notion of "home field advantage."

<img src="images/road_spread_dist.png" alt="History of the Spread">  

<sub>__Figure 1:__ The historical distribution of the Vegas spread for NFL games from the perspective of the visiting team.  Excluding the intentional dip at 0 points, the spread conforms to a roughly normal distribution with a mean close to +2.4 points. </sub>

<BR>

<img src="images/over-under_dist.png" width="600" align="right" alt="History of the Over/Under">  

#### The Over/Under
The Over/Under is simply the total expected number of points scored by both teams in a game.  You can bet the Over or the Under, and will win if the combined score of the teams is either more than (over) or less than (under) the set Over/Under value, depending on your wager.  If the final combined score equals the Over/Under value exactly, the bet ends in a "push" and all money is returned.  

<BR><BR><BR><BR>
<div align="right">
<sub><b>Figure 2:</b> The historical distribution of the Vegas Over/Under for NFL games. The mean is denoted
<BR>
by the small dashed line at 42.2 points. Again, we observe here a gaussian distribution. </sub>
</div>

<BR>

<img src="images/money-line_dist.png" width="600" align="right" alt="History of the Money Line">  

#### The Money Line
The money line is simply the odds that a specific team will win the game, regardless of margin of victory (spread).  The money line is given in odds and like the spread a negative value denotes the favored team, and the odds themselves indicate what the [payout](#interpreting-odds-and-the-payout) will be for a winning bet.  Again owing to the notion of "home field advantage", the average money line for a favored road team is -230, while the average for a home favorite is -313.  

<BR><BR>
<div align="right">
<sub><b>Figure 3:</b> The historical distribution of the Vegas Money Line for NFL games. Split much like
<BR>
the spread, the money line shows the higher value and density of home favorites.
</sub>
</div>  

<BR>

All things considered, you must risk more money when betting on a home team as they're expected to win more frequently.  This trend holds true for underdogs as well; you win more money from the average road underdog (+248) than the average home underdog (+179).



<BR>
<BR>
<BR>



## Wise Bets
Games that pass a user-set threshold of deviation from the model's prediction, either in a point spread or in odds to win, are labeled as __wise bets__.

A game whose actual spread deviates from the predicted spread by the user-set point threshold or more will be labeled a "wise bet".  The underlying approach to finding mis-valued spreads works as follows.  The key factor for a spread is its _flexibility_. As Vegas receives more bets on a particular team at a given spread value, they can adjust the spread in order to balance the wagers on the opposing team, reducing the bookmakers' risk by taking near equal money on both sides.  (Vegas typically does not win big on any given game.  They win small amounts consistently by playing percentages very carefully).  

This flexibility in the line is the key component I aimed to use in snuffing out inefficiency in the spread.  If the betting public has a possibly inaccurate perception about a given team, they will either over- or under-bet for that team, forcing Vegas oddsmakers to compensate by artificially adjusting the spread in order to entice bettors to make wagers against their (inaccurate) perception and even out the money wagered.  

Because of this, the initial aim of this project was simple: I wanted __to identify which factors best predict games that have spreads that are incorrectly set, to label these games as potentially "wise bets," and to examine the results of these games in hopes of finding that a favorable percentage would be winning bets.__

Secondarily, we can do the same for the Over/Under as well as the Money Line.  The Money Line is slightly different, since it is concerned only with the binary outcome of win/lose.








<BR>
<BR>
<BR>

## Model Selection
Four models were tested in this project: two tree ensemble methods, Random Forests (RF) and Gradient Boosted Trees (GBT), as well as Support Vector Machines (SVM) and finally ElasticNet regression.  The GBT models are from __XGBoost__ while the rest are from __Sci-Kit Learn__.

### Data Selection
As mentioned above, there were three divisions of the original dataset features, as well as up to five progressively smaller year ranges of games to inspect.

#### Feature Separation and Importance
The feature-set division arises from my desire to answer the following question: _which single statistics are the most important in predicting X result for an NFL game?_  

The easiest and most direct way to do this is to use a model which has a feature importance attribute.  Gradient Boosted Trees do, and so served this role in this project.  There are two ways of finding the feature importance in ensemble tree models like Random Forests and Gradient Boosted Trees: first, each model has its own attribute which will tell you which features gave the highest return in purity or lowest return in error for the single fitting and run of the model.  Second, a more robust approach is to use Recursive Feature Elimination (RFE), which is calculated by fitting the model with all but one feature, measuring how well it predicts, and then repeating this for all features in the dataset until each one has had a turn being left out.  The features that caused the greatest drop in prediction accuracy are judged to be the most important.

However, if any two features are closely related, the importance for either one will be largely negated by the existence of the other.  For example, pretend we have the two following statistics in this database: _touchdowns in the first three quarters_ and _touchdowns for the entire game_.  When the the stat for the _entire game_ is removed and the model measures how accurate its predictions are, it will still have a feature included that provides three-quarters of the removed _entire game_ stat's information, and the prediction accuracy won't be severely impacted.  As a result, the _touchdowns for the entire game_ statistic will be reported as not being a very important feature.  However, the reality is that this would indeed be an important statistic, but having a correlated or, in this case, partially duplicated feature clouds our ability to determine its true importance.

Because of this fact, I split the feature set into three parts:
1. All statistics
2. Raw statistics
3. Matchup deltas

_Raw_ stats are simply every team's own per-game statistics coming into the given game of interest.  Examples include touchdowns/game, 1st downs/game, turnovers/game, etc.  _Matchup deltas_ are the differences between the road and home teams in these respective stats.  So, the home team might average seven more first downs per game, but 1.3 more turnovers per game.  Since the matchup stats are technically derived from the raw statistics, I wanted to ensure proper evaluation of feature importances which led to their being optionally separated.  The top features for each target will be reported in the [Results](#results) section below.

Ideally, RFE allows you to trim the feature set of the model to only the most important variables in an effort to lower complexity and reduce variance.  This goal was not realistically achieved in this project, as any subset of N most important features (20, 40, 80) failed to match the accuracy of the full feature set, regardless of which subdivision (all, raw, matchup) of features was chosen by RFE.  

Regarding prediction, having more information tends to be better than having less, and we see that here as using _all_ the features did lead to the best prediction accuracy for any given model.  However, the difference between using all the data and only the matchup data was minor, commonly in the range of 0.5% to 1.5%.  The raw features alone were noticeably less predictive, sometimes giving up to 5% worse prediction accuracy.

#### Database Size
As noted above, the more advanced metrics do not extend back to the beginning of the recorded Vegas betting data.  In order to use data ranging back to 1978, the start of the Vegas data, we would have to exclude many statistics which begin in subsequent years.  So, this gives us realistically two options:
1. A 'longer' database that goes back to 1978, but is lacking any information-rich advanced metrics (called the _1978 database_).
    + 8,788 rows (games), 258 columns.  Total of 2,267,304 data points.
2. A 'wider' database that starts in 1994 but has all the info-rich metrics (called _Advanced database_ since all advanced metrics are included).
    + 5498 rows (games), 360 columns.  Total of 1,979,280 data points.  

Experiments with both databases showed the wider, Advanced database to give slightly better predictions in regression, up to 4% improved R<sup>2</sup> accuracy and a tenth of a point lower in MAE.  For predicting the winner of a game, this gap was near 0.5% in AUC but -1.3% in F1-score without SMOTE oversampling, and -3.8% in AUC and -4.7% in F1-score with SMOTE oversampling.  

This suggests that the information-rich advanced metrics more than make up for the loss of sample size when predicting the spread for a game but are either roughly equivalent to (without SMOTE), or do not compensate for the loss of (with SMOTE), fourteen extra years of data for predicting the winner outright.  Looking ahead, however, the advanced metrics will only continue to accrue and if they give better or roughly comparable predictions now with far fewer years of data, it would be sensible to use them going forward.

### Regression vs Classification
Since the Spread and the Over/Under are numeric, regression models were used to predict these targets.  Conversely, classification algorithms were used in modeling the Money Line binary winner/loser of a game.  

#### Regression
There are two possible regression targets: the _spread_ and the _over/under_ for each game. Each target is given in game points.  The spread extends back to 1978 while the over/under starts in 1979.

###### Regression Target Ranges
Target | Min. (abs) | Max. (abs) | Range
-------|------|---------------|----------
Spread | 0.0 | 26.5 | 26.5
Over/Under | 28.0 | 63.0 | 35.0

<sub>__Table 1:__ Range information for the Vegas Spread and Over/Under, from 1978-2016.</sub>

<BR>

#### Classification
Three classification targets are available:
1. Whether a team _covered the spread_ (won the spread bet)
2. Whether a game went _over or under_ the over/under point total
3. Whether a team _won or lost_ the game (the money line bet).  

The cover and over/under classes are very close to being ideally balanced, but the classes for a home team vs. a road team winning are slightly imbalanced.  Regardless of year range, the home team wins at roughly a 58% to 42% rate compared to the road team.  

This imbalance is not drastic, but does mean that stratification during train-test splitting to ensure both the train and the test splits receive an equal ratio of each class is wise.  Other class imbalance fixes attempted were using the cost-minimizing `'balanced'` class weighting in the Random Forest model, which made substantial difference in the efficacy of the model.  Alternatively, the oversampling SMOTE package by __imblearn__ was used for the GBC and SVC models.  Its impact was typically mild, but always positive, offering a few percentage points of improvement for the Receiver Operating Characteristic Area Under the Curve (AUC).  In particular, the Gradient Boosted Tree classifier by __XGBoost__ handled the 58/42 class imbalance rather well out of the box.  The full breakdown of classes is shown below for reference.

###### Class Ratios
Target | Data | Majority Class | Minority Class | Majority % | Minority % | Counts
-------|------|----------------|----------------|------------|------------|-------
Road Cover | 1978 | Yes | No | 51.4% | 48.6% | 4514 - 4274
Road Cover | Advanced | Yes | No | 51.7% | 48.3% | 2843 - 2655
Over/Under Result | 1978 | Over | Under | 50.6% | 49.4% | 4449 - 4338
Over/Under Result | Advanced | Over | Under | 51.0% | 49.0% | 2805 - 2693
Home Team Win | 1978 | Win | Loss | 58.2% | 41.8% | 5114 - 3674
Home Team Win | Advanced | Win | Loss | 57.9% | 42.1% | 3183 - 2315

<sub>__Table 2:__ Breakdown of classification targets within the two primary datasets, 1978-2016 and 1994-2016 (_Advanced_).  Only the __Home Team Win__ target has a moderate class imbalance.  </sub>

<BR>

### Model Selection and Training
In order to choose the models which performed best I optimized for the mean absolute error (MAE).  Compared to the root mean squared error (RMSE), the MAE is consistent across ranges of errors and doesn't 'flare' up in response to larger residuals.  For evaluating how many points a predicted NFL game's spread is  from the actual spread, there is no harsher penalty for being five points away than there is for being four points away.  There _are_ discontinuous jumps in importance of residual values, but these are _not_ progressively increasing with the value of the error itself.  Instead these recurring pronounced importance ranges are vestiges of the discrete scoring nature of football, with the vast majority of scores being multiples of three points or seven points, these being the two most common values of scoring plays (a field goal and a touchdown).  Using the absolute error ensures an easily interpretable metric for evaluating model accuracy with this data: a MAE of 3.5 means we have an average error of 3.5 points.

The two best regression performers were the Support Vector Machine and the Gradient Boosted Trees models, with the Random Forest a small step behind and the ElasticNet behind it.  Initial grid search cross-validation runs with the Gradient Boosted Trees regressor gave an overly optimistic result due to overfitting on the cross-validation set, with a drastically higher cross-validation score than subsequent test score.  This resulted in tweaking the parameters toward fewer trees, a medium tree depth, and a lower learning rate.  

For the point spread, both the SVM and GBT models had an MAE of 2.2 points (and an R<sup>2</sup> of near .740).  For the over/under, they converged to an MAE of 1.8 points and an R<sup>2</sup> of near .715.  This means that at our best, we can predict the point spread of an NFL game to within 2.2 points and the over/under to within 1.8 points.

The best performing model in all classification tasks was the Gradient Boosted Classifier. Classifying whether a team covered the spread or whether a game went over or under the over/under was not particularly responsive to model tuning.  Regardless of the model and its parameters, the AUC hovered close to an even score of 0.500.  I believe this is due to the nature of the categories -- the spread and over/under are designed by oddsmakers to be as close to the break-even point (50/50) as possible, to attract equal bets on both sides.  If anything, these results simply verify that Vegas is quite effective at calculating the expected margin of victory and total points scored per game, _en masse_.  Predicting the winner straight-up, however, is not a result contrived by Vegas and as such does have some appear to have some leeway in determining the outcome via machine learning.  Below are two tables summarizing the results of the model tuning and selection process.  

<BR>

###### Regression Outcomes
Target | Data | Model | Metrics | Score
-----|--------|-------|---------|-----
Game Spread | 1978 | GBR | MAE <br> R<sup>2</sup> | 2.41 <br> 0.699
Game Spread | Advanced | GBR | MAE <br> R<sup>2</sup> | 2.31 <br> 0.731
Over/Under Value | 1978 | GBR | MAE <br> R<sup>2</sup> | 1.85 <br> 0.716
Over/Under Value| Advanced | GBR | MAE <br> R<sup>2</sup> | 1.86 <br> 0.723  

<sub>__Table 3:__ The overview of results from the two regression models and targets in this project.

<BR>
<BR>

###### Classification Outcomes
Target | Data | Model | Metrics | Score
-----|--------|-------|---------|-----
Road Team Cover | 1978 | GBC | AUC <br> AUC (SMOTE) | 0.516 <br> 0.518
Road Team Cover | Advanced | GBC | AUC <br> AUC (SMOTE)  | 0.502 <br> 0.535
Over/Under Result | 1978 | GBC | AUC <br> AUC (SMOTE)  | 0.514 <br> 0.515
Over/Under Result | Advanced | GBC | AUC <br> AUC (SMOTE)  | 0.494 <br> 0.508
Home Team Win | 1978 | GBC | AUC <br> AUC (SMOTE)  | 0.710 <br> 0.775
Home Team Win | Advanced | GBC | AUC <br> AUC (SMOTE)  | 0.711 <br> 0.745

<sub>__Table 4:__ The overview of results from the three classification targets using combinations of datasets and class-balancing oversampling (SMOTE).


<BR>

### Class Inspection
#### Principal Component Analysis
One tactic when struggling to find viable class separation is to analyze your data with dimensional reduction.  A popular method of this type of dimensionality reduction is Principal Component Analysis (PCA), which uses some higher-level mathematics to reduce the input data to core, or principal, components based on the amount of observed variance along a given rotational axis of the data.  The result is _not_ simply a set of input features, but rather the 'fundamental' relationships -- components -- between the features and the variance in the data.  If there exists a way to mathematically represent the data in a way that makes it separable in N-dimensions, PCA can tell us.  We can select for the number of components we want returned, which makes PCA ready-made for 2D and 3D visualizations.  

The results of using PCA to analyze the initial target and driving force of this project, the spread, were not encouraging.  Using the model's predictions for the spread to label games as potential "wise bets" or not, PCA showed a inseparable blob in two dimensions.
![PCA 2D Spread](images/2d3pwisebetPCA.png "2D PCA results for the spread")

<sub>__Figure 1:__ The first two principal components failed to give any viable separation for wise bets derived from the Vegas spread -- there is no line that can be drawn to reasonably divide the two classes.  

<BR>

The classes are clearly inseparable in two dimensions, but what about three?  It is possible that there exists a hyperplane which can divide the classes in three dimensional space.  For example, picture in your mind the Great Pyramid at Giza, Egypt.  Pretend the limestone blocks that make up the pyramid are separated into two classes by being painted either red or blue.  Now, pretend the top fraction of the pyramid's peak is all red, and the rest of the structure is all blue.  We could divide the red from the blue blocks -- the classes -- by putting a massive sheet of, say, thin plywood, between them.  This sheet of plywood is called a _hyperplane_ and would perfectly separate the two classes of blocks, meaning we could predict mathematically whether a brick was in the red or blue class (no word yet on which class of block is filled with grain...).  

Now, imagine floating high directly above the pyramid and looking down upon it.  You'd see a smaller tip of red blocks in the center surrounded by blue blocks, because the pyramid itself would look like a two dimensional square, much the way mountains look 'flat' when you look directly down on them from a plane.  We would be wholly unable to divide the blue and red blocks in this flat, two-dimensional perspective.  This situation demonstrates the process of using PCA in two dimensions versus three dimensions.  Theoretically, PCA can be used for as many dimensions as there are features in your dataset, but we can only effectively visually represent it in two or three dimensions.

Unlike the simplistic pyramid example, applying PCA in three dimensions to the wise bets from the Vegas spread did not reveal any feasible hyperplane of separation.  

<img src="images/3d_pca_gifs1/3D_PCA.gif" width="600" align="middle" alt="3D PCA for Vegas Spread wise bets">

<sub>__Figure 2:__ Three dimensions -- each axis is a principal component -- are unfortunately not enough to find a hyperplane of sufficient division between games that are wise bets and games that aren't.  There is no underlying structure to the classes, here.  They're distributed in a roughly globular manner, and almost randomly so.  The hyperplane was obtained by using a linear SVM model.

<BR>


<img src="images/model_spread_wb_non-pca_tsne/epsilon50/tsne_wise_bet.gif" width="600" alt="t-SNE for Vegas Spread" align="right">  

#### t-SNE
A second dimensional reduction algorithm, or manifold learner, that is commonly used for visualization is t-distributed Stochastic Neighbor Embedding (t-SNE).  Unlike PCA, t-SNE doesn't provide a Rosetta Stone for translating data into its fundamental components.  Instead it seeks to find local groupings of one data point to its neighbors in high dimensions and visually represent them in lower dimensions, illuminating possible separability.  

<BR><BR><BR><BR>
<div align="right">
<sub><b>Figure 354:</b> The results of increasing levels of perplexity for t-SNE dimension reduction on the Vegas
<BR> spread bets. While there is eventual clustering, the classes never become linearly separable.</sub>
</div>


<BR>

t-SNE's results might change slightly every time it is run, it is very sensitive to its parameters, and cannot be used for inference about unused, new data.  With proper tuning, however, it can reveal grouped relationships which might tell you if your data is actually separable.  Like PCA, t-SNE failed to reveal any underlying structure that could be separated.

<BR>

#### Feature Overlaps
With no apparent real success in determining which games should be considered a wise bet, I decided to take a quick glance at the primary [advanced metrics](#advanced-metrics) which routinely are designated the most important features in the model.  The hope is to see horizontal (x-axis) separation, showing that there are distinct means or groupings for the two classes in a given statistic.  As with PCA and t-SNE, the results were not encouraging as both classes occupy very similar regions of each feature.

<img src="images/feature_overlap_vegas_spread.png" align="middle" alt="Important Feature Overlap" >

<sub>__Figure 4:__ The advanced metrics of DVOA, EPA, ANY/A, and PORT show very little horizontal separation for games that were labeled as a "wise bet" and those that weren't.  Ideally we would see a distribution of all blue and a separate one of all red beside it for a given metric.  Note: Axis tick labels are removed to help focus simply on bin separation, and the counts have been normalized since the raw count of "wise bet" games is a mere fraction of the total games.</sub>






<BR>
<BR>
<BR>



## Results
As mentioned in [Wise Bets](#wise-bets) above, the goal of predicting the spread and the over/under was be able to label games that had improperly set lines which could make them appealing bets.  This means we really wanted to regress against these targets in order to ultimately classify them.  Paired with the three classification targets, this results in the final goal for all models in this project being able to classify whether or not a game is one we should bet on.  A quick glance at [Tables 3](#regression-outcomes) and [4](#classification-outcomes) show a fairly pedestrian success rate at correctly predicting two of the three classification targets and a modest but not insignificant error on the regression targets.  

<BR>

### Spread Results
###### Spread Summary Statistics
Statistic | Mean (pts) | Std. Dev. | Coeff. of Var. | Min. (pts) | Max. (pts) | Min. Std. | Max. Std.
----------|:----------:|:---------:|:--------------:|:----------:|:----------:|:---------:|:--------:
Spread    |    2.58    | 5.89      |  2.28          | -23.0      | 26.5       |  -4.34    | 4.06     
Home MoV  |    2.87    | 14.6      |  5.07          | -46.0      | 59.0       |  -3.34    | 3.84   


<sub>__Table 333:__ Summary statistics showing the much wider spread of margin of victory, which is echoed in the rolling averages for each statistic.</sub>

<BR>

#### Spread Accuracy
If we can predict the line accurately, we can identify games that are improperly valued by Vegas and choose to bet those games.  The best result obtained was a MAE of 2.32 points.  The spread average is 2.58 points with a standard deviation of 5.91 points.  (See [Table 333](#spread-summary-statistics).  Unfortunately this means that, on average, any prediction's true value can be within a window of 4.64 points -- not great.  

If we predicted a spread for the road team an upcoming game to be +1.0 point (meaning they were a 1-point underdog), which is the 8th most common out of 47 unique recorded lines, then using our average error, the "true" spread of the game might ought to be 1.0 - 2.32 = -1.32 points, or 1.0 + 2.32 = +3.32 points.  In the case of -1.32 points, the road team would now be favored and would likely cause us to change our bet.  Conversely, in the case of +3.32 points, the home team would now be favored by _over_ a field goal, which is the easiest score to make in football and would likely change our bet.  

With this in mind, I decided to set the threshold for deciding how far off a game's spread was to a minimum +/- 3 points.  This ensured we would not select games that would 'flip' the favored team and that our prediction would be beyond the threshold of a field goal. Games that met or exceeded this threshold are explored in the [Wise Bets Results](#wise-bets-results) section below.

#### Spread Feature Importance
The most important features tend to be the _matchup delta_ features I engineered.  These tell the difference between the two teams in a game in a given metric.  My reasoning was that the raw value, say a high amount of rushing yards per game, would matter little for prediction of one team's superiority if the other team also had a high value in that metric.  If one team was significantly better than the other team in a given metric we would be able to make more accurate predictions.  This finding is shown consistently in this project as the _matchup_ features rank high in importance.

<img src="images/road_spread_feats.png" align="middle" alt="Important features to predict the spread" >

<sub>__Figure 400:__ The 40 most important features in predicting the spread are dominated by the _matchup_ features, including ten of the first twelve.  After the first 15 features, relative importance begins to level off with larger groupings of equally important features. </sub>

<BR>

#### Spread Analysis
The most important statistic is simply the difference in losses between the teams.  This isn't surprising, as the general public is going to base much of their betting on which team has a better record.  Much of the rest of the features are dominated by the [advanced metrics](#advanced-metrics), with _weighted DVOA_ (a measure of how well a team has been playing recently) being the second biggest predictor.  The first non-_matchup_ feature is simply the number of 3rd down conversions the visiting team is averaging per game.  The more 3rd downs a team converts, the more they extend their drives and the more likely those drives are to yield points.   

Apart from these metrics, it is interesting to see _hours of rest_ be so statistically significant.  Vegas is clearly factoring in how long it has been since a team last played into its formula when setting a spread.  The _season_ in which a game is played also has an impact on the spread, which surprised me.  So, I took a look at the spread with regard to change over time.

<img src="images/road_spread_rolling_avg.png" align="middle" alt="Historical Rolling Average for the Spread" >

<sub>__Figure 400:__ The spread has changed measurably over time.  The rolling average was computed with a window of four years and a second order polynomial fit line.

<BR>

I was unsure if the observed change is due to Vegas becoming better at prediction and improving their formulae, or if it reflects a change in the league itself, where home teams were more successful in the mid-1990s.  So -- you guessed it -- I decided to look at the historical averages for home team _margin of victory_.  

<img src="images/home_mov_rolling_avg.png" align="middle" alt="Historical Rolling Average for Home Team Margin of Victory" >

<sub>__Figure 400:__ The average margin of victory for home teams isn't smoothed very well with a four year rolling average window.  Its trend mirrors that of the Vegas spread overall, but has pronounced differences in any given year.

<BR>

We see that overall, the trends follow the same general path of peaking in the early- to mid-1990s and falling thereafter.  But a year-by-year inspection shows significant discrepancies.  Take 1995, for example.  It was the season of the lowest average home margin of victory for 30 years, but that year and the one after both saw Vegas _increase_ the average spread in favor of the home teams!  In general I suspect the average margin of victory has too much variance for Vegas to react with knee-jerk spread dampening or inflating.  While margin of victory trends over time are informative, they are clearly not the sole explanation for the changes with time in the average spread.  We can see the summary statistics in [Table 333](#spread-summary-statistics) back up what the graphs above show us.


<BR>
<BR>


### Over-Under Results
###### Over/Under Summary Statistics
Statistic | Mean (pts) | Std. Dev. | Coeff. of Var. | Min (pts) | Max (pts) | Min Std. | Max Std.
----------|:----------:|:---------:|:--------------:|:---------:|:---------:|:--------:|:-------:|
Over/Under |    41.6   | 4.58      |  0.11          | 28.0      | 63.0      | -2.97    | 4.67    |

<sub>__Table 33:__ Summary statistics showing the much wider spread of margin of victory, which is echoed in the rolling averages for each statistic.</sub>

<BR>

#### Over-Under Accuracy
Predicting the Over/Under is a bit easier for the model, as the data is more tightly clustered around its mean than the Vegas Spread. (See [Table 33](#over/under-summary-statistics)).  The lowest MAE for predicting the Over/Under was 1.86 points.  As with the spread, if we consider the range this gives us for prediction, we have a 3.72-point window.  However, unlike the spread, where a scoring play can be good (if made by the team we've bet on) or bad (if made by their opponent), all scoring plays for an Over/Under bet are either good (if we bet the Over) or bad (if we bet the Under).  

This allows us to use the 1.86-point MAE as our error instead of the 3.72 window.  If the real Over/Under is 43.0 points and our predicted Over/Under is 44.0 points, and we choose the Over, we are not worried about the +1.86-point error in prediction since we are already expecting more than 44 points to be scored. So 44.0 + 1.86 = 45.86 points for the upper bound of prediction is actually _better_ for us, since this says the game should go even further over. The reverse is true for betting the Under.

By far the most obtainable scoring play in football is the field goal, which is worth three points.  As such, it reasons any predicted Over/Under that is +/- more than three points different than the actual Over/Under should be considered a potential "[wise bet](#wise-bets)". If it is beyond this threshold with the error taken into account, even better. See [Wise Bets Results](#wise-bets-results) below.

#### Over-Under Feature Importance
Unsurprisingly the most important features for determining the combined points scored in a game are statistics that relate to how effective a team is at scoring or preventing a score.  We see a strong divergence from important features in predicting the spread, with no _matchup delta_ metrics present.  This fits common sense as we aren't concerned with how much better Team A is than Team B at something, but rather how good or bad both teams combined are.  

###### Over-Under Top Features
Rank | Statistic | Importance (%) | Rank | Statistic | Importance (%)
-----|-----------|:--------------:|------|-----------|:-------------:
1 | Home Pts For/Gm | 6.10   |  9        | Temperature          | 1.89
2 | Road Pts For/Gm | 5.81   |  10       | Wind (mph)           | 1.74
3 | Season          | 4.64   |  11       | Road Off PassTD/Gm   | 1.60
4 | Home Pts Allow/Gm | 3.48 |  12       | Road Wtd. Def DVOA   | 1.60
5 | Road Pts Allow/Gm | 3.34 |  13       | Road Def TotYd/Gm    | 1.45
6 | Road Off TotYd/Gm | 3.05 |  14       | Home Off TotYd/Gm    | 1.45
7 | Home Def TotYd/Gm | 2.18 |  15       | Road Off PassYd/Gm   | 1.31
8 | Wind Chill        | 1.89 |  16       | Roof Dome            | 1.31


<sub>__Table 99:__ Top 16 features for predicting the Over/Under show many stats related to scoring and yardage, but also surprisingly temperature, wind, and whether or not the game is played in a dome.

<BR>

#### Over-Under Analysis
The Over/Under has been climbing since the year 2000.  Initially the change was incremental, but over the last decade the Over/Under has exploded, setting a new record five-year averge for ten of the last eleven years!  I explore some possible causes for this growth below, starting with features that had the biggest impact on predicting the Over/Under.

<img src="images/over-under_rolling_avg.png" align="middle" alt="Historical rolling average of Over/Under" >

<sub>__Figures 411:__ The Over/Under has been on an upward climb since 2000, and has especially skyrocketed the last ten years.

<BR>

Looking at [Table 99](#over-under-top-features), the two most important statistics are the two we'd hope to see: how many points each team scores per game.  Following that is a surprising result -- the season!  This sparked me to investigate the Over/Under change over time as I did with the spread above.  It is examined below.  The remainder of the important statistics can be categorized as either "team related" or "game related".  The team-related statistics are sensible, related to how many points allowed and yards teams average.  But the game-related features are interesting and worth a quick word.

Wind chill and temperature only differ below 50° F, so seeing them paired is partly a consequence of their having the same information for all temps above 50° F.  I think there is also a relationship between the weather variables and the roof variables.  First, a quick graphical glance, then my thoughts below.

<img src="images/temp_with_domes.png" align="middle" alt="Temp distribution with domes" >

<sub>__Figure 4005:__ The distribution of game-time temperatures from 1978-2016 show an expected distribution, except for the occurence of dome games which spikes the count for 67 °F.

<BR>

The poignant aspect of the temperature charts is the towering prevalence of games at 67 °F, the temperature of games in played in a dome.  (See [Acquisition and Error Correction](#acquisition-and-error-correction) for details on this).  Around 21% of all games played from 1978-2016, but these aren't equally distributed across that time span.  Domes have become increasingly popular in recent years.  Because of this, I wondered if there was a possible connection between the increase in games played in domes and the increase in the Over/Under.  Behold!

<img src="images/percent_games_domes.png" align="middle" alt="Percent of games per year in a dome" >

<sub>__Figures 410:__ The trend in percent of games played in a dome is clear: more, more, more.  This trend also mirrors the increase in Over/Unders set by Vegas.

<BR>

Though I am unsure of the cause of the slight dip in the mid-200s (it's possibly due to temporary outdoor stadia being used while newer, domed stadia were being built), the overall trend of an increased percent of games being played in domes is obvious.  Currently one-quarter of the league's stadia are either domed or have retractable roofs.  Once the forthcoming Las Vegas Raiders finish building their new stadium in Nevada, nine of thirty-two teams will have the potential for a roofed stadium. <sup id="a1">[__5__](#fn5)</sup>

Considering this impact on the Over/Under, recall that feature importance only tells us if having either _more_ or _less_ of the given feature increases the prediction of the model, not which one. With this in mind, I propose the following explanation for the apparent value of the weather-related and dome features in predicting the Over/Under: in the NFL, successfully passing the football is the catalyst for consistent scoring. <sup id="a1">[__6__](#fn6)</sup> Extreme weather (very high/low temperature, very windy, rain, snow) adversely affects the passing game more than the rushing game.  In a rainy game, for example, teams will run much more than pass because the ball is slippery, making it both hard to throw and to catch.  This decrease in passing will lead to a decrease in combined scoring.  

But there's no bad weather in a dome.  So, the increase in domes means an increase in the number of games that are guaranteed to have good conditions, and a decrease in the number of games which can have bad ones.  With this reasoning, more dome games should equal more scoring.  There are also a handful of factors which influence the rise of domes, including the ability to draw fans on bad weather days, as well as what might be perceived as a competitive advantage for the home team by gearing their offense to more finesse passing (debatable).  Regardless of origin, the larger trend between incresed number of dome games parallels the increase in the Over/Under set by Vegas.  

While the relationship between an increased number of dome games and the increased Over/Under makes sense and is worth further investigation, there are other reasons which have undoubtedly contributed more to the increase in Over/Under values, primarily the increase in league-wide passing rate and efficiency <sup id="a1">[__6__](#fn6)</sup>, as well as what are perceived to be more "pro-offense" rule changes in the last fifteen years.  With this in mind, I took a quick look at how offense has changed in the NFL over time.  

###### Passing and Rushing Offense Over Time

<img src="images/combined_pass_rush_yard_rolling_avg.png" align="middle" alt="Passing vs. Rushing Yard Trends over time" >



<sub>__Figures 4193:__ Five-year rolling averages of Passing offense (left), which has grown at an alarming rate over the last decade-plus, and rushing offense (right), which dropped precipitously two decades ago and has somewhat stabilized since.</sub>

<BR>

These two rolling average plots of passing offense (left) and rushing offense (right) since 1978 show starkly different trends in league-wide offensive approach.  In 1978 and 1979 major rule changes were implemented that made pass defense more difficult, causing the first initial rise in passing offense.  To my surprise, it leveled off and remained consistent for the remainder of the 1980s and 1990s.  Beginning in 2005, however, the league began to experience its own Cambrian explosion of passing attacks, growing each year for a decade straight.  Conversely, rushing offense plummeted for fifteen consecutive years before coming to a roughly stable resting point.  

As intersting as the topic of how the league changes schematically as a whole over time is, the point of this investigation was to see if we could explain the dramatic rise in Over/Unders.  I think its safe to say we can do so to a large degree with passing offense alone, as we predicted above.  Take a minute to compare [Figure 411](#over---under-analysis) and [Figure 4193](#passing-and-rushing-offense-over-time).  The sharp rise in passing offense parallels that in the Over/Under, while rushing offense seems to have little to no correlation.  When comparing multiple variables at a time, a scatter matrix can help illuminate trends between all combinations of the targets.

###### Offense and Over-Under Scatter Matrix
<img src="images/pass_rush_over-under_matrix.png" align="middle" alt="Scatter Matrix for Passing, Rushing, and Over/Under" >

<sub>__Figure 4443:__ Scatter matrix of five-year rolling average passing offense, rushing offense, and the Over/Under per season since 1978.  Passing offense is clearly positively correlated with the Over/Under, while rushing offense has almost no observed correlation.</sub>



<BR>

At a glance we can see evidence of what we suspected above -- passing offense is strongly and positively correlated with the Over/Under, while rushing offense apparently bears no correlation to it.  Recall the Over/Under is simply the combined total points in a game, and as mentioned above, passing efficiency is the primary means to offensive success and higher scoring (see [Brian Burke's article](https://fifthdown.blogs.nytimes.com/2010/08/31/why-passing-is-more-important-than-running-in-the-n-f-l/) about passing offense efficiency which was written in 2010, right as the furious growth in passing offense began.  His claims would be only more concrete and emphasized if the article were written today).  The data here only bolster this conclusion: more passing means more scoring, all things equal.  But if _everyone_ is better at passing league-wide, then would it not competitively cancel out for everyone?  This question -- which statistics correlate to actually _winning_ -- is one we'll explore a bit more in the next section, where we analyze the Vegas money line.

<BR>
<BR>

### Money Line Results
As noted above in the [NFL Betting Primer](#nfl-betting-primer) section, the money line is merely the odds of Team A beating Team B.  As you might expect, the odds of Team A beating Team B dovetail very nicely with the spread, which is a measure in points (not odds) of how much better Vegas believes Team A is than Team B.  In fact, the two are so closely linked that they are for practical purposes equivalent in significance.  The R<sup>2</sup> correlation between a game's spread and its money line is __0.975__!  


###### Spread and Money Line Linear Relationship
<img src="images/spread_vs_moneyline.png" align="middle" alt="spread_vs_moneyline" width="800">

<sub>__Figure 4443:__ The relationship between the spread and moneyline looks to be perfectly described by a 3rd degree polynomial (not pictured), but a closer look at the _density_ of the spread values shows that over 94% are concentrated between -10 and +10.  This range comprises the center of this plot, and is easily desribed by a linear regression.</sub>

<BR>


<img src="images/spread_vs_moneyline_rolling_avg.png" align="right" alt="spread_vs_moneyline_rolling_avg" width="600">

###### Spread and Money Line Rolling Average
The point to take home is that the spread and the money line are essentially two ways to report the same metric -- how much better one team is than another.  Because of this, regressing against the money line is essentially the same as regressing against the spread.  Results are nearly identical in both feature importance and accuracy, as expected.  As a result, no further testing or modeling was done on the money line.  The remainder of the project was focused on examining how well the results of games labeled "wise bets" turned out.

<BR><BR>
<div align="right">
<sub><b>Figure 4444:</b> Further illustrating the near-identical relationship between the spread and the money
<BR>
line is their five-year rolling averages which appear as nearly carbon copies of each other.</sub>
</div>

<BR>
<BR>
<BR>

### Predicting Bet Outcomes
#### Covers and Over/Unders
Two of the three classification targets, _road covers_ and _over/under results_ proved to be very difficult to tease anything meaningful out of.  My first hunch is that this is due mostly to what the classes are derived from: betting lines set by Vegas.  As discussed in the [NFL Betting Primer](#nfl-betting-primer) section, Vegas has a vested interest in making the results of any bet as close to 50/50 as possible, in order to avoid suffering massive losses.  Vegas takes small wins many times over to turn large profits.  

Since the spread and over/under are set by Vegas to meet this balanced criterion, it is not surprising to find that "hidden gem" metrics which predict the results of these bets were not discovered.  If something does a great job of predicting these results, then Vegas knows about them already -- this is their livelihood, after all -- and is using them in their formulae to set these values.  In a sense, we are merely trying to tease out something Vegas has already baked in.

The reults for predicting these two bets never got much above 51%.  Remembering that the initial motivation of this project was to build a model to predict the spread, identify games that were beyond a spread-deviation threshold, and investigate to see if they were truly "wise bets".  As you can likely guess, the results for this approach were sub-optimal.  That does not make them uninteresting, however, and they are discussed below in the [Wise Bets Results](#wise-bets-results) section.

#### The Winning Team
Recall from [Table 2](#class-ratios) that the lone imbalanced class was the _Home Team Win_ class, which is simply a binary class of "yes" or "no" results depending on if the home team won ("yes") or lost ("no").  Also recall that home teams win at a 58% rate over time, meaning the class is 58% 'yes' and 42% 'no'.  Methods of addressing this moderate imbalance were discussed in the [Classification](#classification) section as well.  Ultimately I decided to use the _advanced_ database and to not use SMOTE oversampling for class psuedo-balancing.  

While using SMOTE did improve the _Home Team Win_ predictions, I didn't like the idea of 'articifial' games being used in the database because it made extrapolating results to the actual games that occurred more difficult.  The idea of having 'false' games included if I were to use a plot detailing the breakdown of how a specific metric impacts the chances of a team winning was unappealing.  If I decided to use only the 'true' games, would the trends match up to what was claimed by the SMOTE-driven results?  Having the luxury of a Gradient Boosted Classifier that handled the 'imbalanced' classes rather well (as noted in [Table 4](#classification-outcomes)) made the decision easier, admittedly.

#### Winning Team Accuracy
When our model makes a prediction that the home team won a game it has labeled the game as a "positive" (e.g. "yes").  If the game it labeled as a positive actually was by the home team, the prediction would then be considered a "true positive".  However, if the game predicted by the model to have been won by the home team was actually won by the visiting team, it would have incorrectly labeled the game as a positive, resulting in a "false positive."  No model will get every prediction right (in the real world), so we need a way to measure how accurate and reliable our model is.  One method, developed during World War II by the British, is the ROC curve.

The ROC tells us the ratio of how many true positives to false positives our model is predicting.  We do this for every threshold of 'confidence' in the prediction, from 1% to 99% confident.  The ratio changes as the threshold changes, and we plot these changes to form a ROC curve.  Random guessing would equate to a 50/50 split of true positives to false positives, so this is our baseline.  Anything below that is _worse than random chance_.  The higher the true positive rate, the better the model.  The actual value we use to evaluate the model is technically the area under the curve (AUC).  As in model tuning, we use a split subsample, or fold, of the data to test how well it performs on each split and average them together.  The ROC curve for predicting whether the home team wins a game is below.

<img src="images/roc_home_winner.png" align="middle" alt="ROC Curve for Home Winner">

<sub>__Figure 5544:__ The ROC curve for predicting if the home team will win.  Some deviation is present between each fold of data.</sub>

<BR>

Any time we have a classification model, we should also examine the confusion matrix for our predictions.  This is another way to visualize how accurate our model is.


<img src="images/cmat_home_winner.png" align="middle" alt="Confusion Matrix for Home Winner" width="600">

<sub>__Figure 55874:__ A confusion matrix of the prediction results for classifying games for the home team being the winner.  The model errs on the side of considering more games to be won by the home team than actually are.</sub>

<BR>
**Proba**





<BR>
<BR>

### Wise Bets Results









Below are the results for each of the five Vegas-related targets investigated in this project.  

__90.4%__ of all spreads are <= +/- 10.



<BR>

## Future Considerations








<BR><BR><BR><BR><BR><BR><BR><BR><BR><BR><BR><BR><BR><BR><BR><BR>




### References:
<a name="fn1">1</a>: http://www.nbcnews.com/news/other/think-sports-gambling-isnt-big-money-wanna-bet-f6C10634316  
<a name="fn2">2</a>: https://www.inc.com/slate/jordan-weissmann-is-illegal-sports-betting-a-400-billion-industry.html  
<a name="fn3">3</a>: https://www.boydsbets.com/super-bowl-how-much-bet/  
<a name="fn4">4</a>: http://mentalfloss.com/article/26730/how-wind-chill-calculated  
<a name="fn5">5</a>: https://en.wikipedia.org/wiki/List_of_current_National_Football_League_stadiums  
<a name="fn6">6</a>: https://fifthdown.blogs.nytimes.com/2010/08/31/why-passing-is-more-important-than-running-in-the-n-f-l/
