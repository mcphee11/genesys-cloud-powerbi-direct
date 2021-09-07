# genesys-cloud-powerbi-direct
How to build out a direct query from powerbi to genesys cloud

This is designed as an example only. While the other project i have [genesys-cloud-powerbi]('https://github.com/mcphee11/genesys-cloud-powerbi') gives an example of how to build out a powerbi "connector" that can be used as a datasource in powerbi. This project example shows how you can query Genesys Cloud directly through a Advanced query without the need to build a connector. This is a great way for users to connect to Genesys Cloud that are not users of VisualStudio and able to deploy connectors using Power Query.

The first step to this is to create a OAuth client inside the Genesys Cloud, this needs to be of the "client credentials" type and will give you a "clientId & secret" that will be used in the powerbi query for authentication. You will need to also provide the correct "scopes" for the credentials to have the required permissions to the API endpoints your wanting to query. The permissions will differ depending on the reports your wanting to run and I suggest you reference the Genesys Cloud permissions documentation to get the required values.

![](/docs/images/oauth.png?raw=true)

    CLIENTID
    CLIENTSECRET

Ensure you copy the above details from your environment and keep them secure.


# PowerBi Desktop

Open PowerBi Desktop and create a new Blank Query

![](/docs/images/powerbi1.png?raw=true)

From here you will need to go into "Advanced Editor"

![](/docs/images/powerbi2.png?raw=true)

Inside the advanced editor delete the text that is in there by default and paste in the below example.

    NOTE: ensure you replace the "CLIENTID & CLIENTSECRET" with your ones from above

This example is for a basic connection and some table formatting you can create your own table formatting and even use a different API endpoint. You will also notice that the API endpoint im using in this example is one of the "JOBS" endpoints this is a endpoint that first requires a POST then you get a JOBID back to which you then do a GET request on that once the data is done. I have simply put a 30sec timer to wait for the data to be populated. Depending on your data query size you may want a longer time frame set.



    let
    //Get the API Token
    login_url = "https://login.mypurecloud.com.au/",
    token_path = "oauth/token",
    ClientID = "ENTER_YOUR_CLIENTID",
    Secret = "ENTER_YOUR_SECRET",

    EncodedCredentials = "Basic " & Binary.ToText(Text.ToBinary(ClientID & ":" & Secret), BinaryEncoding.Base64),

    Token_Response = Json.Document(Web.Contents(login_url,
    [
        RelativePath = token_path,
        Headers = [#"Content-Type"="application/x-www-form-urlencoded",#"Authorization"=EncodedCredentials],
        Content=Text.ToBinary("grant_type=client_credentials")
    ])),

    //API query conversation details job endpoint
    token = Token_Response[access_token],
    api_url = "https://api.mypurecloud.com.au",
    query_path = "/api/v2/analytics/conversations/details/jobs",

    data= Json.Document(Web.Contents(api_url,
    [
        RelativePath = query_path,
        Headers = [#"Authorization"="Bearer "&token,#"Content-Type"="applicaiton/json"],
        Content = Text.ToBinary("{
                    ""conversationFilters"": [
                        {
                            ""type"": ""and"",
                            ""predicates"": [
                                {
                                ""type"": ""dimension"",
                                ""dimension"": ""conversationId"",
                                ""operator"": ""exists""
                                }
                            ]
                        }
                    ],
                    ""order"": ""asc"",
                    ""orderBy"": ""conversationStart"",
                    ""interval"": ""2021-08-31T14:00:00.000Z/2021-09-05T14:00:00.000Z"",
                    ""startOfDayIntervalMatching"": true
                    }")
    ])),

    //returned jobId
    jobId = data[jobId],

    //GET job data
    job_query_path = "/api/v2/analytics/conversations/details/jobs/"&jobId&"/results",

    job_data= Function.InvokeAfter(()=>Json.Document(Web.Contents(api_url,
    [
        RelativePath = job_query_path,
        Headers = [#"Authorization"="Bearer "&token,#"Content-Type"="applicaiton/json"]
    ])), #duration(0,0,0,30)),
    conversations = job_data[conversations],
    #"Converted to Table" = Table.FromList(conversations, Splitter.SplitByNothing(), null, null, ExtraValues.Error)
    in 

    conversations

Ensure that you update the URLS to also be based on your region. In my example im using "mypurecloud.com.au" if your in another region you may need to change it for eg "mypurecloud.com" etc.

Once you add this and click done. You will be prompted to setup credentials

![](/docs/images/powerbi3.png?raw=true)

for this as the file includes the client credentials you can simply select anonymous. This will happen for both the API URL as well as the login token url, so you will get this twice with the different 'https://....' showing.

![](/docs/images/powerbi4.png?raw=true)

You will then also be required to allow the privacy for the endpoints as well.

![](/docs/images/powerbi5.png?raw=true)

Depending on your required privacy select the option that suits. In my lab I have ignored these levels.

![](/docs/images/powerbi6.png?raw=true)

Once this is all done you wont need to do this each time powerbi will remember these details. Now you have the data and can start to build out your reports.

![](/docs/images/powerbi7.png?raw=true)

Here is a simple example I built out based on this data with tables that relate to each other.

![](/docs/images/powerbi8.png?raw=true)