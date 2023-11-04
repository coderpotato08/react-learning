# React执行npm run start 如何启动项目

通过package.json查询指令，可以发现start指令实际执行的是**react-scripts start**这条指令，在node包中的react-script/bin下的react-script.js就是执行react-script后实际运行的脚本

```js
const scriptIndex = args.findIndex(
  x => x === 'build' || x === 'eject' || x === 'start' || x === 'test'
);
const script = scriptIndex === -1 ? args[0] : args[scriptIndex];	
const nodeArgs = scriptIndex > 0 ? args.slice(0, scriptIndex) : [];	// build,eject,start,test以外的参数

if (['build', 'eject', 'start', 'test'].includes(script)) {
  const result = spawn.sync(
    process.execPath,
    nodeArgs
      .concat(require.resolve('../scripts/' + script))	// 这一步可以知道通过start参数，实际运行了../script下的start.js脚本
      .concat(args.slice(scriptIndex + 1)),
    { stdio: 'inherit' }
  );
  
  ...
  
}
```

通过react-script.js这段中可以知道start实际运行了../script下的start.js脚本。

## start.js源码分析：

```js
const fs = require('fs');
const chalk = require('react-dev-utils/chalk');	// 终端样式库 chalk
const webpack = require('webpack');
const WebpackDevServer = require('webpack-dev-server');
const clearConsole = require('react-dev-utils/clearConsole');	// 清空colsole
const checkRequiredFiles = require('react-dev-utils/checkRequiredFiles');
const {	// webpackDevServe需要用到的相关函数
  choosePort,  // 选择可用端口
  createCompiler,	// 创建webpack的compiler
  prepareProxy,	// 准备proxy
  prepareUrls,	// 准备项目启动url
} = require('react-dev-utils/WebpackDevServerUtils');
const openBrowser = require('react-dev-utils/openBrowser'); // 在浏览器打开项目
const semver = require('semver');
const paths = require('../config/paths');
const configFactory = require('../config/webpack.config'); // 获取webpack.config.js的内容
const createDevServerConfig = require('../config/webpackDevServer.config');
const getClientEnvironment = require('../config/env');
const react = require(require.resolve('react', { paths: [paths.appPath] }));
```

从引入中可以见到很多函数都是从react-dev-utils中取过来的，因此react-dev-util这个脚本内的方法都很重要，可以抽空仔细研究，这里我们先只讲那些start.js依赖所用到的一些。

```js

// check index.html 和 index.js是否存在，不存在直接掐断进程
if (!checkRequiredFiles([paths.appHtml, paths.appIndexJs])) {
  process.exit(1);
}
// 默认host 和 端口号
const DEFAULT_PORT = parseInt(process.env.PORT, 10) || 3000;
const HOST = process.env.HOST || '0.0.0.0';

if (process.env.HOST) {
  ...
}
```

```js
const { checkBrowsers } = require('react-dev-utils/browsersHelper');
checkBrowsers(paths.appPath, isInteractive)
  .then(() => {
    return choosePort(HOST, DEFAULT_PORT);	// 选择端口
  })
  .then(port => {
    if (port == null) {
      return;
    }

    const config = configFactory('development');	// 获取../config/webpack.config.js下的webpack配置
    const protocol = process.env.HTTPS === 'true' ? 'https' : 'http';
    const appName = require(paths.appPackageJson).name;

    const useTypeScript = fs.existsSync(paths.appTsConfig);	// 是否要加载tsConfig.json
    const urls = prepareUrls(	// 获取url配置
      protocol,
      HOST,
      port,
      paths.publicUrlOrPath.slice(0, -1)
    );
    const compiler = createCompiler({	// 创建complier实例
      appName,
      config,
      urls,
      useYarn,
      useTypeScript,
      webpack,
    });
    const proxySetting = require(paths.appPackageJson).proxy; // 获取package.json的proxy代理地址
    const proxyConfig = prepareProxy(	// 获取proxy配置
      proxySetting,
      paths.appPublic,
      paths.publicUrlOrPath
    );
    const serverConfig = {	// webpack-server配置
      ...createDevServerConfig(proxyConfig, urls.lanUrlForConfig), // ../config/webpackDevServer.config
      host: HOST,
      port,
    };
    const devServer = new WebpackDevServer(serverConfig, compiler);	// WebpackDevServer实例
    // Launch WebpackDevServer.
    devServer.startCallback(() => {
      if (isInteractive) {
        clearConsole();
      }

      if (env.raw.FAST_REFRESH && semver.lt(react.version, '16.10.0')) {
        console.log(
          chalk.yellow(
            `Fast Refresh requires React 16.10 or higher. You are using React ${react.version}.`
          )
        );
      }

      console.log(chalk.cyan('Starting the development server...\n'));
      openBrowser(urls.localUrlForBrowser);
    });

    ['SIGINT', 'SIGTERM'].forEach(function (sig) {
      process.on(sig, function () {
        devServer.close();
        process.exit();
      });
    });

    if (process.env.CI !== 'true') {
      // Gracefully exit when stdin ends
      process.stdin.on('end', function () {
        devServer.close();
        process.exit();
      });
    }
  })
  .catch(err => {
    if (err && err.message) {
      console.log(err.message);
    }
    process.exit(1);
  });
```



