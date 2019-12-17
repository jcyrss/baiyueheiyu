```py
class WX_Handler(AbstractHandler):
    def __init__(self):

        AbstractHandler.__init__(self,logging.getLogger("django"))

        self.METHOD_TAB = {
            'GET': {
                'wx_authorize':self.handle_authorize,
                'get_oid':self.handle_wx_get_oid_and_info,
            }

        }

    WX_USER_AUTHORIZE  = 'https://open.weixin.qq.com/connect/oauth2/authorize?appid='+ wx_settings.APP_ID + \
                             '&redirect_uri=http%3A%2F%2F<domain>%2Fapi%2Fpayment%2Fwx%2F%3Faction%3Dget_oid&'\
                                 .replace('<domain>',wx_settings.WX_AHTH_REDIRECT_DOMAIN)+ \
                             'response_type=code&scope=snsapi_userinfo&state=<state>#wechat_redirect'
    WX_GET_USER_OID_URL_FMT  = 'https://api.weixin.qq.com/sns/oauth2/access_token?appid=%s&secret=%s&code=%s&grant_type=authorization_code'
    WX_GET_USER_INFO_URL_FMT = 'https://api.weixin.qq.com/sns/userinfo?access_token=%s&openid=%s&lang=zh_CN'

    PHONE_HTML_TMPLT = u'''
    <!DOCTYPE html><html>
    <head>
    <meta http-equiv="content-type" content="text/html; charset=UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, minimum-scale=1.0, maximum-scale=1.0">
    </head>
    <body><div style='margin: 20% 5%;text-align: center;font-size: x-large;color:#127A88;'>
    %s</div>
    </body></html>
    '''


    PHONE_HTML_WX_AUTHOR_OK = u'''
    <!DOCTYPE html><html>
    <head>
    <meta http-equiv="content-type" content="text/html; charset=UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, minimum-scale=1.0, maximum-scale=1.0">
    </head>
    <body><div><img style="width: 100%"  src="http://cdn-ali-static.zgyjyx.com/etc/others/wx_authorize_ok.png"></div></body></html>
    '''

    # 该请求是从用户微信扫描二维码 发起的访问， 需要重定向到腾讯服务器
    def handle_authorize(self,request):
        # 获取该用户信息缓存的key，下面要用到
        self.checkMandatoryParams(request,['ck'])
        usercachekey = request.param_dict['ck']

        url = self.WX_USER_AUTHORIZE.replace('<state>',usercachekey)
        # print 'redirect to  ' + url
        return redirect(url)




    # 该请求是从用户微信从 腾讯服务器 重定向返回到 我们服务器 的访问，
    # 为什么放在common里面呢？ 没有办法，因为微信浏览器用户并没有登录我们系统
    # 这里就是为了给没有登录的请求使用的，不然就放在各种用户的app的 handler 里面了
    def handle_wx_get_oid_and_info(self,request):

        self.checkMandatoryParams(request,['code','state'])
        code = request.param_dict['code']
        getTokenUrl = self.WX_GET_USER_OID_URL_FMT % (wx_settings.APP_ID,
                                                      wx_settings.APP_SECRET,
                                                      code)

        # print 'getTokenUrl ' + getTokenUrl
        r = urllib.urlopen(getTokenUrl)
        content = r.read()
        retObj = json.loads(content)

        if 'openid' not in retObj:
            return HttpResponse(self.PHONE_HTML_TMPLT % u'不能获取用户openid')
        if 'access_token' not in retObj:
            return HttpResponse(self.PHONE_HTML_TMPLT % u'不能获取用户access_token')

        oid = retObj['openid']
        access_token = retObj['access_token']

        # print 'oid: %s' % oid
        # print 'access_token: %s' % access_token

        getInfoUrl = self.WX_GET_USER_INFO_URL_FMT % (access_token,oid)
        r = urllib.urlopen(getInfoUrl)
        content = r.read()
        retObj = json.loads(content)
        # print retObj


        if 'nickname' not in retObj:
            return HttpResponse(self.PHONE_HTML_TMPLT % u'获取用户信息失败')

        # 用户信息放入缓存，等待前端来主动获取
        ck = request.param_dict['state']
        cacheKey = CacheKeys.wx_userinfo %ck

        # print 'set cacheKey' + cacheKey
        cache.set(cacheKey,retObj,300)

        return HttpResponse(self.PHONE_HTML_WX_AUTHOR_OK )

        # return HttpResponse(self.PHONE_HTML_TMPLT % u'微信授权成功<br><br>有效期5分钟<br><br>请在网页上点击 "确认微信帐号" ')
```
