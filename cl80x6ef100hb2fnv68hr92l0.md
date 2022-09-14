## Building a LibGen Search API using AWS Lambda

[LibGen](http://libgen.is/) is an online book database (there are other URLs for it, but this is the one I used). This is a very useful tool when doing research and perusing books to buy. Unfortunately, while they have an [ API ](http://libgen.is/json.php), they do not have a programmatic way to search their database. You need to have IDs to access their API, but you can only get them from the LibGen search page. This leads to an unfortunate gap when using the service programmatically. Seeing this, I began to work through how you might reverse engineer an API here and was able to utilize [ AWS Lambdas ](https://aws.amazon.com/lambda/) to accomplish just that. Lambdas are an awesome AWS tool that you can leverage to very quickly create an HTTP endpoint when connecting with [ API Gateway ](https://aws.amazon.com/api-gateway/). Using this, we'll get an API to return LibGen data when queried with a search term.

### Project Outline

1. Create Lambda and API Gateway
2. Query LibGen for search term
3. Extract IDs from results page
4. Use IDs to make LibGen API Request
5. Do whatever you want with the API response

### 1. Create Lambda and API Gateway

Creating a Lambda with an API Gateway attached is very simple. Simply log into AWS and create a Lambda. Choose "Author from Scratch" and name your function. Use Node.js 10.x and click "Create Function". No need to worry about other settings.

Once the function is created, there's one more step to get an HTTP endpoint for your function. Simply click "API Gateway" from the Add triggers list on the left. Scroll down and you'll see some settings to set up, use "Create a new API" and use the "Open" security for now. Click "Add" and then click "Save" in the top right and you've got your endpoint! You can add API Keys or other authentication in the future if you'd like. Your endpoint URL should be where the settings were.

Notice too that you have a starter query in your Lambda. We'll roughly use this outline to structure our code.

### 2. Querying Libgen

LibGen uses a pretty straightforward query parameters to get search results. Due to this, it's straightforward to get the results page. Here is the URL format:

`http://libgen.is/search.php?req={SEARCH_TERM}&lg_topic=libgen&open=0&view=simple&res=25&phrase=1&column=def`

Simply replace the "SEARCH_TERM" piece with your search term and you've got a results page. To do this, I am using a Lambda with NodeJS and leveraging [ Axios ](https://www.npmjs.com/package/axios) to get the results page. The reason I'm doing this is to very quickly get the script deployed and also to learn Serverless style programming.

Here's the code to get the HTTP response:

```javascript
const axios = require("axios");

module.exports.lginfo = async (event) => {
  const params = JSON.parse(event.body);
  const { search } = params;
  const libgenScrape = await axios.get(
    `http://libgen.is/search.php?req=${search}&lg_topic=libgen&open=0&view=simple&res=25&phrase=1&column=def`
  );
};
```

We're not returning anything yet, but you'll notice that we're getting the HTTP page via a Lambda API.

### 3. Extract IDs

Now you've got your HTTP from the results page coming in from the axios.get command. Next, we need to extract the IDs for use in querying the official LibGen API. Unfortunately, there are no easy identifiers to get the LibGen ID. Fortunately, the search results page is pretty simple HTML and is predictable, so we'll use a simple RegEx expression to get the ID values from the results page! (Well, not too simple, it's still RegEx)

```
/(?<=view.php?id=)[0-9][^']*/g
```

Put it all together and we get a string of our IDs!

```javascript
const axios = require("axios");

module.exports.lginfo = async (event) => {
  const params = JSON.parse(event.body);
  const { search } = params;
  const libgenScrape = await axios.get(
    `http://libgen.is/search.php?req=${search}&lg_topic=libgen&open=0&view=simple&res=100&phrase=1&column=def`
  );
  const regex = /(?<=view.php?id=)[0-9][^']*/g;
  const allIds = libgenScrape.data.match(regex);
};
```

### 4. Make LibGen API Request

Now we've got all the pieces together to make a request to LibGen! This is the fun part. First we need to get our IDs that you got in the last step in allIds as a string. Then we make a second request to LibGen this time as a real API request using the string of IDs.

```javascript
const axios = require("axios");

module.exports.lginfo = async (event) => {
  const params = JSON.parse(event.body);
  const { search } = params;
  const libgenScrape = await axios.get(
    `http://libgen.is/search.php?req=${search}&lg_topic=libgen&open=0&view=simple&res=100&phrase=1&column=def`
  );
  const regex = /(?<=view.php?id=)[0-9][^']*/g;
  const allIds = libgenScrape.data.match(regex);
  const idRequestString = allIds.toString();
  const libgenApiResponse = await axios.get(
    `http://libgen.is/json.php?ids=${idRequestString}&lg_topic=libgen&fields=title,author,year,extension,md5,edition,pages,language,filesize,coverurl`
  );
};
```

In the code above, in the libgenApiResponse, we've got our response from LibGen! Again, we're using Axios to make the call. We take the stringified list of IDs and put that in the request. You can customize what LibGen returns to you, and you can see pretty clearly what I'm asking for in my request. View their [ documentation ](http://libgen.is/json.php) for more details on that.

### 5. Use the API Response

Now that you've got the API response from LibGen, you can do whatever you'd like with it! I'll simply return the response in my Lambda so that I've got an easy to use search API on top of LibGen with an endpoint accepting a search query.

```javascript
const axios = require("axios");

module.exports.lginfo = async (event) => {
  const params = JSON.parse(event.body);
  const { search } = params;
  const libgenScrape = await axios.get(
    `http://libgen.is/search.php?req=${search}&lg_topic=libgen&open=0&view=simple&res=100&phrase=1&column=def`
  );
  const regex = /(?<=view.php?id=)[0-9][^']*/g;
  const allIds = libgenScrape.data.match(regex);
  const idRequestString = allIds.toString();
  const libgenApiResponse = await axios.get(
    `http://libgen.is/json.php?ids=${idRequestString}&lg_topic=libgen&fields=title,author,year,extension,md5,edition,pages,language,filesize,coverurl`
  );
  const response = {
    statusCode: 200,
    headers: {
      "Access-Control-Allow-Origin": "*", // Required for CORS support to work
    },
    body: JSON.stringify(libgenApiResponse),
  };
  return response;
};
```

In this code, I'm simply taking the response that we generated above and returning it in the API response from the Lambda! In order to query this, you must POST to the endpoint with a "search" parameter. Then it will respond after a few seconds with the LibGen API response and you can do with that whatever you'd like.

Enjoy!