## react-dev-utils/chalk

这个方法没什么，其实就是调了下chalk库，一个终端文字样式处理库

```js
var chalk = require('chalk');

module.exports = chalk;
```

## react-dev-utils/clearConsole

通过向控制台输入清屏指令，来实现清屏，其中**“\x1B”**是<ESC>的ascll码，然后根据环境拼接上不同的清屏指令。

```js

function clearConsole() {
  process.stdout.write(
    process.platform === 'win32' ? '\x1B[2J\x1B[0f' : '\x1B[2J\x1B[3J\x1B[H'
  );
}

module.exports = clearConsole;
```

## react-dev-utils/WebpackDevServerUtils.js

### choosePort函数：

通过detect函数（detect-port-alt库）来选择可用端口

```js
function choosePort(host, defaultPort) {
  return detect(defaultPort, host).then(	// 选择可用端口
    port =>
      new Promise(resolve => {
        if (port === defaultPort) {
          return resolve(port);
        }
        const message =	// 判断是否是win32位系统 端口是否 < 1024 是否以root身份运行
          process.platform !== 'win32' && defaultPort < 1024 && !isRoot()
            ? `Admin permissions are required to run a server on a port below 1024.`
            : `Something is already running on port ${defaultPort}.`;
        if (isInteractive) {
          clearConsole();
          // 若当前端口存在活动进程，存放当前端口活动进程信息
          const existingProcess = getProcessForPort(defaultPort);	
          const question = {
            type: 'confirm',
            name: 'shouldChangePort',
            message:
              chalk.yellow(
                message +
                  `${existingProcess ? ` Probably:\n  ${existingProcess}` : ''}`
              ) + '\n\nWould you like to run the app on another port instead?',
            initial: true,
          };
          prompts(question).then(answer => {
            if (answer.shouldChangePort) {
              resolve(port); // 选择y,则返回选择端口
            } else {
              resolve(null);	// 选择n,则返回null
            }
          });
        } else {
          console.log(chalk.red(message));
          resolve(null);
        }
      }),
    err => {	// 无可用端口 报错
      throw new Error(
        chalk.red(`Could not find an open port at ${chalk.bold(host)}.`) +
          '\n' +
          ('Network error message: ' + err.message || err) +
          '\n'
      );
    }
  );
}
```

### react-dev-utils/getProcessForPort

```js
function getProcessForPort(port) {
  try {
    var processId = getProcessIdOnPort(port);	// 获取当前进程号
    var directory = getDirectoryOfProcessById(processId);	// 获取当前进程所在的文件夹目录
    var command = getProcessCommand(processId, directory);	// 获取当前进程指令
    return (
      chalk.cyan(command) +		
      // /usr/local/bin/node /Users/bk/Desktop/学习资料/electron/electron-react-app/node_modules/react-scripts/scripts/start.js
      chalk.grey(' (pid ' + processId + ')\n') +
      // (pid 43979)
      chalk.blue('  in ') +
      // in
      chalk.cyan(directory)
      // /Users/bk/Desktop/学习资料/electron/electron-react-app
    );
  } catch (e) {
    return null;
  }
}
```

#### getProcessCommand函数：

ps指令用于报告当前活动的进程 

-o command 获取指定列command(说明) 

-p processId 指定进程

```js
function getProcessCommand(processId, processDirectory) {
  var command = execSync(
    'ps -o command -p ' + processId + ' | sed -n 2p',	// 通过进程id获得服务名称
    execOptions
  );

  command = command.replace(/\n$/, '');

  ...
}
```

### createCompiler函数：

