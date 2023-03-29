#### 背景
作为一个喜欢搬运 YouTube 视频的网友，我发现将视频下载下来再上传到 B 站十分繁琐，因此我决定开发一个小工具，能够方便快捷地将 YouTube 视频下载并上传至 B 站，以节省我的时间和精力。
<br />
#### 项目实现
语言：Python<br />主要借助：request库、yt-dlp、biliup
<br />
完整代码：
```python
from PIL import Image
import json
import sys
import requests
import yt_dlp
import subprocess
from biliup.plugins.bili_webup import BiliBili, Data

def download(url):
    # 初始化
    video_info = {} 
    audio_ydl_opts = {
        'format': 'bestaudio/best',
        'postprocessors': [{
            'key': 'FFmpegExtractAudio',
            'preferredcodec': 'mp3',
        }],
        'outtmpl': './resource/%(channel)s/%(id)s'
    }
    video_ydl_opts = {
        'format' : 'bv[height<=1080][ext=mp4]+ba[ext=m4a]',
        'outtmpl': './resource/%(channel)s/%(id)s.mp4'
    }

    # 下载音频
    with yt_dlp.YoutubeDL(audio_ydl_opts) as ydl:
        ydl.download(url)

    with yt_dlp.YoutubeDL(video_ydl_opts) as ydl:
        # 获取视频信息
        extract_info = ydl.extract_info(url, download=False)
        video_info['id'] = extract_info['id']
        video_info['title'] = extract_info['title']
        video_info['description'] = extract_info['description']
        video_info['url'] = extract_info['webpage_url']
        video_info['categories'] = extract_info['categories']
        video_info['tags'] = extract_info['tags']
        video_info['video_path'] = f"./resource/{extract_info['channel']}/{extract_info['id']}.mp4"
        video_info['audio_path'] = f"./resource/{extract_info['channel']}/{extract_info['id']}.mp3"
        video_info['thumbnail_path'] = f"./resource/{extract_info['channel']}/{extract_info['id']}.png"
        # 下载视频
        ydl.download(url)
        # 下载壁纸，并转换成可以上传的格式
        imgData = requests.get(extract_info['thumbnail']).content
        with open(f"./resource/{extract_info['channel']}/{extract_info['id']}.webp", "wb") as handler:
            handler.write(imgData)
        im = Image.open(f"./resource/{extract_info['channel']}/{extract_info['id']}.webp").convert("RGB")
        im.save(f"./resource/{extract_info['channel']}/{extract_info['id']}.png")
    return video_info

    # 上传
def upload(video_info):
    return_code = subprocess.run([
        './biliup', 'upload', f"{video_info['video_path']}",
        '--copyright', '2',
        '--cover', f"{video_info['thumbnail_path']}",
        '--desc', f"{video_info['description'][:99]}",
        '--source', f'{video_info["url"][:79]}',
        '--tag', 'youtube',
        '--title', f'{video_info["title"]}',
    ])
    print("return code:", return_code)



if __name__ == "__main__":
    # 获取url
    url = sys.argv[1]
    # 下载
    video_info = download(url)
    # 上传
    upload(video_info)

```
