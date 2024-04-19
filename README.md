# Chord Clash Development

I was on the tech team and worked on game logic and level design. My initial responsibilities included implementing a system for spawning notes timed to the rhythm of a song, moving the notes along a set path, and assessing player accuracy when hitting notes.

## Timing

As a complete beginner to Unreal, I didn't have a definitive idea for what method would work best for timing. Most tutorials I could find with rhythm games and Unreal used plugins, so I was kind of winging it from the start. Minimizing lag was the main goal, both for gameplay and because we intended to have the singers periodically switch depending on which player had higher health. I needed to identify a method where the backing instrumentals, vocals, and spawned notes would not get out of sync in the case of lag. It's also worth noting that I was doing all of the development for this game on an older Macbook, so lag was inevitable (which was both a good and a bad thing for timing testing).

Finding a method involved a lot of brainstorming and experimenting with different variations of similar systems, but the main two methods I tried were the combination of **data tables and a timer**, and spawning notes using a **level sequence event track**.

### Data Table + Timer

My initial attempt involved data tables and timers, using C++ rather than Blueprints. As a team we'd discussed the possibility of using data tables so that the timing could be inputed externally, so I decided to see how viable this method would be as a test.

I first used Reaper to map out all of the times I wanted a note to play at a certain lane. I then inputed all of these times into a Google Sheet and exported the CSV.
<p align="center">
  <img width="506" alt="gnossienne_beatmap" src="https://github.com/jdlogan0/chordclash-dev-portfolio/assets/143477762/229f7bf9-1e7c-4c5c-8a2c-c63a3271e67e">
</p>

While reading from the data table wasn't too difficult, I struggled a lot with types in Unreal (FStrings and FNames weren't exactly intuitive). It took a few tries, but eventually I got the system in place! First, the information from the data table was put into an array:

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

I then set up a timer that would go off every 0.125 seconds. Every time the timer was advanced, the variable for the current beat would be increased by 0.125, and compared to the nextBeat variable, which was set using information from the data table.
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
In Blueprints, I just had the component start moving immediately with Begin Play.

And it worked! Kind of! Here's one of the best tests I ran:


https://github.com/jdlogan0/chordclash-dev-portfolio/assets/143477762/41def0c9-392e-4e87-adf4-ce9dcf061dd9

Unfortunately, the timing didn't always work that well. Even more unfortunately, it was never consistent, so I couldn't just set an offset and call it a day. I'd need to find a new method.

### Sequences

While I couldn't find many tutorials about rhythm games in Unreal, there were tutorials for timing events to music using sequences. I hadn't wanted to use this method at first mostly because I wanted to leave open the possibility of changing note speed (a feature that some rhythm games have so that the notes don't get cramped in more difficult songs), which would be harder to implement in a sequence, where the tracks are pretty much set (the rigidity of sequences did end up coming back to haunt me later). That, and I wanted to see if there was a method where the timing of notes didn't have to be directly edited in Unreal. Neither of these things were a huge priority though, so I gave sequences a try.

I still used Reaper to get the timing of notes correct, then created a sequence with the NoteSpawner bound to it. It had 4 trigger event tracks to easily keep track of which lanes a note was being fired from. I had all of the keys connected to an event in the NoteSpawner, with the lane as the only variable. Besides being a very tedious task, I also had some trouble with getting the keys bound to the event. When I went into properties, the correct event didn't always show up, and I kept accidentally creating new events. Looking back (after doing the same process for multiple songs), I think I had been messing up the way events were bound to tracks.

When all of the keys were in the right spot, I dragged all of them back by two seconds (you can easily move all keys in a track at once) so that they would hit the bar at the correct time after moving downwards for two seconds.

When I tested it, it was obvious that a) the timing worked really well, with no gap between the event firing and the note spawning, and b) lag messing with the sync was a non-issue because both the audio and the note spawning were tied to the sequence.

**insert picture from test here**

## Player Input

My unfamiliarity with Unreal made it really hard for me to figure out player input. I had somehow missed that you couldn't control regular actors with keys the the same way you could with pawns, and I couldn't understand why code that looked exactly the same as tutorials wasn't working.