```js
function createCompiler({
  appName,
  config,
  urls,
  useYarn,
  useTypeScript,
  webpack,
}) {
  let compiler;
  try {
    compiler = webpack(config);	// 创建compiler实例
  } catch (err) {
    console.log(chalk.red('Failed to compile.'));
    console.log();
    console.log(err.message || err);
    console.log();
    process.exit(1);
  }

  compiler.hooks.invalid.tap('invalid', () => { // 监听文件改变
    if (isInteractive) {
      clearConsole();
    }
    console.log('Compiling...');
  });

  let isFirstCompile = true;	// 是否首次编译
  let tsMessagesPromise;			

  if (useTypeScript) {	// 处理tsconfig.json
    forkTsCheckerWebpackPlugin
      .getCompilerHooks(compiler)
      .waiting.tap('awaitingTypeScriptCheck', () => {
        console.log(
          chalk.yellow(
            'Files successfully emitted, waiting for typecheck results...'
          )
        );
      });
  }

  compiler.hooks.done.tap('done', async stats => {	// 编译完成回调
    if (isInteractive) {
      clearConsole();
    }

   	// 处理warning与errors，使其可以被展示出来
    const statsData = stats.toJson({
      all: false,
      warnings: true,
      errors: true,
    });

    const messages = formatWebpackMessages(statsData);
    const isSuccessful = !messages.errors.length && !messages.warnings.length;
    if (isSuccessful) {
      console.log(chalk.green('Compiled successfully!'));
    }
    if (isSuccessful && (isInteractive || isFirstCompile)) {
      printInstructions(appName, urls, useYarn);
    }
    isFirstCompile = false;

    // 如果error存在，则只展示error
    if (messages.errors.length) {
      // 只保留第一条error
      if (messages.errors.length > 1) {
        messages.errors.length = 1;
      }
      console.log(chalk.red('Failed to compile.\n'));
      console.log(messages.errors.join('\n\n'));
      return;
    }

    // 如果error不存在，则展示warnings
    if (messages.warnings.length) {
      console.log(chalk.yellow('Compiled with warnings.\n'));
      console.log(messages.warnings.join('\n\n'));

      // eslint取消的tips
      console.log(
        '\nSearch for the ' +
          chalk.underline(chalk.yellow('keywords')) +
          ' to learn more about each warning.'
      );
      console.log(
        'To ignore, add ' +
          chalk.cyan('// eslint-disable-next-line') +
          ' to the line before.\n'
      );
    }
  });

  // You can safely remove this after ejecting.
  // We only use this block for testing of Create React App itself:
  const isSmokeTest = process.argv.some(
    arg => arg.indexOf('--smoke-test') > -1
  );
  if (isSmokeTest) {
    ...
  }

  return compiler;
}
```

### prepareProxy函数：

创建proxyConfig实例

```js
function prepareProxy(proxy, appPublicFolder, servedPathname) {
  // `proxy` 对特殊请求设置代理服务器
  if (!proxy) {
    return undefined;
  }
  if (typeof proxy !== 'string') {	// proxy参数必须是个string
    console.log(
      chalk.red('When specified, "proxy" in package.json must be a string.')
    );
    console.log(
      chalk.red('Instead, the type of "proxy" was "' + typeof proxy + '".')
    );
    console.log(
      chalk.red('Either remove "proxy" from package.json, or make it a string.')
    );
    process.exit(1);
  }

  const sockPath = process.env.WDS_SOCKET_PATH || '/ws';
  const isDefaultSockHost = !process.env.WDS_SOCKET_HOST;
  function mayProxy(pathname) {
    ...
  }

  if (!/^http(s)?:\/\//.test(proxy)) { // proxy必须以http或https开头
    console.log(
      chalk.red(
        'When "proxy" is specified in package.json it must start with either http:// or https://'
      )
    );
    process.exit(1);
  }

  let target;
  if (process.platform === 'win32') {
    target = resolveLoopback(proxy);
  } else {
    target = proxy;
  }
  return [
    {
      target,
      logLevel: 'silent',
      context: function (pathname, req) { //非GET请求 或 header中accept的类型不包含“text/html”
        return (
          req.method !== 'GET' ||
          (mayProxy(pathname) &&
            req.headers.accept &&
            req.headers.accept.indexOf('text/html') === -1)
        );
      },
      onProxyReq: proxyReq => { // 防止跨域，将header中的origin设置为代理地址
        if (proxyReq.getHeader('origin')) {
          proxyReq.setHeader('origin', target);
        }
      },
      onError: onProxyError(target),
      secure: false,
      changeOrigin: true,
      ws: true,
      xfwd: true,
    },
  ];
}

```

### prepareUrls函数：

