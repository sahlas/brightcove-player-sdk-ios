# Using Sidecar Subtitles With The Brightcove Player SDK for iOS, version 6.1.1.158

Introduction
===================

Sidecar Subtitles is an extension to the Brightcove Player SDK that allows you to add WebVTT subtitles to an HLS manifest from within an iOS/tvOS app. There are two primary uses for Sidecar Subtitles:

-  If you are a legacy Video Cloud customer with captions added in Brightcove Studio, Sidecar Subtitles inserts your supplied WebVTT captions into the HLS manifest for playback.
-  If you have additional files that you want to retrieve and add at runtime in your app, Sidecar Subtitles will let you add these WebVTT caption files just before playing back the video.

**Video Cloud Dynamic Delivery**

If you are a Video Cloud Dynamic Delivery customer, and you set all your captions on the Brightcove servers (Brightcove Studio, for example), then you will have **no need** for the Sidecar Subtitles extension. Dynamic Delivery ensures that your captions are inserted into the HLS manifest when your video is retrieved from the Internet, so there is no need for Sidecar Subtitles, and nothing else you need to do.

Still, if you are using Dynamic Delivery and need to add additional subtitle files *after* you have retrieved the video, you can still use Sidecar Subtitles to add them.


Quick Start
===========

**Legacy Video Cloud**

If you are using legacy Video Cloud and need to make sure the captions you set up on the server are inserted into the manifest, then all you need to do is make sure that you are using a Sidecar Subtitles playback controller or session provider:

```
        NSString *policyKey = <your-policy-key>;
        NSString *accountId = <your-account-id>;
        NSString *videoID = <your-video-id>;

        BCOVPlayerSDKManager *manager = [BCOVPlayerSDKManager sharedManager];
    [1] id<BCOVPlaybackController> controller = [playbackManager createSidecarSubtitlesPlaybackControllerWithViewStrategy:nil];
        [self.view addSubview:controller.view];

        BCOVPlaybackService *playbackService = [[BCOVPlaybackService alloc] initWithAccountId:accoundId
                                                                                    policyKey:policyKey];
    [2] [playbackService findVideoWithVideoID:videoID
                                   parameters:nil
                                   completion:^(BCOVVideo    *video,
                                                NSDictionary *jsonResponse,
                                                NSError      *error) {

                                     [controller setVideos:@[ video ]];
                                     [controller play];

                                 }];
```

BCOVSidecarSubtitles adds some category methods to BCOVPlaybackManager. The first of these is `-createSidecarSubtitlesPlaybackControllerWithViewStrategy:`. Use this method to create your playback controller. Alternately (if you are using more than one session provider), you can create a BCOVSSSessionProvider and pass that to the manager method that creates a playback controller with upstream session providers.

* If you are developing for tvOS, the viewStrategy passed to createSidecarSubtitlesPlaybackControllerWithViewStrategy must be nil.

* Note that `BCOVSSSessionProvider` should come before any session providers in the chain passed to the manager when constructing the playback controller. This extension is **not compatible** with the Widevine plugin.

If you have questions or need help, visit the support forum for Brightcove Native Player SDKs at https://groups.google.com/forum/#!forum/brightcove-native-player-sdks .

Using the Built-In PlayerUI Controls
---
The code snippet above presents a video player without any controls. You can add playback controls to your code like this.

Add a property to keep track of the `BCOVPUIPlayerView`:

    // PlayerUI's Player View
    @property (nonatomic) BCOVPUIPlayerView *playerView;

Create the `BCOVPUIBasicControlView`, and then the `BCOVPUIPlayerView`. This is where we associate the Playback Controller (and thus all the videos it plays) with the controls. Set the player view to match the video container from your layout (`yourVideoView`) when it resizes.

    BCOVPUIBasicControlView *controlView = [BCOVPUIBasicControlView basicControlViewWithVODLayout];
    self.playerView = [[BCOVPUIPlayerView alloc] initWithPlaybackController:controller options:nil controlsView:controlView];

    // Match parent view size
    self.playerView.frame = self.yourVideoView.bounds;
    self.playerView.autoresizingMask = UIViewAutoresizingFlexibleHeight | UIViewAutoresizingFlexibleWidth;

    // Add BCOVPUIPlayerView to your video view.
    [self.yourVideoView addSubview:self.playerView];

If you use the BCOVPUIPlayerView, you also need to remove one line above:

    [self.view addSubview:controller.view]; // no longer needed when using PlayerUI
    
If you want to reuse the player with with another playback controller, you can simply make a new assignment:

	self.playerView.playbackController = anotherPlaybackController;

The player view will automatically add the playback controller's view to its own view hierarchy.

Please see the Brightcove Native Player SDK's README for more information about adding and cumstomizing PlayerUI controls in your app.

