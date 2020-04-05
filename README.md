# MARVYN
MARVYN is **M**astering **A**udio, **R**ipping & **V**ideo tasks automaticall**Y** & **N**icely.


## Key features
MARVYN automates multimedia tasks:
- Rip and decrypt Audio CDs, DVDs & BDs without a single click or keypress
- Compress videos keeping only the main video and a single audio track
- Convert video and audio into smaller formats (e.g. h.264, mp3)
- Remultiplex video directories to files losslessly (Blu-ray or DVD to MKV)

MARVYN is nice:
- Runs in a Docker container to keep the system clean of dependencies
- Uses [VAAPI](https://trac.ffmpeg.org/wiki/Hardware/VAAPI) hardware acceleration for video transcoding
- Runs with low cpu priority
- Allows to send a mail notification after a disc is ripped
- Does not need root permissions

## TL;DR
* Install:
  ```shell
  $ sudo apt install docker-compose
  $ wget -O - https://github.com/git-developer/marvyn/tarball/develop | tar --one-top-level=/opt/docker/marvyn --strip-components 1 -xz
  $ /opt/docker/marvyn/image/build
  ```
* Enable autorip:
  ```shell
  $ sudo cp /opt/docker/marvyn/udev/rules.d/disc-detection.rules /etc/udev/rules.d/
  $ sudo udevadm control --reload
  ```
* Run: _Insert disc_

## Pre-requisites
- An x86 compatible linux system with `docker-compose`

The Docker Compose [documentation](https://docs.docker.com/compose/install/)
contains a comprehensive guide explaining several install options.
On debian-based systems, `docker-compose` may be installed by calling

```shell
$ sudo apt install docker-compose
```

The ARM platform (eg. Raspberry Pi) is not supported as long as
[MakeMKV](https://makemkv.com/) is available for x86 only.


## Installation
1. Download to `/opt/docker/marvyn`, e.g.
    ```shell
    $ git clone --depth 1 https://github.com/git-developer/marvyn
    $ mkdir -p /opt/docker && mv marvyn/opt/docker/marvyn /opt/docker/
    ```
1. Create Docker image (this will take a few minutes):
    ```shell
     $ /opt/docker/marvyn/image/build
    ```
    If you wish to customize the tags and labels, change the values in
    `image/.labels`and `etc/base.yml` accordingly.

## Usage
### Autorip
* Insert a Blu-ray, DVD or Audio CD into your drive.

When autorip is enabled (disabled by default), MARVYN will automatically rip the
disc to your output directory, remove copy protection and tag audio files.
If mail notification is enabled (disabled by default), MARVYN will send a mail
with a log file on finish.

### Video Disc Conversion
* `bin/transcode-disc-directory <path-to-directory>`

The directory must contain a BD or DVD structure (i.e. `VIDEO_TS` or `BDMV`).
All videos will be converted to the configured target format (default: h.264)
containing a single language (default: German) and written to the output
directory as separate files. Video conversion uses hardware transcoding.

### Audio Conversion
* `bin/transcode-audio-directory <path-to-directory>`

The directory or its subdirectories must contain audio files. The audio files
will be converted to the target format (default: mp3) while keeping
directory structure and metadata (tags).

### Video Series Conversion
* `bin/transcode-series-directory <path-to-directory>`

The directory is supposed to hold a (TV or movie) series; it must contain
subdirectories, each holding a Blu-ray or DVD structure.
Every subdirectory will be converted by the _Video Disc Conversion_.

### Video Remultiplexing
* `bin/remux-directory <path-to-directory>`

The directory must contain a Blu-ray or DVD structure (i.e. `VIDEO_TS` or
`BDMV`). Every contained video will be converted to a separate MKV file
losslessly and written to the output directory.

### Video Transcoding
* `bin/vaapi-transcode <path-to-video-file>`

The input video will be converted to the configured target format
(default: h.264) containing a single language (default: German).
Video conversion uses hardware transcoding.


## Configuration

### General
* File: `etc/base.yml`

| Option | Description | Examples | Default |
| -----  | ----------- | -------- | ------- |
| Volume Mount `/output` | Output directory | `/media/newstuff:/output` | `../output:/output` |
| Environment variable `MARVYN_MAKEMKV_KEY` | MakeMKV [registration key](https://www.makemkv.com/buy/). When unset, a beta key is downloaded and used.  | `T-wN2...AE5z` | _none_ |
| Environment variable `MARVYN_NICENESS` | CPU priority (in terms of the `nice` command) | `-19`..`20` | _none_ (effectively: Default of `nice` command)
| Environment variable `TZ` | Timezone | `Europe/Berlin` | _none_ |
| Environment variable `FAKETIME` | System time that MARVYN uses. This option allows to circumvent expiration issues. See [libfaketime](https://github.com/wolfcw/libfaketime) for format specification and details. | `-14d`, `2020-12-24 20:30:00` | _none_ |
| Service Option `cpu_shares` | CPU priority. 1024 means normal, lower value mean lower priority. See [CPU share constraint](https://docs.docker.com/engine/reference/run/#runtime-constraints-on-resources#cpu-share-constraint) for details. | `512` | `512` |

### Autorip
Enable Autorip:

```shell
$ sudo cp /opt/docker/marvyn/udev/rules.d/disc-detection.rules /etc/udev/rules.d/
$ sudo udevadm control --reload
```

You have to to modify the file content only if one or more of the following
cases apply:
* MARVYN is not located in the default directory `/opt/docker/marvyn`.
* Your disc drive is not mounted at the default locations `/dev/sg1` and
  `/dev/sr0`; you also have to modify `etc/rip-disc.yml` in this case.
* You want to run Autorip as a non-root user; change the command to
  `/usr/bin/sudo -E -u <user> /opt/docker/marvyn/bin/rip-disc` in this case.

#### Main configuration
File `etc/rip-disc.yml`

| Option | Description | Examples | Default |
| -----  | ----------- | -------- | ------- |
| Device Options | Available disc drives | `/dev/cdrom:/dev/cdrom` | see `etc/rip-disc.yml` |
| Environment variable `MARVYN_CD_TITLE_DEPTH` | Starting in the output directory, how many subdirectories does MARVYN have to descend to find the directory containing the CD title? Should be set according to the directory count used in `OUTPUTFORMAT` and `VAOUTPUTFORMAT` in the file `data/.abcde.conf` | `1` | `3` |
| Environment variable `MARVYN_EJECT_DISC` | Eject disc on finish | `yes` | `no` |

#### Mail notification
* File: `etc/mail.env`

Notification mails are sent only when `MARVYN_MAIL_RECIPIENT` is set.

| Option | Description | Examples |
| -----  | ----------- | -------- |
| `MARVYN_MAIL_RECIPIENT` | Recipient address | `admin@example.com` |
| `MARVYN_MAIL_SENDER` | Sender address | `marvyn@example.com` |
| `MARVYN_MAIL_SERVER` | Mail server and optional port | `mail.example.com:587` |
| `MARVYN_MAIL_USER` | Username for login on mail server | `mailaccount@mail.example.com` |
| `MARVYN_MAIL_PASSWORD` | Password for login on mail server | `p4§§w0rD` |

#### Audio CD ripping
* File: `data/.abcde.conf`

See the [abcde](https://git.einval.com/cgi-bin/gitweb.cgi?p=abcde.git;a=blob;f=abcde.conf)
documentation for details.

### Video Disc Conversion
* File: `etc/transcode-disc-directory.yml`

| Option | Description | Examples | Default |
| -----  | ----------- | -------- | ------- |
| Volume Mount `/work` | Working directory for temporary files | `/tmp/transcoding:/work` | `../work:/work` |
| Environment variable `MARVYN_CLEANUP` | Delete temporary files after successful finish? | `yes`, `no` | `yes` |

This configuration uses the _Video Transcoding_ configuration as default.
Values in `etc/vaapi-transcode.yml` will be applied to all video transcoding
tasks, including _Video Disc Conversion_. Values in
`etc/transcode-disc-directory.yml` will be applied to _Video Conversion_ only.

### Audio Conversion
* File: `etc/transcode-audio-directory.yml`

| Option | Description | Examples | Default |
| -----  | ----------- | -------- | ------- |
| Environment variable `MARVYN_FFMPEG_OUTPUT_FORMAT` | Output format | `mp3`, `ogg` | `mp3` |
| Environment variable `MARVYN_FFMPEG_OUTPUT_OPTIONS` | Output options for FFmpeg | see [FFmpeg documentation](https://ffmpeg.org/ffmpeg.html#Options) | `-q 5.5 -id3v2_version 3 -c:v copy` |
| Environment variable `MARVYN_AUDIO_INCLUDES` | Comma-separated list of filenames that are considered for conversion. Glob expressions are supported. | `*.flac,*.ogg` | `*.flac,*.ogg,*.mp3` |
| Environment variable `MARVYN_AUDIO_EXCLUDES` | Comma-separated list of filenames that are ignored for conversion. Glob expressions are supported. This option has no effect when `MARVYN_AUDIO_INCLUDES` is set. | `*.jpg,*.png,*.txt` | _none_ |

### Video Series Conversion
* File: `etc/transcode-series-directory.yml`

This configuration uses the _Video Disc Conversion_ configuration as default.
Values in `etc/transcode-disc-directory.yml` will be applied to all video disc
conversion tasks, including _Video Series Conversion_. Values in
`etc/transcode-series-directory.yml` will be applied to
_Video Servies Conversion_ only.

### Video Remultiplexing
* File: `etc/remux-directory.yml`

### Video Transcoding
* File: `etc/vaapi-transcode.yml`

| Option | Description | Examples | Default |
| -----  | ----------- | -------- | ------- |
| Environment variable `MARVYN_VIDEO_BITRATE` | Video bitrate | `1280k` | `1280k` |
| Environment variable `MARVYN_VIDEO_CODEC` | Video format supported by VAAPI | `h264_vaapi` | `h264_vaapi` |
| Environment variable `MARVYN_VIDEO_WIDTH` | Video width | `720` | `720` |
| Environment variable `MARVYN_AUDIO_BITRATE` | Audio bitrate | `128k` | `128k` |
| Environment variable `MARVYN_AUDIO_CODEC` | Audio format | `aac` | `aac` |
| Environment variable `MARVYN_AUDIO_CHANNELS` | Audio channels | `2` | `2` |
| Environment variable `MARVYN_AUDIO_LANGUAGES` | List of allowed audio languages in descending priority. The first language that is found in the video is selected for output. If the video contains only a single language, that one is used and this setting is ignored. | `deu,ger,eng` | `deu,ger,eng` |
| Environment variable `MARVYN_VAAPI_DEVICE` | VAAPI device for hardware transcoding | `/dev/dri/renderD128` | `/dev/dri/renderD128` |
| Device Options | Available VAAPI devices | `/dev/dri/renderD128:/dev/dri/renderD128` | `/dev/dri/renderD128:/dev/dri/renderD128` |


## Advanced Topics

### Management tasks
* Start a service (e.g. Audio Conversion):
  ```shell
  $ bin/transcode-audio-directory /media/audio/
  marvyn-transcode-audio-directory-20200312-061525
  ```
* See the log:
  ```shell
  $ docker logs marvyn-transcode-audio-directory-20200312-061525
  ```
* Check status:
  ```shell
  $ docker ps -a --format={{.Status}} -f name=marvyn-transcode-audio-directory-20200312-061525
  ```
* Stop service:
  ```shell
  $ docker stop marvyn-transcode-audio-directory-20200312-061525
  ```
* Remove a service container (including its logs):
  ```shell
  $ docker rm marvyn-transcode-audio-directory-20200312-061525
  ```
* Remove all MARVYN containers:
  ```shell
  $ docker container ls -a -q -f name=marvyn | xargs docker rm
  ```
* Start a shell within MARVYN:
  ```shell
  $ docker-compose -f etc/base.yml run --rm marvyn-base bash
  ```

### Environment variables
Environment variables may either be declared in a configuration file
or as environment variable. For all variables prefixed with `MARVYN_`,
the environment value takes precedence over the configuration file.
For all other variables, the default `docker-compose` behavior is applied:
* if a variable is not declared in the configuration file, the value is ignored.
* if a variable is declared in the configuration file without value,
  the value of the environment variable is used.
* if a variable is declared in the configuration file with value,
  this value is used and the environment value is ignored.

### Directories for output and work as command line arguments
By default, the directories for `/output` and `/work` are read from the
configuration file. Alternatively, it is possible to hand them over as command
line arguments to a service, e.g.

* `$ bin/transcode-disc-directory /media/input.mp4 /media/output/`
* `$ bin/transcode-disc-directory /media/input.mp4 /media/output/ /tmp/work/`

Unfortunately, this does not work reliably as long as
[docker-compose #7270](https://github.com/docker/compose/issues/7270)
is open: either the command line argument or the configuration file value
will be chosen randomly. MARVYN knows this bug and will cancel when detected.

### Users, groups and permissions
MARVYN services run with userid and groupid of the current user.
Autorip is called by `udev` which runs as `root` by default.
A different user may be configured by using `sudo -u` within the udev rule.

### Home directory
The directory `data` is used as home directory for MARVYN.
It contains all configuration and data needed at runtime, e.g.
* MakeMKV configuration
* abcde configuration
* Keys for disc decryption

### Base image (Linux distribution)
The default Docker image is based on Ubuntu, an alternative Docker image based
on Debian is available in `image/Dockerfile.debian`.

### Ripping of encrypted discs
Detailed information about ripping of encrypted discs may be found
in the resources listed in the file `README.keys`.

### Extending MARVYN
Example: Create a new service called _process-video_

1. Create start script `bin/process-video`:
   ```shell
   ln -s .start bin/process-video
   ```

1. Create configuration file `etc/process-video.yml`:
   ```yaml
   ---
   version: "2.2"
   services:
     marvyn-process-video:
       extends:
         file: base.yml
         service: marvyn-base

       <custom configuration here>
   ```

1. Create executable service script `services/process-video`, e.g.:
   ```shell
   #!/bin/sh
   here="$(dirname ${0})"
   . "${here}/.prepare"
   if [ -z "${1-}" ] || [ ! -d "${1}" ] || [ -z "${2-}" ]; then
     cancel "Syntax: $(basename $0) <input-directory> <output-directory>"
   fi
   mkdir -p "${output_fullpath}"
  
   <service implementation>
   ```

### Components
MARVYN integrates the following applications and libraries:

* [Docker Compose](https://docs.docker.com/compose/)
* [MakeMKV](https://www.makemkv.com/)
* [dvdbackup](https://sourceforge.net/projects/dvdbackup/)
* [abcde](https://abcde.einval.com/)
* [ffmpeg](https://ffmpeg.org/)
* [sendEmail](http://caspian.dotconf.net/menu/Software/SendEmail/)
* [libdvdcss](https://www.videolan.org/developers/libdvdcss.html)
* [libfaketime](https://github.com/wolfcw/libfaketime)
* [udev](http://man7.org/linux/man-pages/man7/udev.7.html)


## Examples
Given the output directory `/media/inbox/` and the following example directory structure:

```
media
+-inbox
| +-CD
| +-DVD
| +-BD
+-audio
| +-single-track.mp3
| +-album
|   +-01.ogg
|   +-02.flac
|   +-folder.jpg
+-video
| +-movie.mp4
| +-movie
| | +-VIDEO_TS
| +-series
|   +-episode-01
|   | +-VIDEO_TS
|   +-episode-02
|     +-VIDEO_TS
+-phone
```

* Rip a Blu-ray disc (1:1 copy) lossless to `/media/inbox/BD`
  and receive an e-mail notification on finish:

  _Insert the disc into the drive_

* Compress DVD directory `/media/video/movie/` to small MKV files
  containing a single language, using VAAPI hardware transcoding:
  ```shell
  $ bin/transcode-disc-directory /media/video/movie/
  ```

* The same for a directory containing several DVDs of a series
  in directory `/media/video/series`:
  ```shell
  $ bin/transcode-series-directory /media/video/series/
  ```

* For a MP4 file `/media/video/movie.mp4` with output format MP4:
  ```shell
  $ bin/vaapi-transcode /media/video/movie.mp4
  ```

* For a MP4 file `/media/video/movie.mp4` with output format MKV:
  ```shell
  $ bin/vaapi-transcode /media/video/movie.mp4 /media/phone/movie.mkv
  ```

* Convert DVD directory `/media/video/movie` losslessly to MKV format:
  ```shell
  $ bin/remux-directory /media/video/movie/
  ```

* Convert audio directory `media/audio` to mp3, keeping tags:
  ```shell
  $ bin/transcode-audio-directory /media/audio/
  ```
