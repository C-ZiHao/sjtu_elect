# sjtu_elect

疫情在家，闲来无事，正好开始第三轮选课，便试着写了一下i.sjtu.edu.cn的选课脚本，毕竟大一的时候蛮想有个选课脚本的，但迫于大一的代码水平问题QAQ，也找不到好的教程，所以用了点时间将写代码时的过程整理了一下，当一个简单的练习教程。

代码质量不高，但也懒得改了（毕竟快大三了，也不怎么需要抢课），发出来也只是当一种选课脚本思路，这样子的话，即便选课网小重构也是可以自己修改的，核心代码都有了，部分小的地方需要自己补充。

教程很简单，没有用很复杂的东西，有一点python基础的就可以写出来，个人觉得最大的难点在于分析选课网的拼音变量命名法（大雾）。

## 程序作用：捡课 
## 编写语言：python
## 用到的python库：

模拟登录：selenium

jaccount验证码识别：pytesseract

发送选课请求：requests


```python
from time import sleep
import re
import datetime 
from PIL import Image
import pytesseract
from selenium.webdriver.chrome.options import Options
from selenium import webdriver
from selenium.webdriver.common.keys import Keys
import requests
import random
```

## 主要过程：

### 1.登录

登录i.sjtu.edu.cn,或者说登录甲亢，甲亢的登录界面如下，登录需要传递三项参数：甲亢账户，密码和验证码。
需要我们获取的也就是验证码，但甲亢的验证码是每次访问都会刷新的，获取链接下载验证码图片再进行识别有点麻烦，索性用selenium模拟登录直接截屏QAQ，简单粗暴。

