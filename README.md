# Chord Clash Development

Chord Clash is a two-player rhythm game where players deal damage to each other based on the accuracy of their hits, using powerups spawned during the games to gain advantages. I was on the tech team and worked on game logic and level design. My initial responsibilities included implementing a system for spawning notes timed to the rhythm of a song, moving the notes along a set path, and assessing player accuracy when hitting notes.

## Timing

As a complete beginner to Unreal, I didn't have a definitive idea for what method would work best for timing. Minimizing lag was the main goal, both for gameplay and because we intended to have the singers periodically switch depending on which player had higher health. I needed to identify a method where the backing instrumentals, vocals, and spawned notes would not get out of sync in the case of lag. It's also worth noting that I was doing most of the development for this game on an older laptop, so lag was inevitable (which was both a good and a bad thing for timing testing).

The main two methods I tried were the combination of **data tables and a timer**, and spawning notes using a **level sequence event track**.

### Data Table + Timer

My initial attempt involved data tables and timers, using C++ rather than Blueprints. As a team we'd discussed the possibility of using data tables so that the timing could be inputed externally, so I decided to see how viable this method would be as a test.

I first used [Reaper](https://www.reaper.fm/) to map out all of the times I wanted a note to play at a certain lane. These times were inputted into a Google Sheet and exported a CSV.
<p align="center">
  <img src = "/images/gnossienne_table.png" alt = "table with the first few notes for Gnossienne No. 1, a test song" height = 200>
</p>

While reading from the data table wasn't too difficult, I struggled a lot with types in Unreal (FStrings and FNames weren't exactly intuitive).

First, the information from the data table was put into an array:

```
SongDataTable = LoadObject<UDataTable>(nullptr, TEXT("/Game/SongData/Gnossienne"));
    beatTimes = SongDataTable->GetRowNames();
    nextBeat = beatTimes[0].ToString();
    beatNumCurrent = 0;
    
    nextBeatSound = beatTimes[0].ToString();
    beatSoundCurrent = 0;
    
    if (SongDataTable) {
        UE_LOG(LogTemp, Warning, TEXT("song loaded"));
    }
    else {
        UE_LOG(LogTemp, Warning, TEXT("could not load song"));
    }
```

A timer would go off every 0.125 seconds. Every time the timer was advanced, the variable for the current beat would be increased by 0.125, and compared to the nextBeat variable, which was set using information from the data table.
```
GetWorldTimerManager().SetTimer(BeatTimerHandle, this, &ABeatCounter::AdvanceTimer, 0.125f, true);
```

If the current beat was equal to the nextBeat (with an offset so that notes spawned a couple seconds before and hit the bar at the actual time), beats would be spawned based on which lanes were marked as true. Then nextBeat would be set to the next time in the table.
```
if (nextBeat == beatString) {
        FSongMapDataTable* notes = SongDataTable->FindRow<FSongMapDataTable>(FName(*beatString), "");
        if (notes != NULL) {
            UE_LOG(LogTemp, Display, TEXT("yippee"));
            if (notes->lane1 == true) {
                FireBeat(30.0);
            }
            if (notes->lane2 == true) {
                FireBeat(10.0);
            }
            if (notes->lane3 == true) {
                FireBeat(-10.0);
            }
            if (notes->lane4 == true) {
                FireBeat(-30.0);
            }
        }
        beatNumCurrent++;
        nextBeat = beatTimes[beatNumCurrent].ToString();
    }
```
In Blueprints, the component starts moving immediately with Begin Play.

Unfortunately, the timing didn't always work well, with the song often playing slightly after notes started spawning. It was never consistent, so I couldn't just set an offset and call it a day.

### Sequences