```js
function prepareUrls(protocol, host, port, pathname = '/') {
  const formatUrl = hostname =>
    url.format({
      protocol,
      hostname,
      port,
      pathname,
    });	// 生成url
  const prettyPrintUrl = hostname =>
    url.format({
      protocol,
      hostname,
      port: chalk.bold(port),
      pathname,
    });	// 生成美观的url （控制台展示的port端口号加粗）

  const isUnspecifiedHost = host === '0.0.0.0' || host === '::';
  let prettyHost, lanUrlForConfig, lanUrlForTerminal;
  if (isUnspecifiedHost) {
    prettyHost = 'localhost';
    try {
      lanUrlForConfig = address.ip(); // 获取本机ip地址
      if (lanUrlForConfig) {
        // 检查是否是私有ip 10.开头 或 172.开头 或 192.168开头
        // A类 10.0.0.0 --10.255.255.255  B类 172.16.0.0--172.31.255.255  C类 192.168.0.0--192.168.255.255
        if (
          /^10[.]|^172[.](1[6-9]|2[0-9]|3[0-1])[.]|^192[.]168[.]/.test(lanUrlForConfig)
        ) {
          lanUrlForTerminal = prettyPrintUrl(lanUrlForConfig); // 美化url
        } else { // 非私有的ip直接弃置
          lanUrlForConfig = undefined;
        }
      }
    } catch (_e) {
      // ignored
    }
  } else {
    prettyHost = host;
  }
  const localUrlForTerminal = prettyPrintUrl(prettyHost);
  const localUrlForBrowser = formatUrl(prettyHost);
  return {
    lanUrlForConfig,	// 本机ip
    lanUrlForTerminal,	// ip地址加端口号
    localUrlForTerminal,	// 本地终端url加端口号
    localUrlForBrowser,	// 本地浏览器url加端口号
  };

```

## 检测 browserslist配置

在start.js中可以发现，在做完一系列系统文件参数是否缺失的判断后，会执行一个叫checkBrowsers的函数，这个函数用来检测 browserslist 的配置文件是否存在，只有**存在browserslist**才会让进程进行下去

browserslist配置智能添加css前缀，jspolyfill垫片来兼容各个浏览器。

### 源码分析：

#### react-dev-utils/browsersHelper.js

```js
const defaultBrowsers = {		// 默认的browserslist配置
  production: ['>0.2%', 'not dead', 'not op_mini all'],
  development: [
    'last 1 chrome version',
    'last 1 firefox version',
    'last 1 safari version',
  ],
};
```

```js

function shouldSetBrowsers(isInteractive) {	// 是否需要设置browserslist
  if (!isInteractive) {		// process.stdout.isTTY === true
    return Promise.resolve(true);
  }

  const question = {
    type: 'confirm',
    name: 'shouldSetBrowsers',
    message:
      chalk.yellow("We're unable to detect target browsers.") +
      `\n\nWould you like to add the defaults to your ${chalk.bold(
        'package.json'
      )}?`,
    initial: true,
  };
	// 如果不存在browserslist配置，会让用户选择是否注入默认browserslist配置，返回一个promise
  return prompts(question).then(answer => answer.shouldSetBrowsers);
}
```

```js
function checkBrowsers(dir, isInteractive, retry = true) {
  // 利用browserslist库，会检测process.env.BROWSERSLIST、process.env.BROWSERSLIST_CONFIG、browserslist、.browserslistrc、package.json、自定义文件路径下是否存在配置
  const current = browserslist.loadConfig({ path: dir });
  if (current != null) {
    return Promise.resolve(current);
  }

  if (!retry) {
    return Promise.reject(
      new Error(
        chalk.red(
          'As of react-scripts >=2 you must specify targeted browsers.'
        ) +
          os.EOL +
          `Please add a ${chalk.underline(
            'browserslist'
          )} key to your ${chalk.bold('package.json')}.`
      )
    );
  }

  return shouldSetBrowsers(isInteractive).then(shouldSetBrowsers => {
    if (!shouldSetBrowsers) {	// 选N，直接掉checkBrowsers，会报错
      return checkBrowsers(dir, isInteractive, false);
    }

    return (
      pkgUp({ cwd: dir })	// 利用pkg-up库操作package.json的内容
        .then(filePath => {
          if (filePath == null) {
            return Promise.reject();
          }
          // 设置默认的browserslist
          const pkg = JSON.parse(fs.readFileSync(filePath));
          pkg['browserslist'] = defaultBrowsers;
          fs.writeFileSync(filePath, JSON.stringify(pkg, null, 2) + os.EOL);

          browserslist.clearCaches();
          console.log(
            `${chalk.green('Set target browsers:')} ${chalk.cyan(
              defaultBrowsers.join(', ')
            )}`
          );
          console.log();
        })
        // Swallow any error
        .catch(() => {})
        .then(() => checkBrowsers(dir, isInteractive, false))
    );
  });
}
```



