## Station Creation Flow

1. The station microservice hears a USER_CREATED event.

2. In response, a station is created with some defaults:

   ```json
   {
   	"id" : "theStationUUID",
   	"playolaUID": "theUsersPlayolaUUID",
   	"deepLink": "innerLinkToStation",
   	"stationStatus": "GATHERING_SONGS",
   	"secsOfAdsPerHour": 360,
   	"rules": {
   		"artistMinimumRestMinutes": 70,
   		"songMinimumRestMinutes": 180,
   		"dayOffsetMinutes": 60 
   	},
     "status": "PENDING"
   }
   ```

3. The `station` service requests a list of 150 songs related to the user from the `spotify` service.

   ```json
   GET /v1/spotify/userImportantTracks?playolaUID=theUsersPlayolaUUID
   
   response:
   {
   	[
     	{
         "title": "Too Much Love",
         "artist": "Rachel Loy",
         "album": "My Album",
         "durationMS": 180000,
         "isrc": "USUM71703861",  // (if available)
         "spotifyID": "11dFghVXANMlKmJXsNCbNl"
      	},
     	{
       	"title": "Fade To Gray",
         "artist": "Rachel Loy",
         "album": "Broken Machine",
         "durationMS": 170000,
         "spotifyID": "11dFghVXANMlKmJXsNCbN2"
     	}
     ]
   }
   ```

4. The `station` service then `POSTS` this list to the `songs` microservice and receives an array of new `Song` objects, most with a `PENDING` status:

   ```
   POST /v1/songs/getOrCreateMany
   
   response:
   {
   	[
     	{
         "title": "Too Much Love",
         "artist": "Rachel Loy",
         "album": "My Album",
         "durationMS": 180000,
         "isrc": "USUM71703861",  // (if available)
         "spotifyID": "11dFghVXANMlKmJXsNCbNl",
         "status": "PENDING"
      	},
     	{
       	"title": "Fade To Gray",
         "artist": "Rachel Loy",
         "album": "Broken Machine",
         "durationMS": 170000,
         "spotifyID": "11dFghVXANMlKmJXsNCbN2",
         "status": "ENABLED"
     	}
     ]
   }
   ```

5. The `songs` microservice fires off a `SONGS_SONG_CREATED` event for each song.  This launches the [__SONG_AQUISITION_FLOW__](./SongAcquisitionFlow.md)

6. The `stations`  microservice listens for the `SONGS_SONG_ENABLED` event at the end of the   [__SONG_AQUISITION_FLOW__](./SongAcquisitionFlow.md).  Each time a song is created, it checks to see if there are enough songs to start the station and, if so, generates a playlist.