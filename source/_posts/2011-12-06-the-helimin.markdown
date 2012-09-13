---
layout: post
title: "The Helimin (or how to threshold based on hue in OpenCV)"
date: 2011-12-06 19:31
comments: true
categories: 
---
{% img left /images/content/helimin.jpg 'The Helimin' 'No this is not stock photography' %}

That toy helicopter is what turned out to be the most viable candidate in a long list of outrageous and impractical musical instruments. The result of my weekend at the Music Hackday London.

I remember first hearing about the Music Hackday shortly after I moved to London, in September 2009. It was presented to me as a bit of a London institution. Being an attendee of the first ever Music Hackday is a bit like saying you saw the Sex Pistols at the Less Free Trade Hall (if the gig had been held in the Guardian offices). New attendees such as myself crowd around and listen to stories of the first event, nodding in revered silence.

You'd be right in thinking the Music Hackday is quite a big event around here. Part of it could be to do that the Sillicon Roundabout "brand" somewhat kicked off through last.fm and songkick's efforts. Or maybe England is just that much in love with its image of the spiritual home of music-that-is-rather-good-if-I-dare-say-so-myself, that a sense of musical enthusiasm has seeped into other industries. In a good way.

{% pullquote %}
{"Now, if the 2011 edition had been your first Music Hackday, you'd be forgiven for mistaking it for a Spotify launch event."} Spotify were handing out (up to now) elusive developer licences for making Spotify apps on their new apps platform. The excitement was almost nauseating. Attendees were genuinely disgusted with themselves.
{% endpullquote %}

Not having the opportunity to do web apps very often these days, this would have been the perfect chance to refresh my HTML 5 skills, some WebGL perhaps, and experiment with some of the APIs we were given access to. Lots of interesting stuff to play with.

Somehow it didn't quite work out that way.

<!-- more -->

### Enter, the helicopter ###

