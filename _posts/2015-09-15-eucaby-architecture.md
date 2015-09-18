---
layout: post
title: Eucaby Architecture
tags: geo, messenger, appengine, python, flask, ionic, angularjs
image: eucaby_architecture/index.png
blogfeed: true
---

Eucaby consists of three separate applications. The core is Eucaby API which serves messages between users and integrates external web services. Mobile application is the main client which sends and receives user location messages. Eucaby website provides general information and allows to view messages sent to email address.

{{ more }}

![Eucaby Architecture][img-architecture]

Both Eucaby API and Eucaby website are running on Google AppEngine which provides 

# Eucaby API

## Authentication

### Authentication Components

![Eucaby Authentication][img-authentication]

### Authentication Flow Diagram

[Facebook Access Tokens][fb-access-tokens]

## Endpoints

Originally I planned to use endpoints.

## Push Notifications


# Mobile Application

Mobile application is implemented with the popular [Ionic][ionic] framework which is one layer on the top of [PhoneGap][phonegap]. It is really easy to use if you get along with AngularJS and allows to generate native code for several mobile platforms out of the box. It supports plugin architecture so you can use pretty much any mobile native functionality with a simple javascript interface. I used a few plugins including [PushPlugin][pushplugin] for receiving push notifications from **GCM** or **APNs** and [Geolocation][geolocation] for accurate geographic location. It is perfect for rapid prototyping and conveying ideas. 

I noticed a few disadvantages building mobile application with Ionic compared to native application:
* Larger memory footprint. PhoneGap runs embedded browser. 
* Slower. 
* Quirky. Some interactions look unusual. 

AngularJS module which communicates with Eucaby API application is `EucabyApi`. It uses [OpenFB][openfb] module to authenticate with Facebook OAuth. Mobile application doesn't use Facebook Graph API directly though. Here is the snippet of the code for `EucabyApi` module:

```
angular.module('eucaby.api', [
    'openfb',
    'eucaby.utils',
    'eucaby.config'
])

.factory('EucabyApi', [
    '$http',
    '$q',
    'OpenFB',
    'utils',
    'storageManager',
    'config',
function ($http, $q, OpenFB, utils, storageManager, config) {

    return {
        init: function(){
            OpenFB.init(config.FB_APP_ID,
                        'http://localhost:8100/oauthcallback.html');
        },
        login: function(force){
            var deferred = $q.defer();
            var accessToken = storageManager.getAccessToken();
            if (accessToken && !force) {
                deferred.resolve(accessToken);
            } else {
                /*
                * Get short-lived Facebook access token
                * Get Facebook user profile with '/me' request
                * Get Eucaby access token
                */
                // ...

                OpenFB.login('email,user_friends,public_profile')
                    .then(function(){
                        // ...
                    });
            }
            return deferred.promise;
        },
        api: function(obj){
            /*
            * Ensure that Eucaby access token exists
            * If Eucaby access token is expired refresh it
            * Make API request
            */
        
            var self = this;
            var method = obj.method || 'GET';
            var params = obj.params || {};
            var data = obj.data && utils.toPostData(obj.data) || '';
            var deferred = $q.defer();
            
            // ...
            
            var apiRequest = function(method, path, token, params){
                return $http({
                    method: method, url: config.EUCABY_API_ENDPOINT + path,
                    params: params, data: data,
                    headers: {'Authorization': 'Bearer ' + token}});
            };
            
            var makeApiRequest = function(data){
                // ...
                apiRequest(method, obj.path, token, params)
                    .success(function(data){
                        deferred.resolve(data);
                    });
            };

            self.login().then(makeApiRequest);
            return deferred.promise;
        },
        logout: function(){
            // ...
            OpenFB.logout();
        }
    };
}]);
```

# Eucaby Website

Eucaby website application not only displays general information but also allows users to view or send location message back to the sender. It is implemented in Django and uses some shared resources with Eucaby API.

[img-index]: /img/eucaby_architecture/index.png
[img-authentication]: /img/eucaby_architecture/authentication.png
[img-authentication-flow]: /img/eucaby_architecture/authentication-flow.png
[img-architecture]: /img/eucaby_architecture/architecture.png
[fb-access-tokens]: https://developers.facebook.com/docs/facebook-login/access-tokens
[ionic]: http://ionicframework.com/
[phonegap]: http://phonegap.com/
[pushplugin]: https://github.com/phonegap-build/PushPlugin
[geolocation]: https://github.com/apache/cordova-plugin-geolocation
[openfb]: https://github.com/ccoenraets/OpenFB
[endpoints]: https://cloud.google.com/appengine/docs/python/endpoints/

