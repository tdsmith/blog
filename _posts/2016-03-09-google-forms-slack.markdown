---
title: Integrating Google Forms and Slack
layout: post
---

Getting submissions to a Google Form to appear in a Slack chat is straightforward. You can use services like [Zapier](https://zapier.com/zapbook/google-sheets/slack/) to do this, but it turns out to be easy to use Google's scripting interface to get instant message-pushing without a third-party tool.

This approach is lifted directly from an [Obake Labs](http://www.obakelabs.com/push-to-slack-channel-on-google-form-submission/) blog post, but the code didn't quite work off the shelf for me.

To implement:

1. Set up a Google Form and link it to a spreadsheet.
1. Set up a [Bot User](https://slack.com/apps/A0F7YS25R-bots) in Slack and /invite it to the channel you want it to speak in. Make a note of the key.
1. Go to the response spreadsheet and open the Script Editor (Tools -> Script editor).
1. Paste in the following code, replacing the values as necessary:

{%highlight javascript%}
function SendSlackMessage(e){
  // Only include form values that are not blank
  var user_info="";
  for (var key in e.namedValues) {
    var val = e.namedValues[key][0].toString();
    if (val !== "") {
      user_info += key + ': ' + val + '\n';
    }
  }
  var url = "https://slack.com/api/chat.postMessage";
  var payload =
  {
    "token" : "BOT_TOKEN_HERE",
    "as_user" :"true",
    "text" : "New response received:\n" + user_info,
    "channel" : "#general", // or whatever
    "type" : "post",
  };
  var options =
  {
    "method" : "POST",
    "payload" : payload,
    "followRedirects" : false,
    "muteHttpExceptions": true
  };
   
  //Hit the Slack API with the request
  var result = UrlFetchApp.fetch(url, options);
   
  //Check the request went through and log errors if necessary
  if (result.getResponseCode() == 200) {
   var params = JSON.parse(result.getContentText());
   Logger.log(params);
  }
}
{%endhighlight%}

Finally, go to Resources -> Current project's triggers, and add a trigger to run SendSlackMessage on a form submit response from a spreadsheet. Save your project and you should be in business!

I'm looking for ways to get responses _in_ to Google Forms from Slack in a reasonable manner. [Botkit](https://github.com/howdyai/botkit)'s approach to conversations is the only thing I've come across so far; let me know if you have a better way.
