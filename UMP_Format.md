# UMP Format

The UMP (likely Universal Media Playback) format is used as the response format by all Google services that play videos. This document details the format for the purposes of interoperability.

There are 2 request formats for /videoplayback:
- VideoPlaybackRequestProto. Gets media data with ump.
- VideoPlaybackAbrRequestProto. Can fetch media data for two itags at the same time. Interlaced data response. Uses server abr streaming url.

## Variable sized integers

The UMP format uses variable size integers in a number of places.

The first 4 bits of the first byte set the size of the integer:

- If the top bit is unset, it's a 1-byte value.
- If the top bit is set, but the next bit is not, it's a 2-byte value.
- If the top two bits are set, but the next bit is not, it's a 3-byte value.
- If the top three bits are set, but the next bit is not, it's a 4-byte value.
- If the top four bits are set, it's a 5-byte value. The remaining bits are ignored.

Getting the size of the integer from the first byte can be implemented as follows:

```rust
fn varint_size(byte: u8) -> u32 {
    byte.leading_ones().min(4) + 1
}
```

The remainder of the bits in the first byte are used as part of the integer, except for in 5-byte integers where those bits are ignored.

The variable integer decoding can be implemented as follows:

```rust
fn read_varint(buf: &mut &[u8]) -> u32 {
    let prefix = buf[0];
    let size = varint_size(prefix);

    let mut shift = 0;
    let mut result = 0;

    if size != 5 {
        shift = 8 - size;

        // compute mask of prefix
        let mask = (1 << shift) - 1;

        result |= prefix as u32 & mask;
    }

    for i in 1..size {
        let byte = buf[i as usize];

        result |= (byte as u32) << shift;
        shift += 8;
    }

    // advance the buffer
    *buf = &std::mem::take(buf)[size as usize..];
    result
}
```

Note that the 5-byte integer case behaves differently, ignoring the bottom 4 bits in the first byte entirely, and just reading a 32-bit little-endian integer from the next four bytes.

## High level structure

The UMP format requires that the Content-Type header in the HTTP response contains "application/vnd.yt-ump".

A UMP response is split into parts. Each part is prefixed by a pair of variable length integers, the first being the part type and the second being the part payload length. In pseudo-C, it'd look like this:

```c
struct UmpPart
{
    varInt type;
    varInt size;
    uint8_t data[size];
};
```

Note that you **must** treat the type field as a variable length integer. The current type numbers are all below 128, which will produce a single-byte encoding, but if you read the type ID as a single byte instead of properly decoding it as a variable length integer your implementation will break if/when new part types are added.

### Part encryption

Some parts in the response may be encrypted. The request may also be encrypted.
The key is found in INNERTUBE_API_BASE/youtubei/v1/config.

Request type:
```protobuf
syntax = "proto2";

package youtube.api.innertube;

import "youtube/api/innertube/inner_tube_context.proto";
import "youtube/api/innertube/response_context.proto";
import "youtube/api/innertube/cold_config_group.proto";
import "youtube/api/innertube/hot_config_group.proto";

option cc_enable_arenas = true;
option java_multiple_files = true;
option java_package = "com.google.protos.youtube.api.innertube";
option objc_class_prefix = "YTI";

message ConfigRequest{
	optional InnerTubeContext context = 1;
}

message ConfigResponse{
	optional ResponseContext responseContext = 1;
	optional ColdConfigGroup rawColdConfigGroup = 4;
	optional HotConfigGroup rawHotConfigGroup = 5;
}
```

Usually, the rawColdConfigGroup and rawHotConfigGroup are not present. Instead, they are serialized in ResponseContext::globalConfigGroup

