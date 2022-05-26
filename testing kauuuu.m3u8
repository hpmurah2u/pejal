use std::ops::Index;
use std::result;
use std::str::FromStr;

use serde::de::{Deserialize, Deserializer};
use {Client, Error, Result};

// pub mod format;
pub mod podcast;
mod radio;
pub mod song;
pub mod video;

pub use self::radio::RadioStation;
use self::song::Song;
use self::video::Video;
// pub use self::podcast::{Podcast, Episode};

// use self::format::{AudioFormat, VideoFormat};

/// A trait for forms of streamable media.
pub trait Streamable {
    /// Returns the raw bytes of the media.
    ///
    /// Supports transcoding options specified on the media beforehand. See the
    /// struct-level documentation for available options. The `Media` trait
    /// provides setting a maximum bit rate and a target transcoding format.
    ///
    /// The method does not provide any information about the encoding of the
    /// media without evaluating the stream itself.
    fn stream(&self, client: &Client) -> Result<Vec<u8>>;

    /// Returns a constructed URL for streaming.
    ///
    /// Supports transcoding options specified on the media beforehand. See the
    /// struct-level documentation for available options. The `Media` trait
    /// provides setting a maximum bit rate and a target transcoding format.
    ///
    /// This would be used in conjunction with a streaming library to directly
    /// take the URI and stream it.
    fn stream_url(&self, client: &Client) -> Result<String>;

    /// Returns the raw bytes of the media.
    ///
    /// The method does not provide any information about the encoding of the
    /// media without evaluating the stream itself.
    fn download(&self, client: &Client) -> Result<Vec<u8>>;

    /// Returns a constructed URL for downloading the song.
    fn download_url(&self, client: &Client) -> Result<String>;

    /// Returns the default encoding of the media.
    ///
    /// A Subsonic server is able to transcode media for streaming to reduce
    /// data size (for example, it may transcode FLAC to MP3 to reduce file
    /// size, or downsample high bitrate files). Where possible, the method will
    /// return the default transcoding of the media (if enabled); otherwise, it
    /// will return the original encoding.
    fn encoding(&self) -> &str;

    /// Sets the maximum bitrate the media will use when streaming.
    ///
    /// The bit rate is measured in Kbps. Higher bit rate media will be
    /// downsampled to this bit rate.
    ///
    /// Supported values are 0, 32, 40, 48, 56, 64, 80, 96, 112, 128, 160, 192,
    /// 224, 256, 320. The method will not error or panic when using a non-legal
    /// value, but the server may not provide the requested bit rate. A bit rate
    /// of 0 will disable a limit (i.e., use the original bit rate.)
    fn set_max_bit_rate(&mut self, bit_rate: usize);

    /// Sets the transcoding format the media will use when streaming.
    ///
    /// The possible transcoding formats are those defined by the Subsonic
    /// server as valid transcoding targets. Default values are `"mp3"`,
    /// `"flv"`, `"mkv"`, and `"mp4"`. The special value `"raw"` is additionally
    /// supported on servers implementing version 1.9.0 of the API.
    ///
    /// The method will not error or panic when using a non-supported format,
    /// but the server may not provide that transcoded format.
    fn set_transcoding(&mut self, format: &str);
}

/// A trait deriving common methods for any form of media.
pub trait Media {
    /// Returns whether or not the media has an associated cover.
    fn has_cover_art(&self) -> bool;

    /// Returns the cover ID associated with the media, if any.
    ///
    /// The ID may be a number, an identifier-number pair, or simply empty.
    /// This is due to the introduction of ID3 tags into the Subsonic API;
    /// collections of media (such as albums or playlists) will typically
    /// have an identifier-number ID, while raw media (such as songs or videos)
    /// will have a numeric or no identifier.
    ///
    /// Because the method has the potential to return either a string-y or
    /// numeric ID, the number is coerced into a `&str` to avoid type
    /// checking workarounds.
    fn cover_id(&self) -> Option<&str>;

    /// Returns the raw bytes of the cover art of the media.
    ///
    /// The image is guaranteed to be valid and displayable by the Subsonic
    /// server (as long as the method does not error), but makes no guarantees
    /// on the encoding of the image.
    ///
    /// # Errors
    ///
    /// Aside from errors that the `Client` may cause, the method will error
    /// if the media does not have an associated cover art.
    fn cover_art<U: Into<Option<usize>>>(&self, client: &Client, size: U) -> Result<Vec<u8>>;

