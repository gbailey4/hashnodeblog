## Create an iOS (and MacOS Big Sur) Widget to show online Call of Duty: Warzone friends

I've recently been very into Call of Duty: Warzone and am constantly logging into their iOS app to see if I have friends online to join. I came across [Scriptable](https://scriptable.app) recently which is a great utility for iOS ([and Mac](https://scriptable.app/mac-beta/)) that allows you to use JavaScript to build native iOS Shortcuts. This includes being able to build a custom widget as well, which is how we will use it for this project. I figured this would be a great first test for this tool to try and list my online friends in a widget so I don't need to log into the Call of Duty app anymore and can - at a glance - see who is online.

You can also see this [GitHub repo](https://github.com/gbailey4/callofdutyscriptable) for all the code from this tutorial.

### Steps to Create Widget

1. Parse your username and password into your script
2. Get the CSRF Token
3. Login with username and password
4. Get list of online friends from the Call of Duty servers
5. Putting it all together and creating the widget

## Parse your username and password into your script

You'll need to get your Activision username and password set in your script. It's not recommended to set these values directly inline (I wouldn't be able to share how to do this if that were so!). To work around that, we'll use the built in functionality that Scriptable has to ingest variables from parameters set in the widget.

To take arguments from a widget you first need to put the arguments in your widget. Create a new script in Scriptable and name it "Online Call of Duty Friends" (or whatever you like!). We'll separate the username and the password with a "|" character. To get your input from the widget and separate out the username and password enter the below JS:

```javascript
    const input = args.widgetParameter;
    // To get the email portion of the string we use this insane RegEx from this website: https://emailregex.com
    const userRe = /(?:[a-z0-9!#$%&'*+\/=?^_`{|}~-]+(?:\.[a-z0-9!#$%&'*+\/=?^_`{|}~-]+)*|"(?:[\x01-\x08\x0b\x0c\x0e-\x1f\x21\x23-\x5b\x5d-\x7f]|\\[\x01-\x09\x0b\x0c\x0e-\x7f])*")@(?:(?:[a-z0-9](?:[a-z0-9-]*[a-z0-9])?\.)+[a-z0-9](?:[a-z0-9-]*[a-z0-9])?|\[(?:(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.){3}(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?|[a-z0-9-]*[a-z0-9]:(?:[\x01-\x08\x0b\x0c\x0e-\x1f\x21-\x5a\x53-\x7f]|\\[\x01-\x09\x0b\x0c\x0e-\x7f])+)\])/
    const username = userRe.exec(input)[0]
    // Then grab the password via the regex below
    const passwordRe = /(\@(.*)\|(.*))/
    const password = passwordRe.exec(input)[3]
```

The above code will grab the argument from the widget and separate out the username - an email - and the password after the "|" character. We now have both inputs required to login in the `username` and `password` variables!

## Get the CSRF Token

I leaned heavily on [this Postman documentation page](https://documenter.getpostman.com/view/7896975/SW7aXSo5) to understand how to interact with the Call of Duty servers. From there you can see that there is CSRF protection on the API. We'll need to first get the CSRF token before logging in. We can do that by creating a asynchronous function to get the token. We use an asynchronous function so that we can wait for the response when we call the function.

```javascript
    async function getCSRF() {
      const url = 'https://profile.callofduty.com/cod/login'
      const r = new Request(url)
      await r.load()
      const csrf = r.response.cookies.find( ({ name }) => name === 'XSRF-TOKEN' ).value;
      return csrf
    }
```

The built in `Request` function in Scriptable makes this easy as we can see. From the function above, we'll hit the login page and get the CSRF token from the cookies set in the response.

## Login with username and password

Now that we've got the CSRF token we can login using the username and password we grabbed from the widget parameters. The parameters we need to get from the response are the `ACT_SSO_COOKIE`, `ACT_SSO_COOKIE_EXPIRY`, and `atkn`. We'll craft the request like a form submit using the Request API from Scriptable again and then grab those values from the response.

```javascript
    async function getLogin(token) {
      const url = 'https://profile.callofduty.com/do_login?new_SiteId=cod'
      const r = new Request(url)
      r.method = 'POST'
      r.addParameterToMultipart('username', username)
      r.addParameterToMultipart('password', password)
      r.addParameterToMultipart('remember_me', 'true')
      r.addParameterToMultipart('_csrf', token)
      r.headers = {Cookie: `XSRF-TOKEN=${token}; new_SiteId=cod;`}
      await r.load()
      const ACT_SSO_COOKIE = r.response.cookies.find( ({ name }) => name === 'ACT_SSO_COOKIE' ).value
      const ACT_SSO_COOKIE_EXPIRY = r.response.cookies.find( ({ name }) => name === 'ACT_SSO_COOKIE_EXPIRY' ).value
      const atkn = r.response.cookies.find( ({ name }) => name === 'atkn' ).value
      const authHeaders = `ACT_SSO_COOKIE=${ACT_SSO_COOKIE}; ACT_SSO_COOKIE_EXPIRY=${ACT_SSO_COOKIE_EXPIRY}; atkn=${atkn};`
      return authHeaders;
    }
```

The above code will create the Request object and then submit the form and grab the necessary values. We'll then return an object with all the needed values to interact with the API as your Call of Duty user.

## Get list of online friends from the Call of Duty servers

Now we have everything we need to request the friends list directly from the Call of Duty API. We'll create another asynchronous function that takes the CSRF token and the authHeaders we generated above and build an array of online friends. If there are no online friends we'll return an array with one value that states there are no online friends.

```javascript
    async function getFriends(csrf, authHeaders) {
      const url = `https://my.callofduty.com/api/papi-client/userfeed/v1/friendFeed/rendered/en/${csrf}`;
      const r = new Request(url);
      r.headers = {Cookie: authHeaders};
      const players = await r.loadJSON();
      const friends = players.data.identities
      const onlineFriends = friends.filter(f => f.status.online === true).map(f => f.username.split('#')[0])
      if (onlineFriends.length === 0) {
        onlineFriends.push('No friends on')
      }
      return onlineFriends;
    }
```

Nothing too crazy going on above, but you'll see that the CSRF is actually used in the URL. A little strange, but easy to work with. The API response is easy since it's a JSON object and we simply grab the online players by their status and map those to an array.

## Putting it all together and creating the widget

Now we'll put it all together by building another asynchronous function that will get the required values from the above functions and build the widget object (complete with a nice gradient). There's also a bit at the top where we build a simple function to format a string to show the time that the widget was last run. This is helpful so you can decide whether you want to manually run the widget to refresh the players online. Finally, we'll call the function putting it all together and put it in a widget. We'll also give the user the ability to manually call the widget and make that possible. Conclude the script with `Script.complete();` to finalize the run.

```javascript
    function formatAMPM(date) {
      let hours = date.getHours();
      let minutes = date.getMinutes();
      let ampm = hours >= 12 ? 'pm' : 'am';
      hours = hours % 12;
      hours = hours ? hours : 12; // the hour '0' should be '12'
      minutes = minutes < 10 ? '0'+minutes : minutes;
      let strTime = hours + ':' + minutes + ' ' + ampm;
      return strTime;
    }
    
    async function generateResponse() {
      const response = {}
      response.widget = new ListWidget();
      const now = await new Date(Date.now());
      const time = `Last run: ${formatAMPM(now)}`;
      const csrf = await getCSRF();
      const authHeaders = await getLogin(csrf);
      const onlineFriends = await getFriends(csrf, authHeaders);
      onlineFriends.map(f => response.widget.addText(f))
      
      console.log(onlineFriends);
      console.log(response)
      
      response.widget.addText("")
      
      let dateText = response.widget.addText(time)
      dateText.font = Font.semiboldSystemFont(11)
      dateText.rightAlignText()
      dateText.textColor  = Color.lightGray()
      
      let gradient = new LinearGradient()
      gradient.locations = [0, 1]
      gradient.colors = [
        new Color("13233F"),
        new Color("141414")
      ]
      response.widget.backgroundGradient = gradient
      
      return response
    }
    
    const presentation = await generateResponse();
    
    if (config.runsInWidget) {
      // The script runs inside a widget, so we pass our instance of ListWidget to be shown inside the widget on the Home Screen.
      Script.setWidget(presentation.widget)
    } else {
      // The script runs inside the app, so we preview the widget.
      presentation.widget.presentSmall()
    }
    
    Script.complete();
```

To create the widget you'll go to your phone and create a widget. Here is [Apple's documentation](https://support.apple.com/en-us/HT207122) on doing so. When you create the widget search for "Scriptable" and select the size you'd like between Small, Medium, and Large. Select the newly created widget and set the parameters. You'll set the Script to the name you used for the script, I used "Online Call of Duty Friends". Then select "Run Widget" for what to do upon clicking the widget. Finally enter your username (email) and password separated by a pipe character "|" in the Parameter field.

You're all done! Enjoy being able to see at a glance online CoD friends!