```protobuf
syntax = "proto2";

package youtube.api.innertube;

import "youtube/api/innertube/service_tracking_params.proto";
import "youtube/api/innertube/global_config_group.proto";

option cc_enable_arenas = true;
option java_multiple_files = true;
option java_package = "com.google.protos.youtube.api.innertube";
option objc_class_prefix = "YTI";

message GlobalConfigGroup{
	oneof coldConfigGroup{
		string serializedColdConfigGroup = 1; // base64url serialized cold config group
		bytes bytesSerializedColdConfigGroup = 6; // raw bytes. decode protobuf directly
	}

	oneof hotConfigGroup{
		string serializedHotConfigGroup = 3; // base64url serialized hot config group
		bytes byteSerializedHotConfigGroup = 7; // raw bytes. decode protobuf directly
	}

	optional string hotHashData = 4;
	optional string coldHashData = 5;
}

message ResponseContext{
	optional string visitorData = 2;
	repeated ServiceTrackingParams serviceTrackingParams = 6;
	optional uint32 maxAgeSeconds = 7;
	optional GlobalConfigGroup globalConfigGroup = 16;
}
```

Hot config group definition
```protobuf
syntax = "proto2";

package youtube.api.innertube;

option cc_enable_arenas = true;
option java_multiple_files = true;
option java_package = "com.google.protos.youtube.api.innertube";
option objc_class_prefix = "YTI";

message MediaHotConfig{
	optional Unnamed3302 mediaQualitySettings = 3;
	optional Unnamed3814 planAwareStreamingConfig = 4;
	optional Unnamed4127 unnamedField5 = 5;
	optional Unnamed4238 unnamedField6 = 6;
	optional Unnamed4254 scriptedPlayerHotConfig = 7;
	optional Unnamed1793 exoCacheHotConfig = 139360648;
	optional OnesieHotConfig onesieHotConfig = 146311580;
	optional Unnamed909 bandwidthModelConfig = 151068309;
	optional Unnamed3757 unnamedField157462099 = 157462099;
	optional Unnamed4134 qoeHotConfig = 158048433;
	optional Unnamed1798 unnamedField171189558 = 171189558;
	optional Unnamed724 unnamedField179572306 = 179572306;
}

message OnesieHotConfig{
    // 32 byte encryption key. first 16 bytes are aes-128-ctr key. last 16 bytes are hmac sha256 key.
	optional bytes clientKey = 1;
    // the encrypted client key, sent between client and server. the server uses this to decrypt the request
	optional bytes encryptedClientKey = 2;
	optional int64 keyExpiresInSeconds = 3;
	optional string reverseProxyConfig = 5;
    // urls for connection prewarming/keepalive
	optional ConnectionPrewarmConfig prewarmConfig = 7;
	repeated int32 audioItagWhitelistArray = 10;
	repeated int32 videoItagWhitelistArray = 11;
	optional Unnamed3701 unnamedField12 = 12;
    // directly copied and sent in the request
	optional bytes onesieUstreamerConfig = 16;
	optional bool disableHostReplacement = 17;
	optional bool retryEnabled = 22;
	optional Unnamed1811 unnamedField23 = 23;
	optional int64 maxRetryTimeoutMs = 24;
	optional string unnamedField25 = 25;
	optional bool disableFallbackToInnertube = 26;
	optional bool sendClientInfoToUstreamer = 27;
    // whether or not to send video_streaming.MediaCapabilities in the request
	optional bool sendMediaCapabilities = 29;
    // whether or not to use the hot config to create the request. not sure of the alternative,
    // but there is also another OnesieConfig in the cold config group. it also contains the same
    // encryption key related fields, but the key is different in the same response
	optional bool useHotConfigToCreateOnesieRequest = 30;
	optional bool useHotConfigWithMissingPlayback = 31;
	optional string fallbackHostname = 32;
	optional string fallbackUrlParams = 33;
}

message HotConfigGroup{
	optional Unnamed5060 upgradeConfig = 63102527;
	optional Unnamed2896 kidsHotConfig = 130475591;
	optional Unnamed3389 mobileInfraHotConfig = 131837742;
	optional Unnamed3641 offlineHotConfig = 132001284;
	optional Unnamed3161 mainAppHotConfig = 132499096;
	optional Unnamed3475 musicHotConfig = 133056033;
	optional Unnamed5102 uploadsHotConfig = 133853435;
	optional Unnamed3108 liveStreamingHotConfig = 135988504;
	optional MediaHotConfig mediaHotConfig = 138536474;
	optional Unnamed3877 playerHotConfig = 138815859;
	optional Unnamed4221 renderingHotConfig = 144376845;
	optional Unnamed3290 mdxHotConfig = 146933132;
	optional Unnamed3132 loggingHotConfig = 147204317;
	optional Unnamed4275 searchHotConfig = 149689357;
	optional Unnamed681 adsHotConfig = 156312220;
	optional Unnamed3610 notificationsHotConfig = 161879134;
	optional Unnamed1307 commerceHotConfig = 167174573;
	optional Unnamed3006 liveChatHotConfig = 167601954;
	optional Unnamed2855 iosCommerceLibHotConfig = 170746148;
	optional Unnamed5182 videoEffectsHotConfig = 176383586;
	optional Unnamed4157 redHotConfig = 199363921;
	optional Unnamed4169 reelHotConfig = 210561608;
	optional Unnamed1713 embeddedPlayerHotConfig = 210797621;
	optional Unnamed3745 pdgHotConfig = 235947118;
	optional Unnamed4709 systemHealthConfig = 246895001;
	optional Unnamed4608 subscriptionHotConfig = 297150804;
	optional Unnamed4441 shortsCreationHotConfig = 298441996;
	optional Unnamed1805 experimentFlags = 346648555;
	optional Unnamed4256 scriptingHotConfig = 346651377;
	optional Unnamed1491 datapushHotConfig = 394298679;
    // more fields
}
```