## React环境变量

项目的根目录添加一系列名为 .env的文件，里面写上变量名和值，打包后，可以在js代码中通过process.env.REACT_APP_XXX读取到对应文件中的变量值。
 注：**文件中的变量必须以REACT_APP_ 开头**，其他的react不识别。

### 多环境设置：

默认，可以在项目根目录下建立如下文件：

- .env：默认。
- .env.local：本地覆盖。除 test 之外的所有环境都加载此文件。
- .env.development, .env.test, .env.production：设置特定环境。
- .env.development.local, .env.test.local, .env.production.local：设置特定环境的本地覆盖。

左侧的文件比右侧的文件具有更高的优先级：

- npm start： .env.development.local, .env.development, .env.local, .env
- npm run build： .env.production.local, .env.production, .env.local, .env
- npm test： .env.test.local, .env.test, .env (注意没有 .env.local )

注：实际测试发现添加完.env文件后，需要重新执行npm start后，代码中获取变量才能生效。

### 源码分析：

#### env.js

```js
// [
//	.env.${NODE_ENV}.local, 
//	.env.local, 
//	.env.${NODE_ENV}, 
//	.env 
// ]
// 环境变量文件名列表
const dotenvFiles = [
  `${paths.dotenv}.${NODE_ENV}.local`,
  // Don't include `.env.local` for `test` environment
  // since normally you expect tests to produce the same
  // results for everyone
  NODE_ENV !== 'test' && `${paths.dotenv}.local`,
  `${paths.dotenv}.${NODE_ENV}`,
  paths.dotenv,
].filter(Boolean);

// 。。。

// 环境变量注入
// 会先判断环境变量文件是否存在，存在再去进行注入
dotenvFiles.forEach(dotenvFile => {
  if (fs.existsSync(dotenvFile)) {
    require('dotenv-expand')(
      require('dotenv').config({
        path: dotenvFile,
      })
    );
  }
});

```

**dotenv-expand**: 可以处理动态字符串 例如：

```
MONGODB_USERNAME=testuserMONGODB_PASSWORD=testpasswordMONGODB_HOST=localhostMONGODB_PORT=27017MONGODB_SERVERNAME=my-mongodb-serverMONGODB_URI=mongodb://${MONGODB_USERNAME}:${MONGODB_PASSWORD}@${MONGODB_HOST}:${MONGODB_PORT}/${MONGODB_SERVERNAME}
```

**dotenv**: 对.env文件中的环境变量进行预加载

#### config/env.js

```js
// 抓去NODE_ENV和以REACT_APP_开头的环境变量并准备通过webpack.DefinePlugin在webpack配置中注入
const REACT_APP = /^REACT_APP_/i;

function getClientEnvironment(publicUrl) {
  const raw = Object.keys(process.env)
    .filter(key => REACT_APP.test(key))
    .reduce(
      (env, key) => {
        env[key] = process.env[key];
        return env;
      },
      {
        NODE_ENV: process.env.NODE_ENV || 'development',
        PUBLIC_URL: publicUrl,
        WDS_SOCKET_HOST: process.env.WDS_SOCKET_HOST,
        WDS_SOCKET_PATH: process.env.WDS_SOCKET_PATH,
        WDS_SOCKET_PORT: process.env.WDS_SOCKET_PORT,
        FAST_REFRESH: process.env.FAST_REFRESH !== 'false',
      }
    );
  const stringified = {
    'process.env': Object.keys(raw).reduce((env, key) => {
      env[key] = JSON.stringify(raw[key]);
      return env;
    }, {}),
  };
  return { raw, stringified };
}
```

#### config/webpack.config.js

```js
  const env = getClientEnvironment(paths.publicUrlOrPath.slice(0, -1));
// 。。。
// Makes some environment variables available to the JS code, for example:
      // if (process.env.NODE_ENV === 'production') { ... }. See `./env.js`.
      // It is absolutely essential that NODE_ENV is set to production
      // during a production build.
      // Otherwise React will be compiled in the very slow development mode.
      new webpack.DefinePlugin(env.stringified),	// webpack.DefinePlugin定义环境变量
```

