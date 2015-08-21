---
layout: post
title: Eucaby&#58; Simple Geo Messenger
tags: geo, messenger, appengine, python, ionic
blogfeed: true
---

Eucaby Architecture

{{ more }}

Eucaby is designed as a simple Geo Messenger that helps you to share and request accurate and authenticated location messages with your friends.

## Eucaby Usage

### Authentication

Eucaby supports Facebook authentication only at the moment. Other types of authentication (G+, Twitter, username/password) will be supported in the future. To login, you click the "Login with Facebook" button and go through the standard Facebook authentication. Eucaby requests only basic information such as user profile and list of friends who use the application.

### Send Location and Request

To send location you clicks "I am here" button and the dialog will be displayed with the optional message field, email field and list of friends. Only your Facebook friends who use the Eucaby application will be in this list. Map in the dialog displays your current location which is relatively accurate depending on your phone settings. It is advised to enable Wi-Fi for a better accuracy. Setting manual location will be supported in the future. 

You can send location either to an arbitrary email address or your friend. Sending location via email is one of the most flexible ways to share your location. Email recipient will receive the link with your current location which is valid for 1 day. She doesn't need to have Eucaby application installed to view the location. You can even send location to your email address to remember where you have been :). Other methods of sharing location will be supported in the future (SMS, Facebook, etc.). Sending location via friend contact will promptly notify your friend about your current location through the application and will not share it on Facebook.

Recent friends or emails will be displayed in recent contacts (up to 3) to make it more convenient for you to find the recent contact. Email input field supports autocomplete, so when you start typing email it will try to autocomplete the email addresses that you used before.

To send request you use "Where are you?" button and in the open dialog you can ask your friend about her current location. She can respond back with the current location either from the application (preferred) or following the link which opens the location form in the browser. Request link will be valid for 1 day. Although recipient can view your location or even send her location back to your request, the requests and initiating locations can only be sent from the Eucaby mobile application. 

... and don't forget to pass a message along :).

### Incoming and Outgoing Messages

Incoming and Outgoing tabs keep track of your location or request messages that you received from (Incoming) or sent to (Outgoing) other users. The most frequently used tab will probably be Incoming tab. When you send location or request message it will be displayed with outline location or bolt icon in Outgoing tab respectively. After user responds with location to your request it will be displayed in Incoming tab with filled location icon. You still can see your original request in Outgoing tab and it will be displayed with filled bolt. Filled icon is a sign that your location or request are complete. So when you see your request complete you don't feel alone :). After you received a request from other user you can send back your current location.

Well, a few more locations for the same request are also possible :). 

### Settings

By default, when someone sends you request or location you will receive an email notification. If you are a popular person and getting too many of them you might want to consider turn off the email notifications from Settings menu.

### Privacy

To use Eucaby you need to allow location access. Eucaby will only use the current location when you send messages to your friends and will not share it with any third part applications.