The client uses the `clientKey` found in the Onesie(Hot)Config for encrypting the request. The IV is randomly generated.

The `Onesie(Hot)Config::encryptedClientKey` is sent in the request, along with the hmac and iv produced  by the function below. Responses are encrypted/hmaced with the same key

```rust
// the key does not deviate from this length. otherwise, consider the response invalid
fn encrypt_request(client_key: [u8; 32], data: &[u8]) -> (Vec<u8>, Vec<u8>) {
    let aes_key = &client_key[0..16];
    let hmac_key = &client_key[16..32];

    let iv = random_bytes(16); // 16 bytes
    let mut aes = new_aes_128_ctr(aes_key, &iv);

    let encrypted = aes.update_final(data);
    let mut hmac = new_sha_256_hmac(hmac_key);

    hmac.update(&encrypted);

    (encrypted, hmac.update_final(&iv))
}
```

The request has the following format

```protobuf
syntax = "proto2";

package youtube.api.innertube;

import "video_streaming/streamer_context.proto";

option cc_enable_arenas = true;
option java_multiple_files = true;
option java_package = "com.google.protos.youtube.api.innertube";
option objc_class_prefix = "YTI";

message EncryptedInnertubeRequest{
    // contains only .youtube.api.innertube.ClientInfo. hl: 'zz', gl: 'zz',
    // debugDeviceIdOverride: '0', clientName: ANDROID or IOS or any other, clientVersion:
    // operating system version. not app version.
	optional InnerTubeContext context = 1;
    // encrypted via the function above. see below for type definition
	optional bytes encryptedOnesieRequest = 2;
    // from Onesie(Hot)Config::encryptedClientKey
	optional bytes encryptedClientKey = 5;
	optional bytes iv = 6; // iv generated above
	optional bytes hmac = 7; // hmac generated above
    // if set to true, the `encryptedOnesieRequest` is gzipped before encrypted
	optional bool useCompression = 8;
    // from Onesie(Hot)Config::reverseProxyConfig
	optional string reverseProxyConfig = 9;
	optional Unnamed7319 unnamedField12 = 12;
}

message OnesieInnertubeRequest{
	repeated string unnamedField1 = 1;
	optional Unnamed6053 unnamedField2 = 2;
	optional EncryptedInnertubeRequest encryptedRequest = 3
    // from Onesie(Hot)Config::onesieUstreamerConfig
	optional bytes ustreamerConfig = 4;
	optional int32 unnamedField5 = 5;
	optional int32 unnamedField6 = 6;
	optional .video_streaming.StreamerContext streamerContext = 10;
	optional Unnamed472 unnamedField13 = 13;
}
```

Streamer Context definition

```protobuf
syntax = "proto2";

package video_streaming;

import "youtube/api/innertube/client_info.proto";

option cc_enable_arenas = true;

message StreamerContext{
    // the same client used for all innertube requests
	optional .youtube.api.innertube.ClientInfo client = 1;
	optional bytes unnamedField2 = 2; // unknown
	optional bytes unnamedField3 = 3;
	optional bytes unnamedField4 = 4;
}
```

