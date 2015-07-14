# Creating a Velocity App in NodeJS

### Setup the project
- `mkdir velocity-demo`
- `npm init`
- `atom package.json`
- Add engines
  ``` json
  "engines": {"node": "0.12.x", "npm": "2.7.x"}
  ```
- `npm install coffee-script --save`
- `mkdir bin`
- `atom bin/www`
``` javascript
#!/usr/bin/env node
require('coffee-script');
require('coffee-script/register');
require('../app');
```
- Talk about what the #! is doing
- `npm install --save express jade cookie-parser body-parser monsanto-velocity-node`
- `atom app.coffee`
``` coffee-script
  express = require 'express'
  jade = require 'jade'
  cookieParser = require 'cookie-parser'
  bodyParser = require 'body-parser'
  velocity = require 'monsanto-velocity-node'
  path = require 'path'

  app = express()

  appBaseUrl = process.env.appBaseUrl || '/velocity-demo'

  app.use bodyParser.json()
  app.use bodyParser.urlencoded({ extended: false })
  app.use cookieParser()
  app.use "#{appBaseUrl}/services", require './routes/services'

  velocity.listen(app, 3000)
```
- Introduce the Velocity Node module
- `mkdir routes`
- `atom routes/services.coffee`
``` coffee-script
  express = require 'express'
  router = express.Router()

  #Basic GET example
  router.get '/something', (req, res, next) ->
    res.json
      something: 'value'

  router.get '/something/else/:something', (req, res, next) ->
    res.json
      somethingelse: req.params.something

  router.post '/something', (req, res, next) ->
    res.json req.body

  module.exports = router
```
- Run the project for the first time
  - `./bin/www`
  - http://localhost:3000/services/something
  - http://localhost:3000/services/something/else/12345
- Try and make a change and show that nothing happens
- Introduce supervisor
  - `npm install supervisor -g`
- Add some scripts to package.json
``` json
  "scripts": {
      "start": "supervisor -w .,routes,views -e coffee,jade,js bin/www"
  }
```
- `npm start`
- Make a change, show that the changes are available right away


### Setting up the UI layer
- Add the following to app.coffee
``` coffee-script
  app.set 'view engine', 'jade'
  app.use "#{appBaseUrl}/styles", express.static(path.join __dirname, 'public/styles')
  app.use "#{appBaseUrl}/scripts", express.static(path.join __dirname, 'public/scripts')

  velocity.addBootstrap app, "#{appBaseUrl}",
      cdnHost: process.env.cdnHost

  velocity.addPage app,
      appBaseUrl: "#{appBaseUrl}"
      title: 'Velocity Demo'
      templatePath: path.resolve __dirname, './views/page.jade'
      stylesheets: [
          'style.css'
      ]
```
- `mkdir ui`
- `atom ui/main.coffee`
``` coffee-script
  do ->
    bootstrap = window.bootstrap
    require.config bootstrap.requireJsConfiguration
    require ['./startup']
```
- `atom ui/startup.coffee`
``` coffee-script
  define (require) ->
    $ = require 'jquery'
    navigation = require 'tps/navigation'
    $ ->
      navigation.installHeader
          applicationInformation:
              appName: 'Velocity Demo'
      $('body').append '<div>We have an app and the nav bar is now installed</div>'
```
- Run `npm start`
- navigate to http://localhost:3000/velocity-demo but nothing happens

