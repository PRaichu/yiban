#易班自动打卡脚本（基于湖南工程学院）
```
#2020-6-1  优化异常处理
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

#userfile = "xxxxxxxxxxxxxx/user.txt"    #原用户文件地址
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
