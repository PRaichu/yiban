易班自动打卡脚本（适用于湖南工程学院健康打卡系统）
=========================
<br><br>

一、基于ChromeDriver和selenium实现的自动打卡程序
------------------------
>语言：Python<br>
>原理：通过模拟浏览器操作，登录打卡网站，并自动打卡<br>
>优点：适应性强，此方法可识别验证码<br>
>缺点：速度慢、受网络等因素影响较大<br>
>>运行环境（其他环境自测）：<br>
>>系统：Linux CentOS7.7<br>
>>Python版本:3.7.4<br><br>


```Python
import os
import shutil
import json
import urllib.request
import datetime
from selenium import webdriver
import time
from datetime import datetime
from concurrent.futures import ThreadPoolExecutor
import threading
import pymysql

tempfile = "xxxxxxxxxxxxxx/usertemp.txt"    #副本文件地址
logsfolder = "xxxxxxxxxxxxxx/logs"  #log文件夹地址，需要自己创建
templogfile = logsfolder + "/templog.txt"   #不要改动
picfolder = "" #不要改动
n = 10   #重复执行上限，至少为1
count = 0
picfileflag = 1

def deluser(_id,_path):
    with open(_path,'r',encoding = 'UTF-8') as r:
        lines=r.readlines()
        r.close()
    with open(_path,'w',encoding = 'UTF-8') as w:
        for l in lines:
            if _id not in l:
                w.write(l)
        w.close()

def wirtelog(log,_path):
    print(log)
    with open(_path,"a+",encoding = "UTF-8") as f:
        f.write(log)
        f.write('\n')
        f.close()

def piclog(driver,_path,picname):
    picpath = _path + "/" + picname + ".png"
    driver.get_screenshot_as_file(picpath)

def picfile(_path):
    global picfolder
    file = _path + "/" +str(time.strftime("%Y-%m-%d" , time.localtime()))
    picfolder = file
    folder = os.path.exists(file)
    if not folder:
        os.makedirs(file)
    else:
        shutil.rmtree(file)
        os.makedirs(file)

def run(_name,_id,_password):
    global count
    global picfileflag
    try: 
        chromeOptions = webdriver.ChromeOptions()
        chromeOptions.add_argument('--headless')  #浏览器无窗口加载
        chromeOptions.add_argument('--disable-gpu')  #不开启GPU加速
        chromeOptions.add_argument('--disable-dev-shm-usage') 
        chromeOptions.add_argument('--no-sandbox')#以根用户打身份运行Chrome，使用-no-sandbox标记重新运行Chrome,禁止沙箱启动
        browser = webdriver.Chrome(options=chromeOptions,executable_path="/usr/bin/chromedriver")
        
        browser.get('http://xggl.hnie.edu.cn/content/menu/student/temp/zzdk?_t_s_=1585105573057')
        time.sleep(3)
        
        browser.find_element_by_css_selector(".btn.btn-primary.signin-loader").click()
        time.sleep(1)
        
        browser.find_element_by_id("username").send_keys(_id)
        browser.find_element_by_id("password").send_keys(_password)
        browser.find_element_by_name("submit").click()
        time.sleep(1)
        
        browser.get("http://xggl.hnie.edu.cn/content/menu/student/temp/zzdk?_t_s_=1585201027691")
        time.sleep(3)
        
        browser.find_element_by_id("dk_btn").click()
        time.sleep(10)
        
        browser.find_element_by_id("save").click()
        time.sleep(1)
        
        count += 1
        logstr = "[" + str(count) + "]" +str(time.strftime("%Y-%m-%d %H:%M:%S：" + _name + "打卡成功！", time.localtime()))
        wirtelog(logstr,templogfile)
        
        deluser(_id,tempfile)
        browser.quit()
    except BaseException as err:
        count += 1
        if picfileflag:
            picfile(logsfolder)
            picfileflag = 0
        if "element not interactable" in str(err) or "element not visible" in str(err):
            logstr = "[" + str(count) + "]" +str(time.strftime("%Y-%m-%d %H:%M:%S：" + _name + "打卡失败！  失败原因：", time.localtime())) + "已打卡！"
            wirtelog(logstr,templogfile)
            time.sleep(2)
            piclog(browser,picfolder,_name + "(已打卡)")
            time.sleep(2)
            deluser(_id,tempfile)
            time.sleep(2)
        else:
            errtime = time.strftime("%Y-%m-%d %H:%M:%S：" , time.localtime())
            temperrtime = time.strptime(errtime, "%Y-%m-%d %H:%M:%S：")
            picnametime = time.strftime("(%Y-%m-%d-%H-%M-%S)", temperrtime)
            logstr = "[" + str(count) + "]" + errtime + _name + "打卡失败！  失败原因：" + str(err).replace('\n', '').replace('\r', '')
            time.sleep(2)
            piclog(browser,picfolder,_name+picnametime)
            time.sleep(2)
            wirtelog(logstr,templogfile)
        browser.quit()


def mainfun():
    pool = ThreadPoolExecutor(max_workers=7) #线程池线程数量
    f = open(tempfile,"r",encoding = 'UTF-8')

    for line in f:
        data_list = line.split()
        future1 = pool.submit(run, data_list[0],data_list[1],data_list[2])

    if future1.done() == False:
        time.sleep(15)
    pool.shutdown()
    f.close()

if __name__ == '__main__':
    #shutil.copyfile(userfile,tempfile)
    db = pymysql.connect(host='xxxxxxxxxxxxxx',port=3306,user='xxxxxxxxxxxxxx',password='xxxxxxxxxxxxxx',db='xxxxxxxxxxxxxx')
    cursor = db.cursor()
    cursor.execute("use xxxxxxxxxxxxxx;")
    cursor.execute("select count(*) from xxxxxxxxxxxxxx;")
    num = cursor.fetchone()
    cursor.execute("SELECT * FROM xxxxxxxxxxxxxx")
    data = cursor.fetchall()
    with open(tempfile,"w",encoding="UTF-8") as f:
        for i in range(0,num[0]):
            f.write(data[i][0]+" "+data[i][1]+" "+data[i][2]+'\n')
    f.close()
    m = 1
    while os.path.getsize(tempfile) and n:
        logstr = "-------------------------------第" + str(m) + "次运行-------------------------------"
        wirtelog(logstr,templogfile)
        mainfun()
        wirtelog('\n',templogfile)
        n -= 1
        m += 1
        count = 0
    #os.remove(tempfile)    #删除副本文件(可有可无，复制文件时会覆盖)
    os.rename(templogfile,logsfolder+"/"+str(time.strftime("%Y-%m-%d-%H-%M-%S",time.localtime()))+ ".txt")

```
<br><br>
二、基于requests实现的自动打卡程序
------------------------
>语言：Python<br>
>原理：发送请求登录打卡网站，获取上次打卡信息，并通过请求提交实现打卡<br>
>优点：速度快（保守估计比第一种方法快五十倍以上）<br>
>缺点：局限性较大，如果网站有验证码，就会很麻烦<br>
>>运行环境（其他环境自测）：<br>
>>系统：Linux CentOS7.7 / Win10<br>
>>Python版本:3<br><br>

