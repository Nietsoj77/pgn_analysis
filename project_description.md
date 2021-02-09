# Estimating playing strength in chess
This analysis aims to produce a model that is capable of predicting a player's strength in terms of rating, based on the information from a game.

## About the dataset
The data for this project is from a collection of games played on one of the largest chess sites lichess.org.<br><br>

The dataset consists of games from a large online tournament, the *yearly classical arena*, that was played on May 15th 2020. About 16,000 games were played in the tournament. The data can be downloaded via the following link.

> https://lichess.org/api/tournament/Y4t9Lk9R/games?evals=true

### Presentation of chess games - the PGN format
The standard for presenting chess games on a computer is what is known as PGN files (Portable Game Notation). These files consists of a number of tags and a sequence of moves. The tags are presented in square brackets, and moves are presented as a string where each move is numbered. Here is an example:

```
[Event "Rated Bullet tournament https://lichess.org/tournament/yc1WW2Ox"]
[Site "https://lichess.org/PpwPOZMq"]
[White "Abbot"]
[Black "Costello"]
[Result "0-1"]
[UTCDate "2017.04.01"]
[UTCTime "11:32:01"]
[WhiteElo "2100"]
[BlackElo "2000"]
[WhiteRatingDiff "-4"]
[BlackRatingDiff "+1"]
[WhiteTitle "FM"]
[ECO "B30"]
[Opening "Sicilian Defense: Old Sicilian"]
[TimeControl "300+0"]
[Termination "Time forfeit"]

1. e4 { [%eval 0.17] [%clk 0:00:30] } 1... c5 { [%eval 0.19] [%clk 0:00:30] }
2. Nf3 { [%eval 0.25] [%clk 0:00:29] } 2... Nc6 { [%eval 0.33] [%clk 0:00:30] }
3. Bc4 { [%eval -0.13] [%clk 0:00:28] } 3... e6 { [%eval -0.04] [%clk 0:00:30] }
4. c3 { [%eval -0.4] [%clk 0:00:27] } 4... b5? { [%eval 1.18] [%clk 0:00:30] }
5. Bb3?! { [%eval 0.21] [%clk 0:00:26] } 5... c4 { [%eval 0.32] [%clk 0:00:29] }
6. Bc2 { [%eval 0.2] [%clk 0:00:25] } 6... a5 { [%eval 0.6] [%clk 0:00:29] }
7. d4 { [%eval 0.29] [%clk 0:00:23] } 7... cxd3 { [%eval 0.6] [%clk 0:00:27] }
8. Qxd3 { [%eval 0.12] [%clk 0:00:22] } 8... Nf6 { [%eval 0.52] [%clk 0:00:26] }
9. e5 { [%eval 0.39] [%clk 0:00:21] } 9... Nd5 { [%eval 0.45] [%clk 0:00:25] }
10. Bg5?! { [%eval -0.44] [%clk 0:00:18] } 10... Qc7 { [%eval -0.12] [%clk 0:00:23] }
11. Nbd2?? { [%eval -3.15] [%clk 0:00:14] } 11... h6 { [%eval -2.99] [%clk 0:00:23] }
12. Bh4 { [%eval -3.0] [%clk 0:00:11] } 12... Ba6? { [%eval -0.12] [%clk 0:00:23] }
13. b3?? { [%eval -4.14] [%clk 0:00:02] } 13... Nf4? { [%eval -2.73] [%clk 0:00:21] } 0-1
```
### About playing strength
Playing strength in chess is usually measured by Elo rating. A beginner usually has a rating of about 1000-1200, whereas the top players in the world have ratings in the high 2700s and above. The Elo rating is a robust indicator of playing strength, but it changes quite slowly, at the most about 10 points per game. Therefore, players who do not play very frequently may seek other indicators of playing strength development.

When computers assess a chess position, they usually estimate a point count often denoted as *centipawns* (CP). In the pgn example above, the scores are presented as full pawns (not centipawns) following the %eval tag. One centipawn is the positional equivalent of one 100th of a pawn. A positive CP value indicates an advantage for White, and a negative value indicates an advantage for black. Large changes of the evaluation indicates poor moves. A change of 100 CP is usually a mistake. A change of 150 or more is usually a serious mistake (blunder). A mistake larger than 300 CP is usually enough to lose a game at a reasonable level of play.

When analyzing a chess game with a computer, a frequently used statistic is the average change of the CP score, average centipawn loss (aCPL). This is often assumed to indicate playing strength. 

## Data exploration
The purpose of this project is to identify variables that can predict playing strength. The estimate for playing strength (dependent variable) will be the players' actual ratings. The assumption is that the following variables may contribute to predicting playing strength. They will therefore be used as predictors (independent variables):
- Average centipawn loss (aCPL)
- Rating difference between players
- Number of moves played
- Number of blunders (big mistakes)
- Blunder rate (number of blunders / number of moves)