While I couldn't find many tutorials related to rhythm games in Unreal (particularly those not involving plugins), there were tutorials for timing events to music using sequences. I hadn't wanted to use this method at first mostly because I wanted to leave open the possibility of changing note speed (a feature that some rhythm games have so that the notes don't get cramped in more difficult songs), which would be harder to implement in a sequence, where the tracks are relatively set in stone. I had also wanted to see if there was a method where the timing of notes didn't have to be directly edited in Unreal. As neither of these things were a huge priority, sequences were worth trying.

I still used Reaper to get the timing of notes correct, then created a sequence with an actor for spawning notes bound to it (this actor, NoteSpawner, handled most note-related functions). It had a trigger event track with 4 sections to easily keep track of which lanes a note was being fired from. All of the keys connected to an event in the NoteSpawner, with the lane as the only variable. When all of the keys were in the right spot, they were dragged back by two seconds so that they would hit the bar at the correct time after moving downwards. Besides being a very tedious task, I also had some trouble with getting the keys bound to the event. When I went into properties, the correct event didn't always show up, and I kept accidentally creating new events. Looking back (after doing the same process for multiple songs), there had likely been a key I was binding incorrectly and then copied, causing the issue to be present throughout the song.

<p align="center">
  <img src = "/images/keyProperties.png" alt = "right clicking to show properties for a key">
</p>

The timing worked with no gap between the event firing and the note spawning, and lag messing with the sync was a non-issue because both the audio and the note spawning were tied to the sequence.

<p align="center">
  <img src = "/images/test.gif" alt = "test with sequences for timing">
</p>

## Note Queue

When an arrow object is spawned, it is added to a queue. There are 8 queues: 1 per lane per player. When a note is spawned, it's added to the correct lane queue for both players. When the note is hit or moves past the area where it's considered a hit, it is removed from the queue. That way, when a player uses a key to hit a note, the note is taken from the correct lane queue so it's position can be checked, and the accuracy assessment is handled accordingly.

I originally wanted to organize the queues with 2D arrays, only to discover that Unreal doesn't allow 2D arrays. You can't just make a 2D array, you need to make an array of structs, and those structs can contain arrays. I used maps instead:


<p align="center">
  <img src = "/images/addToQueueOld.png" alt = "old version of AddToQueue">
</p>
<p align="center">
  <img src = "/images/getQueue0Old.png" alt = "old version of GetQueue0">
</p>

I tried to get this method of accessing the queues to work, with many different variations, but kept running into the issue of the function accessing "none". It's likely that I didn't initialize something correctly, but for the sake of time I needed to move on from using the maps and structs, even if it meant messier code.

I ended up creating 8 arrays that are accessed using switch statements.

<p align="center">
  <img src = "/images/addToQueue.png" alt = "new AddToQueue">
</p>

When a note is accessed using a queue, the current position of the sprite is sent to a function that checks the accuracy. There are 4 ranges for a hit based on how close the current position is to the target position: Perfect, Great, Good, and Bad. Misses are handled separately. After a score is determined based on the ranges, the score is updated with another function. The NoteSpawner has an array of custom structs containing a score for each player. The index of this array corresponds to the number of the note (first note to spawn having a spawn number of 0). When the score is updated for a note, the function checks if both players' scores have been updated. If this is the case, then damage is calculated based on the difference in accuracy.

<p align="center">
  <img src = "/images/checkAccuracy.png" alt = "part of the check accuracy function">
</p>

When a note passes the range where it can be hit and removes itself from the queue, it sets the score to miss for any player that does not have a score already associated with that note.

When accuracy is calculated and scores are updated, the functions also take into account powerups and skills that may affect these calculations. For example, if a player has the Auto Perfect powerup activated, their score will be changed to Perfect before damage is calculated.


## Player Input

My unfamiliarity with Unreal made it difficult for me to figure out player input. I was trying to control regular actors with keys the the same way you would with pawns, and I couldn't tell why code that looked exactly the same as tutorials wasn't working.

I finally made the NoteSpawner into a pawn, and everything fell into place up until I transitioned from my test project into the actual project. Since we had a set camera in the real game, the camera switching after the NoteSpawner was possessed posed a problem. There weren't clear answers for stopping this automatic switch, despite it feeling like something that should be a simple toggle, so I decided to make the NoteSpawner back into a regular actor and handle input a different way. Instead of the NoteSpawner directly firing off events from action mapping, the level blueprint does. It then uses a reference to NoteSpawner to call a function that handles the input. 

## Singer Switching

One feature that I was set on including in our final product was the singers switching based on player health. It may not have been essential to gameplay, but it was one of my favorite quirks of the multiplayer game.

I wanted to avoid using regular audio players for the singers because of lag. If the sequence were to lag and the singers were being controlled elsewhere, the singers would end up out of time with the sequence. Therefore, I tried to prioritize using sequences over other methods. This wasn't an easy task as sequences aren’t really made to be dynamically changed at runtime - there are some options available, but it's not necessarily clear what *can't* be dynamic. 

I was initially able to get the switch to happen smoothly using subsequences. There were two subsequences that each had just the audio for the singer. An event track in the main sequence had keys for every point I wanted the singers to possibly switch. When the event was triggered, the correct subsequence was deactivated using loops to go through tracks and disable every section.

<p align="center">
  <img src = "/images/og_sequence.png" alt = "original sequence with subsequences for singers">
</p>

As it turns out, the set active function is just for development. Unlike functions like Print String that make it very clear they only function in development, I had no idea activating/deactivating sections wouldn't work until we built the project. 

When it became clear that activation wouldn't work, my next attempt involved binding the audio track to Singer actors with audio attenuation, with a new SingerManager actor to move them. When a singer needed to be deactivated, the singer would teleport far enough away that the audio wouldn't be present. This didn't work either because for the duration of a sequence, the transform of a bound actor is controlled by the sequence even if there's no transform track, which I hadn't realized. The singer would only move to the teleported position after the sequence ended.

The last method I tried that still involved sequences was changing the bindings, which can be done at runtime. The idea was that the binding would change from a close actor to an actor far away and vice versa depending on health. There were a couple tutorials on this, so I had an idea of what needed to be done, but I wasn't able to successfully implement it. It's a method that I would like to go back to, as the main reason I moved on was for time's sake. 

Finally, I used audio played by a SingerManager actor, which had an event track on the sequence. On a switch, the volume would change depending on player health. Both audio files were primed before the sequence so they would start immediately in sync with the sequence. When the sequence lagged, and the singers were ahead. To solve this, instead of playing once at the beginning of the sequence and changing volume, the audio is repeatedly played and stopped. The switch event on the sequence passes along the current time so the audio can be played from that moment. That way, if the sequence lags, the singers are only out of sync until the next switch.

<p align="center">
  <img src = "/images/singerswitch.png" alt = "SingerManager switching based on health">
</p>

## Calibration

An offset was added to the notespawner that would affect the calculations for accuracy and the timing for a note being removed from queues. The first step was simple - just subtracting the offset from the location before checking the ranges. It was when I changed the way timing worked to allow for more time before removing a note from the queue that I ran into problems. It was also when more art was added to the game, so working on my laptop went from laggy but playable (and actually good for testing if the offset calculations worked) to slow enough I couldn't always tell if my changes were doing anything. This made testing much more difficult as there were many instances of having to ask my teammates to test my branch to check if things were working. 

The removal from queue call happens after the note finishes moving just past the bar, so I was able to fix the issues with the queue removal timing by changing movement rather than adding delays. The note moves further down, with the distance and timing dependent on the offset. There is no visual difference, but it ensures a note will not be removed before the player has the chance to hit it.

<p align="center">
  <img src = "/images/movementOffset.png" alt = "movement with offset">
</p>

To calibrate, players manually adjust the offset while playing a calibration sequence that plays the same note over and over again. This sequence works exactly the same as the main song sequences, with powerups disabled and no health/damage.

## Version Control

We used GitHub as our version control software, which generally went smoothly for us. With Blueprints, you'll be forced to pick one version of the file over another if there is a merge conflict, so we had to be careful about what was being worked on at the same time. For our game there were enough separate parts that this wasn't a huge issue. We all actively communicated who was working with what Blueprints, so we rarely dealt with merge conflicts, and most were because of slight changes people accidentally made that could be ignored. There were only a few times where someone's work had to be overwritten. We also had it set up so any pull requests needed to be approved by two other people before they could be merged into main.

We used channels in the team Discord server to effectively communicate changes that were made to avoid conflicts. There was a bot that updated us to any changes in the GitHub repository, a channel for telling team members when a pull request was made and asking for reviews, and a channel for people to say when they started and finished working with certain Blueprints.
        
## Architecture 

I worked primarily with the NoteSpawner class Blueprint, Arrow Object class Blueprint, and sequence Blueprints, along with the level Blueprint to connect everything. 

Below is a rough statechart diagram of our game, with more detail for the Play Song state (the states pictured inside this one are referencing the NoteSpawner actor's actions during this period of the game).

<p align="center">
  <img src = "/images/statechart.png" alt = "statechart diagram">
</p>

## Lessons Learned

1. **Search forums often** - If you're having an issue, chances are someone else has too and took to the Unreal forums to ask about it. It's not a guarantee you'll find answers, but Unreal forums are often a good starting point.
2. **Build early** - You don't want to be blindsided by a function behaving perfectly fine up until the build.
3. **Try something else if you're stuck sooner rather than later** - There were a few times I was spending far too long on methods that I thought would work better (sequences rather than audio players for the singer switches being the most time consuming), when if I'd just tested the alternative earlier I would've found it worked fine.
4. **Ask team members to step in** - Asking other team members to help with smaller tasks that I'd find along the way was very important to getting the project done. Worst case, nobody else will have time, but it's good to at least ask.
5. **Sequences are not ideal if things need to change at runtime** - If you know you'll need to make changes at runtime with actors/audio involved in a sequence, it will be an issue. This isn't to say that you can't do anything dynamically with sequences, but it's limited.
