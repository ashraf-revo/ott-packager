# ott-streaming-packager

OTT streaming packager supporting ABR streaming for HLS and DASH

This application is intended to serve as a reliable and scalable OTT streaming repackager (with optional transcoding) to deliver content as part of an overall media streaming platform. There are two key variations of OTT streaming technologies that this software accommodates:

    HLS (HTTP Live Streaming) - Transport Stream HLS and Fragmented MP4 HLS (CMAF style)
    DASH (Dynamic Adaptive Streaming over HTTP) - Fragmented MP4 

With this application, you can ingest *live* MPEG2 transport streams carried over UDP (Multicast or Unicast) for transcoding and/or repackaging into HTTP Live Streaming (HLS) (both TS and MP4) and DASH output container formats.  The application can optionally trancode or just simply repackage.  If you are repackaging then the source streams need to be formatted as MPEG2 transport containing H264/HEVC and AAC audio, however if you are transcoding then you can ingest a MPEG2 transport stream containing other formats as well.  I will put together a matrix layout of the supported modes as I get closer to a v1.0 release.

## Quickstart

The software install guide here is for Ubuntu 16.04 server only, however, you can run this on older/newer versions of Ubuntu as well as in Docker containers for AWS/Google cloud based deployments.

```
cannonbeach@insanitywave:$ sudo apt install git
cannonbeach@insanitywave:$ sudo apt install build-essential
cannonbeach@insanitywave:$ sudo apt install g++
cannonbeach@insanitywave:$ sudo apt install libz-dev
cannonbeach@insanitywave:$ git clone https://github.com/cannonbeach/ott-packager.git
cannonbeach@insanitywave:$ cd ott-packager
cannonbeach@insanitywave:$ make
```
The above steps will compile the application (it is named "fillet"). Please ensure that you already have a basic development environment setup.<br>
<br>

```
The fillet application must be run as a user with *root* privileges, otherwise it will *not* work.

usage: fillet [options]

PACKAGING OPTIONS
       --sources       [NUMBER OF ABR SOURCES - MUST BE >= 1 && <= 10]
       --ip            [IP:PORT,IP:PORT,etc.] (Please make sure this matches the number of sources)
       --interface     [SOURCE INTERFACE - lo,eth0,eth1,eth2,eth3]
                       If multicast, make sure route is in place (see note below)
       --window        [WINDOW IN SEGMENTS FOR MANIFEST]
       --segment       [SEGMENT LENGTH IN SECONDS]
       --manifest      [MANIFEST DIRECTORY "/var/www/html/hls/"]
       --identity      [RUNTIME IDENTITY - any number, but must be unique across multiple instances of fillet]
       --hls           [ENABLE TRADITIONAL HLS TRANSPORT STREAM OUTPUT - NO ARGUMENT REQUIRED]
       --dash          [ENABLE FRAGMENTED MP4 STREAM OUTPUT (INCLUDES DASH+HLS FMP4) - NO ARGUMENT REQUIRED]
       --manifest-dash [NAME OF THE DASH MANIFEST FILE - default: masterdash.mpd]
       --manifest-hls  [NAME OF THE HLS MANIFEST FILE - default: master.m3u8]
       --manifest-fmp4 [NAME OF THE fMP4/CMAF MANIFEST FILE - default: masterfmp4.m3u8]

TRANSCODE OPTIONS (needs to be compiled with option enabled - see Makefile)
       --transcode     [ENABLE TRANSCODER AND NOT JUST PACKAGING]
       --outputs       [NUMBER OF OUTPUT PROFILES TO BE TRANSCODED]
       --vcodec        [VIDEO CODEC - h264 or hevc]
       --resolutions   [OUTPUT RESOLUTIONS - formatted as: 320x240,640x360,960x540,1280x720]
       --vrate         [VIDEO BITRATES IN KBPS - formatted as: 800,1250,2500,500]
       --acodec        [AUDIO CODEC - needs to be aac]
       --arate         [AUDIO BITRATES IN KBPS - formatted as: 128,96]
       --aspect        [FORCE THE ASPECT RATIO - needs to be 16:9, 4:3, or other]
       --scte35        [PASSTHROUGH SCTE35 TO MANIFEST]
       --stereo        [FORCE ALL AUDIO OUTPUTS TO STEREO- will downmix if source is 5.1 or upmix if source is 1.0]
       --quality       [VIDEO ENCODING QUALITY LEVEL 0-3 (0-LOW,1-MED,2-HIGH,3-CRAZY)
                       LOADING WILL AFFECT CHANNEL DENSITY-SOME PLATFORMS MAY NOT RUN HIGHER QUALITY REAL-TIME

H.264 SPECIFIC OPTIONS (valid when --vcodec is h264)
       --profile       [H264 ENCODING PROFILE - needs to be base,main or high]

PACKAGING AND TRANSCODING OPTIONS CAN BE COMBINED                                                                      

```
Simple Repackaging Command Line Example Usage (see Wiki page for Docker deployment instructions which is the recommended deployment method):<br>
```
cannonbeach@insanitywave:$ sudo ./fillet --sources 2 --ip 127.0.0.1:4000,127.0.0.1:4200 --interface lo --window 5 --segment 5 --manifest /var/www/html/hls --identity 1000
```
<br>
This command line tells the application that there are two unicast sources that contain audio and video on the loopback interface. The manifests and output files will be placed into the /var/www/html/hls directory. If you are using multicast, please make sure you have multicast routes in place on the interface you are using, otherwise you will *not* receive the traffic.

