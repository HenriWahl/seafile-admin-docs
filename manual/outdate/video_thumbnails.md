# Video thumbnails

### Install ffmpeg package

You need to install ffmpeg package to let the video thumbnail work correctly:

**Ubuntu 16.04**
```
# Install ffmpeg
sudo apt-get update && sudo apt-get -y install ffmpeg
 
# Now we need to install some modules
pip install pillow moviepy
```

**Centos 7**
```
# We need to activate the epel repos
yum -y install epel-release
rpm --import http://li.nux.ro/download/nux/RPM-GPG-KEY-nux.ro

# Then update the repo and install ffmpeg
yum -y install ffmpeg ffmpeg-devel

# Now we need to install some modules
pip install pillow moviepy
```

**Debian Jessie**
```python
# Add backports repo to /etc/apt/sources.list.d/
# e.g. the following repo works (June 2017)
sudo echo "deb http://httpredir.debian.org/debian $(lsb_release -cs)-backports main non-free" > /etc/apt/sources.list.d/debian-backports.list

# Then update the repo and install ffmpeg
sudo apt-get update && sudo apt-get -y install ffmpeg

# Now we need to install some modules
pip install pillow moviepy
```

### Configure Seafile to create thumbnails

Now configure accordingly in `seahub_settings.py`

```python
# Enable or disable thumbnail for video. ffmpeg and moviepy should be installed first. 
# For details, please refer to https://manual.seafile.com/deploy/video_thumbnails/
# NOTE: since version 6.1
ENABLE_VIDEO_THUMBNAIL = True

# Use the frame at 5 second as thumbnail
THUMBNAIL_VIDEO_FRAME_TIME = 5  

# Absolute filesystem path to the directory that will hold thumbnail files.
THUMBNAIL_ROOT = '/haiwen/seahub-data/thumbnail/thumb/'
```
