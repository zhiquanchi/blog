https://github.com/g1879/DrissionPage/issues/235

page = ChromiumPage()
错误提示：
DrissionPage.errors.BrowserConnectError:
127.0.0.1:9222浏览器无法链接。
请确认：
1、该端口为浏览器
2、已添加'--remote-debugging-port=9222'启动项
3、用户文件夹没有和已打开的浏览器冲突
4、如为无界面系统，请添加'--headless=new'参数
5、如果是Linux系统，可能还要添加'--no-sandbox'启动参数
可使用ChromiumOptions设置端口和用户文件夹路径。

代码已运行三个月，在未修改代码情况下报错


可能原因：
代理问题
网卡端口问题
抓包导致的代理和网络问题。

修复方案，
一般重置网络或者重启电脑后都可以修复。