# Kali-Web安全随记

### 识别WAF

- nmap脚本

  nmap -p 80 --script=http-waf-detect x.x.x.x

  nmap -p 80 --script=http-waf-fingerprint x.x.x.x    (更加精确, 会尝试拦截响应)

- waf00f

  waf00f www.xxxx.com



### Wget:

- 是一个为下载HTTP内容创建的工具
- -r 递归下载
- -P设置保存的目录 
- -l 递归下载的时候，规定遍历深度
- -k 修改所有链接，能使站点在本地浏览
- -p 下载所有图像
- -w 破防自动浏览机制