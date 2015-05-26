# Further Securing Your Sinch Calling Functionality App With REST API

The callback APIs are a really powerful way to get information about your calls and further secure them. This is the final step before you allow calls to be connected. If you have followed our other tutorials, you are likely familiar with the different states of a call: progressing, established, and ended. With the callback API, you get the same events, enabling you to make decisions just before you connect the call to the server. 

For detailed information, see our [documentation](http://www.sinch.com/docs/rest-apis/api-documentation/#callback).

## Prerequisites
1. Complete [this tutorial] (http://www.sinch.com/tutorials/using-delegated-security-application-server-using-c-sinch-sdk/).
2. Deploy the solution to a public accessible web server. This is necessary because the Sinch backend needs to be able to post to your callback handler. 

## Setup
1. To receive callbacks, open your [dashboard](http://www.sinch.com/dashboard) and add the URL that should get callbacks.
<img src="Configure_callback.png" style="width:200">
2. Add a new WebAPI controller named CallbackController to your backend project.

## Responding to callbacks
If you have configured your callback URL, you **must** to respond to callbacks for the call to be connected. The Sinch service expects a response containing **Svaml** to make it easier to work with. In your C# project, create a couple of classes to help out:

```
public class Svaml {
    public Instruction[] Instructions { get; set; }
    public Action Action { get; set; }
}
public class Instruction {
    public string Name { get; set; }
    public string Ids { get; set; }
    public string Locale { get; set; }
}
public class ICEAction:Action {
    public string Number { get; set; }
    public string locale { get; set; }
    public int maxDuration { get; set; }
    public bool callback { get; set; }
    public string cli { get; set; }
}
public class SimpleICEAction : Action {
    public int maxDuration { get; set; }
    public bool callback { get; set; }
}
public class Action {
    public string Name { get; set; }
}
```

Svaml contains instructions and actions. For example, an instruction is playing a sound file to a user, and an action is connecting to a call. For a complete list of instructions and actions, see the [documentation](http://www.sinch.com/docs/rest-apis/api-documentation/#callback).


Next, create this method in your **callbackcontroller**:
 
```
[HttpPost]
public async Task<Svaml> Post(HttpRequestMessage message) {
}
```

This method is called for all callback events; they all look slightly different, but they all contain a JSON request an event.

### Incoming call event (ICE)
This event is triggered right before you get the in progress event in a call client. I think this is the most interesting callback because you can change the call that is about to happen. Here, you can decide if a user has enough credit to make a call, or if the phone number is one you want to connect to, and then make decisions based on that. In case you are using Sinch to make click-to-call buttons, you may want to check the dialed phone number against a database. That's exactly what we are going to do in this tutorial. I don't want to connect any phone calls unless you are trying to call me. In the **post** method, add:

```
public async Task<Svaml> Post(HttpRequestMessage message) {
    //read the response
    String body = await message.Content.ReadAsStringAsync();
    var json = JsonConvert.DeserializeObject<JObject>(body);
    //create a new respone object
    var svaml = new Svaml();
    if (json["event"].ToString() == "ice") {
        //only calls to my personal phone is allowed with this app
        if (json["to"].ToString() == "+15612600684") {
            svaml.Action = new SimpleICEAction() {
                Name = "ConnectPSTN",
                callback = true,
                maxDuration = 600
            };
        } else {
            //else hangup
            svaml.Action = new Action {
                Name = "hangup"
            };
        }
    }
}
```

### Answered call event (ACE)
Since you are using callbacks, you also need to take care of the ACE event, otherwise the call will not connect. In this case, you are going to just let it continue. At the end of the **post**, add:

```
if (json["event"].ToString() == "ace") {
    svaml.Action = new Action() {
    Name = "continue",
   };
}
var svamljson = JsonConvert.SerializeObject(svaml, Formatting.Indented,
    new JsonSerializerSettings { NullValueHandling = NullValueHandling.Ignore });
return svaml;
```

### Disconnected call event (DiCE)
This event is a notification event, where you could adjust the balance of your users' accounts, post duration to a CRM system, etc. No response is needed. 

You can find the finished code for this tutorial [here](https://github.com/sinch/net-backend-sample).