    /// Returns the URL pointing to the cover art of the media.
    ///
    /// # Errors
    ///
    /// Aside from errors that the `Client` may cause, the method will error
    /// if the media does not have an associated cover art.
    fn cover_art_url<U: Into<Option<usize>>>(&self, client: &Client, size: U) -> Result<String>;
}

/// Information about currently playing media.
///
/// Due to the "now playing" information possibly containing both audio and
/// video, compromises are made. `NowPlaying` only stores the ID, title, and
/// content type of the media. This is most of the information afforded through
/// the web interface. For more detailed information, `song_info()` or
/// `video_info()` gives the full `Song` or `Video` struct, though requires
/// another web request.
#[derive(Debug)]
pub struct NowPlaying {
    /// The user streaming the current media.
    pub user: String,
    /// How long ago the user sent an update to the server.
    pub minutes_ago: usize,
    /// The ID of the player.
    pub player_id: usize,
    id: usize,
    is_video: bool,
}

impl NowPlaying {
    /// Fetches information about the currently playing song.
    ///
    /// # Errors
    ///
    /// Aside from the inherent errors from the [`Client`], the method will
    /// error if the `NowPlaying` is not a song.
    ///
    /// [`Client`]: ../struct.Client.html
    pub fn song_info(&self, client: &Client) -> Result<Song> {
        if self.is_video {
            Err(Error::Other("Now Playing info is not a song"))
        } else {
            Song::get(client, self.id as u64)
        }
    }

    /// Fetches information about the currently playing video.
    ///
    /// # Errors
    ///
    /// Aside from the inherent errors from the [`Client`], the method will
    /// error if the `NowPlaying` is not a video.
    ///
    /// [`Client`]: ../struct.Client.html
    pub fn video_info(&self, client: &Client) -> Result<Video> {
        if !self.is_video {
            Err(Error::Other("Now Playing info is not a video"))
        } else {
            Video::get(client, self.id)
        }
    }

    /// Returns `true` if the currently playing media is a song.
    pub fn is_song(&self) -> bool {
        !self.is_video
    }

    /// Returns `true` if the currently playing media is a video.
    pub fn is_video(&self) -> bool {
        self.is_video
    }
}

/// A HLS playlist file.
#[derive(Debug)]
pub struct HlsPlaylist {
    /// The extension of the playlist metadata. Typically `M3U` or `M3U8`.
    pub extension: String,
    /// The version of the HLS specification.
    pub version: usize,
    /// The target duration for HLS slices.
    pub target_duration: usize,
    hls: Vec<Hls>,
}

impl HlsPlaylist {
    /// Returns the number of slices in the playlist.
    pub fn len(&self) -> usize {
        self.hls.len()
    }

    /// Returns the total duration of the playlist.
    pub fn duration(&self) -> usize {
        self.hls.iter().fold(0, |c, h| c + h.inc)
    }
}

/// A slice of a media for use in a HLS playlist.
#[derive(Debug)]
pub struct Hls {
    /// The duration increment of the slice.
    pub inc: usize,
    /// The path of the slice relative to the server.
    pub url: String,
}

impl Hls {
    /// Fetches the raw bytes of the slice from the `Client`.
    ///
    /// Will likely error if the `Client` is not the same one that the HLS slice
    /// was generated from.
    pub fn get_bytes(&self, client: &Client) -> Result<Vec<u8>> {
        client.hls_bytes(self)
    }
}

impl FromStr for HlsPlaylist {
    type Err = Error;
    fn from_str(s: &str) -> result::Result<Self, Self::Err> {
        fn chew<'a, 'b>(s: &'a str, head: &'b str) -> result::Result<&'a str, Error> {
            if s.starts_with(head) {
                return Ok(s.trim_start_matches(head));
            } else {
                Err(Error::Other("missing required field"))
            }
        }

        let mut split = s.split('\n');

        let _ext = split.next().unwrap();
        let extension = chew(_ext, "#EXT")?.to_owned();
        let _ver = split.next().unwrap();
        let version = chew(_ver, "#EXT-X-VERSION:")?.parse::<usize>()?;
        let _tar = split.next().unwrap();
        let target_duration = chew(_tar, "#EXT-X-TARGETDURATION:")?.parse::<usize>()?;

