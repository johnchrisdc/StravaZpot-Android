# StravaZpot [![Build Status](https://travis-ci.org/SweetzpotAS/StravaZpot-Android.svg?branch=master)](https://travis-ci.org/SweetzpotAS/StravaZpot-Android)

A fluent API to integrate with Strava on Android apps.

## Usage

This document explains how to use **StravaZpot** in your Android app. For additional questions, you may want to check **Strava** official documentation [here](https://strava.github.io/api/).

## Authentication

### Login button

StravaZpot includes a custom view to include a login button according to Strava guidelines. To do that, you can add the following code to your XML layout:

```xml
<com.sweetzpot.stravazpot.authenticaton.ui.StravaLoginButton
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/login_button"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:layout_gravity="center"
    app:type="orange"/>
```
The custom attribute `type` accepts two different values: `orange` and `light` (default). `StravaLoginButton` makes use of vector drawables in the support library. Thus, if you are targeting version prior to Android API 21, you would need to add the following code to your Activity:

```java
static {
    AppCompatDelegate.setCompatVectorFromResourcesEnabled(true);
}
```

### Login Activity

Strava uses OAuth 2.0 to authenticate users, and then request them to grant permission to your app. To get a Client ID and Secret from Strava for your app, follow [this link](https://www.strava.com/settings/api).

Once you have your app credentials, you can get an `Intent` to launch the login activity for your app. You can easily do it with:

```java
Intent intent = StravaLogin.withContext(this)
                  .withClientID(<YOUR_CLIENT_ID>)
                  .withRedirectURI(<YOUR_REDIRECT_URL>)
                  .withApprovalPrompt(ApprovalPrompt.AUTO)
                  .withAccessScope(AccessScope.VIEW_PRIVATE_WRITE)
                  .makeIntent();
startActivityForResult(intent, RQ_LOGIN);
```

You need to notice several things with this call:

- `<YOUR_CLIENT_ID>` must be replaced with the Client ID provided by Strava when you registered your application.
- `<YOUR_REDIRECT_URL>` must be in the domain you specified when you registered your app in Strava.
- Refer to `ApprovalPrompt` enum to get more options for this parameter.
- Refer to `AccessScope` enum to get more options for this parameter.
- You need to launch the intent with `startActivityForResult` since the login activity will return a value that you will need later to obtain a token. If login to Strava was successful and the user granted permission to your app, you will receive a `code` that you can retrieve with:

```java
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    super.onActivityResult(requestCode, resultCode, data);

    if(requestCode == RQ_LOGIN && resultCode == RESULT_OK && data != null) {
        String code = data.getStringExtra(StravaLoginActivity.RESULT_CODE);
        // Use code to obtain token
    }
}
```

Finally, you need to add the login activity to your manifest:

```xml
<activity
    android:name="com.sweetzpot.stravazpot.authenticaton.ui.StravaLoginActivity"
    android:label="@string/login_strava" />
```

### Obtain a Token

Every Strava API call needs a token to prove the user is authenticated and the app has permission to access the API. After you have obtained the code from user login, you need to exchange it with Strava to get a token. You can do it with the following code:

```java
AuthenticationConfig config = AuthenticationConfig.create()
                                                  .debug()
                                                  .build();
AuthenticationAPI api = new AuthenticationAPI(config);
LoginResult result = api.getTokenForApp(AppCredentials.with(CLIENT_ID, CLIENT_SECRET))
                        .withCode(CODE)
                        .execute();

```
Notice that in this call you must provide the Client ID and Secret provided by Strava when you registered your application, and the code obtained during the login process. Also, the execution of the previous code involves a network request; **you are responsible for calling this code in a suitable thread, outside the UI thread**. Otherwise, you will get an exception.

If the previous request is successful, you will get a `LoginResult`, which has a `Token` that you can use in your subsequent API calls, and an `Athlete` instance, representing the authenticated user.

### Deauthorize

```java
AuthenticationAPI api = new AuthenticationAPI(config);
authenticationAPI.deauthorize()
                 .execute();
```

## Athlete API

### StravaConfig

Before introducing the Athlete API, we have to talk about `StravaConfig`. `StravaConfig` is a class required by all the APIs in **StravaZpot** to configure the way it is going to interact with Strava. You can create a single instance of `StravaConfig` as soon as you obtain a token, and reuse it during your app lifecycle. To create an instance of `StravaConfig`:

```java
StravaConfig config = StravaConfig.withToken(TOKEN)
                                  .debug()
                                  .build();
```

You must provide the token obtained during the authentication process. The call to `debug()` method will show in the Android Monitor what is going on when you do the network requests.

Once you have the configuration object, you can proceed to use all the APIs.

### Create the Athlete API object

```java
AthleteAPI athleteAPI = new AthleteAPI(config);
```

### Retrieve current athlete

```java
Athlete athlete = athleteAPI.retrieveCurrentAthlete()
                            .execute();
```

### Retrieve another athlete

```java
Athlete athlete = athleteAPI.retrieveAthlete(ATHLETE_ID)
                            .execute();
```

### Update an athlete

```java
Athlete athlete = athleteAPI.updateAthlete()
                            .newCity(CITY)
                            .newState(STATE)
                            .newCountry(COUNTRY)
                            .newSex(Gender.FEMALE)
                            .newWeight(WEIGHT)
                            .execute();
```

### Retrieve athlete's zones

```java
Zones zones = athleteAPI.getAthleteZones()
                        .execute();
```

### Retrieve athlete's totals and stats

```java
Stats stats = athleteAPI.getAthleteTotalsAndStats(ATHLETE_ID)
                        .execute();
```

### List athlete K/QOMs/CRs

```java
List<SegmentEffort> koms = athleteAPI.listAthleteKOMS(ATHLETE_ID)
                                     .inPage(PAGE)
                                     .perPage(ITEMS_PER_PAGE)
                                     .execute();
```

## Friend API

### Create the Friend API object

```java
FriendAPI friendAPI = new FriendAPI(config);
```

### List user's friends

```java
List<Athlete> friends = friendAPI.getMyFriends()
                                 .inPage(PAGE)
                                 .perPage(ITEMS_PER_PAGE)
                                 .execute();
```

### List another athlete's friends

```java
List<Athlete> friends = friendAPI.getAthleteFriends(ATHLETE_ID)
                                 .inPage(PAGE)
                                 .perPage(ITEMS_PER_PAGE)
                                 .execute();
```

### List user's followers

```java
List<Athlete> followers = friendAPI.getMyFollowers()
                                   .inPage(PAGE)
                                   .perPage(ITEMS_PER_PAGE)
                                   .execute();
```

### List another athlete's followers

```java
List<Athlete> followers = friendAPI.getAthleteFollowers(123456)
                                   .inPage(2)
                                   .perPage(10)
                                   .execute();
```

### List common following athletes between two users

```java
List<Athlete> followers = friendAPI.getBothFollowing(ATHLETE_ID)
                                   .inPage(PAGE)
                                   .perPage(PER_PAGE)
                                   .execute();
```

## Activity API

### Create the Activity API object

```java
ActivityAPI activityAPI = new ActivityAPI(config);
```

### Create an activity

```java
Activity activity = activityAPI.createActivity(ACTIVITY_NAME)
                               .ofType(ActivityType.RUN)
                               .startingOn(START_DATE)
                               .withElapsedTime(Time.seconds(SECONDS))
                               .withDescription(ACTIVITY_DESCRIPTION)
                               .withDistance(Distance.meters(METERS))
                               .isPrivate(false)
                               .withTrainer(true)
                               .withCommute(false)
                               .execute();
```

### Retrieve an activity

```java
Activity activity = activityAPI.getActivity(ACTIVITY_ID)
                               .includeAllEfforts(true)
                               .execute();
```

### Update an activity

```java
Activity activity = activityAPI.updateActivity(ACTIVITY_ID)
                               .changeName(ACTIVITY_NAME)
                               .changeType(ActivityType.RIDE)
                               .changePrivate(true)
                               .changeCommute(true)
                               .changeTrainer(true)
                               .changeGearID(GEAR_ID)
                               .changeDescription(ACTIVITY_DESCRIPTION)
                               .execute();
```

### Delete an activity

```java
activityAPI.deleteActivity(321934)
           .execute();
```

### List user's activities

```java
List<Activity> activities = activityAPI.listMyActivities()
                                       .before(Time.seconds(BEFORE_SECONDS))
                                       .after(Time.seconds(AFTER_SECONDS))
                                       .inPage(PAGE)
                                       .perPage(ITEMS_PER_PAGE)
                                       .execute();
```

### List user's friends' activities

```java
List<Activity> activities = activityAPI.listFriendActivities()
                                       .before(Time.seconds(BEFORE_SECONDS))
                                       .inPage(PAGE)
                                       .perPage(ITEMS_PER_PAGE)
                                       .execute();
```

### List related activities

```java
List<Activity> activities = activityAPI.listRelatedActivities(ACTIVITY_ID)
                                       .inPage(PAGE)
                                       .perPage(ITEMS_PER_PAGE)
                                       .execute();
```

### List activity zones

```java
List<ActivityZone> activityZones = activityAPI.listActivityZones(ACTIVITY_ID)
                                              .execute();
```

### List activity laps

```java
List<ActivityLap> laps = activityAPI.listActivityLaps(ACTIVITY_ID)
                                    .execute();
```

## Comment API

### Create the Comment API object

```java
CommentAPI commentAPI = new CommentAPI(config);
```

### List activity comments

```java
List<Comment> comments = commentAPI.listActivityComments(ACTIVITY_ID)
                                   .inPage(PAGE)
                                   .perPage(ITEMS_PER_PAGE)
                                   .execute();
```

## Kudos API

### Create the Kudos API object

```java
KudosAPI kudosAPI = new KudosAPI(config);
```

### List activity kudoers

```java
List<Athlete> athletes = kudosAPI.listActivityKudoers(ACTIVITY_ID)
                                 .inPage(PAGE)
                                 .perPage(ITEMS_PER_PAGE)
                                 .execute();
```

## Photo API

### Create the Photo API object

```java
PhotoAPI photoAPI = new PhotoAPI(config);
```

### List activity photos

```java
List<Photo> photos = photoAPI.listAcivityPhotos(ACTIVITY_ID)
                             .execute();
```

## Club API

### Create the Club API object

```java
ClubAPI clubAPI = new ClubAPI(config);
```

### Retrieve a club

```java
Club club = clubAPI.getClub(CLUB_ID)
                   .execute();
```

### List club announcements

```java
List<Announcement> announcements = clubAPI.listClubAnnouncements(CLUB_ID)
                                          .execute();
```

### List club group events

```java
List<Event> events = clubAPI.listClubGroupEvents(CLUB_ID)
                            .execute();
```

### List user's clubs

```java
List<Club> clubs = clubAPI.listMyClubs()
                          .execute();
```

### List club members

```java
List<Athlete> athletes = clubAPI.listClubMembers(CLUB_ID)
                                .inPage(PAGE)
                                .perPage(ITEMS_PER_PAGE)
                                .execute();
```

### List club admins

```java
List<Athlete> athletes = clubAPI.listClubAdmins(CLUB_ID)
                                .inPage(PAGE)
                                .perPage(ITEMS_PER_PAGE)
                                .execute();
```

### List club activities

```java
List<Activity> activities = clubAPI.listClubActivities(CLUB_ID)
                                   .before(BEFORE)
                                   .inPage(PAGE)
                                   .perPage(PER_PAGE)
                                   .execute();
```

### Join a club

```java
JoinResult joinResult = clubAPI.joinClub(123456)
                               .execute();
```

### Leave a club

```java
LeaveResult leaveResult = clubAPI.leaveClub(123456)
                                 .execute();
```

## Gear API

### Create the Gear API object

```java
GearAPI gearAPI = new GearAPI(config);
```

### Retrieve gear

```java
Gear gear = gearAPI.getGear(GEAR_ID)
                   .execute();
```

## Route API

### Create the Route API

```java
RouteAPI routeAPI = new RouteAPI(config);
```

### Retrieve a route

```java
Route route = routeAPI.getRoute(ROUTE_ID)
                      .execute();
```

### List athlete's routes

```java
List<Route> routes = routeAPI.listRoutes(ATHLETE_ID)
                             .execute();
```

## Segment API

### Create the Segment API object

```java
SegmentAPI segmentAPI = new SegmentAPI(config);
```

### Retrieve a segment

```java
Segment segment = segmentAPI.getSegment(SEGMENT_ID)
                            .execute();
```

### List user's starred segments

```java
List<Segment> segments = segmentAPI.listMyStarredSegments()
                                   .inPage(PAGE)
                                   .perPage(ITEMS_PER_PAGE)
                                   .execute();
```

### List another athlete's starred segments

```java
List<Segment> segments = segmentAPI.listStarredSegmentsByAthlete(ATHLETE_ID)
                                   .inPage(PAGE)
                                   .perPage(PER_PAGE)
                                   .execute();
```

### Star a segment

```java
Segment segment = segmentAPI.starSegment(SEGMENT_ID)
                            .execute();
```

### Unstar a segment

```java
Segment segment = segmentAPI.unstarSegment(SEGMENT_ID)
                            .execute();
```

### List segment efforts

```java
List<SegmentEffort> efforts = segmentAPI.listSegmentEfforts(SEGMENT_ID)
                                        .forAthlete(ATHLETE_ID)
                                        .startingOn(START_DATE)
                                        .endingOn(END_DATE)
                                        .inPage(PAGE)
                                        .perPage(ITEMS_PER_PAGE)
                                        .execute();
```

### Retrieve segment leaderboard

```java
Leaderboard leaderboard = segmentAPI.getLeaderboardForSegment(SEGMENT_ID)
                                    .withGender(Gender.FEMALE)
                                    .inAgeGroup(AgeGroup.AGE_25_34)
                                    .inWeightClass(WeightClass.KG_75_84)
                                    .following(true)
                                    .inClub(CLUB_ID)
                                    .inDateRange(DateRange.THIS_WEEK)
                                    .withContextEntries(CONTEXT_ENTRIES)
                                    .inPage(PAGE)
                                    .perPage(ITEMS_PER_PAGE)
                                    .execute();
```

### Explore segments

```java
List<Segment> segments = segmentAPI.exploreSegmentsInRegion(Bounds.with(Coordinates.at(SW_LAT, SW_LONG), Coordinates.at(NE_LAT, NE_LONG)))
                                   .forActivityType(ExploreType.RUNNING)
                                   .withMinimumClimbCategory(MIN_CLIMB_CATEGORY)
                                   .withMaximumClimbCategory(MAX_CLIMB_CATEGORY)
                                   .execute();
```

## Segment Effort API

### Create the Segment Effort API object

```java
SegmentEffortAPI segmentEffortAPI = new SegmentEffortAPI(config);
```

### Retrieve a segment effort

```java
SegmentEffort segmentEffort = segmentEffortAPI.getSegmentEffort(SEGMENT_EFFORT_ID)
                                              .execute();
```

## Stream API

### Create the Stream API object

```java
StreamAPI streamAPI = new StreamAPI(config);
```

### Retrieve activity streams

```java
List<Stream> streams = streamAPI.getActivityStreams(ACTIVITY_ID)
                                .forTypes(StreamType.LATLNG, StreamType.DISTANCE)
                                .withResolution(Resolution.LOW)
                                .withSeriesType(SeriesType.DISTANCE)
                                .execute();
```

### Retrieve segment effort streams

```java
List<Stream> streams = streamAPI.getSegmentEffortStreams(SEGMENT_EFFORT_ID)
                                .forTypes(StreamType.LATLNG, StreamType.DISTANCE)
                                .withResolution(Resolution.LOW)
                                .withSeriesType(SeriesType.DISTANCE)
                                .execute();
```

### Retrieve segment streams

```java
List<Stream> streams = streamAPI.getSegmentStreams(SEGMENT_ID)
                                .forTypes(StreamType.LATLNG, StreamType.DISTANCE)
                                .withResolution(Resoulution.LOW)
                                .withSeriesType(SeriesType.DISTANCE)
                                .execute();
```

### Retrieve route streams

```java
List<Stream> streams = streamAPI.getRouteStreams(ROUTE_ID)
                                .execute();
```

## Upload API

### Create the Upload API object

```java
UploadAPI uploadAPI = new UploadAPI(config);
```

### Upload a file

Strava allows you to upload files with formats GPX, FIT or TCX. We recommend to use TCXZpot in order to generate TCX files that can be uploaded to Strava.

```java
UploadStatus uploadStatus = uploadAPI.uploadFile(new File(<path_to_file>))
                                     .withDataType(DataType.FIT)
                                     .withActivityType(UploadActivityType.RIDE)
                                     .withName("A complete ride around the city")
                                     .withDescription("No description")
                                     .isPrivate(false)
                                     .hasTrainer(false)
                                     .isCommute(false)
                                     .withExternalID("test.fit")
                                     .execute();
```

### Check upload status

```java
UploadStatus uploadStatus = uploadAPI.checkUploadStatus(16486788)
                                     .execute();
```

## Threading

All the APIs in **StravaZpot** perform network requests in a synchronous manner and without switching to a new thread. Therefore, it is up to the user of the library to invoke the API in a suitable thread, and outside the Android UI thread in order to avoid `NetworkOnMainThreadException`.

## Exceptions

**StravaZpot** methods do not have any checked exceptions, but users of the library should be prepared for them to happen. In particular, the following scenarios can arise:

- Strava may return a `401 Unauthorized` response code. In that case, the network request will throw a `StravaUnauthorizedException`. It is up to the user of the library to reuthenticate with Strava to get a new token and retry the request.
- If any other network error happen, or the request is not successful, it will throw a `StravaAPIException`.

## Download

You can get **StravaZpot** from JCenter using Gradle. Just add this line to your build file:

```groovy
compile 'com.sweetzpot.stravazpot:lib:1.3.1'
```

## License


    Copyright 2018 SweetZpot AS

    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.