I finally made the NoteSpawner into a pawn, and everything fell into place up until I transitioned from my test project into the actual project. Skipping ahead a bit - since we had a set camera in the real game, the camera switching after the NoteSpawner was possessed posed a problem. There weren't clear answers for stopping this automatic switch, despite it feeling like something that should be a simple toggle, so I decided to make the NoteSpawner back into a regular actor and handle input a different way. Instead of the NoteSpawner directly firing off events from action mapping, the level blueprint does. It then uses a reference to NoteSpawner to call a function that handles the input. 

## Note Queue

When an arrow object is spawned, it is added to a queue. There are 8 queues: 1 per lane per player. I wanted to organize the queues with maps and 2D arrays, only to discover that Unreal doesn't play well with 2D arrays. You can't just make a 2D array, you need to make an array of structs, and those structs can contain arrays.

        
> #### GitHub
> 
> This is the point where I finally moved from my test project to the actual repository. I'd used git before (and was using it more than ever with another class this term), but I had no clue how to use it with Unreal.
> I asked a team member for help, and she walked me through setting up GitHub desktop.
> In general, using GitHub for version control went very smoothly for us. We all actively communicated who was working with what Blueprints, so we rarely dealt with merge conflicts, and most were because of slight changes people accidentally made that could be ignored.

## Singer Switching

One feature that I was set on including in our final product was the singers switching based on player health. It may not have been essential to gameplay, but it was one of my favorite quirks of the multiplayer game and I was *going* to make it work.

Sequences aren’t really made for dynamic stuff - there are some options for making it dynamic, but it's not necessarily clear what *can't* be dynamic. 

I was initially able to get the switch to happen smoothly using subsequences. There were two subsequences that each had just the audio for the singer. An event track in the main sequence had keys for every point I wanted the singers to possibly switch (more or less every time, or whenever there was a pause). When the event was triggered, the correct subsequence was deactivated using loops to go through tracks and disable every section.

<p align = "center">
  <img width="1274" alt="og_sequence" src="https://github.com/jdlogan0/chordclash-dev-portfolio/assets/143477762/ff655f84-f2ea-458d-bfe0-e2270690291a">
</p>

This was honestly one of the things I was most excited about figuring out... and one of the things that bothered me most when it stopped working in the build.

As it turns out, the set active function is just for development. Unlike functions like Print String that make it very clear they only function in development, I had no idea activating/deactivating sections wouldn't work until we built the project. 

I wanted to avoid using regular audio players for the singers because of lag. If the sequence were to lag and the singers were being controlled elsewhere, the singers would end up out of time with the sequence. Therefore, I tried to prioritize using sequences over other methods.

When it became clear that activation wouldn't work, my next attempt involved binding the audio track to Singer actors with audio attenuation, with a SingerManager actor to move them. When a singer needed to be deactivated, the singer would teleport far enough away that the audio wouldn't be present. A roundabout way of dealing with audio, but I was running out of ideas. Unfortunately, this didn't work either because of how sequences function. During the sequence, the transform of a bound actor is controlled by the sequence, even if there's no transform track, which I hadn't realized. The singer would only move to the teleported position after the sequence ended.

The last method I tried that still involved sequences was changing the bindings, which can be done at runtime. The idea was that the binding would change from a close actor to an actor far away and vice versa depending on health. There were a couple tutorials on this, so I had an idea of what needed to be done, but I wasn't able to successfully implement. It's a method that I would like to go back to, as the main reason I moved on was for time's sake. By this point I had already spent a couple days just trying to get the singer switch to work, and it wasn't even vital to gameplay. I needed to move on from sequences.

Finally, I used audio played by the SingerManager, which had an event track on the sequence. On a switch, the volume would change depending on player health. I made sure both audio files were primed before the sequence so they would start immediately, and was surprised by how in sync they were. Then the sequence lagged, and the singers were ahead. To solve this, instead of playing once at the beginning of the sequence and changing volume, the audio is repeatedly played and stopped. The event on the sequence passes along the current time so the audio can be played from that moment. That way, if the sequence lags, the singers are only out of sync until the next switch.

## Calibration

## Other Challenges

        
## Diagrams 

## Lessons Learned