This will write the manifests into the /var/www/html/hls directory (this is a common Apache directory).  

<br>
<br>

```
cannonbeach@insanitywave:$ sudo route add -net 224.0.0.0 netmask 240.0.0.0 dev eth0
```

<br>
You should also be aware that the fillet application creates a runtime cache file in the /var/tmp directory for each instance that is run. The cache file is uniquely identified by the "--identity" flag provided as a parameter. It follows the format of: /var/tmp/hlsmux_state_NNNN where NNNN is the identifier you provided. If you want to start your session from a clean state, then you should remove this state file. All of the sequence numbering will restart and all statistics will be cleared as well. It is also good practice when updating to a new version of the software to remove that file. I do add fields to this file and I have not made it backwards compatible.  I am working on making this more configurable such that it can be migrated across different Docker containers.<br>
<br>

```
## Statistics API
An initial restful API is also now available (still working on adding statistics)
curl http://10.0.0.200:18000/api/v1/status
```

<br>

## Packager Operation

The packager has several different optional modes:
- Standard Packaging (H.264/AAC and/or HEVC/AAC)
- Transcoding+Packaging Mode 1 (H.264/AAC) - HLS(TS,fMP4) and DASH(fMP4)
- Transcoding+Packaging Mode 2 (HEVC/AAC) - HLS(fMP4) and DASH(fMP4) - NO TS MODE FOR HEVC
In the transcoding modes, the packaging is bundled together with the transcoding, but I am exploring options to separate the two for a post v1.0 release.

This is a budget friendly packaging/transcoding solution with the expectation of it being simple to setup, use and deploy.  The solution is very flexible and even allows you to run several instances of the application with different parameters and output stream combinations (i.e., have a mobile stream set and a set top box stream set).  If you do run multiple instances using the same source content, you will want to receive the streams from a multicast source instead of unicast.  The simplicity of the deployment model also provides a means for fault tolerant setups.