If you have questions or need help, visit the [Brightcove Native Player SDK support forum](https://groups.google.com/forum/#!forum/brightcove-native-player-sdks).

Manually populating subtitle data
=================================
Whether using legacy Video Cloud, or Video Cloud with Dynamic delivery, you can add your own WebVTT captions files at runtime.

The BCOVSidecarSubtitle extension will look for the presence of an array of subtitle metadata in the `BCOVVideo` object properties, keyed by `kBCOVSSVideoPropertiesKeyTextTracks`. If you are using `BCOVPlaybackService` to retrieve videos and those videos have text tracks associated with them, this will be populated automatically.

If you are providing your own videos or are using Brightcove Player without Video Cloud, you will need to structure the data as shown below:
	
	NSArray *subtitles = @[
	@{
		kBCOVSSTextTracksKeySource: ..., // required
		kBCOVSSTextTracksKeySourceLanguage: ..., // required
		kBCOVSSTextTracksKeyLabel: ..., // required
		kBCOVSSTextTracksKeyDuration: ..., // required/optional [1]
		kBCOVSSTextTracksKeyKind: kBCOVSSTextTracksKindSubtitles or kBCOVSSTextTracksKindCaptions, // required
		kBCOVSSTextTracksKeyDefault: ..., // optional
		kBCOVSSTextTracksKeyMIMEType: ..., // optional
	},
	@{...}, // second text track dictionary
	@{...}, // third text track dictionary
	];
	   
	BCOVVideo *video = [BCOVVideo alloc] initWithSource:<source>
	                         cuePoints:<cuepoints>
	                        properties:@{ kBCOVSSVideoPropertiesKeyTextTracks:subtitles }];

The `kBCOVSSTextTracksKeySource` key holds the source URL of your subtitle track, and can be supplied as either a WebVTT URL or an M3U8 playlist URL.

WebVTT files should have a ".vtt" extension, and M3U8 files should have an ".M3U8" extension. If you cannot follow these conventions, you must include a `kBCOVSSTextTracksKeySourceType` key, and specify either `kBCOVSSTextTracksKeySourceTypeWebVTTURL` or `kBCOVSSTextTracksKeySourceTypeM3U8URL` to indicate the type of file referred to by the URL.

If you are supplying tracks to a video retrieved from Video Cloud, you should **add** your subtitles to any existing tracks rather than **overwriting** them. This code shows how you can add tracks to an existing video:
		
	BCOVVideo *updatedVideo = [video update:^(id<BCOVMutableVideo> mutableVideo) {
		
		// Save the current tracks
		NSArray *originalTracks = video.properties[kBCOVSSVideoPropertiesKeyTextTracks];
		
		// Create your text track dictionary
		NSArray *subtitles = @[
		@{
			kBCOVSSTextTracksKeySource: ..., // required
			kBCOVSSTextTracksKeySourceLanguage: ..., // required
			kBCOVSSTextTracksKeyLabel: ..., // required
			kBCOVSSTextTracksKeyDuration: ..., // required/optional [1]
			kBCOVSSTextTracksKeyKind: kBCOVSSTextTracksKindSubtitles or kBCOVSSTextTracksKindCaptions, // required
			kBCOVSSTextTracksKeyDefault: ..., // optional
			kBCOVSSTextTracksKeyMIMEType: ..., // optional
		},
		@{...}, // second text track dictionary
		@{...}, // third text track dictionary
		];
		
		// Append new tracks to the original tracks, if any
		NSArray *combinedTextTracks = ((originalTracks != nil)
										? [originalTracks arrayByAddingObjectsFromArray:subtitles]
										: subtitles);
						
		// Update the current dictionary (we don't want to lose the properties already in there)
		NSMutableDictionary *updatedDictionary = [mutableVideo.properties mutableCopy];
			
		// Store text tracks in the text tracks property
		updatedDictionary[kBCOVSSVideoPropertiesKeyTextTracks] = combinedTextTracks;
			
		mutableVideo.properties = updatedDictionary;
	  
	}];
Notes
============

* `kBCOVSSTextTracksKeyDuration` is a required key if you are using caption files with a .vtt extension. `kBCOVSSTextTracksKeyDuration` is an optional key if you are using using caption files with a .m3u8 extension.

Please refer to the code documentation in the BCOVSSComponent.h header file for more information on usage of these keys.

Known Issues
============

* Subtitles will not be displayed when viewing 360 degree videos.

* This extension currently does not support integrating with the Widevine Plugin for Brightcove Brightcove Player SDK for iOS.

* When retrieving your videos from the Brightcove Playback API, your renditions must include a master M3U8 playlist. Sidecar Subtitles does not work with single rendition M3U8 playlists.

* If you are providing a subtitle playlist to the Sidecar Subtitles, and that subtitle playlist includes links to web vtt files that respond as 404, playback will fail. This is a bug in Apple's AVPlayer.

* Sidecar Subtitles is not needed when when downloading HLS videos for offline playback, or for playback of offline videos with subtitles.
