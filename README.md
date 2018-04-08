# Easy Node Authentication

Code for the entire scotch.io tutorial series: Complete Guide to Node Authentication

We will be using Passport to authenticate users locally, with Facebook, Twitter, and Google.

#### Upgraded To Express 4.0
This tutorial has been upgraded to use ExpressJS 4.0. See [the commit](https://github.com/scotch-io/easy-node-authentication/commit/020dea057d5a0664caaeb041b18978237528f9a3) for specific changes.

## Instructions

If you would like to download the code and try it for yourself:

1. Clone the repo: `git clone git@github.com:scotch-io/easy-node-authentication`
2. Install packages: `npm install`
3. Change out the database configuration in config/database.js
4. Change out auth keys in config/auth.js
5. Launch: `node server.js`
6. Visit in your browser at: `http://localhost:8080`

## The Tutorials

- [Getting Started and Local Authentication](http://scotch.io/tutorials/easy-node-authentication-setup-and-local)
- [Facebook](http://scotch.io/tutorials/easy-node-authentication-facebook)
- [Twitter](http://scotch.io/tutorials/easy-node-authentication-twitter)
- [Google](http://scotch.io/tutorials/easy-node-authentication-google)
- [Linking All Accounts Together](http://scotch.io/tutorials/easy-node-authentication-linking-all-accounts-together)



## 筆記
https://nodejust.com/nodejs-passport-auth-tutorial/zh-cn/
https://andyyou.github.io/2017/04/11/express-passport/
先去 Facebook Developers開啟一個app, 得到App ID and App Secret

```
var passport = require('passport')
  , FacebookStrategy = require('passport-facebook').Strategy;

passport.use(new FacebookStrategy({
    clientID: FACEBOOK_APP_ID,
    clientSecret: FACEBOOK_APP_SECRET,
    callbackURL: "http://www.example.com/auth/facebook/callback"
  },
  function(accessToken, refreshToken, profile, done) {
    User.findOrCreate(..., function(err, user) {
      if (err) { return done(err); }
      done(null, user);
    });
  }
));
```
- clientID, clientSecret等認證通過後facebook會回傳 accessToken, refreshToken, and profile arguments.
-  profile就是使用者的資料
- callbackURL要跟facebook app的redirect一樣


### Permission 許可
- scope表示取得使用者的資料範圍
```
app.get('/auth/facebook',
  passport.authenticate('facebook', { scope: 'read_stream' })
);

app.get('/auth/facebook',
  passport.authenticate('facebook', { scope: ['read_stream', 'publish_actions'] })
);
```




### auth.js
- 儲存clientID, clientSecret等
- 用在passport.js下面的程式碼
```
passport.use(new FacebookStrategy({
    clientID: FACEBOOK_APP_ID,
    clientSecret: FACEBOOK_APP_SECRET,
    callbackURL: "http://www.example.com/auth/facebook/callback"
  }
```


### user.js
- 統一不同登入方法的資料格式以及加密方式
- 供passport.js儲存從facebook得到的使用者資料


### database.js
- 含database的網址
- 供server.js 連接資料庫


### passport.js
- 以物件的方式定義passport的各種方法
- 當server.js調用
```
require('./app/routes.js')(app, passport)
```
物件passport就被丟進去當作function的參數



### routes.js
- 路由
-  The first route redirects the user to Facebook.
- The second route is the URL to which Facebook will redirect the user after they have logged in.
```
// Redirect the user to Facebook for authentication.  When complete,
// Facebook will redirect the user back to the application at
//     /auth/facebook/callback
app.get('/auth/facebook', passport.authenticate('facebook'));

// Facebook will redirect the user to this URL after approval.  Finish the
// authentication process by attempting to obtain an access token.  If
// access was granted, the user will be logged in.  Otherwise,
// authentication has failed.
app.get('/auth/facebook/callback',
  passport.authenticate('facebook', { successRedirect: '/',
                                      failureRedirect: '/login' }));
```


### passport.intialize()
- passport初始化
- passport.intialize() initialises authentication module.


### passport.session()
-  alters the request object and change the 'user' value that is currently the session id (from the client cookie) into the true deserialised user object.


### serializeUser 和 deserializeUser
- 轉換資料格式
- passport 需要把從 session 取得的資料轉換成帳號資料即 User 的物件實例。


### 建立 mongoose 資料模型
app/models/user.js
```
var mongoose = require('mongoose');
var Schema = mongoose.Schema;
var UserSchema = new Schema({
  username: String,
  password: String,
  firstname: String,
  lastname: String
});
module.exports = mongoose.model('User', UserSchema);
```

### 設定 MongoDB
server.js
```
var mongoose = require('mongoose');
var config = require('./db');
mongoose.connect(process.env.DB_URL);
```


### 設定 passport
server.js
```
var passport = require('passport');
var session = require('express-session');

require('./config/passport')(passport); //  把passport當參數丟進去./config/passport export的function
app.use(session({
    secret: process.env.SESSION_SECRET, // session secret
    resave: true,
    saveUninitialized: true
    // store: new MongoStore({
    //   url: process.env.DB_URL,
    //   ttl: 14 * 24 * 60 * 60 // = 14 days. Default
    // })
}));


app.use(passport.initialize())
app.use(passport.session())
```
- 加上 session 的設定是因為我們希望用戶登入後的相關資料可以透過 session 被保留且一致，所以我們才安裝了 express-session 來幫我們處理這部分。
-  store是指可以把session存在ＤＢ



config/passport.js
```
var LocalStrategy = require('passport-local').Strategy
var FacebookStrategy = require('passport-facebook').Strategy;

module.exports = function(passport){
  passport.use('local-login', new LocalStrategy({
    參數物件
    passReqToCallback = true;
  },
  callback(){}
  ));
  passport.use(new FacebookStrategy({
    參數物件
  },
  function(req, token, refreshToken, profile, done) {}
  ));
}
```
- passport 可以 use 多個策略，接著只要在對應的路由使用 passport.authenticate('<strategy-name>') 即可如下
- 沒有 passReqToCallback 的話，第一個參數就不是 req
- 最後一個參數 done 就是我們用來告訴 passport 模組登入結果的方法，假如登入失敗那麼第一個參數需要傳入錯誤訊息，第二個參數則是 false。登入成功的話，第一個參數則傳入 null，第二個則是任何為 真 的值，這個值會被加到 req 物件上以便我們後續使用。

router.js
```
app.post('/login', passport.authenticate('local-login', {
    successRedirect : '/profile', // redirect to the secure profile section
    failureRedirect : '/login', // redirect back to the signup page if there is an error
    failureFlash : true // allow flash messages
}));


app.get('/auth/facebook', passport.authenticate('facebook', {
  scope : ['public_profile', 'email']
}));
```


### 密碼加密
```
// checking if password is valid
userSchema.methods.validPassword = function(password) {
    return bcrypt.compareSync(password, this.local.password);
};
```

### Facebook strategy
```
passport.use(new FacebookStrategy(fbStrategy,
// facebook will send back the token and profile
function(req, token, refreshToken, profile, done) {
    // asynchronous
    process.nextTick(function() {
      console.log('passport req.user: ', req.user);  
      console.log('passport profile ', profile);
        // 檢查是否登入  check if the user is already logged in
        if (!req.user) {
        // 尚未登入
            User.findOne({ 'facebook.id' : profile.id }, function(err, user) {
              console.log('passport, User.findOne, err: ', err);
              console.log('passport, User.findOne, user: ', user);
                if (err)
                    return done(err);
                // if the user is found, then log them in
                if (user) {
                    // 資料庫找到user但沒有token
                    // if there is a user id already but no token (user was linked at one point and then removed)
                    if (!user.facebook.token) {
                        user.facebook.token = token;
                        user.facebook.name  = profile.name.givenName + ' ' + profile.name.familyName;
                        user.facebook.email = (profile.emails[0].value || '').toLowerCase();
                        console.log('資料庫的user.facebook.token失效，重新存入user資料 存入user資料 user.save, user : ', user);
                        user.save(function(err) {
                            if (err)
                                return done(err);
                            return done(null, user);
                        });
                    }
                    return done(null, user); // user found, return that user
                } else {
                  // 查無user 建立新user
                    // if there is no user, create them
                    var newUser            = new User();
                    newUser.facebook.id    = profile.id;
                    newUser.facebook.token = token;
                    newUser.facebook.name  = profile.name.givenName + ' ' + profile.name.familyName;
                    newUser.facebook.email = (profile.emails[0].value || '').toLowerCase();
                    console.log('資料庫沒有user資料，存入newUser資料 newUser.save, newUser : ', newUser);
                    newUser.save(function(err) {
                        if (err)
                            return done(err);

                        return done(null, newUser);
                    });
                }
            });

        } else {
            // user already exists and is logged in, we have to link accounts
            var user            = req.user; // pull the user out of the session
            user.facebook.id    = profile.id;
            user.facebook.token = token;
            user.facebook.name  = profile.name.givenName + ' ' + profile.name.familyName;
            user.facebook.email = (profile.emails[0].value || '').toLowerCase();
            // console.log('else user.facebook: ', user.facebook);
            console.log('資料庫有user資料，重新存入user資料 user.save, user : ', user);
            user.save(function(err) {
                if (err)
                    return done(err);
                return done(null, user);
            });

        }
    });

}));
```


```
passport profile  {
  id: '',
  username: undefined,
  displayName: undefined,
  name: { familyName: '',
    givenName: '',
    middleName: undefined
  },
  gender: undefined,
  profileUrl: undefined,
  emails: [ { value: '' } ],
  provider: 'facebook',
  _raw: '{"id":"","email":"","last_name":"","first_name":""}',
  _json: {
    id: '',
    email: '',
    last_name: '',
    first_name: ''
  }
}


passport, User.findOne err:  null


passport, User.findOne user:  {
  _id: ,
  __v: 0,
  facebook: {
    email: '',
    name: '',
    token: '',
    id: ''
  }
}


serializeUser, user:  {
  _id: ,
  __v: 0,
  facebook: {
    email: '',
    name: '',
    token: '',
    id: ''
  }
}
```