A key value add to this packager is that source discontinuities are handled quite well (in standard packaging mode as well as the transcoding modes).  The manifests are setup to be continuous even in periods of discontinuity such that the player experiences as minimal of an interruption as possible.  The manifest does not start out in a clean state unless you remove the local cache files during the fast restart (located in /var/tmp/hlsmux...).  This applies to both HLS (handled by discontinuity tags) and DASH outputs (handled by clever timeline stitching the manifest).  Many of the other packagers available on the market did not handle discontinuties well and so I wanted to raise the bar with regards to handling signal interruptions (we don't like them, but yes they happen and the better you handle them the happier your customers will be).

Another differentiator (which is a bit more common practice now) is that the segments are written out to separate audio and video files instead of a single multiplexed output file containing both audio and video.  This provides additional degrees of freedom when selecting different audio and video streams for playback (it does make testing a bit more difficult though).

<br>
In order to use the optional transcoding mode, you must enable the ENABLE_TRANSCODE flag manually in the Makefile and rebuild.  You will also need to run the script setuptranscode.sh which will download and install the necessary third party packages used in the transcoding mode.
<br>

## H.264 Transcoding Example
```
cannonbeach@insanitywave:$ ./fillet --sources 1 --ip 0.0.0.0:5000 --interface eth0 --window 20 --segment 2 --identity 1000 --hls --dash --transcode --outputs 2 --vcodec h264 --resolutions 320x240,960x540 --manifest /var/www/html/hls --vrate 500,2500 --acodec aac --arate 128 --aspect 16:9 --scte35 --quality 0 --profile base --stereo

```

<br>

## HEVC Transcoding Example

```
cannonbeach@insanitywave:$ ./fillet --sources 1 --ip 0.0.0.0:5000 --interface eth0 --window 20 --segment 2 --identity 1000 --hls --dash --transcode --outputs 2 --vcodec hevc --resolutions 320x240,960x540 --manifest /var/www/html/hls --vrate 500,1250 --acodec aac --arate 128 --aspect 16:9 --quality 0 --stereo
````

<br>

## Current Status
(03/04/19) Project is still in active development.  I am still pushing for a v1.0 in the next couple of months.  I pushed up a small update today to clean up a few minor issues:

- MPEG audio decoding
- Fix for audio switching from 5.1 to 2.0 back to 5.1 during commercials.  This is working now, but the audio levels are quieter on these commercials so I am looking at the levels to see if the main downmix can be adjusted to match.  
- Fixed a few small minor code issues (you could call this cleanup)

(02/20/19) As I mentioned in earlier posts, the application is still in active development, but I am getting closer to a v1.0 release.  This most recent update has included some significant transcoding feature improvements.

- 5.1 to stereo downconversion and mono to stereo upconversion added
- SCTE35 passthrough for HLS H.264 output mode with trancoding (added CUE-IN/CUE-OUT to manifest).  Scheduled splices with duration are supported (immediate splices are not implemented yet and will do this with API support)
- Fixed bug where DASH manifest would not output if HLS output mode was not selected
- Finished most of the HEVC encoding path (some minor fixes still needed in the output manifest files for profile/level reporting)
- Fixed DASH a/v sync issue
- Added profile selector for H264 encoding (base,main,high)
- Added quality modes for H264 encoding (low,med,high,crazy)
- Added notification when encoder is unable to make realtime encoding due to limited CPU resources
- Increased internal buffer limits to support a larger number of streams being repackaged

And finally, I am also thinking of putting together a "Pro" version to help me fund the development of this project.  It'll be based on a reasonable yearly fee and provide access to an additional repository that contains a full NodeJS web interface, a more complete Docker integration, benchmarks, cloud deployment examples, deployment/installation scripts, priority support, fully documented API (along with scripts), SNMP traps, and active/passive failover support.

But for those of you that don't wish to take advantage of things like support, the source code for the core application will remain available in the existing repository.  

I also plan to start adapting this current solution over to file version after the v1.0 has been finished and released.

(01/12/19) This application is still in active development and I am hoping to have an official v1.0 release in the next couple of months.  I still need to tie up some loose ends on the packaging as well as complete the basic H.264 and HEVC transcoding modes.  The remaining items will be tagged in the "Issues" section.

I do offer fee based consulting so please send me an email if you are interested in retaining me for any support issues or feature development.  I have several support models available and can provide more details upon request.  You can reach me at: cannonbeachgoonie@gmail.com

See the WIKI page for more information:
https://github.com/cannonbeach/ott-packager/wiki
<br>
