---
title: Predicting CS:GO match winners with logistic regression
date: 2017-02-09 09:27:00 +03:00
categories:
- experiments
- data science
tags:
- python
- games
- classification
comments: true
---

One day I felt I could do something interesting with my data science skills.
I thought: what if I can predict cs:go match winners?
I have never been into esports so I was going into this blind, not knowing what features to look for. 
I welcome you to come along my path in this post.
I calrify this is not a tutorial, rather a story I want to share.

>__For the lazy__
>You can find the model evaluation results at the bottom.
>The csv files you can play around are also at the bottom.

My workhorse was *Python 3.5*
Libraries used: *asyncio*, *aiohttp*, *beautifulsoup*, *pandas*, *sklearn*.

Steps taken, in short:
1. Scraping CS:GO matches from a website, using *asyncio*, *aiohttp* and free proxies to download a lot of web pages.
2. Parsing HTML with *beautifulsoup*, turning collected data into a *csv* file using *pandas*.
3. Extracting features from the collected matches.
4. Training and validating a *logistic regression* model using *sklearn*.

## Scraping
I needed retrospective data.
I found a website that had an archive of csgo matches dating as far as 2012.
19470 html pages containing info about a CSGO match.

It didn't take long to write a script to concurrently download the pages. Concurrency was a implemented using *asyncio* and *aiohttp*. These things are lovely.
> __Side note on web scraping and proxies__
>The first attempt to download that many pages got my IP banned from the site forever. I wrote a script to scrape free proxy lists from the internet and check proxies. Then funnelled requests through them. I looked up to this example: [Multithreaded web scraper with proxy and user agent switching](http://codereview.stackexchange.com/questions/107087/multithreaded-web-scraper-with-proxy-and-user-agent-switching).  The major difference is that I used *asyncio* for concurrency .
>Later I found out about [proxybroker](https://pypi.python.org/pypi/proxybroker/). If you need to make requests with proxies I advice you to use it. It's poorly doumented though so drop me an email if you need help.

I made separate scripts to download the pages and to parse them. It seems intuitive to parse each page right after getting it's html and avoid saving 1.5 GB of html files in a folder. But consider this: what happens if you parse the pages, but later decide to change your parsing code? For example you could initially think that match maps were irrelevant, but later decide that you need that information. If you didn't save the pages you would have to download everything again. If you have the pages saved you can simply alter the parsing code.
To sum it up: when you need to scrape and parse make two scripts, one to scrape, one to parse.

## Parsing
Each match page contained various info. I chose to extract the following:
`date, match_id, team1, team2, team1_score, team2_score, map1, map1_score, map2, map2_score, map3, map3_score`.

`team1` and `team2` are team names. 
I had to preprocess all team names because they had a lot of garbage. Someone had to add a `'` character to their team name. I wonder why nobody thought of adding `;` in the middle of their team name, just to make my life hell.

Here is my team name processing code:
```python
def preprocess_team_name(tname):
    return re.sub(r'[\t\s\[\]\'\.!]','', re.sub(r'[^\x00-\x7f]',r'', tname.strip().lower()).replace(' ', '-'))
```
However ugly it is, it does the following:
1. Reaplce spaces with `-`
2. Lowercase, strip trailing spaces
3. Remove all non latin letters
4. Remove certain blacklisted characters
Example:
```python
>>> preprocess_team_name("vg-сукаzen1999l33t'][")
'vg-zen1999l33t'
```
(All similarities to real team names are accidental and unintended.)

CS:GO matches can be of three types: best of one, best of two, best of three. In best of one two teams compete on one map, the one who wins the map wins the match. In best of two teams compete on two maps, and if each team wins one map, a third map is played to decide the winner. As for best of three, I leave it as an exercise for you to guess what that means.


`map1` and `map1_score` are always present, `map1` is the map name and `map1_score` is the score of format `{team1_score}:{team2_score}` (for example `10:16`)
`map2` and `map2_score` are same as above, but if the match type is best of one take `NaN` values. 
`map3` and `map3_score` are same as above, but if the match type is not best of three take `NaN` values. 

Here's a quick glance at the data:

| date | match_id | team1 | team2 | map1 | map1_score | map2 | map2_score | map3 | map3_score |
| -- | -- | -- | -- | -- | -- | -- | -- | -- | -- |
| 22nd of september 2012 | 8864274 | x6tence | 34united | mirage | 16:9 | NaN | NaN | NaN | NaN |
| 22nd of september 2012 | 8864275 | nip | bemyfrag | mirage | 16:1 | NaN | NaN | NaN | NaN |
| 22nd of september 2012 | 8864276 | nip | x6tence | nuke | 16:7 | NaN | NaN | NaN | NaN |

For convinience during processing I also added the `mtype` column. `mtype` is a categorical variable of match type, it can be `bo1`, `bo2`, `bo3` . The value is derived from `map1`, `map2`, `map3`. I split `map1_score` into separate columns `map1_score1` (team 1 score on map 1) and `map1_score2` (team 2 score on map 1).

## The Y
I had a classification task at hand.
The outcome variable, the class to predict, was choosen as such:
class 1: team number 1 won
class 0: team number 2 won or a tie (in other words: team number 1 didn't win)

I wrote a function to get the match winner from a pandas dataframe row:
```python
def get_winner(row):
    mtype = row.type
    if mtype == 'bo1':
        sum_score1 = row.map1_score1 
        sum_score2 = row.map1_score2
    elif mtype == 'bo2':
        sum_score1 = row.map1_score1 + row.map2_score1
        sum_score2 = row.map1_score2 + row.map2_score2
    elif mtype == 'bo3':
        sum_score1 = row.map1_score1 + row.map2_score1 + row.map3_score1
        sum_score2 = row.map1_score2 + row.map2_score2 + row.map3_score2
    if sum_score1 > sum_score2:
        return 1 #team 1 won
    else:
        return 0 #team 2 won or a tie
```

The reason I haven't used a separate class for ties is that there are very few of them. 
The tie class would be underrepresented, which would require  carefully  tuning the training process . In my case trying to predict the ties significantly lowered accuracy.

## The X (Features)
From my experience futures are the most important in a machine learning task.
No model will be useful if there is no information that explains the outcome you are trying to predict.

To predict which team wins, we need to find variables that describe each team.
These variables should include team capabilities:
1. Overall experience and efficiency
2. Recent overall experience and efficiency
2. Experience and efficiency in specific situations (efficiency on a given map, against a given team, in a given moon phase perhaps)

The main metrics of efficiency is the win rate.

I chose the following features:
`t1_win_pct` - matches wonto matches played ratio of team 1, all time
`t1_matches` - amount of matches played by team 1, all time
`t2_win_pct` - same as team 1
`t2_matches` - same as team 1
`t1_win_pct_3m` -  matches wonto matches played ratio  of team 1, over last 3 months
`t1_matches_3m` - amount of matches  played by  team 1, over last 3 months
`t2_win_pct_3m` - same as team 1
`t2_matches_3m` - same as team 1
`prev_enc_t1win_pct` - amount of wins of team 1 over team 2, divided by the total amount of encouters team 1 and team 2 had
`bo1` - 1 if match type is best of one, 0 otherwise
`bo2` - 1 if match type is best of two, 0 otherwise
`bo3` - 1 if match type is best of three, 0 otherwise
`type_t1_win` - total matches of match type won by team 1, divided by total matches of type played by team 1
`type_t2_win` - same as above for team 2

I tried to use map related futures as well. It's a popular idea that pro teams are better on certain maps, so each team has "their best" and "their worst" maps. In my case using map data made predictions worse. 

I extracted the futures with a few aggregation functions.
Here's the code:


```python
features = ["t1_win_pct", "t1_matches", "t2_win_pct", "t2_matches", "t1_win_pct_3m", "t1_matches_3m", "t2_win_pct_3m", "t2_matches_3m", 'prev_enc_t1win_pct' ]

matches_shifted = matches.query("date > '2012-12'")
df = pd.DataFrame(columns=['match_id', "date", 'winner', 'team1', 'team2']+features, index=range(len(matches_shifted)))

def aggregate_wl(team, dt_end, dt_start = None):
    dt = dt_end
    team_matches = matches.query("(date <= '{0}') and (team1 == '{1}' or team2 == '{1}')".format(str(dt), team))
    if len(team_matches) <= 0:
        return (0, 0, 0, 0)
    team_matches_won = team_matches.query("(team1 == '{0}' and winner == 0) or (team2== '{0}' and winner == 1)".format(team))
    team_wins = len(team_matches_won)/len(team_matches)
    
    team_matches_3m = team_matches.query("date > '{0}' and date <= '{1}' and (team1 == '{2}' or team2 == '{2}')".format(str(dt_start), str(dt_end), team))
    team_matches_won_3m = team_matches_3m.query("(team1 == '{0}' and winner == 0) or (team2== '{0}' and winner == 1)".format(team))
    team_wins_3m = len(team_matches_won_3m)/len(team_matches_3m)
    #print(team_matches)
    #print(team_matches_won)
    
    return team_wins, len(team_matches), len(team_matches_3m), team_wins_3m

def aggregate_prev_enc(team1, team2, dt_end=None, dt_start = None):
    t1_win_pct = 0.5
    try:
        try:
            encounters_1 = pivot.ix[team1].ix[team2]
        except:
            encounters_1 = pd.DataFrame([0,0], index= ['winner_0', 'winner_1'])
            
        try:
            encounters_2 = pivot.ix[team2].ix[team1]
        except:
            encounters_2 = pd.DataFrame([0,0], index= ['winner_0', 'winner_1'])
                
        encounters = encounters_1.sum()+encounters_2.sum()
        t1win = encounters_1.winner_0+encounters_2.winner_1
        win_pct = t1win / encounters
        t1_win_pct = win_pct
    except:
        t1_win_pct = 0.5
            
    return t1_win_pct

def aggregate_prev_enc_df(row):
    team1 = row.team1
    team2 = row.team2
    return aggregate_prev_enc(team1, team2)

def get_match_row(match):
        date = match.date
        team1 = match.team1
        team2 = match.team2
        match_id = match.name
        t1_win_pct, t1_matches, t1_matches_3m,t1_win_pct_3m  = aggregate_wl(team1, date, date - np.timedelta64(3, 'M'))
        t2_win_pct, t2_matches, t2_matches_3m,t2_win_pct_3m  = aggregate_wl(team2, date, date - np.timedelta64(3, 'M'))
        prev_enc_t1win_pct = aggregate_prev_enc(team1, team2)
        winner = match.winner
        return {'match_id':match_id,"date": date, "team1":team1, "team2": team2, 
            "t1_win_pct":t1_win_pct, 
            "t1_matches":t1_matches, 
            "t1_win_pct_3m":t1_win_pct_3m,
            "t1_matches_3m":t1_matches_3m,
            "t2_win_pct":t2_win_pct, 
            "t2_matches":t2_matches,
            "t2_win_pct_3m":t2_win_pct_3m,
            "t2_matches_3m":t2_matches_3m,
            "prev_enc_t1win_pct":prev_enc_t1win_pct,
            "winner":winner,
        }

for i in range(len(matches_shifted)):
    match = matches_shifted.iloc[i]
    df.ix[i] = get_match_row(match)
    if i % 150 == 0:
        print(i, '/', len(matches_shifted))

def aggregate_match_type(row, dt_end=None, dt_start = None):
    try:
        team1 = row.team1
        team2 = row.team2
        match = matches.ix[row.match_id]
        match_type = match.type
        bo1 = int(match_type =='bo1')
        bo2 = int(match_type =='bo2')
        bo3 = int(match_type =='bo3')
        type_matches = matches[matches.type == match_type]
        team1_matches = type_matches.query("(team1 == '{0}' or team2 == '{0}')".format(team1))
        team2_matches = type_matches.query("(team1 == '{0}' or team2 == '{0}')".format(team2))
        team1_matches_won = team1_matches.query("(team1 == '{0}' and winner == 0) or (team2== '{0}' and winner == 1)".format(team1))
        team2_matches_won = team2_matches.query("(team1 == '{0}' and winner == 0) or (team2== '{0}' and winner == 1)".format(team2))
        return pd.Series([bo1, bo2, bo3, len(team1_matches_won)/len(team1_matches), len(team2_matches_won)/len(team2_matches)])
    except Exception as e:
        print(e)
        return pd.Series([None, None, None, None, None])
    
mtype_df = pd.DataFrame(df.apply(aggregate_match_type, axis=1))
mtype_df.columns=["bo1", "bo2", "bo3", "type_t1_win", "type_t2_win"]
features = features + ["bo1", "bo2", "bo3", "type_t1_win", "type_t2_win"]
df = pd.concat([df, mtype_df], axis=1) 
```
It's far from the best code I have written.

Take a glance at the futures:

| match_id | date | winner | team1 | team2 | t1_win_pct | t1_matches | t2_win_pct | t2_matches | t1_win_pct_3m | t1_matches_3m | t2_win_pct_3m | t2_matches_3m | prev_enc_t1win_pct | bo1 | bo2 | bo3 | type_t1_win | type_t2_win
| - | - | - | - | - | - | - | - | - | - | - | - | - | - | - | - | - | - | -
| 764 | 2012-12-02 00:00:00 | 1 | mydgbnet | buykey | 0.4 | 10 | 0.4166666666666667 | 12 | 0.4 | 10 | 0.4166666666666667 | 12 | 0.5 | 0.0 | 1.0 | 0.0 | 0.42857142857142855 | 0.75
| 837 | 2012-12-02 00:00:00 | 0 | alternate-attax | 3dmax | 0.5 | 26 | 0.5294117647058824 | 17 | 0.5 | 26 | 0.5294117647058824 | 17 | 0.5 | 1.0 | 0.0 | 0.0 | 0.4728682170542636 | 0.5251396648044693
| 796 | 2012-12-02 00:00:00 | 0 | nextgaming | fr34kshow | 1.0 | 4 | 0.5 | 6 | 1.0 | 4 | 0.5 | 6 | 0.5 | 1.0 | 0.0 | 0.0 | 1.0 | 0.36363636363636365 |

## Logistic regression, oh my

Now that we have the futures, comes the part with predicting (and testing the model).

I chose logistic regression because it's enough for the task and also shows the significance of features after fitting.
I used `klearn.linear_model.LogisticRegression`.
I established the best parameters I could using grid search:

```python
tol = 0.0009
c = 2.2
```

Loaded the futures:

```python
df = pd.read_csv("processed_data/team_performance_and_match_type_conditional_predicates.csv")
features= df.drop(['date','winner', 'team1', 'team2', 'match_id'], axis=1).columns
```

A quick evaluation of the model using `cross_val_score`:

```python
rs = ShuffleSplit(len(proc_X))
proc_X = pd.DataFrame(scale(df[features]), columns=features)
proc_Y = df['winner']
clf = LogisticRegression(tol=tol, C=c)
scores = cross_val_score(clf, proc_X, list(proc_Y), cv=rs)
print(pd.Series(scores))
```

Output:

```python
0    0.807420
1    0.807965
2    0.828696
3    0.817239
4    0.809056
5    0.824877
6    0.813966
7    0.830333
8    0.828696
9    0.817239
dtype: float64
```

Whoa, predicting if a team wins the match with 80% accuracy.

More detailed info:

```python
rs = ShuffleSplit(len(proc_X))
for train_index, test_index in rs:
    X_train, X_test = proc_X.iloc[train_index], proc_X.iloc[test_index]
    y_train, y_test = proc_Y.iloc[train_index], proc_Y.iloc[test_index]
    clf = LogisticRegression(tol=tol, C=c)
    fit = clf.fit(X_train, list(y_train))
    predictions = pd.Series(clf.predict(X_test), index=test_index)
    print(pd.DataFrame(confusion_matrix(list(y_test.sort_index()), list(predictions.sort_index()))))
    print("features and coefficients")
    print(pd.DataFrame(fit.coef_[0], index=proc_X.columns).sort_values(by=[0]))
    print('accuracy')
    print(len(y_test[y_test==predictions])/len(y_test))
```

Output for one iteration:

```python
[17286 12530  2640 ..., 13144 11121 16165] [15593  8899  5588 ..., 12516 17494 10061]
     0    1
0  502  202
1  157  972
features and coefficients
                           0
t1_win_pct_3m      -1.160522
prev_enc_t1win_pct -0.677553
type_t1_win        -0.551285
t2_matches_3m      -0.250979
t2_win_pct         -0.155482
t2_matches         -0.051284
bo1                -0.034199
bo3                -0.030629
t1_matches          0.050220
bo2                 0.060685
t1_win_pct          0.153829
t1_matches_3m       0.192283
type_t2_win         0.682826
t2_win_pct_3m       1.247174
accuracy
0.8041462084015275
```

Most important here is `confusion_matrix`. It's the easiest tool to assess prediction quality.
You can see that the confusion matrix is diagonal. Of 502+202 total class 0 occurences,  the model got 502  right, of 972+157 class 1 occurences  the model got 972 right. That's a good result, no class appears to be overrepresented.

The coefficients tell interesting things.
`t1_win_pct_3m`, `prev_enc_t1win_pct`, `type_t1_win`, `t2_win_pct_3m`, `type_t2_win` have the highest weights, meaning they are most significant. "Specific experience" features completely dominate the "overall experience" features.

## That's it folks!
Thanks for reading, I hope you found something interesting here.
If you spot a mistake, make sure to point it out.
I welcome emails and comments.

## Give me CSVs already
I can't provide the original data (don't want the wrath of angry cs:go fans upon me), but I am providing obsfuscated samples for you to play with:

[matches_sample.csv]({{site.url}}/files/csgo/matches_sample.csv)

[matches_processed_sample.csv]({{site.url}}/files/csgo/matches_processed_sample.csv)

[team_performance_and_match_type_conditional_predicates_sample.csv]({{site.url}}/files/csgo/team_performance_and_match_type_conditional_predicates_sample.csv)


## What I didn't try or failed at

- It's believed some teams play better on either terrorist or counter terrorist side. It's also believed being on a certain side can give significant advantage on certain maps. Perhaps adding a future `team1_side`would improve accuracy. Also if there was an advantage to certain sides on certain maps, including map data would reveal info about it.
- Information about players. I actually gathered info about players: each player's results in each match, and team player listings for each match. However, to my surprise, including player-related futures only made predictions worse.
- Aggregation could be done with more *numpy*/*pandas* vectorization.
- Inspecting prediction results showed that predicting pro matches was about 65% accurate at best, whilist predicting matches between unexperienced teams was more accurate. But it only means that between two unexperienced teams one  usuallyhas *at least some experience* and the other is completely new, so the prediction is easy to make. Perhaps using boosting to account for different kinds of matches would be wise.
- Match type was calculated from the amount of maps played. So a best of two match that had gone for 3 maps was assigned type `bo3`. This might add confusion.