OnesieRequestProto. The data sent in `EncryptedInnertubeRequest::encryptedOnesieRequest`

```protobuf
syntax = "proto2";

package video_streaming;

option cc_enable_arenas = true;

message HttpHeader{
	optional string key = 1;
	optional string value = 2;
}

message OnesieRequestProto{
    // INNERTUBE_API_BASE/youtubei/v1/(player|otherapi,unknown) (?key=APIKEY&id=VIDEOID)
	optional string url = 1;
    // Content-Type: application/x-protobuf. X-Goog-Api-Format-Version: 2.
    // User-Agent: same user agent sent in player requests, optionally tagged with gzip at the end.
    // If not, the response may be compressed using brotli.
    // Authorization: Bearer [token], if applicable
	repeated HttpHeader headers = 2;
    // the request payload. if its /player, this is a serialized PlayerRequest
	optional bytes body = 3;
    // unknown. always set to true
	optional bool unnamedField4 = 4;
}
```

### Partial parts

Parts are not guaranteed to be wholly contained within one response payload. It is quite common to find that a part's length exceeds the length of the HTTP response payload. This means that the part will continue in the next response payload.

If a response payload is self-contained, i.e. it does not end with a partial part, the last part in the buffer will end precisely at the end of the buffer. Typically the last part will be `MEDIA_END` (type 22) with a single null byte payload, but this is not guaranteed.

If a payload is not self-contained, i.e. its final part has a length exceeding the amount of remaining data in the buffer, its data will continue in the next response payload. In such a case, the next payload will start with a media header part (type 20) followed by a part of the same type as the partial one, whose data is a continuation of the partial part from the previous payload. Parts can be split up over an arbitrary number of response payloads.

This is a little hard to picture, so here's an example. Let's say you've got a part with a length of 2,500,000 bytes, and each response payload can be maximum of 1MB (this is just for example; in practice there is no such hard limit). The resulting response payloads will look something like this:

```
response 1:
	part 20 (media header)
		size=...
		data=...
	part 21 (media data)
		size=2500000
		data=... (len=1000000)

response 2:
	part 20 (media header)
		size=...
		data=...
	part 21 (media data)
		size=1500000
		data=... (len=1000000)

response 3:
	part 20 (media header)
		size=...
		data=...
	part 21 (media data)
		size=500000
		data=... (len=500000)
	part 22 (MEDIA_END)
		size=1
		data=00
```

This gets decoded as a single type 21 part of size 2,500,000 bytes, followed by a type 22 part. Note that there could be a different part type in response 3, after part 21, instead of `MEDIA_END`. The end of a part is determined solely by all of its data being read; the next part type is irrelevant.

Once a partial part begins, responding with a different part type (e.g. sending a partial part 22, then following up with a part 32 before sending the rest of the first part) has undefined behaviour. As far as I could tell from the implementations, error handling is variable here. Some will throw an exception, but some appear to blindly accept the data as a continuation of the data, even if the part type ID is wrong. Fun! I would recommend being rigorous in checking for the correct type when decoding partial parts.

### Reading UMP parts

You can read UMP parts with a state machine:

1. Read a response payload in chunks.
2. If the amount of data remaining in the buffer is zero, go back to step 1.
3. Read the UMP part type as a variable sized integer.
4. Read the UMP part size as a variable sized integer.
5. If the part size is zero, decode the part as a zero-length part (i.e. no payload), passing in an empty buffer, then go back to step 2.
6. If the part size is less than or equal to the remaining buffer size, decode the part from that slice of the buffer, then increment the position within the buffer and go back to step 3.
7. If the part size is greater than the remaining buffer size, keep a copy of the remaining buffer data, read the next buffer, stitch the two together, and continue parsing from step 5.

## Part types

The following part types have been observed.

### Part 10: ONESIE_HEADER

OnesieHeader. Depending on the header type, there is exactly 0-1 ONESIE_DATA parts that follow.

