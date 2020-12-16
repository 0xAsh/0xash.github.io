---
layout: post
title: Farming phone numbers with Python and the Google Places API
excerpt: How I used Python and Google's Places API to dynamically gather a list of phone numbers from multiple address. Or even a really long list of phone numbers from addresses.
tags: Python, APIs
comments: true
social-share: false
gh-repo: 0xash/street2phone.py
gh-badge: [star, fork, follow]
thumbnail-img: /assets/img/street2phone/street2phone.png
---

![headerimage](/assets/img/street2phone/street2phone2.png)

{: .box-note}
Only care about the source code? Checkout the [Github Repo](https://github.com/0xash/street2phone.py)

# Background & Motivation

One day a friend asked me something along the lines of:

_"Do you think you could grab the local phone number code from an address?"_

A simple question. _"Yeah!"_ I answered. A quick Google search can get you that info. Just look around Google Maps and figure out which is most common.

What if you have 100 addresses? What if you not only want the area code, but a list of phone numbers from that area? This would get extremely tedious.

**Enter the Places API.**

# Places API

One of the first thoughts that entered my mind is that this could be automated with Python. A problem I foresaw is that finding an adequate data source would be difficult.

So where would you go if you need to gather a large amount of geographical data? Google Maps seemed like a great candidate.

#### A quick _Google_ search and some digging later, we find the Places API, specifically the Place Details request:

![Place Details](/assets/img/street2phone/1.png)

This looks spot on. So we can issue a Place Details request and gather information including the _phone number_. Perfect.

There's one problem though, if we look at the above screenshot, to issue a Place Details request we will need a `place_id` from a Place Search.

If we remember our original question, we'll need to gather phone numbers for _nearby_ locations, rather than the address itself (which may not possess a phone number entry within Google's data).

#### This brings us to the Nearby Search request:

![Nearby Search](/assets/img/street2phone/2.png)

Essentially, we can gather a long list of nearby `place_id` parameters, then feed them into a Place Details search request to grab the phone number for each nearby location.

#### Let's take a look at the API parameters required for this request:

![Nearby Search Parameters](/assets/img/street2phone/3.png)

#### There are a few things to note here:

