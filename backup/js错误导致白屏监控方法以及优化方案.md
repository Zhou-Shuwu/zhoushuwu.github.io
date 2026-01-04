**问题**
1、定位不到具体是什么原因造成，阿里云错误日志采集不到或者日志并不能标明是否白屏；
2、体验不好，一片空白，界面没有告诉用户接下来该怎么操作；
3、系统没有版本号区别，定位不到是哪个版本导致的问题。
**分析**
1、日志监控不到，错误监控注册时机问题，并且少了应用加载失败导致的阻塞白屏场景，没有白屏场景的业务监控。
2、白屏依赖用户上报，没有数据支撑，用户不知道接下来该怎么操作。
3、由于钉钉微应用对系统的强缓存，对于微应用版本留存情况缺少版本分布数据的支撑。
**方案**
1、对系统错误日志监听逻辑梳理，在index.html页面加上js错误全局监听方法window.addEventListener('error', (errEvent)=>{}, true)，当然也可以将xiangzhu-front-monitor阿里云日志监听放在这里进行初始化。
2、使用文档dom打点对比的方式对html、body、#root、#app节点判断应用或页面是否白屏。js逻辑代码以及白屏后动态添加用户提示代码文件：

[openWhiteScreen.js](https://github.com/user-attachments/files/24420480/openWhiteScreen.js)

<img width="1766" height="736" alt="Image" src="https://github.com/user-attachments/assets/9cae12e2-7608-44e0-974a-4a879b286a88" />

3、event日志埋点封装的http请求类方法：

[common-tools.js](https://github.com/user-attachments/files/24420508/common-tools.js)


4、系统发布上线增加版本号，并将埋点日志增加版本号标记。对webpack打包进行改造，增加版本号生成功能。

<img width="1644" height="808" alt="Image" src="https://github.com/user-attachments/assets/03976f10-2a78-4bed-8235-0496c5f03061" />

该方案依赖html-webpack-plugin这个插件，打包地方导入方法并生成数据，详细配置如下：

<img width="469" height="444" alt="Image" src="https://github.com/user-attachments/assets/f9b53828-3015-4160-be08-ca2d7d128eed" />

plugins: [
  new HtmlWebpackPlugin({
    template:
      process.env.API_ENV == 'prod'
      ? './src/index-prod.ejs'
      : process.env.API_ENV == 'staging'
      ? './src/index-staging.ejs'
      : process.env.API_ENV == 'test'
      ? './src/index-test.ejs'
      : process.env.API_ENV == 'dev'
      ? './src/index-dev.ejs'
      : './src/index.ejs',
    // title: '信贷管理系统', //更改HTML的title的内容
    favicon: `${SRC_PUBLIC_PATH}/favicon.png`,
    chunks: ['runtime', 'vendor', 'app'],
    minify: {
      removeComments: true,
      collapseWhitespace: true,
      removeAttributeQuotes: true,
    },
    buildInfo: BuildInfo('production'),
  })
]

5、index.html页面里初始化以及加载逻辑：

<img width="634" height="201" alt="Image" src="https://github.com/user-attachments/assets/392d44e2-4f4b-4a5e-915e-609be551735d" />

**接入步骤**
第一步：在html上面依次引入：common-tools.js
第二步：初始化http上报埋点方法
<script>
    // 全局系统当前版本
    window.version = "<%= JSON.parse(htmlWebpackPlugin.options.buildInfo).buildDate %>"
    // 全局参数收集器
    try {
      window.GlobalParmas = {
        // 实例化埋点方法
        logTrace: new LogTraceReport('prod'), // prod 根据 <%= JSON.parse(htmlWebpackPlugin.options.nodeEnv) %> 这样取，webpack打包的时候注入
      }
    } catch(e) {
      console.log('加载资源报错', e)
    }
   /**
     * @description: 初始化埋点参数
     * @param {Object} browser: 浏览器环境
     * @return {Object} eventId: 事件id, userId: 用户id, platform: 平台(值限定1、2、3、4), platformDesc: 平台描述
     */
    const initTraceParams = (browser)=>{
      return {
        eventId: 'hr_monitor',
        userId: localStorage.getItem('USERID') || localStorage.getItem('WX-USERINFO'),
        platform: 4,
        platformDesc: browser && browser.versions && (browser.versions.isWechat ? 'hr-weixin' : browser.versions.mobile ? 'hr-dingding' : 'hr-web')
      }
    }
  </script>
第三步：引入openWhiteScreen.js
第四步：加入已定义好的js错误处理方法js-onerror.js

[js-onerror.js](https://github.com/user-attachments/files/24420512/js-onerror.js)

注意：new LogTraceReport('prod') 传参一定要区分环境，'prod'是生产环境，测试环境可传'test'

// 移动终端浏览器版本信息
        isWechat: !!u.match(/MicroMessenger/i),
        trident: u.indexOf('Trident') > -1, // IE内核
        presto: u.indexOf('Presto') > -1, // opera内核
        webKit: u.indexOf('AppleWebKit') > -1, // 苹果、谷歌内核
        gecko: u.indexOf('Gecko') > -1 && u.indexOf('KHTML') == -1, // 火狐内核
        mobile: !!u.match(/AppleWebKit.*Mobile.*/), // 是否为移动终端
        ios: !!u.match(/\(i[^;]+;( U;)? CPU.+Mac OS X/), // ios终端
        android: u.indexOf('Android') > -1 || u.indexOf('Linux') > -1, // android终端或者uc浏览器
        iPhone: u.indexOf('iPhone') > -1, // 是否为iPhone或者QQHD浏览器
        iPad: u.indexOf('iPad') > -1, // 是否iPad
        webApp: u.indexOf('Safari') == -1,
        // 是否web应该程序，没有头部与底部

备注：怎样改变加载图标的颜色
 window.whiteLoadingParams = {
      loadingText: '', // 默认：页面加载中...
      loadingColor: 'red', // 默认：#333
    }

注：uniapp项目怎么接入
接入步骤第二步初始化http上报埋点方法中需引入window.whiteBoxElements自定义白屏判断容器点

window.whiteBoxElements = ['html', 'body', 'uni-app', 'uni-page', 'uni-page-wrapper', 'uni-page-body','uni-view']

<img width="1378" height="552" alt="Image" src="https://github.com/user-attachments/assets/05ced6b0-a183-4c96-989f-a83ca702277b" />

需在App.vue文件中定义onError方法并且条件编译H5中运行handleListenerError方法

<img width="710" height="210" alt="Image" src="https://github.com/user-attachments/assets/abddf0b1-8135-463e-b05c-98508612c9c5" />

备注：vite+ts项目，增加版本号及环境变量：

// 安装
 npm i vite-plugin-html -D

// 在src目录下增加 
addVersion.js

代码如下：
//npm run build打包前执行此段代码
import fs from 'fs';

//返回package的json数据
function getPackageJson() {
  let data = fs.readFileSync('./package.json');//fs读取文件
  return JSON.parse(data);//转换为json对象
}

let packageData = getPackageJson();//获取package的json
let arr = packageData.version.split('.');//切割后的版本号数组
arr[0] = new Date().getMonth() + 1;
arr[1] = new Date().getDate();
arr[2] = new Date().getHours();

packageData.version = arr.join('.');//转换为以"."分割的字符串

//用packageData覆盖package.json内容
fs.writeFile(
  './package.json',
  JSON.stringify(packageData, null, "\t"
  ),
  (err) => { }
);


package.json修改build：
"build": "node ./src/addVersion.js && vue-tsc --noEmit && vite build"

vite.config.ts增加
import {createHtmlPlugin } from 'vite-plugin-html'

export default defineConfig({
plugins: [
  createHtmlPlugin({
      /**
       * 需要注入 index.html ejs 模版的数据
       */
      inject: {
        data: {
          version: require('./package.json').version,
          envStr: process.env.NODE_ENV === 'production' ? 'prod' : 'dev',
        },
      },
    }),
  ]
})


index.html中：
     <script>
      window.version = '<%= version %>';
      const envStr = '<%= envStr %>'
      console.log(window.version, envStr, 'version')
      // 全局参数收集器
      try {
        window.GlobalParmas = {
          // 实例化埋点方法
          logTrace: new LogTraceReport(envStr),
        }
      } catch(e) {
        console.log('加载资源报错', e)
      }
        </script>
  
备注：非js错误状态下如何自动判断白屏
第一步：确保引入openWhiteScreen.js
第二步：执行openWhiteScreen方法，示例如下：
/**
 * 检测页面是否白屏
 * @param {function} callback - 回到函数获取检测结果, status(状态：error(是白屏),ok（不是白屏）,looping（轮询中）), loopTimes(轮询次数 1-10)
 * @param {boolean} skeletonProject - 页面是否有骨架屏
 * @param {array} whiteBoxElements - 容器列表，默认值为['html', 'body', '#app', '#root']
 */
openWhiteScreen(({status, loopTimes})=>{}, { skeletonProject: false, whiteBoxElements })