![登录](https://github.com/C-ZiHao/sjtu_elect/blob/master/%E5%9B%BE%E7%89%871.png)

既然决定使用selenium，首先要启动浏览器，这里我用的chromdriver。 

```python
##chrome_options=Options()  
##chrome_options.add_argument('--headless')
##chrome_options.add_argument('--disable-gpu')  ##浏览器不显示

driver=webdriver.Chrome(r'E:\vscode work\PYTHON\chromedriver.exe') 
 ##建议前期调试让浏览器显示，后期加上参数chrome_options=chrome_options
driver.implicitly_wait(15) 
driver.maximize_window()  
driver.get('https://i.sjtu.edu.cn')
```

 


selenium模拟浏览器登录，那我们怎么登录，就怎么写代码让浏览器这么运行。

打开教学信息服务网后，即下面这一界面，点击jaccount登录
![登录](https://github.com/C-ZiHao/sjtu_elect/blob/master/%E5%9B%BE%E7%89%872.png)

 ```python
driver.find_element_by_xpath('//*[@id="authJwglxtLoginURL"]').click()
```

然后进入甲亢登录界面，开始第一个函数,这样，验证码的图片便被保存到设置的picroot1路径中

```python
input_user = driver.find_element_by_id('user').send_keys(user)
input_pass = driver.find_element_by_id('pass').send_keys(password)
uuid =driver.find_element_by_xpath('/html/body/div/div[2]/div/div[2]/div[2]/div[2]/form/div[3]/div/img')
driver.save_screenshot(picroot1)  
rangle=(left, upper, right, lower)
i = Image.open(picroot1)  
#i.show()
i = i.resize((1920,850))
i = i.crop(rangle)  
i.save(picroot2)  
```

接下来识别验证码，打开灰度调节pytesseract识别，获得验证码信息。
由于识别不一定准确，需要设定简单的循环判断是否登录成功，没有的话，刷新再识别。

```python
image = Image.open(picroot2)
image=image.convert('L')
threshold=230
table=[]
for i in range(256):
     if i <threshold:
         table.append(0)
     else:
         table.append(1)
image=image.point(table,'1')
result = pytesseract.image_to_string(image,lang='eng')
result=result.replace(' ','')
```

然后将账号密码验证码填入，点击登录即可

```python
input_user = driver.find_element_by_id('user').send_keys(user)
input_pass = driver.find_element_by_id('pass').send_keys(password)    
input_code = driver.find_element_by_id('captcha')
input_code.send_keys(result)
input_code.send_keys(Keys.ENTER)
driver.refresh()
```

### 2.选课
主要用到的网站（stu_id是学号）

```python
url1='https://i.sjtu.edu.cn/xsxk/zzxkyzb_cxZzxkYzbIndex.html?gnmkdm=N253512&layout=default&su='+stu_id
url2='https://i.sjtu.edu.cn/xsxk/zzxkyzb_cxJxbWithKchZzxkYzb.html?gnmkdm=N253512&su='+stu_id
url3='https://i.sjtu.edu.cn/xsxk/zzxkyzb_xkBcZyZzxkYzb.html?gnmkdm=N253512&su='+stu_id
```

![登录](https://github.com/C-ZiHao/sjtu_elect/blob/master/%E5%9B%BE%E7%89%873.png)

选课这一部分，我们首先看一下，正常选课是怎么做的，打开F12 Network,随便找一门课程，这里我找了门任选课

![登录](https://github.com/C-ZiHao/sjtu_elect/blob/master/%E5%9B%BE%E7%89%874.png)

很好发现，选课其实就是向url3发送一个post请求，这样子就好办了，只要拿到cookies和传递的data就可以了
### 1.先获取cookie，cookie很好获得，用selenium直接取下来就可以，转一下格式就可以

```python
def getcookie():
    driver.get(url1)
    cookies = driver.get_cookies()
    i=1
    for item in cookies:
        if i==1:
            cookie = item['name'] + '=' + item['value']
            i=i+1
        else:
            cookie = cookie+'; '+item['name'] + '=' + item['value']
    return cookie
```

然后headers就得到了

```python
headers = {
       'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/80.0.3987.106 Safari/537.36',
       'Cookie': cookie,
       'X-Requested-With': 'XMLHttpRequest'
        }
```

### 2.获取data，从网站直接复制下来的data的列表分析一下 
但yysy，这个网站构架的人，似乎有点喜欢用拼音缩写作为参数名，就像这样
```python
jxb_ids: ##隐私
kcmc: (TH201)习近平新时代中国特色社会主义思想概论 - 2.0 学分
rwlx: 1
rlkz: 0
rlzlkz: 1
sxbj: 1
xxkbj: 0
qz: 0
cxbj: 0
xkkz_id: ##隐私
njdm_id: 2018
zyh_id: ###隐私
kklxdm: 30
xklc: 5
xkxnm: 2019
xkxqm: 12
```

然后，我就分析（用输入法盲猜）了将近半个小时的参数名的含义，大概结果是这样的

```python
data={
    'jxb_ids':jxb_ids,##教学班id,实时更新
    'kch_id':kch_id, # 课程号id，在F12自己获取
    'kcmc':kcmc ,  # 教学班名称，自己修改
    'rwlx': '1',  ##什么类型qaq，通选、重修、通识、留学生、民族生是2，板块课（英语、体育）是3，任选主修是1
    'rlkz': '0',   ##自闭，猜不出来了，似乎是定值
    'rlzlkz': '1', ##自闭，猜不出来了，似乎是定值
    'sxbj': '1',  #筛选标记？？第三轮是定值1，前两轮未定
    'xxkbj': '0',  ##选修课标记？？不确定
    'qz': '0',  ##权重，看网站代码，似乎控制选课数目是用权重来的，比如第一轮通识课的限制，第三轮权重自然为0
    'cxbj': '0',##重修标记，默认不改
    'xkkz_id': xkkz_id,  ##与选课轮数，选课类型都有关，自行修改，F12很好获取
    'njdm_id': '2018',     # 年级代码，自己改
    'zyh_id': zyh_id,    # 专业号id,自己改
    'kklxdm': '30',  ##开课类型代码：主修01，任选30，民族生71，留学生72，板块课（英语，体育）06，通识10，重修08，通选11
    'xklc': '5',   ##选课什么，猜不出来了QAQ
    'xkxnm': '2019',  ##选课学年码
    'xkxqm': '12'     ##选课学期码
    }
```

除了jxb_ids，其余参数，是很好获取的，你可以在人满的时候，点一下选课，就可以在发送的post请求中把其它参数复制下来。

至于jxb_ids，由于是实时更新的，所以，我们要获取一下

不难发现，向url2发送post请求，就可以在返回的text中，找到do_jxb_id即jxb_ids，用正则匹配即可

 ![登录](https://github.com/C-ZiHao/sjtu_elect/blob/master/%E5%9B%BE%E7%89%875.png)
 
```python
def getjxbid(headers):
    http=requests.post(url2,headers=headers,data=userData)
    jxb_ids=re.search('"do_jxb_id":"(.*?)"',http.text).group(1)
    return jxb_ids
```

而cookie之后会获得，而这个userdata....

```python
rwlx: 1
xkly: 0
bklx_id: 0
xqh_id: 02
jg_id: 03000
zyh_id: ##隐私
zyfx_id: wfx
njdm_id: 2018
bh_id: ##隐私
xbm: 1
xslbdm: 111
ccdm: 3
xsbj: ##隐私
sfkknj: 1
sfkkzy: 1
sfznkx: 1
zdkxms: 99
sfkxq: 1
sfkcfx: 0
kkbk: 0
kkbkdj: 0
xkxnm: 2019
xkxqm: 12
rlkz: 0
kklxdm: 30
kch_id: 929AF5CE2D6840F1E0530200A8C0DEB1
xkkz_id: ##隐私
cxbj: 0
fxbj: 0
```
又让我不得不吐槽一局变量的命名，又浪费了我不少时间去猜，结果大致如下：

```python
userData = {
    'rwlx': '1',  ##什么类型，通选、重修、通识、留学生、民族生是2，板块课（英语、体育）是3，任选主修是1
    'xkly': '0',   ##选课什么，主修2，其余好像都是0
    'bklx_id': '0', ##板块类型，只对板块课大英体育有用，f12自行获取，其余课程全为0
    'xqh_id': '02', ##猜不出来了，定值？？
    'jg_id': '03000',  # 学院（机构）
    'zyh_id': zyh_id,  # 专业号id
    'zyfx_id': 'wfx',  ##专业方向，无方向？
    'njdm_id': '2018',  ##年级代码
    'bh_id': bh_id,#班级号
    'xbm': '1',  # 性别名
    'xslbdm': '111', ##学生类别
    'ccdm': '3',  # 暂时不知道是什么
    'xsbj': xsbj,
    'sfkknj': '1',  # 是否开课年级
    'sfkkzy': '1',  # 是否开课专业
    'sfznkx': '1',  # 是否开课年级
    'zdkxms': '99', ##任选99，其余全为0
    'sfkxq': '1',  ##定值1？
    'sfkcfx': '0',   ##定值0？
    'kkbk': '0',  ##是否板块课？是1，否0
    'kkbkdj': '0',##是否板块课？是1，否0
    'xkxnm': '2019',  # 选课学年码
    'xkxqm': '12',  #选课学期码
    'rlkz': '0',     ##自闭，猜不出来了，似乎是定值
    'kklxdm': '30',  ##开课类型代码：主修01，任选30，民族生71，留学生72，板块课（英语，体育）06，通识10，重修08，通选11
    'kch_id': kch_id,  # 课程号id，在F12自己获取
    'xkkz_id': xkkz_id,   ##与选课轮数，选课类型都有关，自行修改
    'cxbj': '0',    ##重修标记，默认不改
    'fxbj': '0'  ##方向代码
}
```

### 3.发送post
```python
response = session.post(url3,headers=headers,data=data,verify=False)
flag=re.search(r'"flag":"(-?\d)"',response.text)
return int(flag.group(1))
```
flag返回1，即成功，-1即失败。
之后将，代码写成循环，返回-1，重新获取jxb_id,发送post即可，循环延迟建议用随机函数写的时间长一点，emmmmm，被系统发现就不好了
也就达成了我们最初的目的，捡被退掉的好课。
                                                 
												 
												 写于2020.2.29晚
