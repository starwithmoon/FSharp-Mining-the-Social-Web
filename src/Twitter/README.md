﻿#Mining Twitter

#### Concepts
- Basic frequency distribution : most common
- Lexical diversity
- Average words
- Ploting log-log graph for frequencies of words 
- Histograms

>The first chapter is available [here](http://nbviewer.ipython.org/github/ptwobrussell/Mining-the-Social-Web-2nd-Edition/blob/master/ipynb/__Chapter%201%20-%20Mining%20Twitter%20%28Full-Text%20Sampler%29.ipynb).


####Linq To Twitter
#####Trends
```fsharp
let getTrends trend_id = 
        query {
            for trend in ctx.Trends do
            where (trend.Type = TrendType.Place)
            where (trend.WoeID = trend_id)
            select trend
        }
```
#####Seach
```fsharp
// get search result
let getSearchResult q num = 
    query {
        for searchResult in ctx.Search do
        where (searchResult.Type = SearchType.Search)
        where (searchResult.Query = q)
        where (searchResult.Count = num)     
        select searchResult    
        exactlyOne        
    }

// get search result with specific maxId
let getSearchResultWithMaxId q num maxId = 
    query {
        for searchResult in ctx.Search do
        where (searchResult.Type = SearchType.Search)
        where (searchResult.Query = q)
        where (searchResult.Count = num)     
        where (searchResult.MaxID = maxId)
        select searchResult    
        exactlyOne        
    }

// get statuses from number of batches
let getStatuses q num batches =
    let s2ul (s:string) = Convert.ToUInt64(s)

    let getStatuses q maxId =
        (getSearchResultWithMaxId q num maxId).Statuses |> List.ofSeq |> List.rev

    let combinedStatuses (acc:Status list) _ =
        let maxId =  
            if acc = [] then UInt64.MaxValue
            else (acc |> List.head |> (fun s -> s.StatusID)  |> s2ul) - 1UL
        (getStatuses q maxId) @ acc

    [0..batches] 
        |> List.fold combinedStatuses []
```


####Example 2. Retrieving Trends
```fsharp
let world_woe_id = 1
let us_woe_id = 23424977  

let worldTrends = Trends.getTrends world_woe_id
let usTrends = Trends.getTrends us_woe_id
```

####Example 4. Computing the intersection of two sets of trends
```fsharp
let commonTrends = usTrends |> worldTrends.Intersect
```

####Example 5. Collecting search results
```fsharp
let q = "#fsharp"

// get five batches, 100 tweets each
let statuses = Search.getStatuses q 100 5

statuses    
    |> Seq.distinctBy (fun s -> (s.Text, s.ScreenName)) 
    |> Seq.sortBy (fun s -> -s.RetweetCount)
    |> Seq.mapi (fun i s -> i+1, s.StatusID, s.User.Identifier.ScreenName, s.Text, s.CreatedAt, s.RetweetCount)    
    |> PrettyTable.show "Five batches of results"
```

####Example 6. Extracting text, screen names, and hashtags from tweets
```fsharp
let usersMentioned = 
    [
        for status in statuses do
            for userMentioned in status.Entities.UserMentionEntities ->
                userMentioned.ScreenName
    ]

let statusTexts = [ for status in statuses -> status.Text ]
let screenNames = 
    [
        for status in statuses do
            for userMentioned in status.Entities.UserMentionEntities ->
                userMentioned.ScreenName
    ]
let hashtags = 
    [
        for status in statuses do
            for hashtag in status.Entities.HashTagEntities ->
                hashtag.Tag
    ]

let words = statusTexts |> List.collect (fun s -> (s.Split() |> List.ofArray))

(statusTexts |> Array.ofSeq).[..5] |> PrettyTable.showListOfStrings "Status Texts" 
(screenNames |> Array.ofSeq).[..5] |> PrettyTable.showListOfStrings "Screen Names"
(hashtags |> Array.ofSeq).[..5] |> PrettyTable.showListOfStrings "Hashtags"
(words |> Array.ofSeq).[..5] |> PrettyTable.showListOfStrings "Words"
```

We can also do [List comprehension](http://en.wikipedia.org/wiki/List_comprehension#F.23) in F# like in [Python](http://docs.python.org/2/tutorial/datastructures.html#list-comprehensions).

####Example 7. Creating a basic frequency distribution
```fsharp
let getMostCommon (tokens:seq<string>) count =
    let tokensCount = tokens |> Seq.length
    let len = if tokensCount <= count then tokensCount else count
    query {
        for t in tokens do
        groupBy t into g
        select (g.Key, g.Count())
    }  
        |> Seq.sortBy (fun x -> -snd(x))
        |> Seq.take len
        |> Seq.map (fun x -> 
            { 
                PrettyTable.CountRow.Name = fst(x)
                PrettyTable.CountRow.Count = snd(x)
            }
        )
        |> Array.ofSeq

(getMostCommon words 10) |> PrettyTable.showCounts "Words"
(getMostCommon screenNames 10) |> PrettyTable.showCounts "Screen Names"
(getMostCommon hashtags 10) |> PrettyTable.showCounts "Hashtags"
```

![Words](https://raw.github.com/kimsk/FSharp-Mining-the-Social-Web/master/src/images/Twitter-Example-8-Words.PNG)

![Screen Names](https://raw.github.com/kimsk/FSharp-Mining-the-Social-Web/master/src/images/Twitter-Example-8-ScreenNames.PNG)

![Hashtags](https://raw.github.com/kimsk/FSharp-Mining-the-Social-Web/master/src/images/Twitter-Example-8-HashTags.PNG)


####Example 8. F# PrettyTable using DataGrid
```fsharp
module PrettyTable

open System.Drawing
open System.Windows.Forms

let buildForm title =
    let form = new Form(Visible = true, Text = title,
                        TopMost = true, Size = Size(600,600))

    let data = new DataGridView(Dock = DockStyle.Fill, Text = "F#",
                                Font = new Font("Lucida Console",12.0f),
                                ForeColor = Color.DarkBlue)
 
    form.Controls.Add(data)
    data

let show title dataSource =
    let data = buildForm title
    data.DataSource <- dataSource
```

####Example 9. Calculating lexical diversity for tweets
```fsharp
let lexicalDiversity (tokens:seq<string>) =
        1.0 * (tokens |> Set.ofSeq |> Seq.length |> float)/(tokens |> Seq.length |>float)


// Computing the average number of words per tweet
let averageWords (statusTexts:seq<string>) =
    statusTexts 
        |> Seq.map (fun s -> s.Split()) 
        |> Seq.map (fun s -> s.Length |> float) 
        |> Seq.average

lexicalDiversity words
lexicalDiversity screenNames
lexicalDiversity hashtags
averageWords statusTexts
```

####Example 10. Finding the most popular retweets
```fsharp
type RetweetTable = { Count:int; Handle:string; Text:string}
let retweets = 
    statuses         
        |> Seq.filter (fun s -> s.RetweetCount > 0)
        |> Seq.distinctBy (fun s -> s.Text)
        |> Seq.sortBy (fun s -> -s.RetweetCount)                                
        |> Seq.map (fun s -> (s.RetweetCount, s.User.Identifier.ScreenName, s.Text))                

(retweets |> Seq.map (fun (c,n,t) -> {Count=c; Handle=n; Text=t}))
    |> PrettyTable.show "Most Popular Retweets"  
```
![Most popular retweets](https://raw.github.com/kimsk/FSharp-Mining-the-Social-Web/master/src/images/Twitter-Example-10-MostPopularRetweets.PNG)


####Example 11. Looking up users who have retweeted a status
```fsharp
let mostPopularStatusId = 
    (statuses         
        |> Seq.sortBy (fun s -> -s.RetweetCount)    
        |> Seq.map (fun s -> s.RetweetedStatus.StatusID)).First()

let retweeters = 
    let users = 
        (query {
            for tweet in ctx.Status do
            where (tweet.Type = StatusType.Retweeters)
            where (tweet.ID = mostPopularStatusId)
            select tweet
            exactlyOne
        }).Users
        |> Seq.map (fun u -> u.ToString())
        |> Seq.reduce (fun acc u -> acc + ", " + u)
    
    query{
        for user in ctx.User do
        where (user.Type = UserType.Lookup)
        where (user.UserID = users)
        select user.Identifier.ScreenName
    } |> Array.ofSeq |> Array.mapi (fun i u -> i+1, u)
```

####Example 12. Plotting frequencies of words
```fsharp
let wordCounts = 
    query {
        for word in words do
        groupBy word into g
        select (g.Key, g.Count())
    } 
    
let idxFreqLogLog =  
    wordCounts
        |> Seq.sortBy (fun x -> -snd(x))
        |> Seq.mapi (fun i x ->  log(float (i+1)), log(float <| snd(x)))        

idxFreqLogLog |> PrettyTable.show "Index and Frequency"

(Chart.Line(idxFreqLogLog, Name="Example 12", Title="Frequency Data", YTitle="Freq", XTitle="Word Rank")).ShowChart()
```

![Frequencies of Words](https://raw.github.com/kimsk/FSharp-Mining-the-Social-Web/master/src/images/Twitter-Example-12-FrequenciesOfWords.PNG)


####Example 13. Generating histograms of words, screen names, and hashtags
```fsharp
let getHistogram items =
    items 
        |> Seq.groupBy (snd) 
        |> Seq.map (fun (k,s) -> k, s.Count())
        |> Seq.sortBy (fst)

let wordsHistogram = getHistogram wordCounts |> Seq.take 20
let screenNamesHistogram = 
    query{
        for screenName in screenNames do
        groupBy screenName into g
        select (g.Key, g.Count())
    } |> getHistogram |> Seq.take 12

let hashtagsHistogram = 
    query{
        for hashtag in hashtags do
        groupBy hashtag into g
        select (g.Key, g.Count())
    } |> getHistogram |> Seq.take 12

let yTitle = "Number of items in bin"
let xTitle = "Bins (number of times an items appeared)"
Chart.Column(wordsHistogram, Name="Words", YTitle=yTitle, XTitle=xTitle)
    .ShowChart()
Chart.Column(screenNamesHistogram, Name="Screen Names", YTitle=yTitle, XTitle=xTitle).ShowChart()
Chart.Column(hashtagsHistogram, Name="Hashtags", YTitle=yTitle, XTitle=xTitle).ShowChart()
```

![Words Histogram](https://raw.github.com/kimsk/FSharp-Mining-the-Social-Web/master/src/images/Twitter-Example-13-WordsHistogram.PNG)

![Screen Names Histogram](https://raw.github.com/kimsk/FSharp-Mining-the-Social-Web/master/src/images/Twitter-Example-13-ScreenNamesHistogram.PNG)

![Hashtags Histogram](https://raw.github.com/kimsk/FSharp-Mining-the-Social-Web/master/src/images/Twitter-Example-13-HashtagsHistogram.PNG)


####Example 14. Generating a histogram of retweet counts
```fsharp
let retweetsHistogram = 
    retweets     
    |> Seq.groupBy (fun (c,_,_) -> c)
    |> Seq.map (fun (k,s) -> k, s.Count())
    |> Seq.sortBy (fst)

Chart.Column(retweetsHistogram, Name="Retweets", YTitle=yTitle, XTitle=xTitle).ShowChart()
```

![Retweets Histogram](https://raw.github.com/kimsk/FSharp-Mining-the-Social-Web/master/src/images/Twitter-Example-14-RetweetsHistogram.PNG)

