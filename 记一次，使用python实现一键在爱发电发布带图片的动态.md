#### 1、背景
本人喜欢转载一些youtube上的视频到b站上面，然后就会有些观众想要视频的封面，那我总不可能一个一个发吧，太麻烦了。故打算将资源发布到爱发电上面。但是爱发电却没有公开对应的api，只能自己动手了呀。
<br />
#### 2、思路
发布一条动态，抓包看看调用了哪些api<br />写代码仿照其中的参数调用api<br />封装起来，增加易用性
<br />
#### 3、发布一条动态，抓包看看调用了哪些api
##### 3.1、发布动态
首先，发布动态长这个样子<br />![image.png](https://cdn.jsdelivr.net/gh/Canwaiting/picfornote/202303200905419.png)<br />这里我们直接发布一条动态
##### 3.2、抓包
![image.png](https://cdn.jsdelivr.net/gh/Canwaiting/picfornote/202303200906511.png)<br />可以看到这个发布动态的请求地址为：https://afdian.net/api/post/publish，接下来我们看看我们的请求体是什么<br />![image.png](https://cdn.jsdelivr.net/gh/Canwaiting/picfornote/202303200857005.png)<br />可以看到这个表单挺简单的，但是有一个不太对劲的情况，就是字段"pics"的值不是我上传的地址的值（"./img.jpg")，所以我猜，在调用api/post/publish接口前肯定调用了某个接口，将本地的图片转换成上图的链接，然后再拼接到上图中的字段"pics"中一起提交
##### 3.3、抓取本地图片转换成爱发电链接的api
在发布动态的界面直接点击上传图片<br />![image.png](https://cdn.jsdelivr.net/gh/Canwaiting/picfornote/202303200907585.png)<br />选择好图片，加载完毕之后，你就可以看见，调用了一个接口api/upload/common-pic，到这里我还没有点击发布<br />![image.png](https://cdn.jsdelivr.net/gh/Canwaiting/picfornote/202303200908919.png)<br />![image.png](https://cdn.jsdelivr.net/gh/Canwaiting/picfornote/202303200905420.png)<br />可以看到的确是和猜想的一样，在提交发布动态的表单前，先提交图片，获取到图片的链接之后，再拼接到发布动态的表单，进行发布
##### 3.4、总结
到这里我们可以知道，一个完整的发布动态流程包括两个部分<br />1、将本地图片转换成一个官方链接，对应api（/upload/common-pic)<br />2、将表单中的信息（包括上面提到的链接）进行发布，对应api（/post/publish)
<br />
#### 4、代码实现
首先我们初步选定了技术为python + request库<br />补充：代码中cookies你登陆爱发电后直接按F12，随便点几下有发送请求的按钮，然后在request中翻翻就可以看见了
##### 4.1、将本地图片转换成爱发电官方链接
###### 4.1.1、前期分析
我们先来看看我们需要模拟的request请求<br />![image.png](https://cdn.jsdelivr.net/gh/Canwaiting/picfornote/202303200909452.png)<br />![image.png](https://cdn.jsdelivr.net/gh/Canwaiting/picfornote/202303200910445.png)<br />上网搜了一下，这里大概的意思就是爱发电的这个接口需要的content-type是比较少见的类型：multipart/form-data<br />然后第二张图就是我这次上传的数据的数据体<br />参考：[https://www.jianshu.com/p/902452189ca9](https://www.jianshu.com/p/902452189ca9)，这个给了我大概的思路，request库对于这种类型的支持不是太好，需要借助库requests_toolbelt。然后因为我的数据是文件类型，而该博客中的是文本，所以不能直接照搬<br />参考：[https://www.cnblogs.com/yoyoketang/p/8024039.html](https://www.cnblogs.com/yoyoketang/p/8024039.html)，[https://pypi.org/project/requests-toolbelt/](https://pypi.org/project/requests-toolbelt/)终于把代码写出来了
###### 4.1.2、代码

```python
import requests
from requests_toolbelt import MultipartEncoder

# 1、初始化需要的数据，包括目标网址、请求头、cookies、图片路径等信息
url = 'https://afdian.net/api/upload/common-pic'
headers = {'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/111.0.0.0 Safari/537.36'}
cookies = {
            'auth_token': 'xxxx',
            '_gid':'xxxx',
            '_ga_6STWKR7T9E': 'xxxx',
            '_ga': 'xxxx'
}
img_path = 'img.jpg' 

# 2、使用 MultipartEncoder 创建文件上传的 payload
file_payload = {
    "name":"file",
    "filename":img_path,
    'file':(img_path,open(img_path,'rb'))
}
m = MultipartEncoder(file_payload)

# 3、补充头
headers['Content-Type'] = m.content_type # multipart/form-data;boundary=29cf7f1b13584a73a6630a738be8274a

# 4、发送请求
response = requests.post(url, headers=headers, data = m, cookies=cookies)

# 5、输出响应内容
print(response.text)
```
![image.png](https://cdn.jsdelivr.net/gh/Canwaiting/picfornote/202303200905421.png)<br />注意：这里我开启了转义，实际上我们需要的也是转义后的数据，不过到现在我还没进行处理<br />**值得一提的是，我还没有进行“发布”呢，返回的链接就可以直接打开了，按道理来说，可以用作图床，但是不知道爱发电会不会进行检测，类似上传了的图片在多久没进行发布就会删除。不过，还是不要用作图床了，厚道一点，毕竟爱发电**
###### 4.1.3、进一步封装
因为我们需要的是这个官方链接，所以，这里我对此进行封装成一个函数
```python
import json
import requests
from requests_toolbelt import MultipartEncoder

# 参数：图片地址
# 返回：官方链接
def get_link(img_path):
    # 1、初始化需要的数据，包括目标网址、请求头、cookies、图片路径等信息
    url = 'https://afdian.net/api/upload/common-pic'
    headers = {'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/111.0.0.0 Safari/537.36'}
    cookies = {
                'auth_token': 'xxxx',
                '_gid':'xxxx',
                '_ga_6STWKR7T9E': 'xxxx',
                '_ga': 'xxxx'
    }

    # 2、使用 MultipartEncoder 创建文件上传的 payload
    file_payload = {
        "name":"file",
        "filename":img_path,
        'file':(img_path,open(img_path,'rb'))
    }
    m = MultipartEncoder(file_payload)

    # 3、补充头
    headers['Content-Type'] = m.content_type # multipart/form-data; boundary=29cf7f1b13584a73a6630a738be8274a

    # 4、发送请求
    response = requests.post(url, headers=headers, data = m, cookies=cookies)
    response.encoding = "unicode_escape" # 将utf-8 转换成 unicode

    # 5、返回响应内容
    # 5.1、将响应内容转换成json
    response_json = json.loads(response.text)
    # 5.2、返回想要的字段link
    link = response_json['link']
    return link

if __name__ == "__main__":
    img_path = "img.jpg"
    link = get_link(img_path)
    print(link)

'''
输出：
https://pic1.afdiancdn.com/user/bc58exxxxxxx2540025c377/common/5239daxxxxxxxxxx80_h720_s117.jpg
'''
```
##### 4.2、发布动态
###### 4.2.1、前期分析
![image.png](https://cdn.jsdelivr.net/gh/Canwaiting/picfornote/202303200857009.png)<br />可以看到这里是json类型的数据，对应的数据字段太多，这里就不一一展示，等下在代码中你就知道了
<br />
###### 4.2.2、代码
```python
import json
import requests
from requests_toolbelt import MultipartEncoder


# 参数：图片地址
# 返回：官方链接
def get_link(img_path):
    # 1、初始化需要的数据，包括目标网址、请求头、cookies、图片路径等信息
    url = 'https://afdian.net/api/upload/common-pic'
    headers = {'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/111.0.0.0 Safari/537.36'}
    cookies = {
                'auth_token': 'xxxx',
                '_gid':'xxxx',
                '_ga_6STWKR7T9E': 'xxxx',
                '_ga': 'xxxx'
    }

    # 2、使用 MultipartEncoder 创建文件上传的 payload
    file_payload = {
        "name":"file",
        "filename":img_path,
        'file':(img_path,open(img_path,'rb'))
    }
    m = MultipartEncoder(file_payload)

    # 3、补充头
    headers['Content-Type'] = m.content_type # multipart/form-data; boundary=29cf7f1b13584a73a6630a738be8274a

    # 4、发送请求
    response = requests.post(url, headers=headers, data = m, cookies=cookies)
    response.encoding = "unicode_escape" # 将utf-8 转换成 unicode

    # 5、返回响应内容
    # 5.1、将响应内容转换成json
    response_json = json.loads(response.text)
    # 5.2、返回想要的字段link
    link = response_json['link']
    return link

# 发布函数
# 传入：一个字典类型的数据，包含：标题、内容、图片链接
# 返回：响应数据
def publish(publish_data):
    # 1、初始化需要的数据，包括目标网址、请求头、cookies、请求数据等信息
    url = 'https://afdian.net/api/post/publish'
    headers = {'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/111.0.0.0 Safari/537.36'}
    cookies = {
                'auth_token': 'xxxx',
                '_gid':'xxxx',
                '_ga_6STWKR7T9E': 'xxxx',
                '_ga': 'xxxx'
    }
    title = publish_data['title']
    content = publish_data['content']
    link = publish_data['link']
    data_dict = {
                "post_id":"",
                "vote_id":"",
                "cate":"normal",
                "title":title,
                "content":content,
                "pics":link,
                "is_public":"1",
                "min_price":"0",
                "audio":"",
                "video":"",
                "audio_thumb":"",
                "video_thumb":"",
                "type":"0",
                "cover":"",
                "group_id":"",
                "is_feed":"0",
                "plan_ids":"",
                "album_ids":"",
                "attachment":"[]",
                "timing":"",
                "optype":"publish",
                "preview_text":""
    }


    # 4、发送请求
    response = requests.post(url, headers=headers, data = data_dict, cookies=cookies)

    # 5、返回响应内容
    response.encoding = "unicode_escape" # 将utf-8 转换成 unicode
    return response.text

if __name__ == "__main__":
    img_path = "img.jpg"
    link = get_link(img_path)
    publish_data = {
        "title" : "标题测试",
        "content" : "内容测试",
        "link" : link
    }
    rep = publish(publish_data)
    print(rep)
```
###### 4.2.3、成功
![image.png](https://cdn.jsdelivr.net/gh/Canwaiting/picfornote/202303200905422.png)
<br />
#### 5、总结
##### 5.1、完整代码 Github地址
[https://github.com/Canwaiting/afdian-api](https://github.com/Canwaiting/afdian-api)
##### 5.2、总结
写完这篇博文，自己也学到了不少，这是我写的最长的一篇博客，本来是当作记录自己思路的草稿，写着写着，整理整理，就发出来了。还有很多可以优化的地方，例如每个字段，我也没搞清具体是什么回事，还有上传多张图片要怎么办等等