        let mut hls = Vec::new();
        loop {
            let _inc = split.next().unwrap();
            if _inc == "#EXT-X-ENDLIST" {
                break;
            }
            let inc = chew(_inc, "#EXTINF:")?
                .trim_end_matches(',')
                .parse::<usize>()?;
            hls.push(Hls {
                inc,
                url: split.next().unwrap().to_owned(),
            });
        }

        Ok(HlsPlaylist {
            extension,
            version,
            target_duration,
            hls,
        })
    }
}

impl Index<usize> for HlsPlaylist {
    type Output = Hls;
    fn index(&self, index: usize) -> &Hls {
        self.hls.index(index)
    }
}

impl IntoIterator for HlsPlaylist {
    type Item = Hls;
    type IntoIter = ::std::vec::IntoIter<Hls>;
    fn into_iter(self) -> Self::IntoIter {
        self.hls.into_iter()
    }
}

impl<'de> Deserialize<'de> for NowPlaying {
    fn deserialize<D>(de: D) -> result::Result<Self, D::Error>
    where
        D: Deserializer<'de>,
    {
        #[derive(Deserialize)]
        #[serde(rename_all = "camelCase")]
        struct _NowPlaying {
            username: String,
            minutes_ago: usize,
            player_id: usize,
            id: String,
            is_dir: bool,
            title: String,
            size: usize,
            content_type: String,
            suffix: String,
            transcoded_content_type: Option<String>,
            transcoded_suffix: Option<String>,
            path: String,
            is_video: bool,
            created: String,
            #[serde(rename = "type")]
            media_type: String,
        }

        let raw = _NowPlaying::deserialize(de)?;

        Ok(NowPlaying {
            user: raw.username,
            minutes_ago: raw.minutes_ago,
            player_id: raw.player_id,
            id: raw.id.parse().unwrap(),
            is_video: raw.is_video,
        })
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parse_hls() {
        let hls = hls();
        let p = hls.parse::<HlsPlaylist>().unwrap();

        assert_eq!(p.extension, "M3U");
        assert_eq!(p.version, 1);
        assert_eq!(p.target_duration, 10);
        assert_eq!(p.hls.len(), 23);
    }

    fn hls() -> &'static str {
        "#EXTM3U
#EXT-X-VERSION:1
#EXT-X-TARGETDURATION:10
#EXTINF:10,
/ext/stream/stream.ts?id=1887&hls=true&timeOffset=0&player=19&duration=10&jwt=eyJhbGciOiJIUzI1NiJ9.eyJwYXRoIjoiL2V4dC9zdHJlYW0vc3RyZWFtLnRzP2lkPTE4ODcmaGxzPXRydWUmdGltZU9mZnNldD0wJnBsYXllcj0xOSZkdXJhdGlvbj0xMCIsImV4cCI6MTUxODMxMDEzM30.eLUUHn12c_aU_bof3k3vq7VpCXFAbghufzqGDPlGdl8
#EXTINF:10,
/ext/stream/stream.ts?id=1887&hls=true&timeOffset=10&player=19&duration=10&jwt=eyJhbGciOiJIUzI1NiJ9.eyJwYXRoIjoiL2V4dC9zdHJlYW0vc3RyZWFtLnRzP2lkPTE4ODcmaGxzPXRydWUmdGltZU9mZnNldD0xMCZwbGF5ZXI9MTkmZHVyYXRpb249MTAiLCJleHAiOjE1MTgzMTAxMzN9.1us0LtCuIfxzj_35gNRuxSUMbcECX3OQSMT0cxLvpxM
#EXTINF:10,
/ext/stream/stream.ts?id=1887&hls=true&timeOffset=20&player=19&duration=10&jwt=eyJhbGciOiJIUzI1NiJ9.eyJwYXRoIjoiL2V4dC9zdHJlYW0vc3RyZWFtLnRzP2lkPTE4ODcmaGxzPXRydWUmdGltZU9mZnNldD0yMCZwbGF5ZXI9MTkmZHVyYXRpb249MTAiLCJleHAiOjE1MTgzMTAxMzN9.jFPkjqLWzlC0IwVq4wi2ZUQSjR6nyXIghB5s4UjKoUQ
#EXTINF:10,
/ext/stream/stream.ts?id=1887&hls=true&timeOffset=30&player=19&duration=10&jwt=eyJhbGciOiJIUzI1NiJ9.eyJwYXRoIjoiL2V4dC9zdHJlYW0vc3RyZWFtLnRzP2lkPTE4ODcmaGxzPXRydWUmdGltZU9mZnNldD0zMCZwbGF5ZXI9MTkmZHVyYXRpb249MTAiLCJleHAiOjE1MTgzMTAxMzN9.2otBcvLoPSyoAeyPjPwZvxSoKmoFzhkAQJkiGsI5jpI
#EXTINF:10,
/ext/stream/stream.ts?id=1887&hls=true&timeOffset=40&player=19&duration=10&jwt=eyJhbGciOiJIUzI1NiJ9.eyJwYXRoIjoiL2V4dC9zdHJlYW0vc3RyZWFtLnRzP2lkPTE4ODcmaGxzPXRydWUmdGltZU9mZnNldD00MCZwbGF5ZXI9MTkmZHVyYXRpb249MTAiLCJleHAiOjE1MTgzMTAxMzN9.bbNcug62sXX7tUlo_Iqi7-WJNbZw6shava3kM9-ZFxA
#EXTINF:10,
/ext/stream/stream.ts?id=1887&hls=true&timeOffset=50&player=19&duration=10&jwt=eyJhbGciOiJIUzI1NiJ9.eyJwYXRoIjoiL2V4dC9zdHJlYW0vc3RyZWFtLnRzP2lkPTE4ODcmaGxzPXRydWUmdGltZU9mZnNldD01MCZwbGF5ZXI9MTkmZHVyYXRpb249MTAiLCJleHAiOjE1MTgzMTAxMzN9.i5_XHzLmgqMoHGcuux-U162Tgpx3DHaX4pULYRQDzF4
#EXTINF:10,
/ext/stream/stream.ts?id=1887&hls=true&timeOffset=60&player=19&duration=10&jwt=eyJhbGciOiJIUzI1NiJ9.eyJwYXRoIjoiL2V4dC9zdHJlYW0vc3RyZWFtLnRzP2lkPTE4ODcmaGxzPXRydWUmdGltZU9mZnNldD02MCZwbGF5ZXI9MTkmZHVyYXRpb249MTAiLCJleHAiOjE1MTgzMTAxMzN9.ZNvqTAfTQZUse46c5m2d9R0N_2qIld59zbNkT6yXVZ0
#EXTINF:10,
/ext/stream/stream.ts?id=1887&hls=true&timeOffset=70&player=19&duration=10&jwt=eyJhbGciOiJIUzI1NiJ9.eyJwYXRoIjoiL2V4dC9zdHJlYW0vc3RyZWFtLnRzP2lkPTE4ODcmaGxzPXRydWUmdGltZU9mZnNldD03MCZwbGF5ZXI9MTkmZHVyYXRpb249MTAiLCJleHAiOjE1MTgzMTAxMzN9.pLylLGg6RyNg3ug1LZG0QnzBYwMe3hXYpv2oz8nXOL4
#EXTINF:10,
/ext/stream/stream.ts?id=1887&hls=true&timeOffset=80&player=19&duration=10&jwt=eyJhbGciOiJIUzI1NiJ9.eyJwYXRoIjoiL2V4dC9zdHJlYW0vc3RyZWFtLnRzP2lkPTE4ODcmaGxzPXRydWUmdGltZU9mZnNldD04MCZwbGF5ZXI9MTkmZHVyYXRpb249MTAiLCJleHAiOjE1MTgzMTAxMzN9.lThUtAUp-K2Ty7eUMwQc2RGQBhuckGOoFKfOlrEXkLw
#EXTINF:10,
/ext/stream/stream.ts?id=1887&hls=true&timeOffset=90&player=19&duration=10&jwt=eyJhbGciOiJIUzI1NiJ9.eyJwYXRoIjoiL2V4dC9zdHJlYW0vc3RyZWFtLnRzP2lkPTE4ODcmaGxzPXRydWUmdGltZU9mZnNldD05MCZwbGF5ZXI9MTkmZHVyYXRpb249MTAiLCJleHAiOjE1MTgzMTAxMzN9.JCeL-BBf54awlMEb9k-ziprKYKHmdKWzgwY90ad1u9A
#EXTINF:10,
/ext/stream/stream.ts?id=1887&hls=true&timeOffset=100&player=19&duration=10&jwt=eyJhbGciOiJIUzI1NiJ9.eyJwYXRoIjoiL2V4dC9zdHJlYW0vc3RyZWFtLnRzP2lkPTE4ODcmaGxzPXRydWUmdGltZU9mZnNldD0xMDAmcGxheWVyPTE5JmR1cmF0aW9uPTEwIiwiZXhwIjoxNTE4MzEwMTMzfQ.nPBiEM6cnYXOM3GwAf6lrMBoG9dR45Y2OPPVm0VCfvg
#EXTINF:10,
/ext/stream/stream.ts?id=1887&hls=true&timeOffset=110&player=19&duration=10&jwt=eyJhbGciOiJIUzI1NiJ9.eyJwYXRoIjoiL2V4dC9zdHJlYW0vc3RyZWFtLnRzP2lkPTE4ODcmaGxzPXRydWUmdGltZU9mZnNldD0xMTAmcGxheWVyPTE5JmR1cmF0aW9uPTEwIiwiZXhwIjoxNTE4MzEwMTMzfQ.qSvmeXbmsd9cACBzPdLJQ6y5rc8rWgqZV1KyR0U3I50
#EXTINF:10,
/ext/stream/stream.ts?id=1887&hls=true&timeOffset=120&player=19&duration=10&jwt=eyJhbGciOiJIUzI1NiJ9.eyJwYXRoIjoiL2V4dC9zdHJlYW0vc3RyZWFtLnRzP2lkPTE4ODcmaGxzPXRydWUmdGltZU9mZnNldD0xMjAmcGxheWVyPTE5JmR1cmF0aW9uPTEwIiwiZXhwIjoxNTE4MzEwMTMzfQ.l1eNMaUSFnb_Vlsb1rFEjAo6uukWsRJPZftNl7571bA
#EXTINF:10,
/ext/stream/stream.ts?id=1887&hls=true&timeOffset=130&player=19&duration=10&jwt=eyJhbGciOiJIUzI1NiJ9.eyJwYXRoIjoiL2V4dC9zdHJlYW0vc3RyZWFtLnRzP2lkPTE4ODcmaGxzPXRydWUmdGltZU9mZnNldD0xMzAmcGxheWVyPTE5JmR1cmF0aW9uPTEwIiwiZXhwIjoxNTE4MzEwMTMzfQ.j5eMrPZTVoFqQkIehAjfhAohnRK9d5ZzJm3bEbsZLt0
#EXTINF:10,
/ext/stream/stream.ts?id=1887&hls=true&timeOffset=140&player=19&duration=10&jwt=eyJhbGciOiJIUzI1NiJ9.eyJwYXRoIjoiL2V4dC9zdHJlYW0vc3RyZWFtLnRzP2lkPTE4ODcmaGxzPXRydWUmdGltZU9mZnNldD0xNDAmcGxheWVyPTE5JmR1cmF0aW9uPTEwIiwiZXhwIjoxNTE4MzEwMTMzfQ.301E8HCxYC6f8rw32AKmZh5PIwElck7oPzpseqafY-s
#EXTINF:10,
/ext/stream/stream.ts?id=1887&hls=true&timeOffset=150&player=19&duration=10&jwt=eyJhbGciOiJIUzI1NiJ9.eyJwYXRoIjoiL2V4dC9zdHJlYW0vc3RyZWFtLnRzP2lkPTE4ODcmaGxzPXRydWUmdGltZU9mZnNldD0xNTAmcGxheWVyPTE5JmR1cmF0aW9uPTEwIiwiZXhwIjoxNTE4MzEwMTMzfQ.xSCS47HuPLLzP_Qw1f0Vj1aj4ntRNcw46iTU2UmJUog
#EXTINF:10,
/ext/stream/stream.ts?id=1887&hls=true&timeOffset=160&player=19&duration=10&jwt=eyJhbGciOiJIUzI1NiJ9.eyJwYXRoIjoiL2V4dC9zdHJlYW0vc3RyZWFtLnRzP2lkPTE4ODcmaGxzPXRydWUmdGltZU9mZnNldD0xNjAmcGxheWVyPTE5JmR1cmF0aW9uPTEwIiwiZXhwIjoxNTE4MzEwMTMzfQ.h5bNql36E9nbH3d5Kh9_2QNAOV12MY-c-UzaA7j_N3g
#EXTINF:10,
/ext/stream/stream.ts?id=1887&hls=true&timeOffset=170&player=19&duration=10&jwt=eyJhbGciOiJIUzI1NiJ9.eyJwYXRoIjoiL2V4dC9zdHJlYW0vc3RyZWFtLnRzP2lkPTE4ODcmaGxzPXRydWUmdGltZU9mZnNldD0xNzAmcGxheWVyPTE5JmR1cmF0aW9uPTEwIiwiZXhwIjoxNTE4MzEwMTMzfQ.fc83sb-WJjbinSzA3cXLGnvTkaIbMxTgX1UfM_mjy4g
#EXTINF:10,
/ext/stream/stream.ts?id=1887&hls=true&timeOffset=180&player=19&duration=10&jwt=eyJhbGciOiJIUzI1NiJ9.eyJwYXRoIjoiL2V4dC9zdHJlYW0vc3RyZWFtLnRzP2lkPTE4ODcmaGxzPXRydWUmdGltZU9mZnNldD0xODAmcGxheWVyPTE5JmR1cmF0aW9uPTEwIiwiZXhwIjoxNTE4MzEwMTMzfQ.14-xaOOLGYNmT2ML9szneFLYz9wiYnJZyBW3xD2AnKQ
#EXTINF:10,
/ext/stream/stream.ts?id=1887&hls=true&timeOffset=190&player=19&duration=10&jwt=eyJhbGciOiJIUzI1NiJ9.eyJwYXRoIjoiL2V4dC9zdHJlYW0vc3RyZWFtLnRzP2lkPTE4ODcmaGxzPXRydWUmdGltZU9mZnNldD0xOTAmcGxheWVyPTE5JmR1cmF0aW9uPTEwIiwiZXhwIjoxNTE4MzEwMTMzfQ.3oGWdaQOc6GZ97ihy26rL4onVE0HSoqdJjVyuub9qbw
#EXTINF:10,
/ext/stream/stream.ts?id=1887&hls=true&timeOffset=200&player=19&duration=10&jwt=eyJhbGciOiJIUzI1NiJ9.eyJwYXRoIjoiL2V4dC9zdHJlYW0vc3RyZWFtLnRzP2lkPTE4ODcmaGxzPXRydWUmdGltZU9mZnNldD0yMDAmcGxheWVyPTE5JmR1cmF0aW9uPTEwIiwiZXhwIjoxNTE4MzEwMTMzfQ.zbrfVTgKvj4NAPklbcXEiGczz901tUha8hScjcrCNbo
#EXTINF:10,
/ext/stream/stream.ts?id=1887&hls=true&timeOffset=210&player=19&duration=10&jwt=eyJhbGciOiJIUzI1NiJ9.eyJwYXRoIjoiL2V4dC9zdHJlYW0vc3RyZWFtLnRzP2lkPTE4ODcmaGxzPXRydWUmdGltZU9mZnNldD0yMTAmcGxheWVyPTE5JmR1cmF0aW9uPTEwIiwiZXhwIjoxNTE4MzEwMTMzfQ.MgBWDScW-zQh12N6-NlI3eJ-NQMIm6i0Q5JlDQTaLNg
#EXTINF:7,
/ext/stream/stream.ts?id=1887&hls=true&timeOffset=220&player=19&duration=7&jwt=eyJhbGciOiJIUzI1NiJ9.eyJwYXRoIjoiL2V4dC9zdHJlYW0vc3RyZWFtLnRzP2lkPTE4ODcmaGxzPXRydWUmdGltZU9mZnNldD0yMjAmcGxheWVyPTE5JmR1cmF0aW9uPTciLCJleHAiOjE1MTgzMTAxMzN9.bUpkgSlaNDK2YJvUCf6ucfyOSxMNtPyPudfo0N5qBzA
#EXT-X-ENDLIST"
    }
}