The format is
```protobuf
syntax = "proto2";

package video_streaming;

enum OnesieHeaderType{
    // expect a ONESIE_DATA with an encrypted player response
	PLAYER_RESPONSE = 0;
	OnesieHeaderType_value_1 = 1; // unknown
    // expect a ONESIE_DATA of which the 16 byte data is the decryption key for ONESIE_ENCRYPTED_MEDIA
	MEDIA_DECRYPTION_KEY = 2;
	OnesieHeaderType_value_3 = 3; // unknown
	OnesieHeaderType_value_4 = 4; // unknown
	OnesieHeaderType_value_5 = 5; // unknown
	NEW_HOST = 6;
	OnesieHeaderType_value_7 = 7; // unknown
	OnesieHeaderType_value_8 = 8; // unknown
	OnesieHeaderType_value_9 = 9; // unknown
	OnesieHeaderType_value_10 = 10; // unknown
	OnesieHeaderType_value_11 = 11; // unknown
	OnesieHeaderType_value_12 = 12; // unknown
	OnesieHeaderType_value_13 = 13; // unknown
    // specifies that OnesieHeader::restrictedFormats is set
	RESTRICTED_FORMATS_HINT = 14;
	OnesieHeaderType_value_15 = 15; // unknown
    // specifies that stream metadata, like OnesieHeader::{sequencNumber, startRange} etc are set
	STREAM_METADATA = 16;
	OnesieHeaderType_value_17 = 17; // unknown
	OnesieHeaderType_value_18 = 18; // unknown
	OnesieHeaderType_value_19 = 19; // unknown
	OnesieHeaderType_value_20 = 20; // unknown
	OnesieHeaderType_value_21 = 21; // unknown
	OnesieHeaderType_value_22 = 22; // unknown
	OnesieHeaderType_value_23 = 23; // unknown
	OnesieHeaderType_value_24 = 24; // unknown
	ENCRYPTED_INNERTUBE_RESPONSE_PART = 25; // an encrypted response. purpose is unknown
	OnesieHeaderType_value_26 = 26; // unknown
	OnesieHeaderType_value_27 = 27; // unknown
	OnesieHeaderType_value_28 = 28; // unknown
	OnesieHeaderType_value_29 = 29; // unknown
	OnesieHeaderType_value_30 = 30; // unknown
}

enum CompressionType{
    // unknown. probably deflate. both are handled with gzip in the disassembled code.
    CompressionType_value_0 = 0;
    // unknown. probably gzip. both are handled with gzip in the disassembled code.
    CompressionType_value_1 = 1;
    BROTLI = 2;
}

message CryptoParams{
	optional bytes hmac = 4;
	optional bytes iv = 5;
	optional CompressionType compressionType = 6;
}

message TimestampRange{
	optional int64 startTimestamp = 1;
	optional int64 endTimestamp = 2;
}

message OnesieHeader{
    optional OnesieHeaderType type = 1;
    optional string videoId = 2;
    optional string itag = 3;
    optional CryptoParams cryptoParams = 4;
    optional uint64 lastModified = 5;
    optional int64 startRange = 6;
    optional int64 expectedMediaSizeBytes = 7;
    repeated string restrictedFormats = 11;
    optional TimestampRange unnamedField14 = 14;
    optional string xtags = 15;
    optional int64 sequenceNumber = 18;
}
```

### Part 11: ONESIE_DATA

If present, must be preceded by OnesieHeader (part 10).

#### Type 0: OnesieHeaderType::PLAYER_RESPONSE

The crypto params in the header must be set. If any of the fields mentioned below arent set, panic or throw an exception.
Decrypt the contents using the same clientKey used in the request, and the iv from crypto params.

If the compression type is BROTLI, decompress using BROTLI, otherwise decompress using GZIP.

The contents are now in protobuf.

```protobuf
syntax = "proto2";

package video_streaming;

option cc_enable_arenas = true;

enum ProxyStatus{
	ProxyStatus_value_0 = 0; // unknown
	OK = 1;
	ProxyStatus_value_2 = 2; // unknown
	ProxyStatus_value_3 = 3; // unknown
	ProxyStatus_value_4 = 4; // unknown
	ProxyStatus_value_5 = 5; // unknown
	ProxyStatus_value_6 = 6; // unknown
	ProxyStatus_value_7 = 7; // unknown
	ProxyStatus_value_8 = 8; // unknown
	ProxyStatus_value_9 = 9; // unknown
	ProxyStatus_value_10 = 10; // unknown
	ProxyStatus_value_11 = 11; // unknown
	ProxyStatus_value_12 = 12; // unknown
}

message OnesieInnertubeResponse{
	optional ProxyStatus proxyStatus = 1;
	optional int32 status = 2;
	repeated HttpHeader headers = 3; // same definition as above
	optional bytes body = 4; // the corresponding response for the request above
}
```