```python
import requests
import re
import json
import pymysql

url_login="http://xggl.hnie.edu.cn/website/login"

headers = {
    'Host':"xggl.hnie.edu.cn",
    'Accept-Language':"zh-CN,zh;q=0.9,en;q=0.8",
    'Accept-Encoding':"gzip, deflate",
    'Content-Type':"application/x-www-form-urlencoded; charset=UTF-8",
    'Connection':"keep-alive",
    'Referer':"http://xggl.hnie.edu.cn/index",
    'User-Agent':"Mozilla/5.0 (Linux; Android 6.0; Nexus 5 Build/MRA58N) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/81.0.4044.138 Mobile Safari/537.36"
}

if __name__ == '__main__':
    db = pymysql.connect(host='xxxxxxxxxxxxxx',port=3306,user='xxxxxxxxxxxxxx',password='xxxxxxxxxxxxxx',db='xxxxxxxxxxxxxx')
    cursor = db.cursor()
    cursor.execute("use xxxxxxxxxxxxxx;")
    cursor.execute("select count(*) from xxxxxxxxxxxxxx;")
    num = cursor.fetchone()
    cursor.execute("SELECT * FROM xxxxxxxxxxxxxx")
    user_data = cursor.fetchall()
    for i in range(0,num[0]):
        data = {
            'username': user_data[i][1],
            'password': user_data[i][2],
            'action': "signin"
        }
        #登录
        session_requests = requests.session()
        login_res = session_requests.post(url_login,headers=headers, data=data)  

        #获取上次打卡信息
        last_url = "http://xggl.hnie.edu.cn/content/student/temp/zzdk/lastone"
        last_res = session_requests.get(last_url,headers=headers)
        last_text = last_res.content.decode("utf-8")
        xian = ""
        try:
            xian = json.loads(last_text)['jzdXian']['dm']
        except BaseException:
            xian = ""
        last_data = {
            'operationType' : "Create",
            'sfzx.dm' : json.loads(last_text)['sfzx'],
            'jzdSheng.dm' : json.loads(last_text)['jzdSheng']['dm'],
            'jzdShi.dm' : json.loads(last_text)['jzdShi']['dm'],
            'jzdXian.dm' : xian,
            'jzdDz' : json.loads(last_text)['jzdDz'],
            'jzdDz2' : json.loads(last_text)['jzdDz2'],
            'lxdh' : json.loads(last_text)['lxdh'],
            'grInd' : json.loads(last_text)['grInd'],
            'jcInd' : json.loads(last_text)['jcInd'],
            'jtqk.dm' : json.loads(last_text)['jtqk']['dm'],
            'jtqkXx' : json.loads(last_text)['jtqkXx'],
            'brqk.dm' : json.loads(last_text)['brqk']['dm'],
            'qwhbInd' : json.loads(last_text)['qwhbInd'],
            'qwhbXx' : json.loads(last_text)['qwhbXx'],
            'jchbrInd' : json.loads(last_text)['jchbrInd'],
            'jchbrXx' : json.loads(last_text)['jchbrXx'],
            'lxjlInd' : json.loads(last_text)['lxjlInd'],
            'lxjlXx' : json.loads(last_text)['lxjlXx'],
            'tw' : json.loads(last_text)['tw'],
            'bz' : json.loads(last_text)['bz'],
        }
        #print(last_data)
        #提交打卡
        dk_url = "http://xggl.hnie.edu.cn/content/student/temp/zzdk" + re.findall("welcome(.*?)\"",login_res.content.decode("utf-8"))[0]
        dk_res = session_requests.post(dk_url,headers=headers,data=last_data)
        print(dk_res.content.decode("utf-8"))
```
