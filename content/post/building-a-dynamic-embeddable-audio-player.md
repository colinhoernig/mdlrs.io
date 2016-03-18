---
author: "colin-hoernig"
comments: true
date: 2016-03-15T10:30:34-05:00
draft: false
image: ""
menu: ""
share: true
slug: building-a-dynamic-embeddable-audio-player
tags:
- aws
- api
- javascript
title: Building a Dynamic Embeddable Audio Player With AWS API Gateway, Mustache, and Audio5js
---

As one might guess, we're _very_ passionate about music here at [Music Dealers](https://musicdealers.com).  We love _everything_ about music, but we _especially_ love the incredible community of independent musicians that create the foundation of our company.

Since inception, independent artists from our community have collectively uploaded ~200,000 tracks to the Music Search platform.  In early 2016, we rolled out a <s>new</s> revamped blog _(stay tuned for a future blog post --- hint: static site generation)_.  During development, we saw a need to make it easier to share our artists' content with the world.  In response, we built a dynamic, modular audio player that can be easily added to any page.

![](http://res.cloudinary.com/mdlrs/image/upload/v1458147346/md-player-screenshot_cwztld.png)

##### When building the audio player, we had a few main goals in mind:

- Dynamic: We should be able to take any song from our vast collection, generate its waveform on the fly, and embed the player in any page.
- Fast: When viewing an embedded player, player loading and response times should be as fast as possible.
- Multiple Player Support: Any page should be able to contain multiple players, with only one active player at any given time.
- Secure: To protect our artists, only public facing data should be exposed to the world.

##### There are four main parts making up the audio player:

- AWS API Gateway proxies song requests to a Laravel/ElasticSearch API
- AWS API Gateway proxies waveform requests to a Node.js server and WaveformGenerator module
- [Audio5js](https://github.com/zohararad/audio5js) handles audio events for native HTML5 audio implementations
- [Mustache](https://mustache.github.io/) handles player templating

In the next few sections, we'll dig deeper into the audio player implementation details.

---

#### HTTP Proxy Requests Using AWS API Gateway

Matthew, our Lead Developer at Music Dealers, has been building (and consistenly improving) a robust API to our large database of artist and song data ([register for an API key](http://www.musicdealers.com/developers) / [view the docs](http://docs.mdlrs.apiary.io/#)).

We can easily retreive song data from the Music Dealers API:

```bash
curl -X GET -H "X-Auth-Token: api-key-here" \
-H "Cache-Control: no-cache" "http://api.musicdealers.com/songs/1852526"
```

The `GET /songs` request will hit our API server, where Laravel will authenticate/authorize the user and make a request to our ElasticSearch cluster for the proper `song` data.  Thanks to ElasticSearch's incredible speed, a successful response is returned containing `song` data very quickly.

Our `/songs` API endpoint returns _**a lot of data**_ about the song, as well as the associated artist.  For the new player, we only needed access to small subset of the song/artist data.  This is where AWS API Gateway comes to save the day.

##### Configuring MD API Gateway

AWS API Gateway makes it very simple to set up an API.  In our case, we're using API Gateway as a proxy to pass song requests to a remote Laravel/ElasticSearch API and waveform requests to a remote Node.js WaveformGenerator API.  The MD API Gateway setup is structured as such:

```text
/
  /song         <-- resource
    /{id}       <-- resource
      GET       <-- resource method
    /waveform   <-- resource
      GET       <-- resource method
```

This is to say, if we send a:

- GET request to `/song/{id}` with any ID and associated query params, it should map resources and query params, and then make a request to the remote Laravel/ES API
- GET request to `/song/waveform` with associated query params, it should map resources query params, and then make a request to the Node.js WaveformGenerator API

Each resource method execution lifecycle consists of:

- **client method request** - the API Gateway request endpoint accepts the request, and defines the request paths and URL Query String Parameters
- **integration request** - the API Gateway takes the incoming request paths and URL Query String Parameters and maps them to the target backend (in our case, remote Laravel/ES and Node.js APIs)
- **integration response** - Maps the possible responses from the backend to a specific format (model transformation) based on response type and status
- **method response** - API Gateway sends back the HTTP status code

With an understanding of the API architecture, let's take a quick look at the method executions for MD API Gateway.

###### `GET /song/{id}`

The goal for this method is to accept a song ID via the MD API Gateway and use it to make an authenticated request to the Music Dealers API `/songs/{id}` endpoint.

According to the Music Dealers API [documentation](http://docs.mdlrs.apiary.io/#reference/songs/song/get-a-single-song), the `/songs` endpoint accepts a song ID as the second request path parameter (i.e. `/songs/1852526` where 1852526 is a song ID).  Lucky for us, there's a 1-1 mapping here: the song ID from the MD API Gateway **Method Request** maps directly to Music Dealers API endpoint.

In the **Integration Request**, we define it as an HTTP Proxy and map `method.request.path.id` from MD API Gateway request paths to the `id` URL Path Parameter (`https://api.musicdealers.com/songs/{id}`).  Finally, tack on the `X-Auth-Token` HTTP Header so we can auth against the remote API.

_Note: Constant values can be added in the 'Mapped from' section of Integration Request by wrapping them in single quotes._

In the **Integration Response**, we have a mapping template set up for all `HTTP 200 application/json` responses.  This template transforms Music Dealers API response data, mapping it into a specified format:

```javascript
#set($inputRoot = $input.path('$'))
#if($input.params("callback") != '')$input.params("callback")(#end{
"id" : $inputRoot.results[0].id,
"sku" : "$inputRoot.results[0].sku",
"title" : "$inputRoot.results[0].title",
"duration" : $inputRoot.results[0].duration,
"artist" : {
  "name" : "$inputRoot.results[0].artist.name",
  "sku" : "$inputRoot.results[0].artist.sku",
  "profile_pic_90_url" : "$inputRoot.results[0].artist.profile_pic_90_url",
  "id" : $inputRoot.results[0].artist.id
},
"media" : {
  "streaming" : "$inputRoot.results[0].media.streaming"
}
}#if($input.params("callback") != ''))#end
```

_Note 1: One of our input params is `callback` which we use for jsonp requests.  We use `$inputRoot.results[0]` to pick only the first object in the `results` array._
_Note 2: AWS API Gateway supports Model definitions for easy Mapping Template generation.  Learn more about Models and Mapping Templates [here](http://docs.aws.amazon.com/apigateway/latest/developerguide/models-mappings.html)._

Finally, after deploying the MD API Gateway, we can test the endpoint and view the response.  The JSON contains all of the data that we need to create an audio player instance.

```json
{
  "id": 1852526,
  "sku": "A-C4F8F7-9218-V",
  "title": "Welcome",
  "duration": 129,
  "artist": {
    "name": "RADIO",
    "sku": "A-C4F8F7",
    "profile_pic_90_url": "http://www.musicdealers.com/sites/default/files/imagecache/90x90_scale_crop/profile_pics/radio3000_20141026_rmi_0794.jpg",
    "id": 355467
  },
  "media": {
    "streaming": "//media.musicdealers.com/music-dealers/s/A-C4F8F7/1852526/MDV-A-C4F8F7-9218-V-Welcome"
  }
}
```

Whew...that was a lot to take in.  Onto the next one.

###### `GET /song/waveform`

The goal for this method is to accept a variety of input data via the MD API Gateway URL Query String Parameters and use it to make a request to the Music Dealers Node.js WaveformGenerator API.  I won't go into detail on how our Node.js WaveformGenerator API is built, but I can say that it uses FFMPEG and is inspired by [node-pcm](https://github.com/jhurliman/node-pcm).

Our Node.js WaveformGenerator API accepts a variety of input data which gets transformed into a custom audio waveform.  Dissimilar to the `GET /song/{id}` resource, the MD API Gateway `GET /song/waveform` **Request Method** accepts URL Query String Parameters instead of Request Paths.  Notably, the params are: `bar_width`, `artist_sku`, `peaks`, `catalog_slug`, `bar_gap`, `filename`, `vertical_align`, `round`, `song_id`, `size`, `color`, and `callback`.

In the **Integration Request**, the HTTP Proxy makes a request to our Node.js Waveform API endpoint: `/{catalog_slug}/{artist_sku}/{song_id}/{filename}`.  The `filename`, `song_id`, `artist_sku`, and `catalog_slug` Request Method Query String parameters map to Integration Request URL Path Parameters of the same name, while the rest of the Request Methods' Query String parameters map to URL Query String Parameters in the Integration Request (i.e. `method.request.querystring.peaks => peaks`).

The **Integration Response** execution phase for `GET /song/waveform` is very simple, as it's just an Output Passthrough for all `HTTP 200 application/json` responses. 'Output Passthrough' means that the JSON response from our Node.js WaveformGenerator API is passed through unmodified as the response to the MD API Gateway.

If a `.json` type file is passed in as `filename`, an example response for the same song data in the previous section looks like this:

```json
{
  "count": 250,
  "peaks": [
    50,
    32,
    30,
    43,
    34,
    30,
    61,
    ... // a few hundred peak elements removed for brevity
  ]
}
```

We can then use this JSON to build a dynamic waveform using plain old JS, HTML, and CSS.  For the player use case, we leave out the `peaks` input parameter as the Waveform Generator API will calculate the number of peaks based on the `size` we pass in.  We also leave out `color`, as we use CSS to style the waveform.

_Note: Leaving out the `filename` query param will generate the waveform and return HTML with inline CSS as the response._

With all of the nitty-gritty API details out of the way, let's move on to templating.

---

#### Templating the Audio Player in Mustache

Mustache is logic-less templating language.  We opted to use Mustache.js for our audio player for two main reasons:

- it offers a clean syntax for creating readable template code
- our team is familiar with it, giving increased productivity

Our main player template is very simple, containing only five template variables.  We can drop the player template in any page, and `md-player.js` module will find, parse, and render it using the data returned from the MD API Gateway.

```html
<script type="text/template" id="player-template">
<div class="media md-player-body">
  <div class="media-left">
    <div class="play-img-wrap md-play">
      <img src="[[artist.profile_pic_90_url]]" class="img-rounded artist-image">
      <i class="fa fa-play play-pause-icon play-icon" aria-hidden="true"></i>
      <i class="fa fa-pause play-pause-icon pause-icon" aria-hidden="true"></i>
    </div>
  </div>
  <div class="media-body">
    <h3 class="media-heading">
      <strong>[[ title ]]</strong> <small>by</small> [[ artist.name ]] <small><span class="elapsed">[[ elapsed ]]</span> - [[ duration_formatted ]]</small>
    </h3>
    <div class="audio-player">
      <div class="player-progress">
        <div class="loaded-container"></div>
        <div class="played-container"></div>
        <div class="waveform-container"></div>
      </div>
    </div>
  </div>
</div>
</script>
```

_Note: This particular template uses CSS classes from [Bootstrap 3](http://getbootstrap.com/).  A custom template may be supplied instead._

Notice the `.waveform-container` div?  That's where we inject our generated waveform HTML from within the `md-player.js` module.  Let's take a look.

---

#### md-player.js - Tying Everything Together

So, we have the proxy API built, we have our player template built...how do we tie it all together?  The goal is to use a simple snippet of code and have it magically turn into an audio player on the page:

```html
<div class="md-player" data-song-id="1630299"></div>
```

We use the great Audio5js library to handle player events (think `load`, `play`, `pause`, `progress`, etc).  The creator of Audio5js calls it "The HTML5 Audio Compatility Layer" --- it wraps native HTML5 audio with a cross-browser JavaScript API and uses Flash as a fallback.  Audio5js is a simple yet powerful library.  It provides no UI, supports multiple codecs, and relies on no external libraries.

So, how can we use Audio5js to tie together our API and templates?  Well, it's simpler than one might think.  We first create a new Audio5 instance and define the necessary event handlers:

```javascript
var $current_player = null;
var audio = new audio5({
  swf_path: '/audio5js/audio5js.swf',
  format_time: false, // keep it in seconds for width calculations
});
audio.on('play', function () {
  $current_player.find('.md-play');
  $('.is-playing').removeClass('is-playing');
  $current_player.addClass('is-playing');
});
audio.on('pause', function () {
  $current_player.removeClass('is-playing');
  $current_player.find('.md-play');
});
audio.on('canplay', function() {
  audio.play();
});
audio.on('timeupdate', function (position, duration) {
  $current_player.find('.played-container').width((position/duration)*100 + '%');
  $current_player.find('.elapsed').text(sec_to_time(position));
});
audio.on('progress', function (load_percent) {
  $current_player.find('.loaded-container').width(load_percent + '%');
});
```

That's the basic setup for Audio5js...later on we'll take it a bit futher.  The next thing we need is to fetch data from our API.

The idea is to grab the template markup from the page, find all of the player (`.md-player`) elements on the page, fetch their song and waveform data from the MD API Gateway, and then render the markup inside the player elements.

```javascript
// pull default template from page
var base_template = $('#player-template').html();

// Fetch data for each player, render template into containers
$('.md-player').each(function(idx, el) {

  // set template
  var t = base_template;

  // Fetch song data from MD API Gateway
  $.ajax({
    url: api_url + $(el).data('song-id'),
    jsonp: "callback",
    dataType: "jsonp",
    success: function(data) {
      // We have the song data
      data.elapsed = '00:00';
      $(el).data('song', data);
      data.duration_formatted = sec_to_time(data.duration);

      // Check for custom template
      if (cti = $(el).data('template-id')) {
        // Make sure custom template exists
        if (ctc = $('#'+cti).html()) {
          t = ctc;
        }
      }

      // Render template (we use double brackets instead of double mustaches)
      // to work with Golang templates
      mustache.parse(t, ["[[", "]]"]);
      $(el).html(mustache.render(t, data));

      // Get the width of the waveform container
      var waveformWidth = $(el).eq(0).find('.waveform-container').width();

      // Map properties to data
      var q = {
        callback: "callback",
        color: {
          fg: "BE3026",
          bg: ""
        },
        bar_width: 1,
        bar_gap: 2,
        filename: "waveform.json",
        vertical_align: 1,
        round: 1,
        song_id: data.id,
        artist_sku: data.artist.sku,
        catalog_slug: (data.media.streaming.split('/').filter(function(v) { return v !== ''; }))[1],
        size: {
          height: 55,
          width: waveformWidth
        }
      };

      // Build query param string
      var waveformUrlParams = '?callback='+q.callback+'&bar_width='+q.bar_width
        +'&artist_sku='+q.artist_sku+'&catalog_slug='+q.catalog_slug+'&bar_gap='
        +q.bar_gap+'&filename='+q.filename+'&vertical_align='+q.vertical_align
        +'&song_id='+q.song_id+'&size='+q.size.width;

      // Hit the MD API Gateway waveform endpoint
      $.ajax({
        url: api_url + '/waveform' + waveformUrlParams,
        jsonp: "callback",
        dataType: "jsonp",
        success: function(waveData) {
          // We have the wave data, let's build the HTML waveform
          var waveformDOM = '<div style="overflow: hidden; height: ' + q.size.height + 'px; width: ' + q.size.width + 'px; background-color: #' + (q.color.bg || 'transparent')  + ';">';
          for (var i = 0; i < waveData.peaks.length; i++) {
            var bottom_margin = (!!q.vertical_align) ? (q.size.height - (Math.ceil((waveData.peaks[i]) * (q.size.height/100))))/2 + 'px' : 0;
            waveformDOM += '<div style="margin-bottom: ' + bottom_margin + '; width: ' + q.bar_width + 'px; height: ' + Math.ceil((waveData.peaks[i]) * (q.size.height/100)) + 'px; background: #' + q.color.fg + '; display: inline-block; margin-left: ' + q.bar_gap + 'px; border-radius: ' + q.round + ';"></div>';
          }
          waveformDOM += '</div>';
          // Inject the generated HTML into the player's waveform container
          $(el).find('.waveform-container').html(waveformDOM);
        }
      });
    }
  });
});
```


Earlier, I mentioned that we'd add a bit more functionality to our Audio5js player.  Let's do that now.  It would be nice to pause all other players when another is playing.  Wouldn't it also be nice if we could seek around the audio source by selecting different points in the waveform?  It sure would be.  Let's implement that functionality.

```javascript
// Attach a click event to the play/pause button of each player
$('.md-player').on('click', '.md-play', function(evt) {

  // Keep a reference to any player that's playing (may be null)
  var $prev_player = $current_player;

  // Find the player that had 'play/pause' clicked
  $current_player = $(evt.currentTarget).closest('.md-player');

  // If previous player and current player are the same, play/pause
  if ($prev_player && $prev_player.data('song').id == $current_player.data('song').id) {
    audio.playPause();
  } else {
    // If new player was clicked, load track from API data
    audio.load('http:' + $current_player.data('song').media.streaming + '.mp3');
    audio.seek(0); // Seek back to beginning of new track
  }

// Attach a click event to the progress bar of each player
}).on('click', '.player-progress', function(evt) {

  // Keep a reference to any player that's playing (may be null)
  var $prev_player = $current_player;

  // Find the player that had the progress bar clicked
  $current_player = $(evt.currentTarget).closest('.md-player');
  I
  // Calculate where in the progress bar was clicked
  var p = evt.offsetX / $(evt.currentTarget).width();

  // If the progress bar on the currently active player was clicked, seek to that point
  if ($prev_player && $prev_player.data('song').id == $current_player.data('song').id) {
    audio.seek(p * audio.duration);
  } else {
    // If an inactive player's progress bar was clicked, load song from API data and seek
    audio.load('http:' + $current_player.data('song').media.streaming + '.mp3');
    audio.seek(p * $current_player.data('song').duration); // audio.duration is the old song
  }
});
```

Finally, we have our APIs for fetching song and waveform data, templates for player structure and styling, Audio5js handling audio events, and md-player.js tying it all together.  All that's left is to add the player snippet to the page and watch the magic happen.

```html
<div class="md-player" data-song-id="1852526"></div>
```

<div class="md-player" data-song-id="1852526"></div>

---

#### Closing Thoughts

Building an embeddable, dynamic audio player allows us to easily share our artists' music with the world.  We had a lot of fun working on this project and were able to use some cool technologies and libraries.

If you made it this far, thanks for reading.  I hope that you enjoyed it.

As previously mentioned, keep an eye out for a future blog post by Matthew, our Lead Developer, covering how the new [Music Dealers Blog](http://www.musicdealers.com/blog) was built.

<script src="//mdlrs.com/widgets/md-player/md-player-1.0.js"></script>