If the proxy status is not `OK`, panic or throw an exception.
If the status is not 200, panic or throw an exception. Error codes have a response body in the same format as a Google Cloud API error response.

The body can be deserialized as a `PlayerResponse`.

#### Type 2: OnesieHeaderType::MEDIA_DECRYPTION_KEY

The body is 16 bytes. This is the key used to decrypt ONESIE_ENCRYPTED_MEDIA.
The key may appear after instances of ONESIE_ENCRYPTED_MEDIA.

#### Type 6: OnesieHeaderType::NEW_HOST

Unknown. No ONESIE_DATA follows

#### Type 14: OnesieHeaderType::RESTRICTED_FORMATS_HINT

Indicates that `OnesieHeader::restrictedFormats` is set. A string list of itags that needs to be converted to ints. No ONESIE_DATA follows

#### Type 16: OnesieHeaderType::STREAM_METADATA

Indicates some stream metadata is set in the header. No ONESIE_DATA follows

#### Type 25: OnesieHeaderType::ENCRYPTED_INNERTUBE_RESPONSE_PART

Unknown purpose.

Decrypted in the same way as OnesieHeaderType::PLAYER_RESPONSE.

```protobuf
syntax = "proto2";

package video_streaming;

option cc_enable_arenas = true;

message EncryptedInnertubeResponsePart{
	optional bytes encryptedContent = 1;
	optional bytes hmac = 2;
	optional bytes iv = 3;
	optional CompressionType compressionType = 4;
}
```

### Part 12: ONESIE_ENCRYPTED_MEDIA

Starts with a UMP varint that corresponds to `MediaHeader::headerId`. The rest is encrypted media.

The key is found in `OnesieHeaderType::MEDIA_DECRYPTION_KEY`.
The initialization vector is 16 bytes of zeros. There is no hmac.

All ONESIE_ENCRYPTED_MEDIA chunks are to be decrypted with the same cipher instance, without resetting the IV.
Unknown if an IV reset is used if the `headerId` changes, but likely not.

This data is usually sent under the same `headerId` as MEDIA.
The data should be processed in order of receiving.

### Part 20: MEDIA_HEADER

MediaHeader type definition

```protobuf
syntax = "proto2";

package video_streaming;

option cc_enable_arenas = true;

message MediaHeader{
	enum CompressionType{
		CompressionType_value_0 = 0; // unknown. no-op
		CompressionType_value_1 = 1; // unknown. no-op
		GZIP = 2;
	}

	optional uint32 headerId = 1;
	optional string videoId = 2;
	optional int32 itag = 3;
	optional uint64 lastModified = 4;
	optional string unnamedField5 = 5;
	optional int64 startRange = 6;
	optional MediaHeader.CompressionType compressionType = 7;
	optional bool unnamedField8 = 8;
	optional int64 unnamedField9 = 9;
	optional int64 unnamedField10 = 10;
}
```

### Part 21: MEDIA

Contains the actual media itself. Starts with a UMP varint corresponding to `MediaHeader::headerId`, followed by the media data.

If the header compression type is set to GZIP, decompress using gzip.

If you pull the data out of these and into a webm file, you can play them with VLC!

### Part 22: MEDIA_END

Terminator part. Usually included at the end of all payloads where there is no partial payload at the end.

Size is usually 1, with the data being a UMP varint corresponding to `MediaHeader::headerId`.

### Part 31: LIVE_METADATA

Metadata related to live streams.