### Data wrangling
The dataset is not in a format that can easily be analyzed, so a bit of preparation is needed. The wrangling was done in four steps:

1. Read the pgn file, and extract information
2. Select annotated games (that have computer evaluations)
3. Put the information in a dataframe
4. Calculate values/columns for the predictors

After this processing, the final dataset consisted of roughly 4,600 games.

The pgn format cannot be read directly by pandas, so extracting the information required additional tools. There are several python packages that can read and handle pgn files, but they are written for different purposes. A very common package is chess.pgn, which can iterate over the games and present the moves. However, it can apparently not present the computer evaluations in an easily accessible way. I therefore decided to write my own functions to locate the information and extract it for the purpose of this analysis. The main tool for this is regular expressions.

Tags in the pgn files are presented as square brackets and a keyword. The information is given in quotation marks. The regex is therefore:
```python
key = r"\[(\w+)"
val = r"\"(.+)\"" 
```
Similarly, scores are preceded by the %eval tag, so the regex for the scores is:
```python
score = r"eval (-?[0-9]+\.[0-9]+)"
```
### Initial findings
From the exploration phase, the following observations were made:
- The average rating is about 1770
- The average CPL is just below 120, which is surprisingly high
- The median CPL is considerably lower than the average, indicating an asymmetrical distribution
- On average, a game lasts about 28 moves
- On average, a player blunders 4 times during a game, or about 15% of the moves
- The stats are almost identical for white and black
- Correlation between variables is not very strong

## Data analysis
In the analysis phase, a number of methods were used to investigate the relationships between the variables. First and foremost, the relationship between centipawn loss (CPL) and player rating was investigated with regression plots and contour plots. As the CPL values seem to follow an exponential distribution, a transformation was made using the quadratic root of the values. This gave a better fit between the two variables. Since the values for white and black were practically identical, the white side was used for exploration purposes.

### Linear regression
Several attempts were made in this phase, and several models were discarded.
- Rating vs CPL gives a low R2 (0.082).
- Rating vs CPL, Rating diff gives a higher R2 (0.302).
- Rating vs CPL, Rating diff, number of moves gives even better R2 (0.371).
- Rating vs CPL, Rating diff, number of moves*blunders (interaction) gives the best R2 so far (0.410), but high condition no.
- Rating vs CPL, Rating diff, number of moves, blunder rate gives a slightly lower R2 (0.418), but also a high condition no.
- Rating vs CPL, Rating diff, number of moves, number of blunders gives a good R2 (0.401) and an acceptable condition no.
- The transformed CPL values gave only a slightly better result, so for easier interpretation, the original CPL values are used.

After the initial test (white values only), the complete dataset was used to test the regression model.

The R2 value dropped slightly from the initial model (0.40 to 0.39), but the results were the same overall. All parameters are statistically significant, and we arrive at the following formula:<br><br>
**Rating_pred = 1655 - 0.20*CPL -0.45*RatingDiff + 8.55*nmoves -22*nblunders**
<br><br>
The parameter for CPL suggests that each CPL is worth only 0.2 rating points, and that the number of blunders is the main predictor with 22 rating points per blunder (threshold 1.5) along with the number of moves.

### Validating the model
The linear model is based on predictions from individual games. Anyone who has played chess knows that a player can play well in one game and horribly in another. Therefore, estimating playing strength from a single game is not feasible. A larger set is required to have any sort of validity. I therefore downloaded a pgn of my own games to use as reference. These games were processed with the same approach as described above, and the parameters of the linear model were applied to the dataset to predict the playing strength.

The result of this analysis is that the average of the predicted ratings was about 100 points higher than the ones observed from the games. However, the standard deviation was larger than this difference, which indicates that there is no statistically significant difference between the values.

Another test was performed to check the crude estimate of playing strength from CPL alone. This prediction was completely unreasonable, as some games produced negative values, and the variation was simply too large (stdev > 1,000) for the prediction to be useful.

## Conclusions
This project has shown that it is possible to estimate playing strength from the information in pgn files. However, estimating playing strength from a single game is not reasonable. Neither is using centipawn loss (CPL) alone as a predictor. This project has resulted in a regression model that seems to be capable of estimating playing strength. However, the model is fairly complex for everyday use.

## Resources
- Alliot, Jean-Marc (2017) "Who is the Master?" in *ICGA Journal*, vol 39, no. 1, pp. 3-43
- Coulombe, Patrick (www) "Chess Analytics: Predicting rating from average centipawn loss", https://chessvillage.blogspot.com/2017/11/chess-analytics-predicting-rating-from.html
- Lichess.org (www) "Lichess.org API reference (2.0.0)", https://lichess.org/api
- McKinney, Wes (2017), *Python for Data Analysis* (2nd ed), O'Reilly
- VanderPlas, Jake (2016), *Python Data Science Handbook*, O'Reilly



