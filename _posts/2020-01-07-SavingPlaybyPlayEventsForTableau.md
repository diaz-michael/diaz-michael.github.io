---
title: "Saving Play by Play Events for Tableau Visualizations"
excerpt_separator: "<!--more-->"
categories:
  - Data Visualizations
tags:
  - API
  - R
  - Hockey
  - Data Visualization
  - NHL
  - Tableau
---

Goal: Use R to retrieve selected play by play events from the NHL's undocumented API. The end goal is that this data will be exported to Tableau for visualizations.


<!--more-->
[Github repo](https://github.com/diaz-michael/NHL_API)

## save_events(season_list, event_list)

Season_list takes a vector of strings. Coordinate data was first tracked in the 2010 season so current options are 
- 20102011
- 20112012
- 20122013
- 20132014
- 20142015
- 20152016
- 20162017
- 20172018
- 20182019
- 20192020

Event_list takes a vector of strings. Currently supported options are:
- Goal
- Shot
- Missed_shot
- Hit

## Looping through selected seasons
To Do: Add option to choice gameType

Gets a schedule of all regular season and playoff games and saves the data to a tibble.
```
for (season in season_list) {
    games_schedule <- content(GET(paste0("https://statsapi.web.nhl.com/api/v1/schedule/?season=", season,"&gameType=R&gameType=P")))
    print(season)
    for (day in games_schedule$dates) {
      for (game in day$games) {
        game_data <- add_row(game_data,
                             gamePk = game$gamePk,
                             gameType = game$gameType,
                             gameSeason = game$season,
                             teamHome = game$teams$home$team$name,
                             teamAway = game$teams$away$team$name,
                             gameDate = game$gameDate
        )
      }
    }
  }
```
## Looping through games
Loops through each game in the schedule and gets the play by play data. Checks if a play is desired (event_list).
```
  for (game in 1:nrow(game_data)) {
    ...
    gamePk <- game_data[game, "gamePk"]
    data_pbp <- content(GET(paste0("https://statsapi.web.nhl.com/api/v1/game/",gamePk,"/feed/live")))
    print(paste("Game ID: ",gamePk, data_pbp$gameData$datetime$dateTime))
    
    # playCount = 1
    for (play in data_pbp$liveData$plays$allPlays) {
      # print(playCount)
      # playCount <- playCount + 1
      if (play$result$event %in% event_list){
        
```

## Special case for playerType
The NHL API uses playerType "Assist" for both the first and second assist. In order to keep track of the two unique players, each playerType input is initalized as NA. Then playerTypes are assigned. If a playerType is already assigned (first assist), then the player is assigned to playerTypeIn2.

```
for (type in c("ScorerIn","AssistIn","AssistIn2","GoalieIn","ShooterIn","HitterIn","HitteeIn")) {
          assign(type,NA)
        }
        for (player in play$players) {
          
          typeIn <- paste0(player$playerType,"In")
          
          if (is.na(get(typeIn))) {
            assign(typeIn,player$player$fullName)
          } else {
            assign(paste0(typeIn,"2"),player$player$fullName)
          }
        }
```
## Catching Missing Data
```
event_data <- add_row(event_data,
                              eventTypeId = play$result$eventTypeId,
                              secondaryType =  play$result$secondaryType %>%
                                if(length(.) == 0) NA  else .,
                              period = play$about$period,
                              periodTime = play$about$periodTime,
                              coordX = play$coordinates$x %>%
                                if(length(.) == 0) NA else .,
                              coordY = play$coordinates$y %>%
                                if(length(.) == 0) NA else .,
```

## Saving as CSV
To Do: Add ability to name output file in function
The data is appended to the .csv export file after each game loop. 
```
  write.table(event_data,file = "19.csv", append = TRUE, sep = ",", quote = FALSE,row.names = FALSE, col.names = FALSE)
```

API copyright notice: NHL and the NHL Shield are registered trademarks of the National Hockey League. NHL and NHL team marks are the property of the NHL and its teams.