```protobuf
syntax = "proto2";

package video_streaming;

option cc_enable_arenas = true;

message LiveMetadata{
	optional string unnamedField1 = 1;
	optional int64 headSequenceNumber = 3;
	optional int64 headTimeMillis = 4;
	optional int64 walltimeMs = 5;
	optional string unnamedField6 = 6;
	optional bool unnamedField8 = 8;
	optional int64 unnamedField12 = 12;
	optional int32 unnamedField13 = 13;
	optional int64 unnamedField14 = 14;
	optional int32 unnamedField15 = 15;
}
```

### Part 32: HOSTNAME_CHANGE_HINT

Unknown format.

### Part 33: LIVE_METADATA_PROMISE

SABR Live Metadata Promise, related to "SABR Live Protocols".

Possibly streaming related?

```protobuf
syntax = "proto2";

package video_streaming;

option cc_enable_arenas = true;

message LiveMetadataPromise{
	optional string videoId = 1;
}
```

### Part 34: LIVE_METADATA_PROMISE_CANCELLATION

Cancellation of a SABR Live Metadata Promise (originally sent as part 33), related to "SABR Live Protocols".

Possibly streaming related?

Same format as LIVE_METADATA_PROMISE.

### Part 35: NEXT_REQUEST_POLICY

Unknown purpose and format.

### Part 36: USTREAMER_VIDEO_AND_FORMAT_DATA

Probably related to signalling the media format information for livestreams.

Unknown format.

### Part 37: FORMAT_SELECTION_CONFIG

Format selection config. Related to the user changing format preferences (e.g. force 1080p).

```protobuf
syntax = "proto2";

package video_streaming;

option cc_enable_arenas = true;

message FormatSelectionConfig{
	repeated int32 itags = 2;
	optional string videoId = 3;
}
```

### Part 38: USTREAMER_SELECTED_MEDIA_STREAM

Clearly streaming related but not sure what this is for.

Unknown format.

### Part 39: ???

Non existent.

### Part 40: ???

Non existent.

### Part 41: ???

Non existent.

### Part 42: FORMAT_INITIALIZATION_METADATA

Unknown purpose and format. Unused.

### Part 43: SABR_REDIRECT

Unknown purpose and format.

### Part 44: SABR_ERROR

Unknown purpose and format.

### Part 45: SABR_SEEK

Unknown purpose and format.

### Part 46: RELOAD_PLAYER_RESPONSE

Likely tells the player that the page needs to be reloaded.

Unknown format.

### Part 47: PLAYBACK_START_POLICY

Unknown purpose and format.

### Part 48: ALLOWED_CACHED_FORMATS

Unknown purpose and format.

### Part 49: START_BW_SAMPLING_HINT

Unknown purpose and format.

### Part 50: PAUSE_BW_SAMPLING_HINT

Unknown purpose and format.

### Part 51: SELECTABLE_FORMATS

Unknown purpose and format.

### Part 52: REQUEST_IDENTIFIER

Unknown purpose and format.

### Part 53: REQUEST_CANCELLATION_POLICY

Unknown purpose and format.

### Part 54: ONESIE_PREFETCH_REJECTION

Unknown purpose and format.

### Part 55: TIMELINE_CONTEXT

Unknown purpose and format.

### Part 56: REQUEST_PIPELINING

Unknown purpose and format.

### Part 57: SABR_CONTEXT_UPDATE

Unknown purpose and format.

### Part 58: STREAM_PROTECTION_STATUS

Unknown purpose and format.

### Part 59: SABR_CONTEXT_SENDING_POLICY

Unknown purpose and format.

### Part 60: LAWNMOWER_POLICY

Unknown purpose and format.

### Part 61: SABR_ACK

Unknown purpose and format.

### Part 62: END_OF_TRACK

Unknown purpose and format.

### Part 63: CACHE_LOAD_POLICY

Unknown purpose and format.

### Part 64: LAWNMOWER_MESSAGING_POLICY

Unknown purpose and format.

### Part 65: PREWARM_CONNECTION

Likely to generate 204s and keep alive the connection for faster streaming.

Unknown format.

# Meta

## License

All information published in this document is hereby released in the public domain.

## Special Thanks

Thanks to the folks from #invidious on libera.chat for their invaluable assistance and insight.

Additional thanks to the maintainers of Invidious, Piped, youtube-dl, yt-dlp, uBlock Origin, and Sponsorblock for all their hard work in making YouTube a far more tolerable platform to interact with.