Now, I could come up with all manner of clever reasons *why* I decided to make something with this year's default christmas toy bought by step-dad's all over the country (hey, at least it's not a [pooping dachshund](http://www.guardian.co.uk/money/2011/oct/26/top-christmas-toys-2011-doggie-doo "The Guradian's Top Christmas toys 2011"))... truth is I was looking for an excuse to buy one. *Nothing* could stop me in my plan to make something out of this. I'd stay up all night, through tears of joy, counting down to the day I'd fly a tiny RC helicopter around the lovely, *Le Corbusier-inspired,* Barbican.

### Impure data ###

My goal, ultimately, was to make a musical instrument. My big break into the world of producing coherent noises (EMI was present there, after all... I mean who knows, right?).

Unfortunately I had never made a musical instrument before. I had, however, been wanting to play with [PureData](http://puredata.info/), and thought this event -- In which I'd be surrounded by musically inclined programmers -- would be the perfect time to have a play around. 

I soon managed to generate some horrible sine waves (don't debug with headphones, folks). Lucky for me I spoke to a gentleman who introduced me to the wonders of [rjlib](https://github.com/rjdj/rjlib/ "rjlib Github page"), a third party library of incredibly useful musical functions. Not long after, I wrote a puredata *Patch* which could take an integer and generate a nice reverbery tone modulated to the pentatonic scale. If Bobby McFerrin has tought me something, it's that [everyone loves the pentatonic scale](http://www.youtube.com/watch?v=ne6tB2KiZuk "Bobby McFerrin's lecture at the World Science Festival 2009"). I also made it generate some random noise convolved with an oscillator controled by a second input number (why not?). The result sounded horrifically acceptable.

{% pullquote %}
So that was two variables to control the instrument. {"The pitch would be controlled by the altitude of the helicopter, and the weird oscillating noise would be controlled by the helicopter's horizontal position."} But how could I make this happen?
{% endpullquote %}

### Finding a helicopter in a haystack ###

Short if modifying the helicopter's hardware in some manner, my laptop's webcam seemed the most sensible input device. I had experimented with [openFrameworks](http://www.openframeworks.cc/ "openFrameworks homepage") before, so this seemed like an obvious place to start. OpenFrameworks is a collection of c++ libraries which makes writing code to render visualisations and manipulate video and sound very easy. It includes [OpenCV](http://opencv.willowgarage.com/wiki/ "OpenCV wiki"), everyone's favourite computer vision library.

With openFrameworks's excellent [OSC addon](http://opensoundcontrol.org/implementation/ofxosc "Homepage for OfxOSC"), wiring up a helicopter detection binary to the PD instrument would be easy.

But how could I track the helicopter visually?

I first thought of hanging an easily-identifiable LED from the helicopter. Unfortunately that rendered it non-flight-worthy. I did however notice that I happened to have an ostentatious helicopter and that the decorators of the Barbican had taste. I tried a simple background subtraction method to try and classify individual pixels as either helicopter or non-helicopter. Background subtraction works by capturing the scene viewed by the webcam with no helicopter in it (we'll call it **B**), then flying the helicopter in the scene and having the program capture a frame **A**, regularly. Any difference between **A** and **B** greater than a threshold, can be considered potential helicopter pixels.

``` cpp A very simple way of doing thresholding on background subtraction without any other libraries.
void bg_subtract(const unsigned char* sourceBw_p, const unsigned char* backBw_p,
                 const unsigned int monoimagesize, float thresh, unsigned char* diff_p) {
    //a good candidate for #pragma omp, if you are that way inclined
    for (int i=0; i<monoimagesize; i++) {
        float diff_v  = sourceBw_p[i] - backBw_p[i];
        if (diff_v < 0) diff_v = -diff_v;

        if (diff_v>thresh) diff_p[i]=255;
        diff_p[i] = 0;
    }
}
```

It's then simply a case of using OpenCV's contour finder to find the first blob of continuous pixels big enough to be a helicopter (there should hopefully only be one), and finding its centre of mass.

``` cpp using openCV to find blobs of a certain size
ofxCvContourFinder helicopterFinder;
helicopterFinder.findContours(diff, 10, 5000, 2, true, false);
if (helicopterFinder.blobs.size() > 0) {
    int x = helicopterFinder.blobs[0].centroid.x;
    int y = helicopterFinder.blobs[0].centroid.y;
}
```

This wasn't good enough. Movement in the background was to be predicted as the operator would likely stand in the field of view. Certain solutions exist to mitigate issues with changing background, such as updating the background by using a temporal mean of frames **A** as **B**. This works quite well in situations such as CCTV, with a mostyl unchanging background, but would likely not have worked in this situation, where demoing the system would not be done over a long enough time to let a temporal mean settle.

The fact the helicopter was of a very uniform and unique colour, game me an idea. I measured the hue of the paint job as picked up by my webcam on the HSV scale. The HSV scale specifies colours by their hue, saturation and value, rather than RGB components. As long as the webcam is white balanced for the current lighting, thresholding pixels based on their difference in hue with respect to the helicpoter can then easily be done, generating image **G**.

``` cpp image thresholding based on hue in openFrameworks, using OpenCV
cvCvtColor(source.getCvImage(), imgHSV, CV_RGB2HSV);

imgThreshed = cvCreateImage(cvGetSize(imgHSV), 8, 1);
cvInRangeS(imgHSV, cvScalar(hue-10, low_sat, low_bright),
           cvScalar(hue, 255, 255), imgThreshed);
//low_sat and low_brightness are configurable offsets set at runtime
```

by segmenting the picture by a convolution of hue thresholding and of the foreground image **F * G**, and running the contour finder on the result, I was able to pick out the helicopter with supprising accuracy, in varying light conditions.

Of course, when it came to demo time, I flew the daft thing into the ceiling and stage lights.

You can see it in action in this YouTube video (the fact I can get it airborn for more than 3 seconds is nothing short of miraculous) and find a mention of it in [this Wired article](http://www.wired.co.uk/news/archive/2011-12/05/music-hack-day-london?page=2 "Spotify apps dominate at Music Hack Day London") by [@duncangeere](http://twitter.com/duncangeere).

{% youtube CnEP8uHZL6c %}

The event itself was tremendous fun. The sheer amount of talent and cleverness present in the venue over the weekend was inspiring, if not intimidating. I hope I get to go again.

Maybe next year I'll make something useful.

(Also, a bit more info on the Helmin at the [Music Hackday wiki](http://wiki.musichackday.org/index.php?title=Helimin))