- We'll need to specify an output type in the URL. I'll choose `JSON` since I don't care to mess around with XML.
- If we look at the required parameters we can specify a `radius` (in meters) to gather results from. This is cool because we can custom tailor the amount of results we want (or don't want) to gather.
- Lastly, apart from the usual `key` parameter, there is also a `location` parameter that is required to issue the request. This must be served in the _longitude,latitude_ format. Crap, so one last hurdle. How do we get a latitude/longitude from an address?

Luckily, an additional API method exists for us to gather the latitude and longitude values we'll need to issue a Nearby Search request for our address. This is a simple text search where we can use our original address as the arugment, therefore compelting the whole process.

#### Taking a look at the Text Search Request:

![Text Search](/assets/img/street2phone/4.png)

#### Now let's see what kind of output this will return:

![Text Search Example Output](/assets/img/street2phone/5.png)

Okay, kind of confusing, but with all this information gathered, the flow for gathering phone numbers will look something like this:

![Automation Process Flow](/assets/img/street2phone/6.png)

Essentially, we'll take an address, translate that to latitude/longitude, find the nearby list of `place_id` parameters, and then _finally_ grab a list of phone numbers from those nearby places. 

You might be thinking that we can simply issue an original Place Details request from the `place_id` parameter that's tied to our address, but this wouldn't work since we want nearby phone numbers, and to grab the phone number of _nearby_ locations, we'll need the `place_id` parameter for _all_ the nearby locations. 

It's a little convuluded, but it's the simplest way I could orchestrate it. If I'm missing a step that simplifies (and reduces the cost) of the whole process, Tell me! 

# Automating the Process with Python

Now for the fun part; scripting this out and automating it.

Let's start with the fundamental stepping stone we'll need to kick the process off; getting latitude and longitude. I'll call this function `getLatLng`:

#### getLatLng():

```python
def getLatLng(addr):
   latlongPayload = {'key': args.APIKEY, 'query': addr, 'fields': "lat,lng"}
   longlat = requests.get("https://maps.googleapis.com/maps/api/place/textsearch/json?", params=latlongPayload)
   try:
      lat = longlat.json()['results'][0]['geometry']['location']['lat']
      lng = longlat.json()['results'][0]['geometry']['location']['lng']
      return str(lat) + "," + str(lng)
   ## Catch errors
   except (IndexError, UnboundLocalError):
         print("No lat long found")
         pass
```

For context, I've already setup a parser that is taking arguments that we can access via `args.<value>`. 

I'll next use the Python `requests` module to issue a Text Search request using the `latlongPayload` array we've constructed. Note the `field` value in that array, which tells Google that we only care about the `lat` and `lng` values.

After some indicing we can simply return the latitude and longitude of this address, in a pre-composed format that the Nearby Search request will need in our next step.

Lastly, I'll throw in some error handling to catch errors that may occur if Google can't find a lat/lng value for a given address.

{: .box-warning}

To issue a request to Google APIs you'll need a _devloper key_. For more information on how to get one check out [this documentation](https://developers.google.com/maps/documentation/javascript/get-api-key).

#### With this written, I'll go ahead and give the getLatLng method a try:

![Running getLatLng](/assets/img/street2phone/7.png)

Nice! So we have a working Lat/Lng. Let's move on to scripting the Nearby Search request.

#### getPlaceIDs():

```python
def getPlaceIDs(addr):
   nearbyPayload = {'key': args.APIKEY, 'location': getLatLng(addr), 'radius': 150}
   nearbyReq = requests.get("https://maps.googleapis.com/maps/api/place/nearbysearch/json?", params=nearbyPayload)
   nearbysearches = nearbyReq.json()['results']
   placeIDList = []
   
   ## Iterate through JSON and add place_ids to list if not empty
   for i in nearbysearches:
      if i['place_id'] is not None:
         placeIDList.append(i['place_id'])

   return placeIDList
```

{: .box-warning}
Nearby Search requests can quickly become expensive. If you'd like your search to return specific values (and cut down on prices), I would recommend checking out the [Find Place request documentation](https://developers.google.com/places/web-service/search#FindPlaceRequests)

We can reuse our getLatLng method by simply including it within our passed along payload of `nearbyPayload`. We'll also set the radius to 150 for a decent amount of results.

After our `nearbyPayload` value is set, we can simply issue the Nearby Search request. Now we will indice to the `results` portion of the JSON, and itereate through the data. If the `place_id` value does not possess a `NoneType` value, we will append it to our `placeIDList` list that will be returned once the entire list of nearby locations has been iterated through.

#### With our function completed, let's try to grab a list of `place_id` parameters:

![Running getPlaceIDs](/assets/img/street2phone/8.png)  

Awesome. This brings us to the final stretch, with our working list of nearby `place_id` values, all we will need to do is issue a Place Details request for each and gather the phone number varaible.

I'll create a `makePhoneNumberList` function to automate this final process.

#### makePhoneNumberList():

```python
def makePhoneNumberList():
   phoneNumsList = []
   for i in getPlaceIDs(address):
      placeIDpayload = {'key': args.APIKEY, 'place_id': i, 'fields': "formatted_phone_number"}
      phoneQuery = requests.get('https://maps.googleapis.com/maps/api/place/details/json?', params=placeIDpayload)
      phoneNumbs = phoneQuery.json()['result']
      phoneNumsList.append(phoneNumbs.get("formatted_phone_number"))
      
   return phoneNumsList
```

We begin by iterating through our list of Nearby Place IDs (gathered through use of our previous functions). 

Notice that we specify the `formatted_phone_number` field as the only desired return value for the request.

For each Place ID in our list, we issue a Place Details request and append the returned phone number to a list called `phoneNumsList`. 

#### Running the makePhoneNumberList function:

![Running makePhoneNumberList](/assets/img/street2phone/9.png)

A little clunky with the returned `NoneType` values, but it gets the job done.

## The Final Product.

With all of our functions in place, and some extra scripting to provide meaningful output, our final result looks like the following:

![Running the final script](/assets/img/street2phone/10.png)

# Conclusion

And that's it. A lightweight (_and potentially costly_) way to mass grab phone numbers from a single address. You can check out the full documentation from the [Github Repo](https://github.com/0xash/street2phone.py).

How useful is this? I have no idea. It was a great excuse to work on learning Python. Maybe you'll find a cool use for the script, but hopefully at least it provides a decent guide on using some of the Google Places APIs.

