### User Creation Flow

1. User signs into spotify.  We must provide the token exchange /redirect service for that.  In order to sign in, the user must first open a browser pointed at:

`GET /v1/spotify/auth/authorize?redirect_uri=playola-oauth%3A%2F%2Fspotify`

	* `redirect_uri` -- the name of the callback for spotify to send the final success redirect to.
	* The callback will include a `code` that needs to be exchanged for an `access_token` and `refresh_token` in the next step.

---

2. Client obtains an `access_token` and `refresh_token` from the spotify api (through Playola's `spotify` microservice as middleman).

`POST /v1/spotify/auth/token/swap`

```json
json payload:
{
  "code": "A1223EAFAADCE",
  "redirect_uri": "playola-oauth://spotify"   // this is just for validation,
  																						// but is required by Spotify
}
```

----

 response:

```json
{
   "access_token": "NgCXRKMzYjw",
   "token_type": "Bearer",
   "scope": "user-read-private user-read-email",
   "expires_in": 3600,
   "refresh_token": "NgAagAUmSHo"
}
```

---

3. The client POSTS the new Spotify `refresh_token` and `access_token` to `/v1/users` and receives a `Bearer` Token for use with the Playola API.


```json
POST /v1/users
{
  "spotifyRefreshToken": "NgAagAUmSHo",
  "spotifyAccessToken": "AlsoASpotifyAccessToken"
}
```
   
   * the `users` microservice uses the tokens to get profile data from the `spotify` micro service:

```json
GET /v1/spotify/playolaUserProfile?refresh_token=NgAagAUmSHo&access_token=AlsoASpotifyAccessToken
     
response:
{
	"displayName":"Bob",
	"email":"bob@bob.com",
  "profileImageURL": "https://pics.pics.com/pic_of_bob.jpg"
}
```

  * The `users` service creates a user from the profile info and fires off a `USER_CREATED` event:
   

```json
{
	"user": {
    "displayName": "Bob",
    "email": "bob@bob.com",
    "id": "aPlayolaUUID"
  },
  "creationData": {
    "spotifyRefreshToken": "NgAagAUmSHo",
    "spotifyAccessToken": "asdfasdfdsaf"
  }
}
```
     
The `spotify` service hears that event and creates a `SpotifyUser` in it's own db:
     
```json
{
  "id": "wizzler",
  "email": "bob@bob.com",
  "refresh_token": "NgAagAUmSHo",
  "playola_uid": "aPlayolaUUID"
}
```
     
The `station` service also hears that event and begins building a station for the user:
     
  * Note: Possible race condition b/c the `station` microservice needs the `spotify` service's user to have been created... 
  * This kicks off the [__StationCreation Flow__](./StationCreationFlow.md)

newly created station model:

```json
{
  "email": "bob@bob.com",
  "displayName": "Bob",
  "stationStatus": "PENDING"
}
```

   

Finally, the `users` microservice responds to the client with a `Bearer` token for making requests.

```json
responds with Bearer token:
{
  "token": "thisisabearertoken"
}
```