### Time to introduce Gulp
- `npm install -g gulp`
- `npm install --save-dev gulp gulp-coffee gulp-less gulp-sourcemaps gulp-util gulp-supervisor monsanto-velocity-gulp-tools`
- `atom gulpfile.coffee`
``` coffee-script
  gulp = require 'gulp'
  coffee = require 'gulp-coffee'
  gutil = require 'gulp-util'
  sourcemaps = require 'gulp-sourcemaps'
  supervisor = require 'gulp-supervisor'
  require 'coffee-script/register'
  gulp.task 'coffee', ->
    gulp.src ["./ui/**/*.coffee"]
    .pipe sourcemaps.init()
    .pipe coffee(bare: true).on 'error', gutil.log
    .pipe sourcemaps.write()
    .pipe gulp.dest './public/scripts/'

  gulp.task 'supervise', ->
    supervisor options.main or './bin/www',
      args: []
      watch: ['.', 'views', 'routes']
      ignore: ['/ui', '/public']
      pollInterval: 500
      extensions: ['coffee', 'js', 'jade']
      exec: 'node'
      debug: false
      debugBrk: false
      harmony: true
      noRestartOn: false
      forceWatch: false
      quiet: false

  gulpTools = require 'monsanto-velocity-gulp-tools'
  gulp.task 'coffee', gulpTools.coffee()
  gulp.task 'less', gulpTools.less()
  gulp.task 'supervise', gulpTools.supervisor()
  gulp.task 'watch', ['build'], ->
    gulpTools.watch 'less', 'less'
    gulpTools.watch 'coffee', 'coffee'
  gulp.task 'default', ['build', 'watch', 'supervise']
```
- update our start script
``` json
  "scripts": {
      "start": "gulp"
  }
```
- Run the application again
- `gulp`

### Webpack time
- talk about the strugles of mainatining large number of libraries in the CDN
- talk about how there are a lot of really cool modules written for NodeJS that would be great in a UI, but we cannot use them because there isnt a good way to load them (either as globals or as AMD modules)
- introduce webpack as a tool to bridge the two module systems and allow them both to play together in the broswer environment
- `npm install -g webpack`
- `npm install --save-dev webpack coffee-loader`
- `atom webpack.config.js`
``` coffee-script
webpack = require('webpack');
  require('coffee-script/register');
  module.exports = {
      entry: "./ui/startup.coffee",
      output: {
          path: __dirname + '/public/scripts/',
          filename: 'bundle.js',
          libraryTarget: 'umd'
      },
      module: {
          loaders: [
              { test: /\.coffee$/, loader: "coffee" }
          ]
      },
      devtool: "#source-map",
      resolve: {
          extensions: [
              '',
              '.js',
              '.json',
              '.coffee'
          ]
      },
      externals: require('monsanto-velocity-node/externals'),
      plugins: [
          new webpack.optimize.UglifyJsPlugin({minimize: true})
      ]
  };
```
- Explain what all that does
- Modify main.coffee to look for bundle instead of startup
- Time for a new script
``` json
  "start": "./bin/www",
  "build": "gulp build && webpack",
  "start:dev": "localDevelopment=true gulp"
```
- Build then run the application to make sure it's still working
  - `npm run build && npm start`
  - http://localhost:3000/velocity-demo
- Lets get our auto rebuilding going again
- Add the following to app.coffee
``` coffee-script
  if velocity.localDevelopment()
      velocity.proxyVelocityServices(app)
      app.use velocity.webPackDevMiddleware {appBaseUrl}
```

### Introducing our first CommonJS module
- `npm install --save chance`
- `atom ui/addRandomPerson.coffee`
``` coffee-script
  Chance = require 'chance'
  chance = new Chance()

  module.exports = ->
    chance.name(middle: true)
```
- Modify startup.coffee to make use of chance.js as well as to be a CommonJS module
``` coffee-script
  $ = require 'jquery'
  addRandomPerson = require './addRandomPerson'
  navigation = require 'tps/navigation'
  $ ->
    navigation.installHeader
      applicationInformation:
        appName: 'Velocity Demo'

    $('body').append("<div class='person'>#{addRandomPerson()}</div>") for num in [1..10]
```

### Creating a basic test
- `mkdir test`
- `npm install mocha -g`
- `npm install mocha chai --save-dev`
- `atom basicSpec.coffee`
``` coffee-script
  require('chai').should()
  addRandomPerson = require '../ui/addRandomPerson'
  describe 'Add random person', ->
    runTest = ->
      it 'should return a persons name, including their middle name', ->
        addRandomPerson().split(' ').should.have.length 3

    #run it 50 times to see the nyan cat
    runTest() for num in [1..50]
```
- add a test script to our package.json
``` json
  "test": "mocha --compilers coffee:coffee-script/register --recursive --colors --reporter list test",
  "test:dev": "mocha --compilers coffee:coffee-script/register --recursive --colors --watch --reporter nyan test",
```
